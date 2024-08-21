---
layout: post
title: "
<div style='display: flex; align-items: center; justify-content: space-between;'>
    <a href='/' style='font-size: 28px; font-weight: normal; color: #c1c1c1; text-decoration: none; margin-top: -50px;'>Home</a>
    <img src='https://raw.githubusercontent.com/pligonstein/pligonstein.github.io/main/images/logo.gif' alt='Logo' style='height: 48px; width: 48px; border-radius: 50%; object-fit: cover; margin-top: -50px;'>
</div>"
post_title: "Rust in Embedded Systems - Memory Game"
date: 2024-08-21
categories: blog
---

## Quick Overview

For the last part of this series, I'll present you with a small Memory Game that I had to program using all the information that we have accumulated so far. From GPIO, PWM, all the way to writing some text on the LCD using SPI. Let's get right into the challenge.

## Challenge

### **Memory game**

> Write a memory game using the EEPROM module, two LEDs (one green and one red), and the LCD display:
>
> The program must display 4 random letters, that can be either A, B, X or Y (corresponding to the buttons on the Pico Explorer Base); each letter will be displayed for 500ms. After they are displayed, the user must recreate the sequence in the exact order (the sequence can have repeating letters), by pressing the buttons on the Explorer Base.
>
> If the sequence is inserted correctly, the green LED should be turned on, else, the red LED should turn on to indicate a failure and should reset the score to 0 (the score represents the number of consecutive rounds played without failure).
>
> The game should keep track of the highest score by storing it at address 0x00 in the EEPROM (we assume that its representation doesn't exceed a byte). If the score of a player is higher than the stored value, the highest score will be updated in the memory.

We start off as usual, by specifying we won't use neither the standard main function as our entrypoint nor the standard output. Also, I'll be adding all the imports of the functions and crates used in here as well.

```rust
#![no_main]
#![no_std]

use core::fmt::Write;
use cyw43::new;
use embassy_executor::Spawner;
use defmt::info;
use embassy_futures::select::{select4, Either4};
use embassy_time::Timer;
use {defmt_rtt as _, panic_probe as _};
use ipw_embedded::display;
use embedded_graphics::{pixelcolor::Rgb565, text::Text};
use embedded_graphics::prelude::Point;
use embedded_graphics::mono_font::ascii::FONT_7X13_BOLD;
use embedded_graphics::mono_font::MonoTextStyle;
use embedded_graphics::Drawable;
use embassy_rp::{bind_interrupts, i2c::{Config as I2cConfig, I2c, InterruptHandler as I2CInterruptHandler}};
use embedded_hal_async::i2c::{Error, I2c as _};
use embassy_rp::peripherals::I2C0;
use embassy_rp::gpio::{Level, Pull, Output, Input};
use rand::{Rng, RngCore};
use embassy_rp::clocks::RoscRng;
```

Then, I pasted the code for the SPI controller for our LCD from the last part, plus the introduction of the asynchronous `main` task.

```rust
#[embassy_executor::main]
async fn main(_spawner: Spawner) {

    // SPI Controller Setup

    let peripherals = embassy_rp::init(Default::default());
    let miso = peripherals.PIN_4;
    let display_cs = peripherals.PIN_17;
    let mosi = peripherals.PIN_19;
    let clk = peripherals.PIN_18;
    let rst = peripherals.PIN_0;
    let dc = peripherals.PIN_16;
    let mut display_config = embassy_rp::spi::Config::default();
    display_config.frequency = 64_000_000;
    display_config.phase = embassy_rp::spi::Phase::CaptureOnSecondTransition;
    display_config.polarity = embassy_rp::spi::Polarity::IdleHigh;

    // Init SPI
    let spi: embassy_rp::spi::Spi<'_, _, embassy_rp::spi::Blocking> =
        embassy_rp::spi::Spi::new_blocking(
            peripherals.SPI0,
            clk,
            mosi,
            miso,
            display_config.clone(),
        );
    let spi_bus: embassy_sync::blocking_mutex::Mutex<
        embassy_sync::blocking_mutex::raw::NoopRawMutex,
        _,
    > = embassy_sync::blocking_mutex::Mutex::new(core::cell::RefCell::new(spi));

    let display_spi = embassy_embedded_hal::shared_bus::blocking::spi::SpiDeviceWithConfig::new(
        &spi_bus,
        embassy_rp::gpio::Output::new(display_cs, embassy_rp::gpio::Level::High),
        display_config,
    );

    let dc = embassy_rp::gpio::Output::new(dc, embassy_rp::gpio::Level::Low);
    let rst = embassy_rp::gpio::Output::new(rst, embassy_rp::gpio::Level::Low);
    let di = display::SPIDeviceInterface::new(display_spi, dc);

    // Init ST7789 LCD
    let mut display = st7789::ST7789::new(di, rst, 240, 240);
    display.init(&mut embassy_time::Delay).unwrap();
    display
        .set_orientation(st7789::Orientation::Portrait)
        .unwrap();
    use embedded_graphics::draw_target::DrawTarget;
    display.clear(embedded_graphics::pixelcolor::RgbColor::BLACK).unwrap();
```

After that, I continued with the I2C part of the code, defining the PINs for the SDA and SCL. Then, I specified the default I2C configuration and created the variables that store the device address, the highest score address, and last but not least, the highest score itself. Finally, I used `write_read` to specify where to read the highest score from.

```rust
// I2C
    
    let sda = peripherals.PIN_20; // PIN_20 is the SDA pin
    let scl = peripherals.PIN_21; // PIN_21 is the SCL pin

    let mut bus = I2c::new_async(peripherals.I2C0, scl, sda, Irqs, I2cConfig::default()); // Default I2C configuration
    
    const TARGET_ID: u8 = 0x50u8; // I2C address of the target device
    const HIGHEST_SCORE_ADDR: [u8; 2] = [0x00u8; 2]; 
    let mut highest_score: u8 = 0;

    Timer::after_secs(2).await;
 
    bus.write_read(TARGET_ID, &HIGHEST_SCORE_ADDR, &mut [highest_score]).await.unwrap();

    let mut current_score: u8 = 0;
```

Before diving into the important part of the code, we also have to define the PINs for each of the buttons, plus the two LEDs.

```rust
// GPIO
    let mut button_a = Input::new(peripherals.PIN_12, Pull::Up);
    let mut button_b = Input::new(peripherals.PIN_13, Pull::Up);
    let mut button_x = Input::new(peripherals.PIN_14, Pull::Up);
    let mut button_y = Input::new(peripherals.PIN_15, Pull::Up);

    let mut pin_gren = Output::new(peripherals.PIN_6, Level::Low);
    let mut pin_red = Output::new(peripherals.PIN_1, Level::Low);
```

Ok, now for the key part, we are first spinning it off with generating the random values that are going to be printed out on the display.

> Also, remember that from this point on, all the code needs to placed in the loop, as for MCUs that don't run any operating systems like in our case, they don't return anything.

```rust
loop {
        // Generate random sequence
        let mut sequence: [u8; 4] = [0; 4];
        let mut rng = RoscRng;
        for i in 0..4 {
            sequence[i] = rng.gen_range(0..4);
        }
```

Next, we'll print the sequence on the display and define what to match each of the numbers to.

```rust
let color_yellow = Rgb565::new(255, 255, 0);
        let color_black = Rgb565::new(0, 0, 0);

        // Display the sequence
        let style = MonoTextStyle::new(&FONT_7X13_BOLD, color_yellow);
        for &s in &sequence {
            let letter = match s {
                0 => "A",
                1 => "B",
                2 => "X",
                _ => "Y",
            };
            Text::new(letter, Point::new(120, 120), style).draw(&mut display).unwrap();
            Timer::after_secs(1).await;
            display.clear(color_black).unwrap();
            Timer::after_secs(1).await;
        }
```

Now, we'll specify the user input and match each of the buttons to the previously defined numbers. Plus, I added the `info` for making it easier to debug when we flash our code onto the board.

```rust
// User input
        let mut user_input: [u8; 4] = [0; 4];
        for i in 0..4 {
            let button = select4(button_a.wait_for_falling_edge(), button_b.wait_for_falling_edge(), button_x.wait_for_falling_edge(), button_y.wait_for_falling_edge()).await;
            match button {
                Either4::First(_) => user_input[i] = 0,
                Either4::Second(_) => user_input[i] = 1,
                Either4::Third(_) => user_input[i] = 2,
                Either4::Fourth(_) => user_input[i] = 3,
            }
                info!("Selected value: {}", user_input[i]);
            Timer::after_secs(1).await;
        }
```

For our last part, we'll add the checker for the sequence and we'll write to the EEPROM the highest score if the user input matches the sequence. Also, we'll light the green LED if it's correct.

```rust
// Check if user input matches the sequence
        if user_input == sequence {
            current_score += 1;
            if current_score > highest_score {
                highest_score = current_score;
                let buf: [u8; 3] = [HIGHEST_SCORE_ADDR[0], HIGHEST_SCORE_ADDR[1], highest_score];
                bus.write(TARGET_ID, &buf).await.unwrap();
            }
            pin_gren.set_high();
            Timer::after_secs(1).await;
            pin_gren.set_low();
        } else {
            pin_red.set_high();
            Timer::after_secs(1).await;
            pin_red.set_low();
            current_score = 0;
        }
        Timer::after_secs(1).await;
```

Now, a little surprise, I also added a smalll part at the end of each that mentions our current and highest score.

> Because MCUs don't have heap, we'll have use the library heapless that just basically writes the string we want to print onto the stack.

```rust
let mut res: heapless::String<100> = heapless::String::new();
        write!(&mut res, "Current score: {}\nHighest score: {}", current_score, highest_score).unwrap();

        Text::new(&res, Point::new(20, 120), style).draw(&mut display).unwrap();
        Timer::after_secs(3).await;
        display.clear(color_black).unwrap();
        Timer::after_secs(1).await;
    }
}
```

I really hope you enjoyed this small series of tutorial cause I sure did, and feel free to connect with me if you got any questions. See you in the next post! 👋

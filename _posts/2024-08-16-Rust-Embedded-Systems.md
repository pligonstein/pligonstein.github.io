---
layout: post
title: "
<div style='display: flex; align-items: center; justify-content: space-between;'>
    <a href='/' style='font-size: 28px; font-weight: normal; color: #c1c1c1; text-decoration: none; margin-top: -50px;'>Home</a>
    <img src='https://raw.githubusercontent.com/pligonstein/pligonstein.github.io/main/images/logo.gif' alt='Logo' style='height: 48px; width: 48px; border-radius: 50%; object-fit: cover; margin-top: -50px;'>
</div>"
post_title: "Rust in Embedded Systems - GPIO and PWM"
date: 2024-08-16
categories: blog
---

## Quick Overview

In this article I'll be covering Rust in Embedded Systems, more specifically GPIO(General Purpose Input Output) and PWM(Pulse Width Modulation) protocols. I'll be talking about how to implement those in Rust, using Embassy and I'll be also going through two different challenges that I had to go through recently in a workshop.

> Embassy-rs is short for Embassy Asynchronous Rust

## Setup

We first need to define the entrypoint and bootloader so we'll specify it like this:

```rust
use embassy_executor::Spawner;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let peripherals = embassy_rp::init(Default::default());
}
```

The `init` function takes care of all the peripheral initializations so that we don't need to worry about all these.

We also have to specify:

```rust
#![no_main]
#![no_std]
```

That's because Embassy-rs works in an asynchronous way and so we need to define main as `async`. Also because the microcontrollers that we are going to be using don't run any framework/operating system we can't rely on `std`.

## Challenges

For those challenges I had a pico debug probe and a pico explorer that I had different tasks for.

### **First Task**

<p></p>

> Connect a green LED to GP4 and a red LED to GP5. Blink the LEDs consecutively, for 1 second each (i.e. when one LED is on, the other is off).

This one was fairly easy, we can just use GPIO to program the LED after we set it up on the breadboard and connect the pins accordingly. First, let's go through each part of the code:

```rust
#![no_main]
#![no_std]

use embassy_executor::Spawner;
use embassy_rp::gpio::{Level, Output};
use embassy_time::Timer;
use {defmt_rtt as _, panic_probe as _};
```

The first two lines were explained a little earlier, so there is no need to explain them again. Then, we import all the functions from the crates that we need. I'll be explaining what each function does:

```rust
#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let peripherals = embassy_rp::init(Default::default()); // Defines the peripherals
    let mut pin_green = Output::new(peripherals.PIN_4, Level::Low); // Defines the green LED and sets it to low
    let mut pin_red = Output::new(peripherals.PIN_5, Level::Low); // Defines the red LED and sets it to low
    loop {
        pin_green.set_high(); // Turns on the green LED
        pin_red.set_low(); // Turns off the red LED
        Timer::after_secs(1).await; // Waits for 1 second
        pin_green.set_low();
        pin_red.set_high(); 
        Timer::after_secs(1).await;
    }
}
```

This is the `main` function. The most important part consists in defining the `Output` for each pin and then the `loop` is needed, because as explained earlier the MCU does not run any operating system so there is nothing to return.


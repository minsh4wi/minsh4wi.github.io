---
title: "1 - FLAreCAPTCHA"
date: 2026-10-24 18:06:00 +0300
categories: [CTF, Flare-On 13]
tags: [reverse-engineering, flare-on, ctf]
description: Write-up for the FLAreCAPTCHA challenge from Flare-On 13.
---

The challenge has a file named `flarecaptcha.html`, I opened it in a browser and started analyzing the source code.

I found this condition:

```js
if (e.button === (Math.pow(5, 0) + Math.pow(1, 1)))
```

It is equivalent to:

```js
if (e.button === 2)
```

So, we know that the code checks which mouse button was pressed.

I checked the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/button) and found that `e.button === 2` means the right mouse button was pressed.


![](</assets/img/flare-on-13/Pasted image 20260929070957.png>)


```js
window.addEventListener('mouseup', (e) => {

if (e.button === 2) {

cancelHold();

}

});
```

This code calls `cancelHold()` when the right mouse button is released, I needed to hold the right mouse button for 2 seconds without releasing it.

After holding the button for 2 seconds, the page displays the flag.

![](</assets/img/flare-on-13/Pasted image 20260929071403.png>)

**flag:**

```
triskaidekaphobia@flare-on.com
```

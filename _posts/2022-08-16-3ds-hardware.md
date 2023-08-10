---
layout: post
title:  "Hardware"
date:   2022-08-16 17:30:00 +0200
categories: 3ds
---

This is part two of the [Nintendo 3DS Homebrew series.]({% post_url 2022-08-12-3ds-intro %})

Today, I will go over the hardware we are dealing with - while you can easily find info about models and hardware revisions on the internet, there are many gotchas when it comes to actually utilizing these resources in software.

*Note: I am writing this guide as I go. There is a high chance I got something wrong, so do not hesitate to open an issue/mail me.*


# Models
There were many, different models released in the console's lifetime. Of most interest to us is whether we are dealing with the original or the refresh "New" console (Nintendo naming strikes again).

Visually, the _New_ consoles can be recognized by the small, secondary analog stick just over the ABXY buttons. The refresh, apart from having additional peripherals, greatly expands the hardware capabilities - two times the CPU cores, three times the CPU clock, and two times the memory of the original release. The only thing not changed is the gpu processing power.

Apart from that, both original and _New_ come in a variety of editions: base 3DS, 2DS lacking the stereoscopy screen, XL with larger screens, and weird combinations of thereof, such as 2ds with large screens or one lacking hinge. All these amount to is presence of the 3D effect, screen size (_not the resolution_), hinge, and battery capacity.

# CPU
3DS contains two ARM cores:
* ARM11 - Runs application and system code.
* ARM9 - Mostly here for backwards compatibility with NDS, but also seems to run various housekeeping for the OS.

When developing applications and games, we almost certainly want to run them on the ARM11 MCU. In the baseline 3DS system, it has two cores running at 268MHz. _New 3DS_ expands that to four cores, that can be enabled to run at 804MHz.

However, there is a catch: We cannot use all the cores for the app. The core allocation on 3DS is as follows:

| ID | Usage |
|----|-------|
| 1 | Available for application |
| 2 | System Exclusive |
| 3 | New: Available for application |
| 4 | New: System Exclusive (Head tracking) |

A base 3DS can actually ask the OS to yield a 30% timeslice of the second code and create a thread on it. Thus, base 3DS has, practically, 1.3 cores available.

On the new 3DS, the third core is free for application use and enables true multithreading. The app can again ask the OS for a timeslice of the second core, and when running with the increased clock speed, it can actually get more than the original 30% - The reports vary, but even 70/80% seems achievable. Thus, we can assume we actually get around ~2.5 cores on the _New_ system. Given the increased clock speed, we get over 5x increase in raw processing power.

The fourth core is permanently tied up in *New 3DS*'s head tracking CV process - even when the 3D mode is disabled. Apparently, new Homebrew firmware can actually grant access of the core back to userspace, which means one additional core for multithreading.

Lastly, the original CPU only has L1 cache. *New* models get additional 2 MiB L2 cache.

ARM11 is, by current standards, getting rather long in the tooth. It is the last architecture of the classic series - before the introduction of the Cortex series (Same as first Raspberry Pi, for example). Do not expect NEON extensions, but the basic FPU is there.

# Memory

[See the wiki page for memory mapping.](https://www.3dbrew.org/wiki/Memory_layout)

The base 3DS has 128 MiB of **generic RAM**. _New_ version expands it to 256 MiB.
A portion of it is reserved for system.

*Note: I will hopefully update this section with exact values when I find some time to write device tests.*

RAM is virtualised. Virtual addresses do not directly correspond to physical memory layout, the OS can reorder and move the memory as it sees fit. This is problematic when you want hardware devices to access the memory directly - to perform DMA to peripherals, or pull 3D model data from buffers in RAM - because only the CPU seems to see the virtual addressing.

To solve this, a section of memory known as **linear memory** is allocated. Linear allocation are guaranteed to be contiguous when seen from the hardware. Thus, bulk data sent to peripherals has to go through the linear memory.

The last memory type of interest is the **VRAM**. This is the area where data commonly used by the GPU lives, such as framebuffers and textures. 3DS has a rather lacking amount of dedicated VRAM, somewhat made up by being able to put data in linear memory. Base 3DS has two banks of 3 MiB each, for 6 MiB total - which is available for app use. *Warning: My wild unconfirmed conjecture follows.* As for *New* consoles, additional 4 MiB of "VRAM" is added. However, from what I gather, either none or half of it is available for games (Homebrew might be able to bypass it, although it is not directly possible through `libctru`), as it is/might be used for the head tracking CV. [Read more here.](https://www.3dbrew.org/wiki/Memory_layout#0x1F000000_(New_3DS_only))

Additionally, apps can get access to the **SD card** to operate on arbitrary files there. However, compile-time / read-only files can be baked directly into the binary and exposed as a virtual file system.

I will not cover things like NAND memory here, as they are not very relevant to us now.

# Display

The 3DS comes with two screens.

**Top screen** at 800x240 with anisotropic pixels, can either emulate a regular 15:9 400x240 screen, a wide-mode 800x240 with double horizontal pixel density, or a stereoscopy mode on enabled models, where a lattice layer filters odd and even pixels to both eyes of the user.

In the *New* 3DS, the stereoscopy grid is essentially a 1-pixel tall, monochrome display that turns on and off bars that shadow fractions of pixels below. This allows for adjusting the parallax depending on the head position.

**Bottom screen** is a fixed 320x240 display with a resistive touch-screen layer.

# Input
One analog stick, 2 system buttons, 4 action buttons, 2 shoulder buttons. Standard fare.

*New* 3DS adds another analog stick, and another two shoulder buttons.

3DS has an internal IMU (Accelerometer + Gyro). Apparently.

# To be continued
I will update this post with more information as I go, or split it into multiple parts as necessary.

[Read on to the next part, where I introduce the basic app structure.]({% post_url 2022-08-30-3ds-app-loop %})
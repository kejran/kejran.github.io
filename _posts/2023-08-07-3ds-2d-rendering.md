---
layout: post
title:  "2D Rendering"
date:   2023-08-07 17:30:00 +0200
categories: 3ds
---

This is part five of the [Nintendo 3DS Homebrew series.]({% post_url 2022-08-12-3ds-intro %})

In this entry I shall introduce the basics of 2d rendering with `citro2D`.

# 3DS and 2D graphics

There are generally two approaches to rendering 2D graphics in hardware. 

First, is using traditional, dedicated hardware for tilemaps and free sprites. Works by directly evaluating the pixel that is about to be sent to the display. This type of engine has extremely low latency and never drops frames, but it enforces hard limits (such as no rotation, max sprites per line, etc). This was used on consoles such as NES or Game Boy.

The second, modern approach, is to create a fully capable 3D rendering engine, and then adapt it to drawing aligned, textured rectangles for sprites. The advantage is that, for modern hardware capable of realtime 3d graphics, there is no need to include extra chip space or coprocessor for 2D, since it can be just emulated with 3D - with the perspective correction and other 3D features unused. 

This is how the 3DS (and virtually any other modern console) does it.<sup>1</sup>

<sup>1</sup><sub>Interestingly, this is not the case with the DS: While it had a 3D engine, it was too weak (and power hungry) to rasterise 2D, especially on dual screen.</sub>

## citro2d and citro3d

The primary library for interacting with the 3DS GPU is `citro3d`. It is quite low level, more akin to opengl 3 rather than a game rendering framework. Meanwhile, `citro2d` is built on top of it, and it exposes a very high level interface that just lets you draw shapes onto the screen. 

This means you can mix and match both 2d and 3d - and you can switch from `citro2d` to your own solution if you need more performance or features.

# Application structure

First, we need some boilerplate to get us going. A main application loop would look like this:
```cpp
#include <3ds.h>
#include <citro2d.h>

int main() {

    // gpu init code 
    ...

    while (aptMainLoop()) {

        // render a single frame
        ...
    }

    // cleanup
    ...
}
```

## Setup 

**The following code will only run once, at the startup.**

To start with, initialise the framebuffers with default pixel format:
```cpp
gfxInitDefault();
```
Next, initialise the core 3d renderer the with command buffer size, which defines the size of the staging area that holds all low-level register writes to the gpu - using the default is fine:
```cpp
C3D_Init(C3D_DEFAULT_CMDBUF_SIZE);
```
Next, initialise the 3d renderer with the maximum number of objects that you expect to draw in a single frame - again, just use the default for now:
```cpp
C2D_Init(C2D_DEFAULT_MAX_OBJECTS);
```

Lastly, we need a buffer to render to, one for every screen you use - for now, just the top screen, and "left eye only" - which is used for both eyes when the stereo view is disabled:
```cpp
C3D_RenderTarget *top = C2D_CreateScreenTarget(GFX_TOP, GFX_LEFT);
```

# Render loop

**The following code will run once per each loop iteration.**

We already have the init code, now it is time to handle frame rendering. 

First, we will signal that we want to start rendering a new frame, and that we want vsync - this call will block to throttle down the main loop to 60Hz:
```cpp
C3D_FrameBegin(C3D_FRAME_SYNCDRAW);
```
Next, reset the core renderer for drawing in 2d: 
```cpp
C2D_Prepare();
```
The frame has old data inside; clear it to a color of your choosing:
```cpp
// build a color from R, G, B, A
u32 clr = C2D_Color32(0x22, 0x22, 0x22, 0xff);
C2D_TargetClear(top, clr);
```
Now set the top framebuffer as a rendering target:
```cpp
C2D_SceneBegin(top);
```
You are then ready to draw. All draw calls for this screen should happen now. For example, let us draw a simple rectangle:
```cpp
// alternatively, directly assign ABGR hex:
// u32 fill = 0xff'ee'ee'ee;
u32 fill = C2D_Color32(0xee, 0xee, 0xee, 0xff);

C2D_DrawRectSolid(50, 50, 0, 50, 50, fill);
```

You can then draw to any other screens as you wish.

After you are done drawing, signal end of the frame:
```cpp
C3D_FrameEnd(0);
```

## Cleanup
**The following code will only run once, after the main loop ends.**

If you want to be well-behaved, you can clean up after yourself after exiting the main loop:
```cpp
C2D_Fini();
C3D_Fini();
gfxExit();
```
It might not matter whether you do this or not, as the OS will usually clear it up on its own. Still, it is a good practice to include it.

# Compiling 
Make sure you add the libraries as dependencies to the makefile: 
```
LIBS := -lcitro2d -lcitro3d -lctru -lm
```

Then compile and deploy to the emulator. You should end up with a blank black screen and a white rectangle in the corner. 

![First rectangle](/assets/first_rect.png)

Utterly gripping results, no doubt. Still, you are now set with a base to expand the game upon. 

[In the next section, I will go over importing and drawing images.]({% post_url 2023-08-08-3ds-images %})

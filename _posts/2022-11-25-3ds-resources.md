---
layout: post
title:  "Resources"
date:   2022-11-25 17:30:00 +0200
categories: 3ds
---

This is part five of the [Nintendo 3DS Homebrew series.]({% post_url 2022-08-12-3ds-intro %})

Today I will cover handling files, so we can ship assets such as textures, and save game state.

This post contains some info-dumps that hopefully will not be necessary in most cases, you do not really need to read about texture compression internals if you just want to use a tool. 

# Resource types

While shipping a game, it often makes sense to optimise your assets, convert them to a format supported by hardware, and precalculate whatever data needed. Some data (such as config jsons, ogg sounds, etc) can be stored directly - other data needs to be processed. 

It is useful to make the process of baking your assets into a part of the pipeline. Some support for that already exists in the makefiles.

There are three kinds of "compilable" resources you will interact with when writing homebrew:
* Textures - converted to a format handled internally by `citro3d`
* Shaders - compiled into binary payloads to load onto the gpu
* Models - vertex and index data flattened and optimised to load directly into a buffer

## Textures
Textures contain image data to display on the models. You cannot directly use .png or similar files - the gpu only supports raw data or block compression and needs pixels in morton-code interleaved order.

### Compression formats
The 3DS GPU is surprisingly lacking when it comes to image formats it supports. 
There are the regular `R8`/`RGBA8`/`R4`/`RGB565` and friends when it comes to uncompressed formats, but the only supported block compression is `ETC1`. see `GPU_TEXCOLOR` in `3ds/gpu/enums.h`.

**There is no DXT support whatsoever**, so virtually all existing game textures will need conversion - take this into account when porting some old games.

### Block morton ordering
Most image file formats store data packed in scanlines. This is not the case on modern GPUs - they are usually reordered for better cache locality. And while a desktop GPU driver would happily take a linear image and convert it internally into whatever pattern it needs, the 3DS gpu will not - and we need to do that manually. 

The pattern seems to be a regular morton-curve 8x8 block for every uncompressed format. The blocks are laid out linearly.

### Converting textures
Regular (I recommend using .png for assets) images can be converted (and compressed if necessary) into native textures during a build step. The `tex3ds` tool converts images to final texture payloads. 

The default makefiles will look for `.t3s` files (in the `gfx` folder), which in turn contain actual image filename and arguments to the compressor - you can specify the pixel format and compression level there. See the `gpu/textured_cube` example, and help of `tex3ds` for more info on formats.

If you end up with a lot of textures, you might want to write your own rules to automatically build given textures with necessary settings, instead of duplicating a setting file for each image - for example, have one rule for a folder.

## Shaders
Shaders define operation of the programmable vertex and geometry hardware on the GPU. They are raw binary "executables" that are compiled offline from an assembly program description.

There is not much to configure there - the default makefile will compile text `.v.pica` and `.g.pica` into binary `.shbin` files you can include in your app. You can also do it manually/set up custom task to convert them by calling the `picasso` assembler.

## 3D Models
If you want to use authored triangle meshes (world, characters, etc) you will probably want to convert them to a flat format you can write directly to a linear buffer, to skip any file parsing on load. I am not aware of a tool that would do that, nor if there is any code in homebrew libraries to handle these files.
You will probably end up writing your own format, or you can just write a simple .obj parser on the 3DS side.

*I will revisit this section if I end up working with such meshes in the future.*

# Storing data
There are generally two approaches to shipping static data with the app, both ending up baking the binary payloads into the executable. 

## Linked file
You can use the compiler to link the files and use them in the code directly as byte arrays. This is the default for makefiles. Any compatible file, such as texture or shader, will be turned into a pair of `.o` and `.h` files that will link and expose the data pointer and length to the code.

This approach does not scale well with multiple files, because you need to manually include all headers. It is good for cases where there is a few odd files that need specific handling anyway, such as compiled shaders.

## ROM FS
Instead of linking the files directly, you can use a virtual filesystem. The data is still compiled into the binary, but you can use typical file interfaces, such as `FILE` and `std::fstream`. This way you can iterate directories, dynamically list files, et cetera. This is the way to go for larger projects with multiple textures. 

To use it, you need to enable `romfs` in the makefile by uncommenting the line:
```makefile
ROMFS		:=	romfs
```
This way, any file in the `romfs` folder will be compiled in.

You can also redirect the graphics compilation folder to the `romfs` folder, so you do not have to copy them manually:
```makefile
GFXBUILD	:=	$(ROMFS)/gfx
```

To use the filesystem, just init it before use:
```cpp
Result r = romfsInit();
if (R_SUCCEEDED(r)) {
    // handle files...
    romfsExit();
}
```

Then, all files in the `romfs` folder will be visible in the filesystem under `romfs:/`.

## SD Card
Mutable data that needs to persist, such as saves, data caches, and downloadable content, needs to be stored on the SD card. 

The SD card is also useful for development, since you do not need to link and transfer assets over and over again. 
You can also use it to avoid some legal issues when porting old games - you can instruct users to put original game assets they own onto the SD card instead of distributing them yourself. Maybe you could use it to store ROMs for a custom emulator.

**The SD filesystem is automatically initialised and mounted** before the application start, any file operations should work automatically. With Homebrew, it seems like you get direct access to the root of the SD card, so tread carefully.

When debugging on Citra, local files are visible in the system in `Citra/sdmc` folder.

## Filesystem info
When dealing with filesystems, you can use additional functionality to inspect directory contents:

```cpp
    // open directory info as a generic handle
    Handle handle;
	Result r = FSUSER_OpenDirectory(
        &handle, sdArchive, 
        fsMakePath(PATH_ASCII, "/3ds/mypath")
    );

    // check if the directory exists and is visible
	if (R_SUCCEEDED(r)) {
        
        // create a result buffer and read max 32 items from it
        // you can also query the number of items and create it dynamically
        FS_DirectoryEntry entries[32];
        u32 numRead;
        Result r2 = FSDIR_Read(sdDirH, &numRead, 32, entries);

        // in general this should be a success, but check to be sure
        if (R_SUCCEEDED(r2)) {

            for (int i = 0; i < (int)numRead; ++i) {
                
                // names are local and in utf16; handle as you need
                char name[64] = { "/3ds/mypath/" };
                int len = strlen("/3ds/mypath/");
                utf16_to_utf8((uint8_t*)name + len, entries[i].name, 63-len);
                
                // do something with the path
                printf("%s\n", name);
            }
        }

        // finish up
    	svcCloseHandle(handle);
	}
```

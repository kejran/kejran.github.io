---
layout: post
title:  "Environment"
date:   2022-08-12 17:30:00 +0200
categories: 3ds
---

This is part one of the [Nintendo 3DS Homebrew series.]({% post_url 2022-08-12-3ds-intro %})

Here I will go over necessary steps to get the software stack running. Again, this is not a step-by-step guide, but more of an overview.

## Toolchain setup
To start with developing software, we need an appropriate toolchain to cross-compile to ARMv6, link necessary libraries, and package into cfw-digestible file.

**devkitPRO** provides a complete toolchain - **devkitARM**. [Follow the instructions on the site to install it.](https://devkitpro.org/wiki/Getting_Started) Make sure you reopen session/terminal to update env variables. 

Once installed, navigate to the `examples/3ds/templates` folder, copy the application project to your workspace, and run `make`. Hopefully, you will end up with a `.elf` binary and a `.3dsx` package.

## Build system
Unfortunately, the entire devkitPRO flow is based on handcrafted makefiles, which is rather painful for modern software development. If you can get by with just `make`, I recommend just sticking with it unless you really need something else.

The toolchain does have some degree of support for `cmake`. Sadly, it is not used in examples or open-source homebrew projects, which means that using it is a large reverse-engineering effort. I will dedicate a later post to getting a project running with `cmake`.

## Emulator
It is best to run your application in an emulator first, as it will allow  you to iterate faster and spot various issues with your code. 

The most popular and complete emulator for the 3DS system is [Citra](https://citra-emu.org/). Download it and install. On Windows, make sure to enable the log window in `Emulation > Configure > General > Debug`. 

You can manually open your built `.3dsx` directly in Citra. Alternatively, make a shortcut or bash file to `make` and launch the emulator with the binary.  

## IDE 
Developing from terminal or with just a dumb text editor does not scale well, so I highly suggest you setup a code editor that can handle Makefile projects, or setup a separate project that will provide some form of linting and will dispatch `make` targets. 

### Visual Studio Code
Personally, I decided to go with *Visual Studio Code*. It is honestly good enough for most of my programming needs, and has extensions for pretty much any build workflow. You just need to accept that you will use 128000 Apollo Guidance Computers worth of memory to run it. Or to make things personal, four original Nintendo 3DSes. But I digress.

To start, open the folder containing the `Makefile`. You can either use vscode *tasks*, or a *Makefile plugin*. Personally, I recommend using *tasks*, or a mix of both - tasks will let you run custom commands, such as running the emulator with a built binary.

### Basic Task Setup
Create a `.vscode/tasks.json` file, with the following content:
```json
{
    "version": "2.0.0",
    "tasks": [

    ]
}
```
We will now fill it with necessary tasks. 
### Build task
First, create a task that will build the entire project. This pretty much just calls `make` on the project. Add the following object to the `tasks` array:
```json
{
    "label": "make release",
    "type": "process",
    "command": "make",
    "group": {
        "kind": "build",
        "isDefault": true
    }
}
```
### Clean task
In the same way, add any other `make` targets you need, such as `clean`: 
```json
{
    "label": "clean",
    "type": "process",
    "command": "make",
    "args": [ "clean" ]
}
```
### Run task
Lastly, create a task that will launch the emulator with the built binary. By adding the `dependsOn` field, vscode will first always run `make` before each run - which in turn will only compile when your code changed.
```json
{
    "label": "run",
    "type": "shell",
    "isBackground": true,
    "command": "PATH_TO_CITRA/citra-qt.exe",
    "args": [
        "${workspaceFolder}/${workspaceFolderBasename}.3dsx"
    ],
    "dependsOn": ["make release"]
}
```

### Using the tasks
After you define tasks, you can access them from `Ctrl+Alt+T` or `Ctrl+Shift+P > Run Tasks`. Find the `run` task and start it. 

By now, you should have a complete setup that allows you to build your code and launch directly to the emulator.

### Adding 3ds includes
To vscode stop complaining on imports and enable intellisense, you need to add a `c_cpp_properties.json` file. Fill it with something along these lines, make sure to fixup your paths:

```json
{
  "configurations": [
    {
      "name": "3DS",
      "includePath": [
        "${workspaceFolder}",
        "${workspaceFolder}/build",
        "C:/devkitPro/libctru/include/**",
        "C:/devkitPro/devkitARM/arm-none-eabi/include/**"
      ],
      "defines": [
        "_DEBUG",
        "_UNICODE",
        "WIN32",
        "ARM7",
        "ARM9",
        "_GNU_SOURCE"
      ],
      "browse": {
        "path": [
          "${workspaceFolder}",
          "${workspaceFolder}/include/**",
          "C:/devkitPro/libctru/include/**",
          "C:/devkitPro/devkitARM/arm-none-eabi/include/**"
        ],
        "limitSymbolsToIncludedHeaders": true,
        "databaseFilename": ""
        },
      "cStandard": "c17",
      "cppStandard": "c++20",
      "compilerPath": "C:/devkitPro/devkitARM/bin/arm-none-eabi-g++.exe"
    }
  ],
  "version": 4
}
```
*note to self: this file needs to be udpated to use env variables.*


## Deploying on real hardware
It is not very exciting to use your Homebrew on an emulator. If you already have a CFW console, you can directly run the compiled `.3dsx` files. 

There are many workflows, most commonly using [ftpd](https://github.com/mtheall/ftpd). 
Personally, I start a HTTP server in my build folder with `python3 -m http.server`, then on console, I use *FBI* > *Remote Install*. 

Once you have the package on the SD card, navigate to it in the *Homebrew Launcher* and run it. 

If you ran the template project, you should see a `Hello World` message.

[Read on to the next part, where I go over the hardware on all 3DS models.]({% post_url 2022-08-16-3ds-hardware %})
# LVGL ported to Renesas EK-RA8D1 using Ac6 and Arm2D

## Overview

This project is pre-configured to use Ac6 and Arm2D for Renesas EK-RA8D1 in e2 Studio.

The EK-RA8D1 evaluation kit enables users to effortlessly evaluate the features of the RA8D1 MCU Group and develop embedded systems applications using Renesas’ Flexible Software Package (FSP) and e2 studio IDE. Utilize rich on-board features along with your choice of popular ecosystem add-ons to bring your big ideas to life.

The MCU has a Cortex-M85 core which utilizes the Helium (SIMD) instruction set of Arm. Besides that the chip is equipped with a GPU (called DAVE2D) to off load the MCU. 

## Buy

You can purchase the Renesas EK-RA8D1 board from many distributors. See the sources at https://renesas.com/ek-ra8d1

## Specification

### CPU and Memory
- **MCU:** R7FA8D1BHECBD (Cortex-M85, 480MHz)
- **RAM:** 1MB internal, 64MB external SDRAM
- **Flash:** 2MB internal, 64MB External Octo-SPI Flash
- **GPU:** Dave2D

### Display and Touch
- **Resolution:** 480x854
- **Display Size:** 4.5”
- **Interface:** 2-lane MIPI
- **Color Depth:** 24-bit
- **Technology:** IPS
- **DPI:** 217 px/inch
- **Touch Pad:** Capacitive

### Connectivity
- Camera expansion board
- Micro USB device cable (type-A male to micro-B male)
- Micro USB host cable (type-A male to micro-B male)
- Ethernet patch cable

## Getting started

### Hardware setup
- Attach the MIPI LCD PCB to the main PCB
- On SW1 DIP switched (middle of the board) 7 should be ON, all others are OFF
- Connect the USB cable to the `Debug1` (J10) connector

### Software setup
- Add Ac6
    - Download and install the **Ac6** compiler from from [Arm's website](https://developer.arm.com/Tools%20and%20Software/Arm%20Compiler%20for%20Embedded)
    - To register a community license go to `bin` folder of the compiler and in a Terminal run `armlm.exe activate -server https://mdk-preview.keil.arm.com -product KEMDK-COM0`
    - In E2 Studio open `Window` -> `Preferences`. Select `Toolchains` on the left, click `Add...` and browse the `bin` folder of the Ac6 compiler.
- Get this project
    - Clone the repository with the following command:
        ```
        git clone https://github.com/lvgl/lv_port_renesas_ek-ra8d1_llvm.git --recurse-submodules
        ```
        Downloading the `.zip` from GitHub doesn't work as it doesn't download the submodules.
    - Follow the *RA family* section of the [*documentation*](https://docs.lvgl.io/master/integration/chip/renesas.html#get-started-with-the-renesas-ecosystem) to prepare your environment and import the project


## Setting up a project manually 

### Use Ac6

Although this project is already pre-configured for Ac6 you might be interested in knowing what are main steps of changing toolchain. First, be sure that Ac6 is added to e² Studio as toolchain as described above.

Note that, e² Studio's Ac6 support has some bug and migrating a project to Ac6 requires several non trivial steps.

1. Click `File` -> `Properties` -> `C/C++ Build` -> `Tool Chain Editor`
2. In `Current Toolachain` select `Arm compiler 6.22` and confirm the change of the Toolchain
3. Still in Project properties select `C/C++ Build` -> `Settings`-> `ARM C Compiler 6.22` -> `Target` set
   - `target`: `arm-arm-none-eabi`
   - `mcpu`: `cortex-m85`
4. In `ARM C Compiler 6.22` -> `Miscellaneous` set the `Other flags` to `-Omax -flto -Wno-int-conversion -Wno-deprecated-non-prototype -Wno-implicit-function-declaration -Wno-unused-but-set-variable -Wfloat-equal -Waggregate-return -Wshadow -Wpointer-arith -Wconversion -Wmissing-declarations -Wuninitialized -Wunused -Wno-license-management -Wextra -Wno-implicit-int-conversion -Wno-sign-conversion -mfloat-abi=hard`
5. In `ARM C Linker 6.22` -> `Target` set
    - linker target: Cortex-M85
6. In `Linker` -> `Image Layout` set `Scatter File` to `${workspace_loc:/${ProjName}}/script/fsp.scat`
7. In `Linker` -> `Miscellaneous` the `Other flags` should be `--lto --library_type=standardlib --no_startup --via="${workspace_loc:/${ProjName}/script}/ac6/fsp_keep.via"`
8. In `ARM Assembler 6.22` -> `Miscellaneous` the `Other flags` to `-x assembler-with-cpp` 


### Use Arm2D
Arm2D directly uses the Helium SIMD (a.k.a. vector) instructions. With Ac6 and Arm2D you can expect 20-30% performance boost compared to GCC. Note that only Ac6 can can utilize Arm2D's full power. GCC and LLVM might not be able to compile not or might not be able to use it well. 

1. In order to utilize its power be sure to `mcpu` to `cortex-m85`. To test without SIMD use `cortex-m85+nomve`
2. In `C/C++ Build` -> `Settings`-> `ARM C Compiler 6.22` -> `Target` check the `Vectorization` checkbox
3. Arm2D can be downloaded from `https://github.com/ARM-software/Arm-2D`
4. CMSIS DSP library needs to be added to the project
5. For better performance be sure the `LTO` is enabled and `-Omax` is used. `-Omax` cannot be selected from the list, but needs to be added manually as compiler flag (see above)
6. Arm2D tries to read/write multiple data with a single instruction. Therefore it's important to use the fastest memory for LVGL's buffer. Usually it's the BSS and on this board it's 64kB. An array can be placed to this section like this: `static uint8_t partial_draw_buf[64 * 1024] BSP_PLACE_IN_SECTION(".bss.dtcm_bss") BSP_ALIGN_VARIABLE(BSP_STACK_ALIGNMENT);`

### Other notes
- In Debugger configuration on  the `Debugger` tab set the `GDB Command` to `arm-none-eabi-gdb --fix-segments`. Else you will get a DSI error. 
- For reference the compiler log for a file should look like this (with a different path to the project):
```
armclang --target=arm-arm-none-eabi -mcpu=cortex-m85 -fvectorize -D_RENESAS_RA_ -D_RA_CORE=CM85 -D_RA_ORDINAL=1 -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/src" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/src/Arm-2D/Helper/Include" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/src/Arm-2D/Library/Include" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/arm/CMSIS_5/CMSIS/Core/Include" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/arm/CMSIS_5/CMSIS/DSP/PrivateInclude" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/arm/CMSIS_5/CMSIS/DSP/Include" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/fsp/inc" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/fsp/inc/api" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/fsp/inc/instances" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/fsp/src/rm_freertos_port" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/aws/FreeRTOS/FreeRTOS/Source/include" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra_gen" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra_cfg/fsp_cfg/bsp" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra_cfg/fsp_cfg" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra_cfg/aws" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/ra/tes/dave2d/inc" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/src/lvgl" -I"/home/kisvegabor/projects/lvgl/e2_studio-workspace/lv_port_renesas_ek-ra8d1_ac6_arm2d/src/lvgl/tests" -I"." -O3 -g -Omax -flto -Wno-int-conversion -Wno-deprecated-non-prototype -Wno-implicit-function-declaration -Wno-unused-but-set-variable -Wfloat-equal -Waggregate-return -Wshadow -Wpointer-arith -Wconversion -Wmissing-declarations -Wuninitialized -Wunused -Wno-license-management -Wextra -Wno-implicit-int-conversion -Wno-sign-conversion -mfloat-abi=hard -MD -MP -c -o "src/board_init.o" "../src/board_init.c"
```

## Contribution and Support

If you find any issues with the development board feel free to open an Issue in this repository. For LVGL related issues (features, bugs, etc) please use the main [lvgl repository](https://github.com/lvgl/lvgl). 

If you found a bug and found a solution too please send a Pull request. If you are new to Pull requests refer to [Our Guide](https://docs.lvgl.io/master/CONTRIBUTING.html#pull-request) to learn the basics.


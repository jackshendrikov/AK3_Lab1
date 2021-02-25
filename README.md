# Assembly Simple
Creating a minimal software project in the assembly language.

## Prerequisites

To perform the work you will need the Linux operating system and the following software:
1. Install the qemu-system-gnuarmeclipse computer emulator:
https://xpack.github.io/qemu-arm/

2. Download the archive from:
https://github.com/xpack-dev-tools/qemu-arm-xpack/releases/
and run the following commands:
```sh
>>> mkdir -p ~/opt
>>> cd ~/opt
>>> tar xvf ~/Downloads/xpack-qemu-arm-2.8.0-12-linux-x64.tar.gz
>>> chmod -R -w xpack-qemu-arm-2.8.0-12
```

Check performance:
```sh
>>> ~/opt/xpack-qemu-arm-2.8.0-12/bin/qemu-system-gnuarmeclipse --version
```

The result of successful execution:
```
xPack 64-bit QEMU emulator version 2.8.0-12 (v2.8.0-4-20190211-47-g109b69f49a-dirty)
Copyright (c) 2003-2016 Fabrice Bellard and the QEMU Project developers
```

3. Install the toolchains:
	* arm-none-eabi-gcc:
`>>> sudo apt-get install arm-none-eabi-gcc ` 
or
`>>> sudo apt-get install gcc-arm-none-eabi`

	* arm-none-eabi-newlib: 
`>>> sudo apt-get install arm-none-eabi-newlib`
or
`>>> sudo apt-get install libnewlib-arm-none-eabi`

	* arm-none-eabi-gdb:
`>>> sudo apt-get install arm-none-eabi-gdb`
or
`>>> sudo apt-get install gdb-arm-none-eabi`
or
`>>> sudo apt-get install gdb-multiarch`

	* arm-none-eabi-binutils:
`>>> sudo apt-get install arm-none-eabi-binutils`
or
`>>> sudo apt-get install binutils-arm-none-eabi`

4. Also install (if not done before):

```sh
>>>  sudo apt-get install stlink-tools
>>>  sudo apt-get install make
```


## Task
1. Create a [start.S](./start.S) file in the project directory (create a directory
project). This file is required to store the exception vector table and the hard_reset tag from which the program starts.

<details>
<summary>Content of <cite>start.S</cite> (with comments)</summary><p align="left">

```assembly
.syntax unified
.cpu cortex-m4
//.fpu softvfp
.thumb

// Global memory locations.
.global vtable
.global reset_handler
/*
 * vector table 
 */
.type vtable, %object
vtable:
    .word __stack_start
    .word __hard_reset__+1
    .size vtable, .-vtable
__hard_reset__:
    ldr r0, =__stack_start
    mov sp, r0
    b __hard_reset__
```
</details>

2. Create a `lscript.ld` file. This is a linker script that indicates how much memory the program can use.

<details>
  <summary>Content of <cite>lscript.ld</cite> (with comments)</summary><p align="left">
  
```
// linker script for stm32f1 platforms 
// Define the end of RAM and limit of stack memory

MEMORY
{   
// We mark flash memory as read-only, since that is where the program lives. STM32 //chips map their flash memory to start at 0x08000000, and we have 32KB of flash //memory available. 
    FLASH ( rx )      : ORIGIN = 0x08000000, LENGTH = 1M
//  We mark the RAM as read/write, and as mentioned above it is 4KB long starting at //address 0x20000000. 
    RAM ( rxw )       : ORIGIN = 0x20000000, LENGTH = 128K
}
__stack_start = ORIGIN(RAM) + LENGTH(RAM); // Start of the stack address
```
</details>

3. Assemble the project, to do this:
```sh
>>> arm-none-eabi-gcc -x assembler-with-cpp -c -O0 -g3 -mcpu=cortex-m4 -mthumb -Wall start.S -o start.o
>>> arm-none-eabi-gcc start.o -mcpu=cortex-m4 -mthumb -Wall --specs=nosys.specs -nostdlib -lgcc -T./lscript.ld -o firmware.elf
>>> arm-none-eabi-objcopy -O binary -F elf32-littlearm  firmware.elf  firmware.bin
```
4. You can then run this code in qemu using the gdb debugger.
Write in PATH where qemu is, note that this must be done every time you start the console, so it is best to add the following line to, for example, /home/user/.bashrc file:
`PATH=$PATH:~/opt/xpack-qemu-arm-2.8.0-12/bin/`

Perform:
```sh
>>> qemu-system-gnuarmeclipse --verbose --verbose --board STM32F4-Discovery --mcu STM32F407VG -d unimp,guest_errors --image firmware.bin --semihosting-config enable=on,target=native -s -S
```

With flags -s -S qemu awaits connection of external debugging software with port tcp::1234. Open a new terminal window and perform:
```sh
>>> arm-none-eabi-gdb firmware.elf 
```

If successful:
```
 For help, type "help".
     Type "apropos word" to search for commands related to "word"...
     Reading symbols from firmware.elf...
     (gdb)
```

Enter `target extended-remote:1234`. The program is now ready for debugging. Then the program waits for the entry of gdb commands. In order to perform step by step enter:
```sh
>>> step
```
Then press Enter and run the program.
5. Now you can automate the creation of firmware. Create a GNU Makefile:

<details>
<summary>Content of <cite>Makefile</cite></summary><p align="left">

```make
SDK_PREFIX?=arm-none-eabi-
CC = $(SDK_PREFIX)gcc
LD = $(SDK_PREFIX)ld
SIZE = $(SDK_PREFIX)size
OBJCOPY = $(SDK_PREFIX)objcopy
QEMU = qemu-system-gnuarmeclipse
BOARD ?= STM32F4-Discovery
MCU=STM32F407VG
TARGET=firmware
CPU_CC=cortex-m4
TCP_ADDR=1234
deps = \
	   start.S	\
	   lscript.ld
all: target
target:
	$(CC) -x assembler-with-cpp -c -O0 -g3 -mcpu=$(CPU_CC)  -Wall start.S -o start.o
	$(CC) start.o -mcpu=$(CPU_CC)  -Wall --specs=nosys.specs -nostdlib -lgcc -T./lscript.ld -o $(TARGET).elf
	$(OBJCOPY) -O binary -F elf32-littlearm  $(TARGET).elf  $(TARGET).bin
qemu:
	$(QEMU)  --verbose --verbose --board $(BOARD) --mcu $(MCU) -d unimp,guest_errors --image $(TARGET).bin --semihosting-config enable=on,target=native -gdb tcp::$(TCP_ADDR)  -S
clean:
	-rm *.o
	-rm *.elf
	-rm *.bin
```

The `make` command will create a firmware, the `make qemu` command will launch an emulator with the above settings.
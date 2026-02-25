---
layout: page
title: "Step 1: Bare Metal Assembly on RP2040"
description: "Write ARM assembly directly on the microcontroller. Nothing between your instructions and the silicon."
permalink: /level-0/step-01/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-0/">Level 0</a> / Step 1'
---

## Overview

In this step, you will write ARM assembly language and run it directly on the RP2040. There is no operating system. There is no runtime. There is no compiler generating code on your behalf. You will write instructions, and the processor will execute them — one at a time, in order, exactly as written.

This is the most fundamental form of programming that exists. Everything else is built on top of it.

> **Key Insight:** There is nothing between your instructions and the silicon. When you write `MOV R0, #5`, the processor literally loads the value 5 into register R0. There is no interpretation, no virtual machine, no garbage collector. Understanding this changes how you think about every layer above it.

## What You Will Learn

- The ARM Thumb instruction set (the subset used by the Cortex-M0+)
- How registers, the stack, and the program counter work
- Memory-mapped I/O: controlling hardware by reading and writing memory addresses
- The RP2040 boot sequence: what happens from power-on to your first instruction
- How to blink an LED by directly manipulating GPIO registers

## Background: The ARM Cortex-M0+

The RP2040 contains two ARM Cortex-M0+ cores. The M0+ implements the **ARMv6-M** architecture, which uses the **Thumb** instruction set — a compact, 16-bit encoding of ARM instructions. This means most instructions are 2 bytes wide, with a few 32-bit instructions mixed in.

### Registers

The Cortex-M0+ has 16 registers:

| Register | Name | Purpose |
|---|---|---|
| `R0`–`R7` | Low registers | General-purpose; accessible by all Thumb instructions |
| `R8`–`R12` | High registers | General-purpose; limited Thumb instruction access |
| `R13` | `SP` | Stack Pointer — points to the top of the current stack |
| `R14` | `LR` | Link Register — holds the return address after a branch-with-link |
| `R15` | `PC` | Program Counter — address of the next instruction to execute |

There is also the **Program Status Register (xPSR)** which contains condition flags (zero, negative, carry, overflow) set by arithmetic and comparison instructions.

### Memory-Mapped I/O

On the RP2040, hardware peripherals are controlled by reading and writing to specific memory addresses. The GPIO controller, for example, lives at address `0x40014000`. To turn on an LED connected to GPIO pin 25 (the onboard LED on the Pico), you write a specific bit to a specific address. There are no special I/O instructions — it is all just memory access.

Key addresses you will use:

| Address | Register | Purpose |
|---|---|---|
| `0x40014000` | `GPIO_OUT` | Set output levels for GPIO pins |
| `0x40014008` | `GPIO_OE` | Enable output mode for GPIO pins |
| `0x4000C000` | `IO_BANK0_BASE` | Pin function select (mux control) |
| `0x4000F000` | `PADS_BANK0_BASE` | Pad control (drive strength, pull-ups) |

## Project A: Add Two Numbers

Start with the simplest possible program. Load two values into registers, add them, store the result.

```armasm
.thumb                    @ Use Thumb instruction set
.global _start

_start:
    MOV R0, #7            @ Load 7 into R0
    MOV R1, #13           @ Load 13 into R1
    ADD R2, R0, R1        @ R2 = R0 + R1 = 20

    @ Store the result in memory
    LDR R3, =result       @ Load address of 'result' into R3
    STR R2, [R3]          @ Store R2 at that address

    @ Infinite loop (halt)
loop:
    B loop                @ Branch to self forever

.data
result: .word 0           @ Reserve a 32-bit word, initialized to 0
```

### What is happening here

1. `MOV R0, #7` — The processor writes the literal value 7 into register R0. This takes one clock cycle.
2. `ADD R2, R0, R1` — The ALU (arithmetic logic unit) adds the contents of R0 and R1 and writes the result to R2.
3. `LDR R3, =result` — This is a pseudo-instruction. The assembler replaces it with a PC-relative load that fetches the address of the `result` label from a literal pool.
4. `STR R2, [R3]` — Write the contents of R2 to the memory address held in R3.
5. `B loop` — Branch to itself. Without an OS, there is nowhere to "return" to. The processor must always be executing something.

### Building and Flashing

```bash
# Assemble
arm-none-eabi-as -mcpu=cortex-m0plus -mthumb -o add.o add.s

# Link (using a linker script for the RP2040 memory map)
arm-none-eabi-ld -T memmap.ld -o add.elf add.o

# Convert to UF2 for drag-and-drop flashing
elf2uf2 add.elf add.uf2

# Hold BOOTSEL on the Pico, plug in USB, copy the .uf2 file
cp add.uf2 /media/RPI-RP2/
```

## Project B: Blink an LED

Now use memory-mapped I/O to control hardware. You will toggle GPIO pin 25 (the onboard LED) by writing directly to the GPIO registers.

```armasm
.thumb
.global _start

.equ RESETS_BASE,       0x4000C000
.equ IO_BANK0_BASE,     0x40014000
.equ SIO_BASE,          0xD0000000
.equ GPIO_OE_SET,       0xD0000024
.equ GPIO_OUT_SET,      0xD0000014
.equ GPIO_OUT_CLR,      0xD0000018

.equ LED_PIN,           25

_start:
    @ Step 1: Take IO_BANK0 out of reset
    LDR R0, =RESETS_BASE
    LDR R1, [R0, #0]         @ Read RESET register
    LDR R2, =~(1 << 5)       @ Clear bit 5 (IO_BANK0)
    AND R1, R1, R2
    STR R1, [R0, #0]         @ Write back

wait_reset:
    LDR R1, [R0, #8]         @ Read RESET_DONE
    LDR R2, =(1 << 5)
    TST R1, R2
    BEQ wait_reset            @ Wait until IO_BANK0 reset is done

    @ Step 2: Set GPIO 25 function to SIO (function 5)
    LDR R0, =IO_BANK0_BASE
    MOV R1, #5               @ Function 5 = SIO
    LDR R2, =(LED_PIN * 8 + 4)  @ Offset for GPIO25 CTRL register
    STR R1, [R0, R2]

    @ Step 3: Enable output on GPIO 25
    LDR R0, =GPIO_OE_SET
    LDR R1, =(1 << LED_PIN)
    STR R1, [R0]

blink_loop:
    @ Turn LED on
    LDR R0, =GPIO_OUT_SET
    LDR R1, =(1 << LED_PIN)
    STR R1, [R0]

    @ Delay
    BL delay

    @ Turn LED off
    LDR R0, =GPIO_OUT_CLR
    LDR R1, =(1 << LED_PIN)
    STR R1, [R0]

    @ Delay
    BL delay

    B blink_loop

delay:
    LDR R0, =0x100000        @ ~1M iterations
delay_loop:
    SUB R0, R0, #1
    BNE delay_loop
    BX LR                     @ Return
```

### What is happening here

This program does exactly what an operating system or HAL library would do, but you are doing it yourself:

1. **Reset the peripheral** — The RP2040 boots with most peripherals held in reset. You must release IO_BANK0 from reset before using any GPIO pins.
2. **Configure the pin mux** — Each GPIO pin can be connected to different internal peripherals. You select function 5 (SIO — Single-cycle IO) to control it directly.
3. **Enable output** — Set the output enable bit for pin 25.
4. **Toggle the pin** — Write to the set/clear registers to turn the LED on and off.
5. **Delay** — A busy-wait loop. The processor does nothing useful for ~1 million cycles. (Later you will learn about timer interrupts as a better approach.)

## The Boot Sequence

When you power on the RP2040, the following happens:

1. The internal ROM bootloader runs (burned into the chip at the factory)
2. It looks for a valid flash image (checks a CRC in the first 256 bytes)
3. It configures the XIP (execute-in-place) flash interface
4. It reads the **vector table** from the start of flash:
   - Word 0: Initial stack pointer value
   - Word 1: Address of the reset handler (your `_start`)
5. The processor sets SP and jumps to your code

This is why your linker script matters — it tells the linker where to place the vector table, your code, and your data in the RP2040's memory map.

## Exercises

1. **Modify the blink rate.** Change the delay loop count. What happens at `0x10000`? At `0x1000000`? Can you estimate the relationship between loop count and time?

2. **Blink a pattern.** Make the LED blink SOS in Morse code (three short, three long, three short).

3. **Use two LEDs.** Connect an external LED to another GPIO pin. Alternate blinking between the onboard and external LED.

4. **Read a button.** Wire a push button to a GPIO pin configured as input. Only blink the LED while the button is held down.

5. **Count in binary.** Wire 4 LEDs to 4 GPIO pins. Count from 0 to 15, displaying the count in binary on the LEDs.

## Key Takeaways

- A processor executes instructions from memory, one at a time, in sequence (unless a branch changes the flow)
- Registers are the fastest storage — everything flows through them
- Hardware is controlled by reading and writing specific memory addresses
- There is no magic: an LED blinks because you wrote a 1 to bit 25 of a 32-bit register at address `0xD0000014`
- Without an OS, your program is the only thing running — you control everything, and you are responsible for everything

---

<p>
  <a href="/level-0/">&larr; Level 0 Overview</a> · <a href="/level-0/step-02/">Step 2: C, Compilation, and the Toolchain &rarr;</a>
</p>

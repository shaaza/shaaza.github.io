---
layout: page
title: "Step 2: C, Compilation, and the Toolchain"
description: "Rewrite your assembly programs in C. Then open the hood and see what the compiler actually generates."
permalink: /level-0/step-02/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-0/">Level 0</a> / Step 2'
---

## Overview

In Step 1 you wrote raw assembly — you controlled every register, every memory access, every branch. It worked, but it was tedious. In this step you will rewrite the same programs in C and then use the compiler to reveal what happens between your C code and the machine instructions that actually run.

The goal is not just to learn C. It is to understand the **compilation pipeline** — the series of transformations that turns human-readable source code into a binary that a processor can execute. Every language you use in this course (Python, JavaScript, Rust, Haskell) goes through some version of this process. Understanding it once, concretely, gives you a mental model that transfers everywhere.

## What You Will Learn

- The four stages of C compilation: preprocessor, compiler, assembler, linker
- What header files are and why `#include` exists
- What the linker does and why linking errors are different from compile errors
- What `crt0` is and why your `main()` function is not actually the first thing that runs
- How compiler optimizations transform your code — and how to inspect the results

## The Compilation Pipeline

When you run `gcc -o blink blink.c`, four separate programs run in sequence:

```
  Source code (.c)
       |
       v
  [1. Preprocessor]  — handles #include, #define, #ifdef
       |
       v
  Preprocessed source (.i)
       |
       v
  [2. Compiler]       — translates C to assembly
       |
       v
  Assembly (.s)
       |
       v
  [3. Assembler]      — translates assembly to machine code
       |
       v
  Object file (.o)
       |
       v
  [4. Linker]         — combines objects, resolves symbols, produces binary
       |
       v
  Executable (.elf)
```

You can stop at any stage and inspect the output. This is how you learn what the compiler is actually doing.

### Stage 1: The Preprocessor

The preprocessor is a text substitution engine. It runs before the compiler sees anything.

```c
#include <stdio.h>          // Paste the contents of stdio.h here
#define LED_PIN 25           // Replace every "LED_PIN" with "25"
#define DELAY_COUNT 1000000  // Replace every "DELAY_COUNT" with "1000000"

#ifdef USE_WIFI
#include "wifi.h"            // Only include if USE_WIFI is defined
#endif
```

To see the preprocessor output:

```bash
arm-none-eabi-gcc -E blink.c -o blink.i
```

Open `blink.i` and you will see thousands of lines — all the contents of every included header, with all macros expanded. Your original code is at the bottom, transformed by substitution.

### Stage 2: The Compiler

The compiler translates preprocessed C into assembly language. This is where the real work happens — parsing, type checking, optimization, register allocation, instruction selection.

```bash
arm-none-eabi-gcc -S -mcpu=cortex-m0plus -mthumb blink.c -o blink.s
```

Open `blink.s` and you will see ARM Thumb assembly — the same kind you wrote by hand in Step 1.

### Stage 3: The Assembler

The assembler translates assembly into machine code — the actual binary encoding of each instruction. The output is an **object file** (`.o`), which contains machine code but is not yet a complete program. Symbol references (function calls, global variables) are left as placeholders.

```bash
arm-none-eabi-as -mcpu=cortex-m0plus -mthumb -o blink.o blink.s
```

### Stage 4: The Linker

The linker combines one or more object files, resolves all symbol references, and produces the final executable. It uses a **linker script** to determine where each section goes in memory.

```bash
arm-none-eabi-ld -T memmap.ld -o blink.elf blink.o
```

If you call a function that does not exist in any object file, the linker will fail. This is why linker errors look different from compiler errors — they happen at a different stage.

## What is crt0?

When you write a C program with a `main()` function, `main()` is not the entry point. The actual entry point is `_start`, which comes from a file called **crt0** (C runtime zero). This startup code:

1. Sets up the stack pointer
2. Copies initialized data from flash to RAM (the `.data` section)
3. Zeros out uninitialized data (the `.bss` section)
4. Calls any global constructors (in C++)
5. Calls `main()`
6. If `main()` returns, enters an infinite loop (on bare metal, there is nowhere to return to)

In Step 1, you wrote this startup code yourself in assembly. Now the toolchain provides it.

## Project: LED Blinker in C

Rewrite the assembly blinker from Step 1 using C and the Pico SDK:

```c
#include "pico/stdlib.h"

#define LED_PIN 25

int main() {
    gpio_init(LED_PIN);
    gpio_set_dir(LED_PIN, GPIO_OUT);

    while (1) {
        gpio_put(LED_PIN, 1);
        sleep_ms(500);
        gpio_put(LED_PIN, 0);
        sleep_ms(500);
    }

    return 0;  // Never reached
}
```

### Compile and inspect the assembly

```bash
# Generate assembly output
arm-none-eabi-gcc -S -mcpu=cortex-m0plus -mthumb -O0 \
    -I${PICO_SDK_PATH}/src/common/pico_stdlib/include \
    -I${PICO_SDK_PATH}/src/rp2_common/hardware_gpio/include \
    blink.c -o blink_O0.s

# Now compile with optimization
arm-none-eabi-gcc -S -mcpu=cortex-m0plus -mthumb -O2 \
    -I${PICO_SDK_PATH}/src/common/pico_stdlib/include \
    -I${PICO_SDK_PATH}/src/rp2_common/hardware_gpio/include \
    blink.c -o blink_O2.s
```

Compare the two outputs. The difference is striking.

## Exercise: Compare -O0 and -O2

This is the core exercise of Step 2. Compile the same C code with different optimization levels and study what changes.

### What to look for in -O0 (no optimization)

```armasm
@ Typical -O0 output for a simple function
push    {r7, lr}          @ Save frame pointer and link register
sub     sp, sp, #8        @ Allocate stack space for local variables
str     r0, [sp, #4]      @ Store argument to stack (even though it was in R0!)
ldr     r3, [sp, #4]      @ Load it right back into R3
mov     r0, r3            @ Move to R0 for the function call
bl      gpio_init         @ Call gpio_init
```

Notice: the compiler stores the argument to the stack and immediately loads it back. This is intentional — at `-O0`, each C statement is compiled independently, with no awareness of what comes before or after. This makes debugging easier (every variable is in memory where a debugger can find it) but produces terrible code.

### What to look for in -O2 (optimization enabled)

```armasm
@ Typical -O2 output for the same function
push    {r4, lr}          @ Save only what's needed
mov     r4, r0            @ Keep argument in register — no stack spill
bl      gpio_init         @ Call directly
```

The optimizer:
- Eliminates redundant loads and stores
- Keeps values in registers instead of spilling to the stack
- Inlines small functions
- Reorders instructions for pipeline efficiency
- Eliminates dead code (code that can never run)

### The diff command

```bash
diff blink_O0.s blink_O2.s
```

Or for a side-by-side view:

```bash
diff -y --width=120 blink_O0.s blink_O2.s | head -80
```

Count the instructions in each version. The `-O2` version will typically be 30–60% smaller.

## Understanding the Output

Here are patterns to look for when reading compiler-generated assembly:

| C Construct | Assembly Pattern |
|---|---|
| Function call | `BL function_name` (branch with link) |
| Return from function | `BX LR` or `POP {PC}` |
| Local variable | Stack-relative load/store: `LDR R0, [SP, #offset]` |
| Global variable | PC-relative load of address, then load/store through that address |
| `if/else` | Compare (`CMP`) followed by conditional branch (`BEQ`, `BNE`) |
| `while` loop | Conditional branch at the top or bottom of a block |
| `for` loop | Same as `while` — the compiler does not distinguish them |
| Struct field access | Base register + constant offset: `LDR R0, [R1, #8]` |

## Header Files and Linking in Detail

### Header files (`.h`)

A header file contains **declarations** — it tells the compiler that a function or variable exists, and what its type is. It does not contain the implementation.

```c
// gpio.h — declaration only
void gpio_init(uint gpio);
void gpio_set_dir(uint gpio, bool out);
void gpio_put(uint gpio, bool value);
```

When you write `#include "gpio.h"`, the preprocessor pastes these declarations into your file. The compiler can then check that you call the functions correctly. But the actual machine code for these functions is elsewhere — in a library object file.

### Linking

The linker takes your object file and the library object files, matches every function *call* to a function *definition*, and produces the final binary. If you call `gpio_init()` but forget to link the GPIO library, you get:

```
undefined reference to `gpio_init'
```

This is a **linker error**, not a compiler error. The compiler was happy — it saw the declaration in the header. The linker is the one that noticed the definition is missing.

## Exercises

1. **Write a "multiply" program in C.** Multiply two numbers. Compile to assembly with `-O0` and `-O2`. How does the Cortex-M0+ handle multiplication? (Hint: it has a hardware multiplier, unlike some other M0 cores.)

2. **Inspect crt0.** Find the `crt0.S` file in the Pico SDK. Read it. Identify where it sets the stack pointer, copies `.data`, zeros `.bss`, and calls `main()`.

3. **Count instructions.** For the LED blinker, count the total number of instructions in the `-O0` and `-O2` assembly output. Calculate the percentage reduction.

4. **Try -Os (optimize for size).** Compile with `-Os` and compare to `-O2`. Where do they differ? When would you prefer size over speed?

5. **Preprocessor archaeology.** Run `gcc -E` on a file that includes `pico/stdlib.h`. How many lines of preprocessed output do you get? How many header files are transitively included?

## Key Takeaways

- C is not assembly, but it compiles *to* assembly — you can always look
- The compilation pipeline has four distinct stages, each with its own errors and outputs
- The compiler does not generate the code you wrote — it generates code that has the same *behavior*
- Optimization levels dramatically affect the generated code
- `main()` is not the beginning — `crt0` runs first
- Header files are declarations; object files are definitions; the linker connects them

---

<p>
  <a href="/level-0/step-01/">&larr; Step 1: Bare Metal Assembly</a> · <a href="/level-0/step-03/">Step 3: Peripherals — Build a Computer &rarr;</a>
</p>

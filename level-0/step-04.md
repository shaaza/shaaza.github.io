---
layout: page
title: "Step 4: Write a Minimal OS Kernel"
description: "Build a bare-metal operating system kernel from scratch — memory management, task scheduling, and syscalls."
permalink: /level-0/step-04/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-0/">Level 0</a> / Step 4'
---

## Overview

You have written assembly, compiled C, and wired up peripherals. You have a working computer. But in Step 3, your programs ran one at a time, and you managed everything manually — initializing hardware, reading inputs, updating the display, all in one big loop.

Real computers run multiple programs. They protect one program from crashing another. They provide services (print to screen, read a file, allocate memory) through a standard interface so that programs do not need to know the details of the hardware.

The piece of software that does all of this is the **operating system kernel**. In this step, you will write one.

It will be minimal — a few hundred lines of C and assembly. But it will implement the core functions that every OS, from your Pico to a Linux server, must provide: **boot and hardware initialization**, **memory management**, **task scheduling**, **interrupt handling**, and a **syscall interface**.

## What You Will Learn

- What an operating system kernel actually does (and what it does not do)
- How context switching works at the register level
- Cooperative vs. preemptive scheduling
- Timer interrupts and how the OS reclaims control from running tasks
- Memory management on a system without an MMU
- How syscalls provide a controlled interface between user code and kernel code

## What Does an OS Kernel Do?

An OS kernel has exactly four jobs:

| Job | What It Means | Why It Matters |
|---|---|---|
| **Hardware abstraction** | Initialize and manage peripherals | Programs do not need to know register addresses |
| **Memory management** | Allocate and free memory, prevent corruption | One program cannot overwrite another's data |
| **Task scheduling** | Decide which code runs and when | Multiple programs (or threads) share one processor |
| **Syscall interface** | Provide controlled entry points for kernel services | Programs can request resources without direct hardware access |

Everything else — file systems, networking, graphics — is built on top of these four primitives.

## Project: A Kernel for the RP2040

You will implement each of the four kernel functions in C, with small amounts of assembly for the parts that must manipulate registers directly.

### Part 1: Boot and Hardware Initialization

The kernel starts with a boot sequence. You wrote a simpler version in Step 1; now you formalize it.

```c
// boot.c — Kernel entry point

#include "kernel.h"

// Vector table — must be at the start of flash
__attribute__((section(".vectors")))
const uint32_t vector_table[] = {
    (uint32_t)&_stack_top,      // Initial stack pointer
    (uint32_t)kernel_main,       // Reset handler
    (uint32_t)nmi_handler,       // NMI
    (uint32_t)hardfault_handler, // Hard fault
    // ... additional exception vectors
};

void kernel_main(void) {
    // 1. Initialize system clocks
    clock_init();

    // 2. Initialize GPIO subsystem
    gpio_init_all();

    // 3. Initialize the UART for debug output
    uart_init(UART0, 115200);
    uart_puts(UART0, "Kernel booting...\n");

    // 4. Initialize memory allocator
    mem_init();

    // 5. Initialize the scheduler
    scheduler_init();

    // 6. Create initial tasks
    task_create(task_blink, "blink", 256);
    task_create(task_serial, "serial", 512);
    task_create(task_idle, "idle", 128);

    // 7. Start scheduling — this never returns
    uart_puts(UART0, "Starting scheduler\n");
    scheduler_start();
}
```

### Part 2: Memory Management

The RP2040 has 264 KB of SRAM with no MMU (Memory Management Unit). You cannot use virtual memory or hardware-enforced protection. Instead, you implement a simple allocator that carves the available RAM into chunks.

```c
// mem.c — Simple block allocator

#define HEAP_SIZE 0x20000  // 128 KB reserved for heap
#define BLOCK_SIZE 64      // Minimum allocation unit

static uint8_t heap[HEAP_SIZE];
static uint8_t block_map[HEAP_SIZE / BLOCK_SIZE];  // 0=free, 1=used

void mem_init(void) {
    for (int i = 0; i < HEAP_SIZE / BLOCK_SIZE; i++) {
        block_map[i] = 0;
    }
}

void* kmalloc(size_t size) {
    int blocks_needed = (size + BLOCK_SIZE - 1) / BLOCK_SIZE;
    int consecutive = 0;
    int start = -1;

    for (int i = 0; i < HEAP_SIZE / BLOCK_SIZE; i++) {
        if (block_map[i] == 0) {
            if (consecutive == 0) start = i;
            consecutive++;
            if (consecutive == blocks_needed) {
                // Mark blocks as used
                for (int j = start; j < start + blocks_needed; j++) {
                    block_map[j] = 1;
                }
                return &heap[start * BLOCK_SIZE];
            }
        } else {
            consecutive = 0;
        }
    }
    return NULL;  // Out of memory
}

void kfree(void* ptr) {
    if (ptr == NULL) return;
    int index = ((uint8_t*)ptr - heap) / BLOCK_SIZE;
    // Free consecutive blocks (simplified — real impl tracks size)
    while (index < HEAP_SIZE / BLOCK_SIZE && block_map[index] == 1) {
        block_map[index] = 0;
        index++;
    }
}
```

> **Why no MMU matters:** Without hardware memory protection, a bug in one task can corrupt another task's memory — or even the kernel itself. This is the reality of embedded systems and the reason why memory-safe languages (Rust, for example) are increasingly used in firmware. When you later use Linux, the MMU provides hardware-enforced isolation that makes this class of bug impossible (between processes, at least).

### Part 3: Task Scheduling

This is the heart of the kernel. The scheduler decides which task runs and performs **context switching** — saving one task's registers and restoring another's.

#### The Task Control Block

Every task has a data structure that holds its state:

```c
// task.h

typedef enum {
    TASK_READY,
    TASK_RUNNING,
    TASK_BLOCKED,
    TASK_DEAD
} task_state_t;

typedef struct {
    uint32_t *stack_ptr;       // Saved stack pointer
    uint32_t stack[MAX_STACK]; // Stack memory
    task_state_t state;
    char name[16];
    uint8_t priority;
    uint32_t ticks;            // Total ticks executed
} task_t;

#define MAX_TASKS 8
extern task_t tasks[MAX_TASKS];
extern int current_task;
```

#### Cooperative Scheduling

Start with cooperative scheduling — tasks voluntarily yield control:

```c
// scheduler.c — Cooperative scheduler

void yield(void) {
    // Find the next ready task (round-robin)
    int next = current_task;
    do {
        next = (next + 1) % MAX_TASKS;
    } while (tasks[next].state != TASK_READY && next != current_task);

    if (next != current_task) {
        int prev = current_task;
        current_task = next;
        tasks[prev].state = TASK_READY;
        tasks[next].state = TASK_RUNNING;
        context_switch(&tasks[prev].stack_ptr, tasks[next].stack_ptr);
    }
}
```

#### Context Switching

The context switch is the one piece that *must* be in assembly. It saves all registers to the current task's stack and restores them from the next task's stack:

```armasm
@ context_switch.s — Save and restore task context
@ void context_switch(uint32_t **old_sp, uint32_t *new_sp)
@ R0 = pointer to old task's saved SP
@ R1 = new task's saved SP

.thumb_func
.global context_switch
context_switch:
    @ Save current context
    PUSH {R4-R7, LR}         @ Save callee-saved registers + return address
    MOV  R4, R8
    MOV  R5, R9
    MOV  R6, R10
    MOV  R7, R11
    PUSH {R4-R7}              @ Save high registers

    @ Save current stack pointer
    MOV  R2, SP
    STR  R2, [R0]             @ *old_sp = SP

    @ Restore new context
    MOV  SP, R1               @ SP = new_sp

    POP  {R4-R7}              @ Restore high registers
    MOV  R8, R4
    MOV  R9, R5
    MOV  R10, R6
    MOV  R11, R7
    POP  {R4-R7, PC}          @ Restore low registers and return
```

This is what a context switch looks like at the lowest level. When `POP {R4-R7, PC}` executes, the program counter loads from the new task's stack — and execution resumes exactly where that task last yielded. To the task, nothing happened. It called `yield()`, and then `yield()` returned. It does not know that another task ran in between.

#### Preemptive Scheduling

Cooperative scheduling only works if every task calls `yield()` regularly. A bug in one task (an infinite loop, for example) freezes the entire system. Preemptive scheduling fixes this using **timer interrupts**.

```c
// systick.c — Timer interrupt for preemptive scheduling

#define SYSTICK_CSR   (*(volatile uint32_t*)0xE000E010)
#define SYSTICK_RVR   (*(volatile uint32_t*)0xE000E014)

void systick_init(uint32_t ticks) {
    SYSTICK_RVR = ticks - 1;         // Reload value
    SYSTICK_CSR = 0x07;              // Enable, interrupt, use processor clock
}

// Called automatically by hardware every N ticks
void SysTick_Handler(void) {
    tasks[current_task].ticks++;
    schedule();  // Pick the next task and switch
}
```

The SysTick timer fires an interrupt at a fixed interval (typically every 1 ms). The interrupt handler saves the current task's state, picks the next task, and restores that task's state. The interrupted task has no idea it was interrupted — when it resumes, it continues from exactly where it was.

This is how every modern OS works. Linux, Windows, macOS — all use timer interrupts to ensure that no single process can monopolize the CPU.

### Part 4: Syscall Interface

Tasks should not directly access hardware — they should ask the kernel. A syscall (system call) is a controlled entry point into the kernel.

On the Cortex-M0+, you use the `SVC` (Supervisor Call) instruction to trigger a software interrupt:

```c
// syscall.c — Syscall handler

// Task calls: svc_call(SYSCALL_WRITE, buffer, length)
void SVC_Handler(void) {
    // The SVC number and arguments are on the task's stack
    uint32_t *stack = get_task_stack();
    uint32_t syscall_num = stack[0];
    uint32_t arg1 = stack[1];
    uint32_t arg2 = stack[2];

    switch (syscall_num) {
        case SYSCALL_YIELD:
            schedule();
            break;
        case SYSCALL_WRITE:
            uart_write((char*)arg1, arg2);
            break;
        case SYSCALL_ALLOC:
            stack[0] = (uint32_t)kmalloc(arg1);  // Return value in R0
            break;
        case SYSCALL_FREE:
            kfree((void*)arg1);
            break;
    }
}
```

Now tasks call `svc_call(SYSCALL_WRITE, "hello", 5)` instead of directly accessing UART registers. The kernel validates the request and performs it. This is the foundation of the security and stability model in every operating system.

## Putting It All Together

When your kernel is complete, it boots like this:

1. Processor starts at `kernel_main()` from the vector table
2. Kernel initializes clocks, GPIO, UART, memory, and the scheduler
3. Kernel creates tasks (blink LED, read serial input, idle)
4. Scheduler starts — picks the highest-priority ready task and runs it
5. SysTick fires every 1 ms, preempting the running task and scheduling the next
6. Tasks communicate through shared memory and syscalls

You now have a working (if minimal) operating system.

## Exercises

1. **Add a `sleep()` syscall.** Implement `task_sleep(uint32_t ms)` that blocks a task for the specified number of milliseconds. The scheduler skips blocked tasks; the SysTick handler counts down and unblocks them when their time expires.

2. **Priority scheduling.** Modify the scheduler to run higher-priority tasks first instead of using round-robin. What happens when a high-priority task never blocks? (This leads to starvation — and motivates Step 5.)

3. **Memory fragmentation.** Allocate and free blocks of different sizes in random order. After many cycles, try to allocate a large block. What happens? Draw a memory map showing the fragmentation.

4. **Debug output.** Add a `kprintf()` function that formats strings and outputs over UART. Use it to log every context switch with the task name and tick count.

5. **Crash handler.** Implement the `hardfault_handler` to dump the register state over UART when a task crashes. Deliberately cause a fault (null pointer dereference, stack overflow) and verify your handler catches it.

## Key Takeaways

- An OS kernel is not mysterious — it is a program that manages hardware, memory, scheduling, and provides services
- Context switching is saving and restoring registers — the processor does not know it is switching tasks
- Timer interrupts are what make preemptive multitasking possible — without them, a buggy task hangs the system
- Syscalls are the boundary between untrusted user code and the trusted kernel
- Without an MMU, memory protection is purely a software convention — any bug can corrupt anything
- Every concept here (scheduling, context switches, syscalls) scales directly to Linux, Windows, and every OS you will use

---

<p>
  <a href="/level-0/step-03/">&larr; Step 3: Peripherals — Build a Computer</a> · <a href="/level-0/step-05/">Step 5: FreeRTOS — A Real RTOS &rarr;</a>
</p>

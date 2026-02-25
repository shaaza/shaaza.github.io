---
layout: page
title: "Week 3: OS Project — MVP & Integration"
description: "Build the minimum viable version of each subsystem, then wire them together and discover what breaks."
permalink: /level-1/week-03/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / Week 3'
---

## Objective

This is the most important week of Level 1. You will do two things:

1. **Build an MVP of each subsystem** — the simplest version that does something real
2. **Integrate the subsystems** — wire them together and make the OS boot

The integration phase is where you learn the most. It is also where everything breaks. That is the point.

## Part 1: MVP of Each Subsystem

### What "MVP" Means Here

MVP does not mean "half-finished." It means **the simplest complete version** of each subsystem that fulfills its interface contract.

| Subsystem | MVP Scope | What It Does NOT Do Yet |
|---|---|---|
| **Scheduler** | Round-robin with fixed time quantum, 4 task slots | Priority scheduling, dynamic task creation |
| **Memory Manager** | Simple free-list allocator, first-fit | Paging, virtual memory, defragmentation |
| **Filesystem** | Flat file structure, in-memory only | Directories, persistence, permissions |
| **I/O Subsystem** | Serial output (printf-like), basic keyboard input | Interrupt-driven I/O, device drivers |
| **Shell** | Read a command, parse it, execute built-ins | Scripting, piping, background jobs |

### The Breadth-First Rule

> **You don't have something working until you've built all your subsystems.**

Do not spend the entire week perfecting the scheduler. Get a thin slice of each subsystem working. A minimal scheduler + a minimal memory manager + a minimal filesystem + minimal I/O + a minimal shell = **an OS that boots and responds to commands**. That is infinitely more valuable than a perfect scheduler with nothing else.

### Agent Strategy for MVPs

For each subsystem, give your agent a tightly scoped task:

```bash
# Memory Manager MVP
claude

> Implement the Memory Manager for my OS based on this header:
> [paste memory.h]
>
> Requirements for the MVP:
> - Simple free-list allocator using first-fit strategy
> - Heap is a statically allocated array of 1MB
> - mem_alloc returns aligned pointers (8-byte alignment)
> - mem_free adds blocks back to the free list
> - No coalescing of free blocks in the MVP (we will add that later)
>
> Write mem_init, mem_alloc, mem_free, and mem_stats.
> Include a test file that exercises all operations.
```

```bash
# Scheduler MVP
claude

> Implement the Scheduler for my OS based on this header:
> [paste scheduler.h]
>
> Requirements for the MVP:
> - Fixed array of 4 task slots
> - Round-robin scheduling with 10ms time quantum
> - Each task is a function pointer with a stack
> - Context switching via setjmp/longjmp (we will replace with
>   assembly later)
> - sched_create, sched_yield, sched_exit, sched_run
>
> Write the implementation and a test that creates 3 tasks
> that each print their task ID in a loop.
```

Repeat for filesystem, I/O, and shell. Each MVP should take 1–2 hours of agent-directed development.

## Part 2: Integration

### The Integration Plan

Integration is not "copy all files into one directory and compile." It requires a plan.

```bash
claude

> I have five OS subsystems, each developed independently:
> - Memory Manager (memory/)
> - Scheduler (scheduler/)
> - Filesystem (filesystem/)
> - I/O Subsystem (io/)
> - Shell (shell/)
>
> Each has its own source files and tests. I need to integrate
> them into a single OS that boots. Create:
> 1. A kernel/main.c that initializes subsystems in the correct
>    order and starts the shell
> 2. A Makefile that compiles everything together
> 3. A list of likely integration issues I should watch for
```

### The Boot Sequence

Your `main()` function orchestrates the boot:

```c
#include "memory.h"
#include "io.h"
#include "filesystem.h"
#include "scheduler.h"
#include "shell.h"

int main(void) {
    // 1. Initialize memory first — everyone needs it
    mem_init(heap_area, HEAP_SIZE);

    // 2. Initialize I/O — we need output for debugging
    io_init();
    io_print("Memory initialized.\n");

    // 3. Initialize filesystem
    fs_init();
    io_print("Filesystem initialized.\n");

    // 4. Initialize scheduler
    sched_init();
    io_print("Scheduler initialized.\n");

    // 5. Create shell as the first task
    sched_create(shell_main, "shell");
    io_print("Starting shell...\n");

    // 6. Start the scheduler (this never returns)
    sched_run();

    return 0;  // unreachable
}
```

### Integration Pain Points

These are the problems you **will** encounter. Knowing them ahead of time does not prevent them, but it helps you diagnose them faster.

| Problem | Symptom | Likely Cause |
|---|---|---|
| **Linker errors** | `undefined reference to ...` | Header files promise functions that were not implemented, or naming mismatches |
| **Initialization order** | Crash on boot | Subsystem A calls subsystem B before B is initialized |
| **Memory conflicts** | Corruption, mysterious crashes | Two subsystems allocate from the same memory region, or stack overflow |
| **Interface mismatches** | Wrong behavior, subtle bugs | Agents interpreted the interface contract differently |
| **Naming collisions** | Compiler errors | Two subsystems define functions or globals with the same name |

### Debugging Integration Issues

When integration fails (and it will), use your agent systematically:

```bash
claude

> I am integrating my OS subsystems and getting a crash during
> filesystem initialization. The memory manager and I/O subsystem
> initialize fine. Here is the crash output:
> [paste output]
>
> Here are the relevant source files:
> [paste kernel/main.c, filesystem/fs.c, memory/memory.c]
>
> What is likely going wrong?
```

**Key insight:** Integration bugs are often **interface misunderstandings**, not implementation bugs. The scheduler assumed `mem_alloc` returns zeroed memory; the memory manager does not zero it. The filesystem assumes block sizes of 512 bytes; the I/O subsystem uses 1024. Check the assumptions.

## API Contracts and Modularity

This week teaches the value of **explicit contracts**. When subsystems are built independently (by different agents, or by different people), the only thing that holds the system together is the interface specification.

### What Makes a Good API Contract

```c
/**
 * Allocate a block of memory.
 *
 * CONTRACT:
 * - Pre:  size > 0
 * - Post: Returns a pointer to at least `size` usable bytes,
 *         or NULL if insufficient memory
 * - The returned memory is NOT zeroed (caller must initialize)
 * - The returned pointer is aligned to 8 bytes
 * - The caller is responsible for calling mem_free exactly once
 *
 * THREAD SAFETY: Not thread-safe in MVP. Caller must hold
 * the memory lock if called from multiple tasks.
 */
void *mem_alloc(size_t size);
```

Notice what is specified:
- Preconditions (what must be true before calling)
- Postconditions (what is true after calling)
- What is *not* done (memory is not zeroed)
- Alignment guarantees
- Ownership semantics
- Thread safety

Every unspecified detail is a potential integration bug.

## Deliverables

By the end of Week 3:

- [ ] MVP implementation of all five subsystems
- [ ] Each subsystem passes its own unit tests independently
- [ ] Integrated OS that compiles and boots (even if buggy)
- [ ] Boot sequence in `kernel/main.c` initializing all subsystems
- [ ] Unified Makefile that builds the complete OS
- [ ] List of integration issues discovered and how they were resolved
- [ ] All work committed to Git on an `integration` branch

> **The measure of success this week is not "does everything work perfectly?" It is "does something boot?" Get to a booting system, no matter how minimal. Then iterate.**

---

<p style="text-align: center;">
  <a href="/level-1/week-02/">&larr; Week 2: Decompose & Parallelize</a> · <a href="/level-1/week-04/">Week 4: Iterate & Differentiate &rarr;</a>
</p>

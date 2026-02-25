---
layout: page
title: "Week 2: OS Project — Decompose & Parallelize"
description: "Break the OS into subsystems, define interfaces, and assign each to a separate agent session."
permalink: /level-1/week-02/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / Week 2'
---

## Objective

This week you become the architect. You will decompose your operating system into distinct subsystems, define the interfaces between them, analyze dependencies, and begin working with multiple agent sessions in parallel — each building a different piece of the system.

This is the week where the "vibe-coding" philosophy meets real system design. You are not coding. You are **thinking, decomposing, specifying, and orchestrating**.

## Step 1: Decompose the OS

An operating system is a collection of cooperating subsystems. Before you write (or rather, before your agents write) a single line of code, you need to decide what those subsystems are and what each one does.

### The Five Core Subsystems

| Subsystem | Responsibility | Key Data Structures |
|---|---|---|
| **Scheduler** | Manage processes/tasks, time-slice between them, handle priorities | Process control block (PCB), ready queue, timer |
| **Memory Manager** | Allocate and free memory, track what is in use, prevent corruption | Free list, page table, allocation bitmap |
| **Filesystem** | Store and retrieve files, manage directories, handle persistence | Inode table, directory entries, block allocation |
| **I/O Subsystem** | Handle input/output — keyboard, display, disk, serial | Device table, I/O buffers, interrupt handlers |
| **Shell** | User interface — accept commands, parse them, execute them | Command parser, built-in commands, environment |

### Decomposition Exercise

Open a Claude Code session and work through the decomposition:

```bash
claude

> I am building a minimal operating system in C as a learning
> project. I need to decompose it into subsystems. For each of
> the following subsystems, define:
> 1. Its exact responsibility (what it does and does NOT do)
> 2. The public API it exposes (function signatures)
> 3. What it depends on from other subsystems
> 4. What data structures it owns
>
> Subsystems: Scheduler, Memory Manager, Filesystem, I/O, Shell
```

The agent's response gives you a starting point. **Your job is to refine it.** Ask follow-up questions:

- "What happens if the scheduler needs memory? Who allocates it?"
- "How does the shell invoke a filesystem operation? Direct call or syscall?"
- "What is the boot sequence? Which subsystem initializes first?"

## Step 2: Define Interfaces

Interfaces are the contracts between subsystems. They are the most important design artifact you will produce this week.

### Interface Definition Format

For each subsystem, create a header file that defines its public API:

```bash
claude

> Create a C header file for the Memory Manager subsystem of my
> OS. It should expose these operations:
> - Initialize the memory system with a given heap size
> - Allocate a block of memory (like malloc)
> - Free a block of memory (like free)
> - Report memory usage statistics
>
> Use clear naming conventions. Add comments explaining each
> function's contract: what it expects, what it returns, what
> can go wrong.
```

Example output — `include/memory.h`:

```c
#ifndef OS_MEMORY_H
#define OS_MEMORY_H

#include <stddef.h>
#include <stdint.h>

/**
 * Initialize the memory management subsystem.
 * Must be called before any other memory operations.
 *
 * @param heap_start  Pointer to the beginning of the heap
 * @param heap_size   Size of the heap in bytes
 * @return 0 on success, -1 on failure
 */
int mem_init(void *heap_start, size_t heap_size);

/**
 * Allocate a contiguous block of memory.
 *
 * @param size  Number of bytes to allocate
 * @return Pointer to allocated memory, or NULL if insufficient
 */
void *mem_alloc(size_t size);

/**
 * Free a previously allocated block of memory.
 *
 * @param ptr  Pointer returned by mem_alloc. NULL is a no-op.
 */
void mem_free(void *ptr);

/**
 * Report current memory usage.
 *
 * @param total_out   Receives total heap size
 * @param used_out    Receives bytes currently allocated
 * @param free_out    Receives bytes available
 */
void mem_stats(size_t *total_out, size_t *used_out, size_t *free_out);

#endif
```

**Do this for all five subsystems.** The header files are your contract. Every agent session will code against these interfaces.

## Step 3: Dependency Analysis

Before you can parallelize, you need to understand what depends on what.

### Dependency Map

```
Boot Sequence:
  1. Memory Manager (first — everyone needs memory)
  2. I/O Subsystem (second — need output for debugging)
  3. Filesystem (depends on memory + I/O)
  4. Scheduler (depends on memory)
  5. Shell (depends on everything above)

Runtime Dependencies:
  Scheduler  --> Memory Manager (allocate PCBs)
  Filesystem --> Memory Manager (allocate buffers)
  Filesystem --> I/O Subsystem (read/write to disk)
  Shell      --> Scheduler (launch processes)
  Shell      --> Filesystem (file commands)
  Shell      --> I/O Subsystem (read input, write output)
```

Ask your agent to help analyze dependencies:

```bash
claude

> Given these five OS subsystems and their interfaces, create a
> dependency graph. Identify:
> 1. Which subsystems can be built completely independently?
> 2. Which have circular dependencies that need breaking?
> 3. What is the correct initialization order?
> 4. Which can be developed in parallel without blocking?
```

## Step 4: Parallelize with Multiple Agent Sessions

Now the interesting part. You will open **multiple terminal windows**, each running a separate Claude Code session, each working on a different subsystem.

### The Orchestration Pattern

```
Terminal 1 (Agent A): Scheduler
Terminal 2 (Agent B): Memory Manager
Terminal 3 (Agent C): Filesystem
Terminal 4 (Agent D): I/O Subsystem
Terminal 5 (Agent E): Shell
```

### Setup for Each Session

Each agent session needs context. Start each one with:

```bash
# Terminal 1 — Scheduler
claude

> You are building the Scheduler subsystem for a minimal OS.
> Here is the interface you must implement: [paste scheduler.h]
> Here are the interfaces of subsystems you depend on: [paste memory.h]
> Build the implementation in scheduler/scheduler.c.
> Start with the simplest possible version — round-robin with
> fixed time quantum.
```

### Coordination Rules

| Rule | Why |
|---|---|
| **Every agent gets the same header files** | They all code against the same contracts |
| **No agent modifies another subsystem's code** | Clean boundaries prevent conflicts |
| **You review before committing** | The architect catches interface mismatches |
| **Commit each subsystem on its own branch** | Isolation until integration in Week 3 |
| **Keep a coordination log** | Track what each agent has built and any issues |

### Practical Tips

- **Do not try to run all five agents simultaneously** on your first attempt. Start with two (Memory Manager and Scheduler), get comfortable, then add more.
- **Copy-paste interface files** to each session. Agents do not share context unless you provide it.
- **When an agent has a question about another subsystem**, answer it yourself based on the interface definition. Do not let agents invent assumptions.

## System Decomposition Thinking

This week exercises a specific skill: **seeing a complex system as a collection of simpler parts with defined interactions**. This skill transfers to every engineering project, not just OS development.

Questions to sit with:

> Where do you draw the boundary between "scheduler" and "memory manager"? There is no single right answer — but there are wrong answers that create circular dependencies and unmaintainable code.

> How thin can each interface be? The thinner the interface, the more independently each subsystem can evolve. But too thin means too many round trips between subsystems.

> What assumptions is each subsystem making about the others? Write them down. Unwritten assumptions become integration bugs in Week 3.

## Deliverables

By the end of Week 2:

- [ ] Full decomposition document: five subsystems with responsibilities defined
- [ ] Header files (`.h`) for all five subsystem interfaces
- [ ] Dependency graph with initialization order
- [ ] At least two subsystems under active development in parallel agent sessions
- [ ] Coordination log tracking what each agent is building
- [ ] All work committed to Git on subsystem-specific branches

---

<p style="text-align: center;">
  <a href="/level-1/week-01/">&larr; Week 1: Environment & Agent Tooling</a> · <a href="/level-1/week-03/">Week 3: MVP & Integration &rarr;</a>
</p>

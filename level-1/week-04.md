---
layout: page
title: "Week 4: OS Project — Iterate & Differentiate"
description: "Deepen each subsystem, add a unique feature, and make the OS yours."
permalink: /level-1/week-04/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / Week 4'
---

## Objective

Your OS boots. It is minimal, probably buggy, and definitely not impressive yet. This week, two things happen:

1. **Iterate** — deepen each subsystem, fix bugs, improve robustness
2. **Differentiate** — add something unique that makes your OS *yours*

The skeleton works. Now you give it substance and personality.

## Part 1: Iterate — Deepen Each Subsystem

### From MVP to Functional

Each subsystem's MVP was the simplest thing that could work. Now you push each one further.

| Subsystem | MVP (Week 3) | Iteration Target (Week 4) |
|---|---|---|
| **Scheduler** | Round-robin, 4 fixed slots | Priority levels, dynamic task creation/destruction, proper context switching |
| **Memory Manager** | First-fit free list | Free block coalescing, allocation statistics, memory leak detection |
| **Filesystem** | Flat in-memory files | Directories, file metadata (size, timestamps), basic persistence |
| **I/O Subsystem** | Serial print, basic input | Buffered I/O, formatted output, input line editing |
| **Shell** | Parse and execute built-ins | Command history, tab completion, piping between commands |

### Iteration Strategy

The key to iterating with agents is **full-system context**. Unlike Week 2, where each agent worked in isolation, now every change must consider the whole system.

```bash
claude

> I have a working (minimal) OS. I want to improve the scheduler.
> Currently it is round-robin with 4 fixed task slots.
>
> Here is the current code:
> [paste scheduler.c, scheduler.h]
>
> Here is how it is called from the kernel:
> [paste relevant parts of main.c]
>
> I want to add:
> 1. Priority levels (high, medium, low)
> 2. Dynamic task creation (not fixed 4 slots)
> 3. A proper task termination mechanism
>
> Modify the implementation. Make sure the existing shell task
> still works after these changes.
```

**Critical rule:** After every change to a subsystem, **rebuild and boot the entire OS**. Do not make three changes and then try to boot. Make one change, verify the system still boots, commit, then make the next change.

```bash
# The iteration loop
make clean && make         # Build
./os                       # Boot and test
git add -A && git commit   # Commit if it works
# Repeat
```

### Common Iteration Pitfalls

| Pitfall | What Happens | How to Avoid |
|---|---|---|
| **Changing interfaces** | Other subsystems break | Add new functions rather than modifying existing signatures |
| **Over-engineering** | Spending 4 hours on perfect memory coalescing | Time-box. 2 hours per subsystem max. |
| **Losing the boot** | A change breaks something and you cannot figure out what | Commit after every working change. Use `git bisect` if needed. |
| **Agent context loss** | Agent forgets the state of the system | Re-provide interface files and key code at the start of each session |

## Part 2: Differentiate — Make It Yours

This is the creative part. Every student's OS should have something unique — a quirk, a novel feature, an unexpected capability. This is not optional polish; it is a design exercise.

> **Personality in engineering is underrated.** The best systems have a point of view. Your OS should have one too.

### Differentiation Ideas

| Category | Example Features |
|---|---|
| **Unusual shell** | A shell that only accepts commands as haikus. A shell with a built-in calculator. A shell that speaks back to you in ASCII art. |
| **Creative filesystem** | A filesystem where files have moods. A versioned filesystem that keeps every edit. A filesystem organized by tags instead of directories. |
| **Novel scheduler** | A scheduler that assigns priority based on how recently a task was starved. A lottery scheduler. A scheduler with an "emergency" mode. |
| **I/O personality** | Boot messages that tell a story. Error messages that are unusually helpful (or unusually sarcastic). A built-in music player using terminal beeps. |
| **System features** | A built-in text editor. A tiny game. A system monitor that shows all subsystem stats in real-time. A scripting language for the shell. |

### Implementing Your Feature

Pick one idea (or invent your own) and build it:

```bash
claude

> My OS has a working shell. I want to add a unique feature:
> when the user types a command wrong, instead of just saying
> "command not found", the shell should suggest the closest
> matching valid command using Levenshtein distance and display
> it with a helpful message.
>
> Here is my current shell code:
> [paste shell.c]
>
> Here are the valid commands:
> [paste command list]
>
> Implement this feature.
```

The differentiation feature is your chance to practice **product thinking** — not just "what can I build?" but "what would be delightful, surprising, or useful?"

### Creative Engineering

Differentiation is also a test of your agent-direction skills. Creative features require more precise specification because there is no standard pattern for the agent to follow.

Compare these prompts:

**Vague (bad):**
> Make the shell more interesting.

**Specific (good):**
> Add a command called `fortune` to my shell that, when invoked, prints a random programming quote from a hardcoded array of 20 quotes. The quotes should be displayed in a box made of ASCII characters. Include the author's name right-aligned below the quote.

The more creative the feature, the more precise your specification needs to be. This is a general principle of working with AI agents: **creativity requires specificity**.

## Full-System Context

Week 4 is where you transition from thinking about subsystems in isolation to thinking about the **system as a whole**. Every change to the scheduler affects the shell. Every change to the memory manager affects the filesystem. You cannot iterate on one piece without considering its impact on everything else.

This is what makes OS development (and system design in general) hard — and why it is such a good exercise for learning agent-directed development. You are forced to:

1. Maintain a mental model of the whole system
2. Communicate that model to the agent
3. Verify that changes in one area do not break another
4. Think about interactions and side effects

No agent can hold this full-system context as well as you can. **This is your value as the architect.**

## Deliverables

By the end of Week 4:

- [ ] Each subsystem improved beyond MVP level
- [ ] At least one unique differentiation feature implemented
- [ ] OS boots reliably and demonstrates improved functionality
- [ ] Each iteration committed separately with descriptive messages
- [ ] Brief write-up: What is your OS's "personality"? What makes it different?
- [ ] All work committed to Git, merged to main

---

<p style="text-align: center;">
  <a href="/level-1/week-03/">&larr; Week 3: MVP & Integration</a> · <a href="/level-1/week-05/">Week 5: User Testing & Feedback &rarr;</a>
</p>

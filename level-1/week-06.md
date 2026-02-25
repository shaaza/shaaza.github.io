---
layout: page
title: "Week 6: Polish & Ship"
description: "Polish based on feedback, make it fun, make it yours, and ship it."
permalink: /level-1/week-06/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / Week 6'
---

## Objective

This is the final week of Level 1. You will take the feedback from Week 5, polish your OS, and **ship it**. Shipping means: someone else can download it, build it, and run it without your help.

> "Personality in engineering is underrated."

Your OS should not just work. It should have character. It should feel like *yours*. The best technical projects are the ones where the builder's taste and perspective are visible in the output.

## Part 1: Fix What Matters

Go back to your prioritized feedback from Week 5. You have limited time. Focus on the items that matter most.

### The Fix List

Work through your priorities:

```bash
# Priority 1: Crashes and freezes
claude

> My OS crashes when the user types a command longer than 128
> characters. Here is the shell input handling code:
> [paste code]
>
> Fix the buffer overflow. Make sure the shell can handle
> arbitrarily long input (or at least truncate gracefully
> with a clear message).
```

```bash
# Priority 2: First 30 seconds experience
claude

> When my OS boots, the user sees a blank prompt with no
> instructions. Based on user testing, people don't know
> what to do. Improve the boot experience:
>
> 1. Show a brief welcome message with the OS name
> 2. List 3-4 commands to get started
> 3. Make it clear that 'help' shows all commands
> 4. Keep it short — no more than 5 lines
>
> Here is my current boot code:
> [paste code]
```

```bash
# Priority 3: Error messages
claude

> Here are the 3 most common errors users hit in my OS:
> 1. Unknown command → currently shows "error: unknown command"
> 2. File not found → currently shows "error: file not found"
> 3. Invalid arguments → currently shows "error: bad args"
>
> Rewrite these error messages to be:
> - Specific about what went wrong
> - Suggestive of what to do instead
> - Consistent in format
>
> Here is where these messages are generated:
> [paste code]
```

### The Polish Pass

After fixing critical issues, do a polish pass. These are small changes that add up to a significantly better experience.

| Area | Polish Task |
|---|---|
| **Boot sequence** | Add a clean startup message, maybe with ASCII art |
| **Help system** | Make sure every command has a help entry with usage examples |
| **Error messages** | Every error should say what went wrong AND what to do about it |
| **Shell prompt** | Something more interesting than `> ` — show the current directory, or the OS name |
| **Exit experience** | A clean shutdown message instead of just terminating |
| **Command names** | Consistent naming — do not mix `list-files` and `rm` styles |

## Part 2: Make It Yours

This is your last chance to add personality. What makes your OS memorable?

### The "Show Don't Tell" Principle

Do not just add features. Add **character**. Some ideas:

```bash
claude

> Add an Easter egg to my OS: when the user types "hello", the
> OS responds with a randomly selected greeting in a different
> programming language's syntax. For example:
> - puts "Hello, World!" (Ruby)
> - System.out.println("Hello, World!") (Java)
> - fmt.Println("Hello, World!") (Go)
> - console.log("Hello, World!") (JavaScript)
>
> Include at least 10 languages.
```

```bash
claude

> Add a 'stats' command to my shell that displays a real-time
> dashboard of OS internals:
> - Memory: used/total with a visual bar
> - Tasks: count and names of running tasks
> - Filesystem: number of files and total size
> - Uptime: how long the OS has been running
>
> Format it cleanly with box-drawing characters.
```

The point is not that these features are technically impressive. The point is that they show **taste and care**. They transform your OS from a homework assignment into something with a personality.

## Part 3: Ship It

Shipping means making your project accessible to others. This is a skill in itself.

### The README

Every shipped project needs a README. Ask your agent to draft one:

```bash
claude

> Write a README.md for my OS project. Include:
> 1. A one-line description
> 2. A screenshot or ASCII rendering of the OS in action
> 3. How to build it (exact commands)
> 4. How to run it
> 5. Available commands (brief list)
> 6. The unique feature and why I built it
> 7. Known limitations
> 8. What I learned building this
>
> Keep it concise. No one reads long READMEs.
```

### The Build Process

Make sure someone else can build and run your OS:

```bash
# Test the build from scratch
cd /tmp
git clone <your-repo>
cd <your-os>
make
./os

# If this doesn't work, fix it until it does
```

The build should:
- Work with a single `make` command
- Not require obscure dependencies (or clearly document them)
- Produce clear error messages if something is missing
- Work on at least one common platform (Linux, macOS)

### Ship Checklist

| Item | Status |
|---|---|
| OS builds from clean checkout with `make` | |
| OS boots and presents a working shell | |
| `help` command lists all available commands | |
| All commands documented in the README | |
| Known bugs listed honestly | |
| Unique feature works and is mentioned in README | |
| Code is committed and pushed to a public repository | |
| Someone else has successfully built and run it | |

## What Shipping Teaches

Shipping is the hardest part of any project. It requires:

- **Ruthless prioritization** — you cannot fix everything, so you fix what matters
- **Honesty about limitations** — listing known bugs is a sign of maturity, not weakness
- **Care for the user** — the README, the build process, the error messages all show whether you care about the person on the other end
- **Letting go** — at some point, it is good enough. Ship it. You can always improve it later.

> "Real artists ship." — Steve Jobs

## Reflection

At the end of Level 1, step back and consider what you have learned:

1. **About AI agents:** When are they most useful? When do they struggle? What makes a good prompt vs. a bad one?
2. **About system design:** How do you decompose a complex system? What makes a good interface? Why does integration always hurt?
3. **About yourself:** What is your natural role — architect, debugger, polish person? Where are you strong? Where do you need growth?
4. **About shipping:** What does it take to go from "it works on my machine" to "anyone can use this"?

These lessons carry forward into Level 2, where you will build agentic systems — AI agents that work *for* you, not just *with* you.

## Deliverables

By the end of Week 6:

- [ ] Critical bugs from user testing fixed
- [ ] Polish pass completed (boot experience, error messages, help system)
- [ ] Unique feature refined and working
- [ ] README written and accurate
- [ ] Build process tested from clean checkout
- [ ] Code pushed to a public repository
- [ ] Final reflection written (1 page): What did you learn about building with AI agents?

---

<p style="text-align: center;">
  <a href="/level-1/week-05/">&larr; Week 5: User Testing & Feedback</a> · <a href="/level-2/">Level 2: Agentic Engineering &rarr;</a>
</p>

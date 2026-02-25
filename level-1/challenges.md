---
layout: page
title: "CLI AI Challenges"
description: "Practical challenges to build fluency with AI coding agents before the OS capstone."
permalink: /level-1/challenges/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / CLI AI Challenges'
---

## Why Challenges First?

Before you attempt to build an operating system with AI agents, you need **fluency**. Not fluency in a programming language — fluency in *directing an AI agent*. You need to develop intuition for:

- How to phrase requests that get good results
- When the agent is confident vs. when it is guessing
- How to verify output you did not write
- What kinds of tasks agents excel at, and where they fall apart

The only way to build this intuition is through **repetition across varied tasks**. These challenges are designed to give you that repetition.

## The Rules

1. **100% agent-generated code.** You do not type code. You direct the agent.
2. **Pick 2–3 challenges** from the list below. Do not try to do all of them.
3. **Document your process.** Keep a log of your prompts, the agent's responses, and where things went wrong.
4. **Time-box each challenge** to 3–5 hours. The goal is breadth of experience, not completion.

## Challenge Menu

Pick 2–3 from the following:

| # | Challenge | Difficulty | What You Will Learn |
|---|---|---|---|
| 1 | **Learn a new language from scratch** | Medium | How agents teach, where they hallucinate syntax |
| 2 | **Debug a deep, hard bug** | Hard | Agent reasoning limits, verification strategies |
| 3 | **Set up a Kubernetes cluster** | Medium | Infrastructure automation, config generation |
| 4 | **Full deployment on unfamiliar OS** | Medium-Hard | Agent knowledge boundaries, system administration |

---

### Challenge 1: Learn a New Language from Scratch

**Pick a language you have never used:** Rust, Elixir, OCaml, Haskell, Zig, or another language that is genuinely unfamiliar.

**The task:** Using only your AI agent, go from zero to a working program — something non-trivial like a simple HTTP server, a CLI tool, or a data structure library.

```bash
# Example session start
claude

> I want to learn Rust from scratch. I have experience with Python
> and JavaScript but have never written Rust. Walk me through
> setting up a project and building a CLI tool that counts word
> frequencies in a text file.
```

**What to pay attention to:**

- Does the agent explain concepts correctly, or does it gloss over important details?
- When you ask follow-up questions, does it stay consistent?
- Does the generated code actually compile? What percentage of the time?
- How does it handle language-specific idioms (e.g., Rust's borrow checker, Elixir's pattern matching)?

**What you will learn:** Agents are remarkably good at teaching programming language basics. They are less reliable on advanced idioms, edge cases, and the *why* behind language design decisions. You will develop a sense for when to trust the agent's explanation and when to verify against official documentation.

---

### Challenge 2: Debug a Deep, Hard Bug

**Find or create a program with a subtle bug:** a concurrency race condition, a memory leak, a deadlock, or an off-by-one error buried in complex logic.

**The task:** Give the buggy code to your agent and ask it to find and fix the bug. Do not tell it what the bug is — just describe the symptoms.

```bash
# Example: a program that deadlocks intermittently
claude

> I have a multi-threaded C program that hangs about 30% of the
> time when processing more than 1000 items. It works fine with
> small inputs. Here is the code: [paste code]
>
> Find the bug and explain what is happening.
```

**What to pay attention to:**

- Does the agent identify the actual root cause, or does it suggest surface-level fixes?
- How many rounds of conversation does it take to converge on the real issue?
- Does it suggest the *right* fix, or a fix that masks the problem?
- Can it explain the bug clearly enough that you understand it?

**What you will learn:** Agents can spot many common bug patterns instantly. But for subtle concurrency bugs, memory corruption, or issues that depend on timing, they often suggest plausible-sounding fixes that do not address the root cause. You will learn to **ask the agent to explain its reasoning** before accepting a fix.

---

### Challenge 3: Set Up a Kubernetes Cluster

**The task:** Using only your agent, set up a working Kubernetes cluster — either locally with minikube/kind or on a cloud provider. Deploy a simple application with multiple services.

```bash
claude

> Help me set up a local Kubernetes cluster using kind. I want to
> deploy a simple web application with a frontend, a backend API,
> and a PostgreSQL database. Generate all the YAML manifests and
> walk me through the deployment.
```

**What to pay attention to:**

- Does the generated YAML actually work, or are there version mismatches and typos?
- How does the agent handle debugging when `kubectl` reports errors?
- Does it understand networking, services, and ingress correctly?
- Can it troubleshoot a pod that will not start?

**What you will learn:** Infrastructure configuration is a strong suit for AI agents — they have seen thousands of Kubernetes manifests. But they sometimes mix API versions, generate deprecated fields, or produce configs that are syntactically valid but semantically wrong. You will learn to **always apply and test**, never trust YAML on sight.

---

### Challenge 4: Full Deployment on an Unfamiliar OS

**Pick an operating system you have never administered:** FreeBSD, NixOS, Alpine Linux, OpenBSD, or another system outside your comfort zone.

**The task:** Using only your agent, install the OS (in a VM is fine), configure it, and deploy a working web application on it.

```bash
claude

> I have a fresh FreeBSD 14 installation in a VM. Help me:
> 1. Set up the package manager
> 2. Install nginx, PostgreSQL, and Python 3
> 3. Configure a basic Flask application behind nginx
> 4. Set it up to start on boot
```

**What to pay attention to:**

- Does the agent know the correct package manager, file paths, and service management for this OS?
- Where does it fall back on Linux assumptions that do not apply?
- How does it handle OS-specific configuration (e.g., FreeBSD's rc.conf, NixOS's declarative config)?
- When commands fail, can the agent diagnose OS-specific issues?

**What you will learn:** Agents are trained heavily on Linux (particularly Ubuntu/Debian). Their knowledge of other operating systems is thinner and more error-prone. You will learn to **detect when the agent is pattern-matching from a different OS** rather than giving you correct information for the one you are using.

---

## What You Learn From All Challenges

Across all challenges, you are building the same meta-skills:

| Skill | Description |
|---|---|
| **Prompt specificity** | Vague prompts get vague results. You learn to be precise. |
| **Verification discipline** | You learn to never trust output without testing it. |
| **Hallucination detection** | You develop a feel for when the agent is confident vs. confabulating. |
| **Iterative refinement** | You learn to steer through multi-turn conversations, correcting course. |
| **Knowing when to stop** | Sometimes the agent cannot solve the problem. You learn to recognize that. |

> "The challenges are not about the specific task. They are about building a working relationship with your AI agent — learning its strengths, its limits, and how to get the best out of it."

## Deliverables

For each challenge you attempt:

1. **A brief write-up** (1–2 pages) describing what you did, what worked, and what did not
2. **Your prompt log** — the actual prompts you used and key agent responses
3. **An honest assessment** of the agent's performance: where it excelled, where it struggled, where it was wrong

These deliverables feed directly into your approach for the OS capstone. The better you understand your tools, the better architect you will be.

---

<p style="text-align: center;">
  <a href="/level-1/">&larr; Level 1 Overview</a> · <a href="/level-1/week-01/">Week 1: Environment & Agent Tooling &rarr;</a>
</p>

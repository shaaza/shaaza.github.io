---
layout: page
title: "Level 0.5 — Programming Languages & Software Engineering"
description: "Languages and disciplines to tame the machine. Before AI agents write code, you need to understand the design space."
permalink: /level-05/
breadcrumb: '<a href="/">Schedule</a> / Level 0.5'
---

## Why This Module Exists

Level 0 taught you what the machine **is** — registers, memory, interrupts, the raw hardware that executes every instruction. But knowing what the machine is and knowing how to *write correct, maintainable software for it* are two very different things.

This module teaches the **languages and disciplines** that tame the machine.

Before AI agents write code for you (Level 1), you need to understand:

- **The programming language design space** — what trade-offs different languages make and why
- **Software maintainability principles** — modularity, testing, clear interfaces
- **Formal methods for correctness** — how to move beyond "it seems to work" to "it provably works"

> "You can't effectively direct an AI agent to build software if you don't understand what good software looks like."

---

## The Direct Connection to Level 1

Everything in this module directly feeds into Level 1, where you will use AI agents to build real software. Here is why each section matters:

| Section | What You Learn | Why It Matters for AI Agents |
|---------|---------------|------------------------------|
| [Languages](/level-05/languages/) | How different paradigms express computation | You need to evaluate and guide agent-generated code across languages |
| [Haskell Deep Dive](/level-05/haskell/) | Pure functions, composition, type safety | FP principles map directly to structuring work for agents |
| [Modularity](/level-05/modularity/) | Independent, interchangeable modules | Modular code fits in context windows and enables parallel agent work |
| [Testing](/level-05/testing/) | Verification at every level | AI-generated code demands even more rigorous testing |
| [Formal Methods](/level-05/formal-methods/) | Mathematical proof of correctness | Safety-critical AI-generated code needs formal guarantees |
| [Synthesis](/level-05/synthesis/) | How it all connects | The complete framework for AI-era software engineering |

---

## The Arc of This Module

The sections build on each other in a deliberate sequence:

1. **Languages** — Understand the *design space* of programming languages. See how the same problem looks in C, Python, Java, Haskell, Rust, and JavaScript. Learn that syntax is noise; paradigm is signal.

2. **Haskell** — Go deep into the paradigm that is most different from what you already know. Pure functions, monads, type classes, and lazy evaluation will rewire how you think about computation.

3. **Modularity** — Zoom out from individual functions to *system structure*. Learn the principle that has governed good software from Unix pipes in 1973 to Netflix microservices today.

4. **Testing** — Learn to verify that code does what it should. Unit tests, integration tests, property-based tests, mutation tests — and why AI-generated code makes testing *more* important, not less.

5. **Formal Methods** — Go beyond testing to *proof*. CSP, Z notation, TLA+ — tools that let you mathematically demonstrate your system is correct.

6. **Synthesis** — Tie it all together. See how languages, modularity, testing, and formal methods form a chain that makes AI-agent-driven development possible.

---

## What You Will Be Able to Do

By the end of Level 0.5, you will be able to:

- Read and reason about code in multiple paradigms (imperative, functional, OOP, systems)
- Design modular systems with clean interfaces
- Write comprehensive test suites including property-based and mutation tests
- Apply formal specification techniques to critical components
- Articulate *why* these principles matter even more in the AI era
- Confidently move into Level 1, where AI agents become your primary development tool

---

## Sections

1. [Languages: Shared Constructs, Different Paradigms](/level-05/languages/)
2. [Deep Dive: Functional Programming with Haskell](/level-05/haskell/)
3. [Modularity: From Unix Pipes to AI Agents](/level-05/modularity/)
4. [Testing: Unit, Integration, Property-Based, Mutation](/level-05/testing/)
5. [Formal Methods: CSP, Z, TLA+ & Industry Case Studies](/level-05/formal-methods/)
6. [Synthesis: How These Principles Enable AI-Agent Development](/level-05/synthesis/)

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a> · Next: <a href="/level-05/languages/">Languages: Shared Constructs, Different Paradigms →</a>
</p>

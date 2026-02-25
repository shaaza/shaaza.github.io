---
layout: page
title: "About This Course"
description: "Philosophy, principles, and the reasoning behind the design."
permalink: /about/
breadcrumb: '<a href="/">Schedule</a> / About'
---

## The Core Idea

**Software Engineering in the AI Era** is a project-driven course that starts at the silicon and ends at intelligence. It is built on one premise: the engineers who will matter most in the coming decade are not the ones who can prompt an AI — everyone will be able to do that — but the ones who understand computation deeply enough to direct AI agents effectively, evaluate their output critically, and build systems that are correct, maintainable, and ambitious.

The course is hands-on first. You build things every week. The things you build get progressively more complex, and the tools you use to build them get progressively more powerful. By the end, you are training your own language models on custom data and deploying them to edge hardware.

---

## The Full Arc

The course moves through five levels, each one a step up in abstraction:

| Level | Title | What You Do | Abstraction Layer |
|-------|-------|-------------|-------------------|
| **0** | Foundations | Write assembly, wire hardware, build a kernel | **Hardware** — the machine itself |
| **0.5** | Languages & SE | Learn paradigms, modularity, testing, formal methods | **Languages** — how we express computation |
| **1** | Vibe-Coding | Build a real OS project using AI agents | **Use** — AI as a development tool |
| **2** | Agentic Engineering | Design and deploy multi-step agent workflows on AWS | **Orchestrate** — AI as infrastructure |
| **3** | ML Foundations | Train neural nets from scratch, fine-tune, deploy | **Build** — AI from the inside |

The progression is deliberate: **Hardware → Languages → Use → Orchestrate → Build.** Each level depends on the ones before it. You cannot orchestrate what you do not understand. You cannot build what you have not used.

---

## Pedagogical Principles

Eight principles guide the design of this course:

1. **Build first, explain after.** You write code and wire circuits before you read theory. Understanding follows from doing, not the reverse.

2. **Ambitious projects over toy exercises.** Every project is something you would actually want to use or show to someone. No contrived homework problems.

3. **Breadth before depth.** See the entire landscape first. Go deep later, when you know what is worth going deep on.

4. **The full stack is one stack.** A keypress that becomes a pixel on a screen traverses every layer of computing. This course traces that path.

5. **AI agents are tools, not magic.** You will use AI agents extensively, but you will also understand what they can and cannot do, and why they fail when they fail.

6. **Documentation is work.** Writing down what you did, what broke, and what you learned is not busywork — it is the primary artifact of your education.

7. **Failure is data.** A project that fails with excellent documentation teaches more than a project that succeeds without explanation. Dead ends are reported, not hidden.

8. **Progressive autonomy.** Early levels provide detailed guidance. Later levels give you a goal and get out of the way. By Level 3, you are making your own architectural decisions.

---

## Progressive Abstraction

The course is structured as a ladder of abstraction. Each level builds on the previous one and reveals the next:

```
Level 3: Build      — Train and deploy neural networks
Level 2: Orchestrate — Design agent workflows and pipelines
Level 1: Use        — Build with AI agents as tools
Level 0.5: Languages — Express computation, verify correctness
Level 0: Hardware   — The machine itself, from silicon up
```

You start at the bottom. You understand what a computer actually does before you ask an AI to write code for one. You understand what good software looks like before you let an agent generate it. You use agents as tools before you build the infrastructure that runs them. You orchestrate agents before you understand how they work internally.

By the end, you can trace a path from a transistor switching state to a language model generating text — and you have built working systems at every level along the way.

---

## Open Questions

This course does not pretend to have all the answers. Several questions remain genuinely open, and we will revisit them throughout:

- **How do you evaluate AI-generated code?** Testing helps, but current test suites cannot cover all failure modes. What does "good enough" mean when the code was written by a model?

- **What skills remain essential?** If AI agents can generate boilerplate, refactor code, and write tests, what should human engineers spend their time on? The answer is not obvious and is changing fast.

- **How do you teach debugging when the code is not yours?** Traditional debugging assumes you wrote the code and understand its intent. AI-generated code breaks that assumption. New debugging intuitions are needed.

- **Where is the boundary between using AI and understanding AI?** This course teaches both, but the balance will shift as the tools improve. Level 1 today may look very different in two years.

- **Can formal methods scale to AI-generated systems?** Formal verification works on small, well-specified components. Can it be applied to the output of probabilistic code generators? We do not know yet.

These are not rhetorical questions. They are active research problems. Part of this course is developing your own answers.

---

## Why It Looks Like This

This site is intentionally plain. No JavaScript frameworks, no CSS libraries, no build pipeline beyond Jekyll. The design is inspired by the early web — simple HTML, readable typography, fast loading, no distractions.

This is a deliberate pedagogical choice. The course teaches you to build complex systems; the course website demonstrates that complexity is not always necessary. A static site generated from Markdown files, served from GitHub Pages, is sufficient for everything we need. It loads instantly on any device, works without JavaScript, is fully accessible to screen readers, and will still be readable in ten years.

The tools should serve the content, not the other way around.

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a>
</p>

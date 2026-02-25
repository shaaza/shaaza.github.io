---
layout: page
title: "Resources"
description: "Tools, references, and reading for every level of the course."
permalink: /resources/
breadcrumb: '<a href="/">Schedule</a> / Resources'
---

## CLI AI Tooling — Level 1

The primary tool for this course is a CLI-based AI coding agent. You will live in the terminal. The following tools are all worth trying; we recommend starting with **Claude Code** and branching out.

| Tool | Type | Pricing | Context Window | Key Strengths |
|------|------|---------|----------------|---------------|
| **Claude Code** (recommended) | CLI agent | Anthropic API usage | 200K tokens | Deep reasoning, large context, agentic file editing, tool use |
| **Codex CLI** | CLI agent | OpenAI API usage | 128K tokens | Fast iteration, GPT-4 family models, Unix-native |
| **Cursor** | IDE (VS Code fork) | Free tier + $20/mo Pro | Model-dependent | Rich GUI, inline diffs, tab-completion, multi-model support |
| **Windsurf** | IDE (VS Code fork) | Free tier + $15/mo Pro | Model-dependent | Cascade agentic flow, deep codebase indexing |
| **Aider** | CLI agent (open source) | BYOK (any provider) | Model-dependent | Open source, Git-native, supports many LLM backends |

> **Advice:** Try your own codebases with each. Keep switching. No tool is permanent. The landscape changes monthly. Develop your own taste for what works on different problem shapes — some tools are better for greenfield code, others for refactoring, others for debugging. The goal is fluency with the *category*, not loyalty to a single product.

---

## AWS & Cloud — Level 2

Level 2 uses AWS as the deployment target for agentic workflows. Core services you will use:

- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/) — Managed foundation model access; agents, knowledge bases, guardrails
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/) — Serverless compute for agent actions
- [AWS Step Functions Developer Guide](https://docs.aws.amazon.com/step-functions/latest/dg/) — Orchestration of multi-step agent workflows
- [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/) — HTTP endpoints for your agent pipelines
- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/) — Infrastructure as code (TypeScript/Python)

You will receive AWS Academy access at the start of Level 2. All resources fit within the free tier or the provided lab credits.

---

## ML Foundations — Level 3

Level 3 follows Andrej Karpathy's neural network series. These are the primary resources:

### Video Lectures

- [Neural Networks: Zero to Hero](https://www.youtube.com/playlist?list=PLAqhIrjkxbuWI23v9cThsA9GvCAUhRvKZ) — Karpathy's full lecture series

### Code Repositories

- [karpathy/micrograd](https://github.com/karpathy/micrograd) — A tiny autograd engine and neural net library. Weeks 13–14.
- [karpathy/makemore](https://github.com/karpathy/makemore) — Character-level language models. Weeks 14–15.
- [karpathy/nanoGPT](https://github.com/karpathy/nanoGPT) — The simplest, fastest repository for training GPT-2. Weeks 15–16.
- [karpathy/minbpe](https://github.com/karpathy/minbpe) — Minimal BPE tokenizer implementation.

### Framework Documentation

- [PyTorch Documentation](https://pytorch.org/docs/stable/) — The framework used throughout Level 3
- [PyTorch Tutorials](https://pytorch.org/tutorials/) — Official tutorial collection
- [Hugging Face Transformers](https://huggingface.co/docs/transformers/) — For fine-tuning and deployment in Weeks 17–18

---

## Hardware — Level 0

Level 0 is built on the Raspberry Pi RP2040 microcontroller. Key references:

- [RP2040 Datasheet (PDF)](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf) — The complete hardware reference. 600+ pages. You will read sections of this, not all of it.
- [Raspberry Pi Pico C/C++ SDK](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html) — Official SDK documentation
- [Pico SDK GitHub Repository](https://github.com/raspberrypi/pico-sdk) — Source code and examples
- [Getting Started with Pico (PDF)](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf) — Setup guide for your development environment
- [FreeRTOS Documentation](https://www.freertos.org/Documentation/RTOS_Fundamentals) — Real-time OS used in Step 5
- [FreeRTOS on RP2040](https://github.com/FreeRTOS/FreeRTOS-SMP-Demos/tree/main/FreeRTOS/Demo/CORTEX_M0%2B_RP2040) — SMP demo for dual-core Pico
- [ARM Cortex-M0+ Technical Reference Manual](https://developer.arm.com/documentation/ddi0484/latest/) — The processor core architecture

---

## General Software Engineering References

### Books

- **The Mythical Man-Month** — Fred Brooks. On why adding people to a late project makes it later. The original essay on software complexity.
- **A Philosophy of Software Design** — John Ousterhout. On complexity as the root problem in software and how to fight it with deep modules and clean interfaces.
- **Structure and Interpretation of Computer Programs** — Abelson & Sussman. The classic MIT text on computation as a way of thinking.
- **The Pragmatic Programmer** — Hunt & Thomas. Practical wisdom for working programmers. The "tracer bullet" and "broken windows" metaphors still hold.
- **Designing Data-Intensive Applications** — Martin Kleppmann. The definitive guide to distributed systems, databases, and data pipelines.

### Papers

- **No Silver Bullet** — Fred Brooks (1986). On essential vs. accidental complexity. Read this before you assume AI solves everything.
- **Out of the Tar Pit** — Moseley & Marks (2006). On the causes of software complexity and the case for functional-relational programming.
- **On the Criteria To Be Used in Decomposing Systems into Modules** — David Parnas (1972). The paper that defined modularity. Still cited, still relevant.
- **Time, Clocks, and the Ordering of Events in a Distributed System** — Leslie Lamport (1978). The foundation of distributed systems reasoning.

### Online References

- [The Missing Semester of Your CS Education (MIT)](https://missing.csail.mit.edu/) — Shell, version control, debugging, and other practical tools
- [Teach Yourself CS](https://teachyourselfcs.com/) — A curated guide to self-studying computer science
- [Build Your Own X](https://github.com/codecrafters-io/build-your-own-x) — Tutorials for building your own database, OS, compiler, etc.

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a>
</p>

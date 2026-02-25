---
layout: page
title: "Level 3 — Foundations: Train & Tune Models"
description: "Understand the layer under the layer. You've used models, orchestrated models — now build them."
permalink: /level-3/
breadcrumb: '<a href="/">Schedule</a> / Level 3'
---

## Philosophy

You have used models (Level 1). You have orchestrated models (Level 2). Now you build them.

Level 3 exists because **you cannot effectively supervise what you do not understand at a fundamental level.** When you prompt a language model, fine-tune an adapter, or deploy a model to production, dozens of design decisions are silently shaping the output. Tokenization strategy. Attention mechanism. Loss function. Quantization scheme. If these are black boxes to you, you are guessing. After Level 3, you are reasoning.

This level follows [Andrej Karpathy's pedagogical sequence](https://karpathy.ai/zero-to-hero.html) — starting from raw backpropagation and building up to a complete GPT implementation. We then extend beyond Karpathy's series into pre-training, fine-tuning, and edge deployment, connecting the theory to real-world production workflows.

> "What I cannot create, I do not understand." — Richard Feynman

---

## The Progression

Each week builds on the previous one. By the end, you will have built every layer of the modern language model stack from scratch.

| Week | Topic | What You Build | Key Concepts |
|------|-------|----------------|--------------|
| [Week 13](/level-3/week-13/) | **Autograd** | micrograd | Backpropagation, computational graphs, automatic differentiation |
| [Week 14](/level-3/week-14/) | **Language Modeling** | makemore | Tokenization, bigrams, MLPs, sampling strategies |
| [Week 15](/level-3/week-15/) | **Transformers** | nanoGPT | Attention, positional encoding, transformer architecture |
| [Week 16](/level-3/week-16/) | **Pre-training** | Custom LM | Data pipelines, distributed training, hyperparameter tuning |
| [Week 17](/level-3/week-17/) | **Fine-tuning** | Task-specific model | LoRA, RLHF, evaluation metrics and frameworks |
| [Week 18](/level-3/week-18/) | **Edge Deployment** | Capstone project | Quantization, ONNX, latency optimization |

---

## Why This Sequence

The sequence is deliberate. Each step removes a layer of abstraction:

1. **micrograd** strips neural networks down to scalar values and derivatives. You see exactly what a gradient is — no tensors, no frameworks, no magic.

2. **makemore** introduces the problem that language models actually solve: predicting the next token. You start with bigrams, build up to MLPs, and see how model architecture affects generation quality.

3. **nanoGPT** assembles the transformer — the architecture behind GPT, Claude, and every modern language model. You implement multi-head attention, layer normalization, and the training loop.

4. **Pre-training** confronts the reality of training at scale: data quality, compute costs, loss curves that plateau, and the engineering required to keep a training run stable.

5. **Fine-tuning** teaches you to adapt a pre-trained model to your specific task — and equally important, to recognize when prompt engineering is sufficient and fine-tuning is unnecessary overhead.

6. **Edge deployment** brings the model out of the data center and onto constrained hardware, teaching the compression and optimization techniques that make real-time inference possible.

---

## Prerequisites

Level 3 assumes you have completed Levels 0 through 2, or have equivalent experience with:

- **Python fluency** — you should be comfortable reading and writing Python without hesitation
- **Basic linear algebra** — matrix multiplication, dot products, transpose operations
- **Basic calculus** — derivatives, the chain rule, what a gradient means geometrically
- **AI agent experience** — you have used and orchestrated LLMs (Levels 1 and 2)

If your calculus is rusty, that is fine. Week 13 (micrograd) rebuilds the intuition from first principles. But you should at least recognize the notation.

---

## What You Will Know Afterward

After completing Level 3, you will be able to:

- Implement backpropagation from scratch and explain what every gradient represents
- Build a character-level language model and reason about its predictions
- Implement the transformer architecture — attention, positional encoding, layer norm — from raw operations
- Set up a pre-training pipeline with data loading, distributed training, and hyperparameter search
- Fine-tune a pre-trained model using LoRA and evaluate it with appropriate metrics
- Quantize and export a model for edge deployment with measured latency budgets
- Make informed architectural decisions when building AI systems, because you understand what is happening inside the model

---

## Weeks

1. [Week 13: micrograd — Backpropagation from Scratch](/level-3/week-13/)
2. [Week 14: makemore — Character-Level Language Models](/level-3/week-14/)
3. [Week 15: nanoGPT — Building Transformers](/level-3/week-15/)
4. [Week 16: Pre-training on Custom Data](/level-3/week-16/)
5. [Week 17: Fine-tuning & Evaluation](/level-3/week-17/)
6. [Week 18: Edge Deployment & Final Project](/level-3/week-18/)

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a> · Next: <a href="/level-3/week-13/">Week 13: micrograd — Backpropagation from Scratch →</a>
</p>

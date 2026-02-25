---
layout: page
title: "Level 0 — Foundations: The Machine Itself"
description: "Before you generate code at scale, understand what a computer actually is."
permalink: /level-0/
breadcrumb: '<a href="/">Schedule</a> / Level 0'
---

## Why Start Here?

This course teaches software engineering in the AI era. You will spend most of your time working with AI agents that generate, review, and deploy code. But before you delegate to machines, you need to understand what those machines actually *are*.

Level 0 exists for a single reason: **you cannot effectively supervise what you do not understand.** When an AI agent generates code that manipulates memory, spawns threads, or talks to hardware, you need a mental model of what is happening beneath the abstractions. Not a vague metaphor — a concrete, testable understanding.

You will not become a systems programmer in these six steps. That is not the goal. The goal is to build a working model of computation from the silicon up, so that every abstraction you encounter later in the course has a foundation you can reason about.

## The RP2040: Our Platform

Everything in Level 0 runs on the **Raspberry Pi RP2040**, a microcontroller that costs less than a cup of coffee. We chose it deliberately.

| Feature | Detail | Why It Matters |
|---|---|---|
| **Processor** | Dual-core ARM Cortex-M0+ @ 133 MHz | Real ARM architecture, same instruction set family as phones and servers |
| **Operating System** | None pre-installed | You see *everything* — no kernel hiding what happens at boot |
| **Cost** | $4–6 per board (Raspberry Pi Pico) | Low-stakes experimentation; buy extras without hesitation |
| **Peripherals** | GPIO, SPI, I2C, UART, PWM, ADC, PIO | Rich enough to build a real computer from components |
| **Documentation** | 600+ page datasheet, official C/C++ SDK, MicroPython support | You can look up every register, every bit, every timing constraint |
| **Community** | Large hobbyist and professional ecosystem | When you get stuck, answers exist |

The RP2040 is not a toy. It is a production microcontroller used in commercial products. But it is simple enough that one person can understand the entire system — and that is exactly what you will do.

## The Six Steps

Each step builds on the last. By the end, you will have a complete mental model of how software runs on hardware.

| Step | Title | What You Build |
|---|---|---|
| [Step 1](/level-0/step-01/) | Bare Metal Assembly on RP2040 | Add two numbers and blink an LED in raw ARM assembly |
| [Step 2](/level-0/step-02/) | C, Compilation, and the Toolchain | Rewrite the same programs in C; inspect generated assembly |
| [Step 3](/level-0/step-03/) | Peripherals — Build a Computer | Wire up an LCD, buttons, SD card, and USB to create a workstation |
| [Step 4](/level-0/step-04/) | Write a Minimal OS Kernel | Implement memory management, task scheduling, and syscalls |
| [Step 5](/level-0/step-05/) | FreeRTOS — A Real RTOS | Port your projects to a production real-time OS |
| [Step 6](/level-0/step-06/) | The Full Stack Map | Trace a keypress from electrical signal to screen pixel |

## What You Will Know Afterward

After completing Level 0, you will be able to:

- Explain what happens when a processor executes an instruction
- Read and write basic ARM assembly
- Describe the compilation pipeline from C source to binary
- Wire up common peripherals and communicate over SPI, I2C, and UART
- Explain what an operating system kernel actually does (and why it exists)
- Reason about concurrency, scheduling, and real-time constraints
- Trace a high-level operation through every layer of the stack

These are not academic exercises. Every concept maps directly to problems you will encounter when building, debugging, and reviewing AI-generated systems in Levels 1–3.

## Hardware Kit — Bill of Materials

You will need the following components for Level 0. Total cost is approximately $30–50 depending on sourcing.

| Component | Quantity | Approx. Cost | Used In |
|---|---|---|---|
| Raspberry Pi Pico (or Pico W) | 1 | $4–6 | All steps |
| USB Micro-B cable | 1 | $3 | All steps |
| Breadboard (half-size or full-size) | 1 | $3–5 | Steps 1–5 |
| Jumper wires (M/M and M/F assorted) | 1 pack | $4–6 | Steps 1–5 |
| LEDs (assorted colors) | 5+ | $1–2 | Steps 1–2 |
| Resistors (220 ohm, 10K ohm assorted) | 10+ | $1–2 | Steps 1–3 |
| Push buttons (tactile) | 4+ | $1–2 | Steps 3–5 |
| SSD1306 OLED display (I2C, 128x64) | 1 | $3–5 | Steps 3–6 |
| MicroSD card breakout (SPI) | 1 | $2–4 | Step 3 |
| MicroSD card (any small capacity) | 1 | $3–5 | Step 3 |
| Rotary encoder | 1 | $1–3 | Step 3 |
| Piezo buzzer | 1 | $1 | Steps 3–5 |

> **Note:** If you have the Pico W variant (with WiFi), you can attempt the wireless stretch goals in Step 3. The standard Pico works for everything else.

Order early. Some components ship from overseas and may take 1–2 weeks to arrive.

---

<p style="text-align: center;">
  <a href="/level-0/step-01/">Begin Step 1: Bare Metal Assembly on RP2040 &rarr;</a>
</p>

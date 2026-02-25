---
layout: page
title: "Step 6: The Full Stack Map"
description: "Trace a single keypress from electrical signal to screen pixel. Map every layer of the abstraction stack."
permalink: /level-0/step-06/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-0/">Level 0</a> / Step 6'
---

## Overview

In Steps 1 through 5, you built each layer of a computing system separately: bare metal instructions, compiled C, peripheral communication, an OS kernel, and a real-time operating system. Now you put it all together into a single, coherent mental model.

The project is simple to describe: **trace a single keypress from the moment the key is pressed to the moment a character appears on the display.** But doing it thoroughly means accounting for every layer — electrical, hardware, firmware, kernel, scheduler, application — and understanding what each layer does, why it exists, and what it costs.

This is the capstone of Level 0. When you finish, you will have a complete map of the abstraction stack that every computer uses.

## What You Will Learn

- How to trace an operation through every layer of the stack
- The responsibility of each layer and why it exists
- The cost of every abstraction — what overhead each layer adds
- Why operating systems exist (and when they should not)
- How to think about systems as layered compositions of simpler parts

## The Trace: One Keypress, Every Layer

You have a USB keyboard connected to the RP2040 (or buttons wired to GPIO, which is more instructive for this exercise). When you press a key, here is what happens:

### Layer 0: Electrical Signal

The button is wired between a GPIO pin and ground, with an internal pull-up resistor enabled.

```
    3.3V
     |
    [Pull-up R]
     |
     +------> GPIO Pin (reads HIGH when open)
     |
    [Button]
     |
    GND
```

- **Button not pressed:** The pull-up resistor holds the pin at 3.3V (logic HIGH, digital 1).
- **Button pressed:** The pin connects to ground through the button (logic LOW, digital 0).
- **Voltage transition:** ~3.3V drops to ~0V in nanoseconds. But the mechanical contacts bounce — the signal oscillates between HIGH and LOW for 1–10 milliseconds before settling.

**Cost:** Nanoseconds for the electrical transition. Milliseconds of bounce that software must handle.

### Layer 1: GPIO Hardware

The RP2040's GPIO controller samples the pin state. When configured for interrupt mode, the controller detects the falling edge (the transition from HIGH to LOW) and:

1. Sets a bit in the GPIO interrupt status register
2. Asserts the GPIO interrupt line to the Cortex-M0+ NVIC (Nested Vectored Interrupt Controller)
3. The NVIC compares the interrupt priority to the current execution priority
4. If the interrupt has sufficient priority, the processor:
   - Pushes R0–R3, R12, LR, PC, and xPSR onto the current stack (8 registers, 32 bytes)
   - Loads the interrupt handler address from the vector table
   - Jumps to the handler

```
GPIO pin falls → GPIO controller sets IRQ → NVIC signals processor →
Processor saves context (8 cycles) → Jumps to ISR vector
```

**Cost:** 12–16 clock cycles from interrupt assertion to first instruction of the handler (on Cortex-M0+ at 133 MHz, that is approximately 90–120 nanoseconds).

### Layer 2: Interrupt Service Routine (ISR)

The ISR runs in interrupt context. It must be fast — interrupts are disabled for the current priority level while the ISR runs, meaning other events of the same or lower priority are delayed.

The ISR does the absolute minimum:

```c
void gpio_isr(void) {
    // 1. Clear the interrupt flag (so it doesn't fire again immediately)
    gpio_acknowledge_irq(BUTTON_PIN, GPIO_IRQ_EDGE_FALL);

    // 2. Signal the kernel that a button event occurred
    BaseType_t higher_woken = pdFALSE;
    xSemaphoreGiveFromISR(button_sem, &higher_woken);

    // 3. Request a context switch if a higher-priority task woke up
    portYIELD_FROM_ISR(higher_woken);
}
```

Three operations. Clear the flag, signal the semaphore, possibly trigger a context switch. Then the ISR returns, the processor pops the saved context, and execution resumes — either the interrupted task or a newly woken higher-priority task.

**Cost:** Approximately 1–2 microseconds for the ISR itself, plus the context save/restore overhead.

### Layer 3: Kernel Scheduler

If `portYIELD_FROM_ISR` triggered a context switch, the kernel scheduler runs:

1. Marks the current task's state as READY
2. Scans the ready queue for the highest-priority task
3. The `input_task` (which was blocked on `button_sem`) is now READY
4. If `input_task` has higher priority than the previously running task, the scheduler selects it
5. Performs a context switch:
   - Saves the full register set of the old task to its stack
   - Restores the full register set of `input_task` from its stack
   - Execution continues in `input_task` as if `xSemaphoreTake()` just returned

**Cost:** Context switch takes approximately 5–10 microseconds on the RP2040, depending on FreeRTOS configuration.

### Layer 4: Input Task (Application Logic)

The input task wakes up from its semaphore wait and processes the event:

```c
void input_task(void *params) {
    while (1) {
        xSemaphoreTake(button_sem, portMAX_DELAY);  // Block until press

        // Debounce: wait for signal to stabilize
        vTaskDelay(pdMS_TO_TICKS(20));

        // Read which button is pressed
        uint8_t key = scan_buttons();

        // Map physical button to character
        char ch = keymap[key];

        // Send to display task via queue
        xQueueSend(char_queue, &ch, portMAX_DELAY);
    }
}
```

The task debounces (waits 20 ms for bounce to settle), reads the button state, maps it to a character, and sends it through a queue to the display task.

**Cost:** 20 milliseconds for debounce (deliberate), plus microseconds for the scan and queue send.

### Layer 5: Display Task (Output)

The display task receives the character and renders it:

```c
void display_task(void *params) {
    char line[17] = {0};  // 16 chars + null
    int cursor = 0;

    while (1) {
        char ch;
        xQueueReceive(char_queue, &ch, portMAX_DELAY);  // Block until char

        // Add character to line buffer
        line[cursor++] = ch;
        if (cursor >= 16) cursor = 0;  // Wrap

        // Render to display
        oled_clear();
        oled_draw_text("> ", 0, 0);
        oled_draw_text(line, 12, 0);
        oled_show();  // Send framebuffer over I2C
    }
}
```

`oled_show()` transfers 1024 bytes over I2C at 400 kHz. Each byte takes 9 bit-times (8 data + 1 ACK), so the full transfer takes:

```
1024 bytes * 9 bits/byte / 400,000 bits/second = 23 milliseconds
```

**Cost:** ~23 milliseconds for the I2C display transfer. This is by far the most expensive step in the entire chain.

### The Complete Trace

| Layer | What Happens | Time |
|---|---|---|
| Electrical | Pin voltage drops from 3.3V to 0V | ~10 ns |
| GPIO hardware | Edge detected, interrupt asserted | ~100 ns |
| NVIC + context save | Processor saves state, jumps to ISR | ~120 ns |
| ISR | Clear flag, signal semaphore | ~2 us |
| Context switch | Scheduler selects input task, restores context | ~8 us |
| Debounce wait | Deliberate delay for signal stability | 20 ms |
| Button scan + keymap | Read GPIO, look up character | ~10 us |
| Queue send + context switch | Transfer to display task | ~10 us |
| Display render | Clear buffer, draw text | ~50 us |
| I2C transfer | Send 1024 bytes to OLED controller | ~23 ms |
| **Total** | **Button press to pixel change** | **~43 ms** |

The electrical signal traverses the entire path in under 50 milliseconds. But notice the distribution: over 99% of the time is spent in two deliberate waits (debounce and I2C transfer). The actual processing — interrupts, scheduling, application logic — takes microseconds.

## Project: Create the Diagram

Your deliverable for Step 6 is a detailed diagram that traces this path. It should include:

1. **A vertical stack diagram** showing every layer from hardware to application
2. **Arrows showing the flow** of a keypress through each layer
3. **Time annotations** at each transition
4. **Register-level detail** for the context switches (which registers are saved, where they go)
5. **Protocol-level detail** for the I2C transfer (start condition, address byte, data bytes, ACK/NACK)
6. **Annotations explaining why each layer exists** — what problem does it solve?

### Suggested Format

```
ELECTRICAL SIGNAL
  Button closes → pin voltage: 3.3V → 0V
  Bounce: ~5ms of oscillation
         │
         ▼
GPIO HARDWARE (RP2040 peripheral)
  Edge detector triggers on falling edge
  Sets bit in IO_BANK0 interrupt status register
  Asserts IRQ to NVIC
         │
         ▼
NVIC + PROCESSOR (ARM Cortex-M0+ core)
  Compares IRQ priority to current execution priority
  Pushes {R0-R3, R12, LR, PC, xPSR} to stack
  Loads handler address from vector table at offset N
  Branches to ISR
         │
         ▼
ISR (interrupt context — kernel code)
  Clears GPIO interrupt flag
  Gives binary semaphore
  Requests context switch via PendSV
         │
         ▼
SCHEDULER (FreeRTOS kernel)
  PendSV handler saves full context of interrupted task
  Scans ready list for highest-priority task
  Restores context of input_task
  Returns from exception → input_task resumes
         │
         ▼
INPUT TASK (application code)
  xSemaphoreTake returns
  Debounce delay (20ms)
  Scans GPIO pins for button state
  Maps button to character via keymap table
  Sends character to queue
         │
         ▼
DISPLAY TASK (application code)
  xQueueReceive returns with character
  Updates line buffer
  Renders text to framebuffer (128x64 pixel array)
  Calls oled_show()
         │
         ▼
I2C DRIVER (HAL / peripheral driver)
  Generates I2C start condition
  Sends device address (0x3C) + write bit
  Sends command byte (0x40 = data mode)
  Sends 1024 bytes of framebuffer data
  Generates I2C stop condition
         │
         ▼
SSD1306 DISPLAY CONTROLLER (external hardware)
  Receives pixel data over I2C
  Writes to internal GDDRAM
  Drives OLED pixel matrix
  Character appears on screen
```

## Why Does Each Layer Exist?

Every layer of abstraction exists to solve a specific problem. But every layer also has a cost.

| Layer | Problem It Solves | Cost It Introduces |
|---|---|---|
| GPIO hardware | Converts analog voltages to digital signals | Pin configuration complexity, interrupt latency |
| NVIC + ISR | Responds to events without polling | Context save/restore overhead, ISR execution time constraints |
| Kernel scheduler | Allows multiple tasks to share one processor | Context switch overhead, scheduling algorithm complexity |
| Queues / semaphores | Safe communication between concurrent tasks | Memory overhead (queue buffers), blocking delays |
| Application logic | Translates hardware events into user-meaningful actions | Processing time, code complexity |
| Display driver | Abstracts display protocol details | I2C transfer time, framebuffer memory |

> **The fundamental lesson:** Every abstraction trades a concrete cost for a reduction in complexity. The art of systems engineering is knowing which abstractions are worth their cost — and which ones you can skip.

## When the OS Should Not Exist

Not every system needs an operating system. For the LED blinker in Step 1, an RTOS would be absurd overhead. The decision depends on complexity:

| Scenario | OS Needed? | Why |
|---|---|---|
| Blink an LED | No | A single `while` loop suffices |
| Read a sensor, display value | Maybe | A super-loop might work; tasks help if timing is critical |
| Multiple peripherals, timing constraints | Yes | Task isolation and preemptive scheduling prevent interference |
| Network connectivity, file systems | Yes | The protocol stacks assume an OS threading model |
| Safety-critical with certification | Yes | Certified RTOS provides documented timing guarantees |

Understanding when *not* to use an abstraction is just as important as understanding the abstraction itself.

## Exercises

1. **Trace a different operation.** Instead of a keypress, trace an SD card file read: from the `open()` call through the filesystem layer, SPI driver, electrical signals on the bus, and back to the application with the data.

2. **Measure every transition.** Toggle GPIO pins at each layer boundary. Use an oscilloscope or logic analyzer to capture the actual timing. Compare your measurements to the theoretical estimates above.

3. **Remove a layer.** Implement the keypress-to-display path without FreeRTOS — just bare-metal C with a super-loop and direct peripheral access. How much simpler is the code? How much faster is the path? What do you lose?

4. **Add a layer.** Implement a simple text editor on top of the shell from Step 3's stretch goal. Trace a keypress through the additional application layer. What does the editor layer add in terms of both capability and overhead?

5. **Map to a desktop system.** Take your RP2040 stack diagram and draw the equivalent for a Linux desktop system: USB HID driver, Linux kernel input subsystem, X11/Wayland display server, GUI toolkit, application. How many more layers are there? Why does each one exist?

## Key Takeaways

- A keypress traverses 7+ layers from electrical signal to screen pixel
- Over 99% of the latency comes from deliberate waits (debounce) and I/O (display transfer), not from processing
- Every layer of abstraction exists to solve a specific problem and has a measurable cost
- Context switches, interrupt handling, and scheduling are fast (microseconds) compared to I/O (milliseconds)
- The operating system exists because managing concurrency and hardware access manually does not scale
- Understanding the full stack means you can diagnose problems at the right layer — not guessing, but knowing where to look
- This mental model applies to every computing system, from microcontrollers to cloud servers — only the specifics change, not the structure

---

You now have a complete picture of the machine. Every abstraction from Level 1 onward — programming languages, frameworks, AI agents, neural networks — runs on top of what you built and traced in Level 0.

---

<p>
  <a href="/level-0/step-05/">&larr; Step 5: FreeRTOS — A Real RTOS</a> · <a href="/level-0/">Back to Level 0 Overview</a>
</p>

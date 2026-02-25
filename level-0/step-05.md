---
layout: page
title: "Step 5: FreeRTOS — A Real RTOS"
description: "Port your projects to a production real-time operating system. Learn tasks, queues, semaphores, and what 'real-time' actually means."
permalink: /level-0/step-05/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-0/">Level 0</a> / Step 5'
---

## Overview

In Step 4 you wrote a minimal kernel. It worked — but it was fragile, limited, and missing features you would need for any real application. In this step you will replace your kernel with **FreeRTOS**, one of the most widely deployed real-time operating systems in the world.

FreeRTOS runs on billions of devices — from medical instruments to satellite subsystems to the ESP32 in your WiFi router. Amazon maintains it. The source code is readable (about 9,000 lines of C for the core). It provides everything your Step 4 kernel provided, plus much more, battle-tested across decades of production use.

By porting your peripheral code from Step 3 to run as FreeRTOS tasks, you will learn the standard concurrency primitives that apply across all operating systems: tasks, queues, semaphores, and mutexes. You will also learn what "real-time" means — and why it is different from "fast."

## What You Will Learn

- FreeRTOS task model: creation, priorities, states, and deletion
- Queues: safe data passing between tasks
- Semaphores and mutexes: coordinating shared resource access
- Priority-based preemptive scheduling
- Inter-task communication patterns
- What "real-time" guarantees actually mean
- Dual-core operation on the RP2040

## What Does "Real-Time" Mean?

> A real-time system is not a fast system. It is a **predictable** system.

A real-time OS guarantees that if a high-priority event occurs, the system will respond within a **bounded, known time**. This is called the **worst-case response time**.

| Type | Guarantee | Example |
|---|---|---|
| **Hard real-time** | Missing a deadline causes system failure | Airbag deployment, pacemaker timing |
| **Firm real-time** | Missing a deadline degrades quality but is not catastrophic | Video frame rendering, audio playback |
| **Soft real-time** | Deadlines are desired but not strictly enforced | UI responsiveness, network timeouts |

FreeRTOS provides the mechanisms for hard real-time — but only if you use them correctly. A single poorly written task can violate timing guarantees for the entire system.

Linux, by contrast, is not a real-time OS. When you call `sleep(1)` on Linux, you will wake up *approximately* 1 second later — maybe 1.001 seconds, maybe 1.05 seconds. The kernel makes no guarantee. On FreeRTOS with proper configuration, `vTaskDelay(pdMS_TO_TICKS(1))` will wake your task within microseconds of the requested time.

## FreeRTOS Core Concepts

### Tasks

A FreeRTOS task is equivalent to a thread. Each task has its own stack, its own program counter, and a priority level. The scheduler always runs the highest-priority ready task.

```c
#include "FreeRTOS.h"
#include "task.h"
#include "pico/stdlib.h"

void blink_task(void *params) {
    const uint LED_PIN = 25;
    gpio_init(LED_PIN);
    gpio_set_dir(LED_PIN, GPIO_OUT);

    while (1) {
        gpio_put(LED_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(500));  // Block for 500ms — scheduler runs other tasks
        gpio_put(LED_PIN, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void serial_task(void *params) {
    while (1) {
        printf("Tick: %lu\n", xTaskGetTickCount());
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main() {
    stdio_init_all();

    xTaskCreate(blink_task, "Blink", 256, NULL, 1, NULL);
    xTaskCreate(serial_task, "Serial", 512, NULL, 1, NULL);

    vTaskStartScheduler();  // Never returns

    return 0;
}
```

Compare this to Step 4: you did not write a context switcher, a timer interrupt handler, or a scheduler. FreeRTOS handles all of it. Your code just defines tasks and calls `vTaskDelay()` to yield.

### Task States

Every task is always in exactly one of these states:

```
                    ┌─────────┐
        create ───> │  Ready  │ <──── event/timeout
                    └────┬────┘
                         │ scheduler selects
                         v
                    ┌─────────┐
                    │ Running │ ──── only ONE task at a time (per core)
                    └────┬────┘
                    ╱         ╲
          vTaskDelay()    yield/preempt
           wait on queue       │
                 │              │
                 v              v
           ┌──────────┐  ┌─────────┐
           │ Blocked  │  │  Ready  │
           └──────────┘  └─────────┘
```

A blocked task consumes zero CPU — the scheduler skips it entirely. This is fundamentally different from the busy-wait `delay()` loops you wrote in earlier steps.

### Queues

Queues are the primary mechanism for passing data between tasks safely. They are thread-safe by design — the kernel handles the locking.

```c
#include "queue.h"

QueueHandle_t sensor_queue;

void sensor_task(void *params) {
    sensor_queue = xQueueCreate(10, sizeof(int));  // Queue of 10 integers

    while (1) {
        int reading = read_adc();       // Read analog sensor
        xQueueSend(sensor_queue, &reading, portMAX_DELAY);  // Block if full
        vTaskDelay(pdMS_TO_TICKS(100)); // Read every 100ms
    }
}

void display_task(void *params) {
    int value;

    while (1) {
        // Block until data is available
        if (xQueueReceive(sensor_queue, &value, portMAX_DELAY) == pdTRUE) {
            char buf[32];
            snprintf(buf, sizeof(buf), "Sensor: %d", value);
            oled_draw_text(buf, 0, 0);
            oled_show();
        }
    }
}
```

The sensor task produces readings; the display task consumes them. They run independently. If the sensor task is faster, the queue buffers the excess. If it is slower, the display task blocks and waits. No explicit synchronization code, no race conditions.

### Semaphores

A semaphore is a signaling mechanism. A **binary semaphore** is like a flag: one task sets it, another waits for it.

```c
#include "semphr.h"

SemaphoreHandle_t button_sem;

// Interrupt handler — runs in ISR context
void gpio_callback(uint gpio, uint32_t events) {
    BaseType_t higher_woken = pdFALSE;
    xSemaphoreGiveFromISR(button_sem, &higher_woken);
    portYIELD_FROM_ISR(higher_woken);
}

void button_task(void *params) {
    button_sem = xSemaphoreCreateBinary();
    gpio_set_irq_enabled_with_callback(BUTTON_PIN, GPIO_IRQ_EDGE_FALL,
                                        true, &gpio_callback);
    while (1) {
        // Block until button interrupt fires
        xSemaphoreTake(button_sem, portMAX_DELAY);
        printf("Button pressed!\n");
        handle_button_press();
    }
}
```

This is the correct pattern for handling hardware interrupts in an RTOS: the ISR does minimal work (give the semaphore), and the task does the real processing. The ISR runs in microseconds; the task can take as long as it needs.

### Mutexes

A mutex (mutual exclusion) protects a shared resource so that only one task can access it at a time.

```c
SemaphoreHandle_t spi_mutex;

void init_spi_mutex(void) {
    spi_mutex = xSemaphoreCreateMutex();
}

void sd_card_task(void *params) {
    while (1) {
        xSemaphoreTake(spi_mutex, portMAX_DELAY);  // Lock
        sd_write_block(data, block_num);
        xSemaphoreGive(spi_mutex);                  // Unlock
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void display_task(void *params) {
    while (1) {
        xSemaphoreTake(spi_mutex, portMAX_DELAY);  // Lock
        oled_refresh();
        xSemaphoreGive(spi_mutex);                  // Unlock
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

Both the SD card and the OLED display use SPI. Without the mutex, their SPI transactions could interleave, corrupting both. The mutex ensures only one task uses the SPI bus at a time.

## Project: Port Peripherals to FreeRTOS Tasks

Take your Step 3 peripheral code and restructure it as FreeRTOS tasks:

| Task | Priority | Responsibility |
|---|---|---|
| `display_task` | 3 (high) | Update the OLED display |
| `input_task` | 2 (medium) | Read buttons and encoder, send events to queue |
| `storage_task` | 1 (low) | Read/write SD card on request |
| `blink_task` | 0 (idle) | Heartbeat LED blink |

Use queues to pass events from `input_task` to `display_task`. Use a mutex to protect the SPI bus shared between the display and SD card.

### Adding the Second Core

The RP2040 has two Cortex-M0+ cores. FreeRTOS SMP (Symmetric Multi-Processing) can schedule tasks across both:

```c
// Launch a task pinned to core 1
xTaskCreateAffinitySet(
    sensor_task,        // Task function
    "Sensor",           // Name
    512,                // Stack size
    NULL,               // Parameters
    2,                  // Priority
    (1 << 1),           // Core affinity — core 1 only
    NULL                // Task handle
);
```

Now your system truly runs tasks in parallel. The display updates on core 0 while the sensor reads on core 1. But this introduces real concurrency bugs that single-core systems never see — and that is the point.

## Exercise: The Priority Inversion Bug

This is the most important exercise in Step 5. You will deliberately create a famous concurrency bug, observe its effects, and then fix it.

### Background: Mars Pathfinder (1997)

The Mars Pathfinder spacecraft landed on Mars on July 4, 1997. Shortly after landing, its computer began resetting repeatedly. The cause: **priority inversion**.

A low-priority task held a mutex on a shared data bus. A high-priority task needed that mutex and blocked. A medium-priority task, which did not need the mutex, preempted the low-priority task — preventing it from releasing the mutex. The high-priority task starved, a watchdog timer fired, and the system reset.

The fix: **priority inheritance**. When a low-priority task holds a mutex that a high-priority task needs, the low-priority task temporarily inherits the high priority, runs to completion, releases the mutex, and drops back to its original priority.

### Recreate the Bug

```c
SemaphoreHandle_t shared_mutex;

// Low priority — holds the mutex for a long time
void low_task(void *params) {
    while (1) {
        xSemaphoreTake(shared_mutex, portMAX_DELAY);
        printf("[LOW] Acquired mutex, doing long work...\n");
        busy_work(5000);  // Simulate slow I/O
        xSemaphoreGive(shared_mutex);
        printf("[LOW] Released mutex\n");
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

// Medium priority — does NOT use the mutex
void medium_task(void *params) {
    while (1) {
        printf("[MED] Running CPU-intensive work...\n");
        busy_work(10000);  // This preempts LOW, blocking HIGH
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

// High priority — needs the mutex
void high_task(void *params) {
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(200));
        printf("[HIGH] Waiting for mutex...\n");
        uint32_t start = xTaskGetTickCount();
        xSemaphoreTake(shared_mutex, portMAX_DELAY);
        uint32_t wait = xTaskGetTickCount() - start;
        printf("[HIGH] Got mutex after %lu ticks!\n", wait);
        xSemaphoreGive(shared_mutex);
    }
}

int main() {
    stdio_init_all();

    // BUG: using a binary semaphore (no priority inheritance)
    shared_mutex = xSemaphoreCreateBinary();
    xSemaphoreGive(shared_mutex);

    xTaskCreate(low_task,    "Low",    512, NULL, 1, NULL);
    xTaskCreate(medium_task, "Medium", 512, NULL, 2, NULL);
    xTaskCreate(high_task,   "High",   512, NULL, 3, NULL);

    vTaskStartScheduler();
    return 0;
}
```

Run this and observe: the HIGH task waits far longer than it should — blocked not by its own work, but by the MEDIUM task preempting the LOW task that holds the mutex.

### Fix It

Replace `xSemaphoreCreateBinary()` with `xSemaphoreCreateMutex()`. FreeRTOS mutexes implement priority inheritance by default:

```c
// FIX: use a mutex (which supports priority inheritance)
shared_mutex = xSemaphoreCreateMutex();
```

Now when HIGH blocks on the mutex, LOW inherits HIGH's priority, preempts MEDIUM, finishes its work, and releases the mutex. HIGH gets the mutex promptly.

## Exercises

1. **Measure context switch time.** Toggle a GPIO pin in two alternating tasks. Use an oscilloscope or logic analyzer to measure the switching overhead. How many microseconds does a context switch take?

2. **Queue overflow.** What happens when you send to a full queue with `xQueueSend(..., 0)` (zero timeout)? With `portMAX_DELAY`? Design a producer-consumer system where the producer is faster than the consumer and handle the overflow gracefully.

3. **Stack overflow detection.** Enable `configCHECK_FOR_STACK_OVERFLOW` in `FreeRTOSConfig.h`. Create a task that recurses deeply. What does FreeRTOS do when it detects the overflow?

4. **Dual-core synchronization.** Run a producer on core 0 and a consumer on core 1, sharing a queue. Verify that the queue operations are safe across cores.

5. **Deadline monitoring.** Create a task that must complete its work within 10 ms. Add instrumentation to measure actual execution time. Introduce load that causes deadline misses. How would you detect and report this in a production system?

## Key Takeaways

- "Real-time" means predictable, not fast — worst-case response time is what matters
- FreeRTOS provides tasks, queues, semaphores, and mutexes — the same primitives found in every concurrent system
- Priority inversion is a real bug that crashed a Mars mission; priority inheritance is the fix
- ISRs should be short: signal a semaphore and return; let a task do the real work
- Dual-core adds real parallelism but also real concurrency bugs
- Every concurrency concept here (mutexes, priority scheduling, producer-consumer queues) applies directly to concurrent programming on any platform

---

<p>
  <a href="/level-0/step-04/">&larr; Step 4: Write a Minimal OS Kernel</a> · <a href="/level-0/step-06/">Step 6: The Full Stack Map &rarr;</a>
</p>

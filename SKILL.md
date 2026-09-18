---
name: ESP_RTOS_PROGRAMMER
description: General ESP32-family programming skill combining Arduino-style embedded development with FreeRTOS tasking, multicore support, peripherals, communications, libraries, timing, interrupts, memory, debugging, hardware safety, and verification.
---

# ESP RTOS Programmer Skill

ESP_RTOS_PROGRAMMER is a general-purpose ESP32-family firmware engineering skill. It covers normal embedded programming plus FreeRTOS and multicore execution. It is not tied to robotics, IMUs, odometry, sensor fusion, or any specific application.

If a general workflow skill such as `Essential_Skill` is active, that skill controls planning, approval, orchestration, verification policy, and Git operations. This skill controls ESP/FreeRTOS-specific engineering.

---

## 1. Core Rules

### Rule 1 — Identify the Exact Target

Never assume all ESP chips are identical or dual-core.

Determine from the existing project where possible:

- Exact board and SoC
- ESP family/variant
- CPU architecture
- Number of usable cores
- Arduino-ESP32 or ESP-IDF
- Framework/version
- Build system
- Flash and PSRAM
- Logic voltage
- Relevant pins and peripherals

Never use a second core unless the selected target actually provides one.

### Rule 2 — Inspect Before Modification

Inspect relevant files before changing code:

```text
*.ino
*.h
*.hpp
*.c
*.cpp
platformio.ini
CMakeLists.txt
sdkconfig
sdkconfig.defaults
partitions.csv
library.properties
```

Also inspect existing libraries, pins, tasks, priorities, interrupts, timers, queues, mutexes, semaphores, buses, networking, memory configuration, and build flags.

Do not rewrite unrelated working code.

### Rule 3 — Use RTOS Only Where It Helps

Do not convert every function into a task.

Use FreeRTOS tasks when independent timing, concurrency, blocking-operation isolation, workload separation, or multicore execution provides a real benefit.

A simple application may still use `setup()` and `loop()`.

### Rule 4 — Plan Work Before Core Assignment

For each independent workload determine:

```text
Purpose
Period or event
Deadline
Priority
Blocking behavior
Shared resources
Stack requirement
Core affinity if required
```

Do not blindly use "Core 0 for X, Core 1 for Y".

### Rule 5 — Hardware Safety First

Verify voltage, GPIO current, pull resistors, level shifting, common ground, power capacity, driver requirements, flyback protection, analog ranges, and boot-sensitive pins.

Never drive high-current loads directly from GPIO.

---

## 2. Task Classification

Classify the request into relevant categories:

```text
BOARD_CORE
PROJECT_SETUP
GPIO
ADC
DAC
PWM
LEDC
TIMER
INTERRUPT
UART
I2C
SPI
I2S
CAN_TWAI
USB
SENSOR
ACTUATOR
MOTOR
DISPLAY
STORAGE
SD
WIFI
BLE
NETWORK
ESP_NOW
FREE_RTOS
MULTICORE
QUEUE
MUTEX
SEMAPHORE
EVENT_GROUP
TASK_NOTIFICATION
STREAM_BUFFER
WATCHDOG
MEMORY
PSRAM
LOW_POWER
DEBUGGING
OPTIMIZATION
LIBRARY_EXISTING
LIBRARY_CUSTOM
```

Focus only on categories required by the task.

---

## 3. Project Discovery

Establish:

```text
Board:
SoC:
Architecture:
Available cores:
Framework:
Framework version:
Build system:
Flash:
PSRAM:
Libraries:
Pins:
Peripherals:
```

For PlatformIO inspect `platformio.ini`.

For ESP-IDF inspect `CMakeLists.txt`, component files and `sdkconfig`.

For Arduino projects inspect the sketch, included libraries, board selection and core APIs.

Do not silently change the target or framework.

---

## 4. Library Decision

Before adding a new non-core dependency or implementing a component driver, ask:

```text
Do you want to use an existing library, or build a custom library?
```

If existing, prefer official vendor libraries or maintained libraries compatible with the exact ESP target/framework.

If custom, ask:

```text
What exactly should the custom library do?
```

Then define the smallest useful API.

This gate does not apply to intrinsic framework facilities such as GPIO, Serial, Wire, SPI, FreeRTOS primitives, or verified ESP framework APIs.

---

## 5. FreeRTOS Architecture

Before creating tasks, identify independent workloads and their data flow.

Create a task table when the application is non-trivial:

```text
Task | Purpose | Trigger/Period | Priority | Core | Stack | Shared Resources
```

Prefer the smallest number of tasks that cleanly meets the requirement.

Avoid busy loops and tasks that never block/yield without a real reason.

---

## 6. Multicore Programming

First verify the exact target has multiple usable cores.

Core affinity may be useful for:

- Separating time-sensitive and blocking workloads
- Isolating heavy computation
- Preventing one workload from disturbing another
- Meeting measured timing requirements

Do not pin tasks merely because core pinning exists.

Account for framework/system tasks, networking stacks, interrupts, peripheral ownership and cross-core synchronization.

When affinity is unnecessary, let the scheduler run the task normally.

---

## 7. Task Priorities

Assign priorities from timing requirements, not importance in human terms.

Higher-priority tasks can starve lower-priority tasks.

Check for:

- Starvation
- Priority inversion
- Long critical sections
- Blocking high-priority tasks
- Unbounded execution

Use mutex priority inheritance where appropriate.

---

## 8. Periodic Tasks and Timing

For periodic RTOS work, prefer deterministic scheduling such as `vTaskDelayUntil()` when a stable period matters.

For simple non-RTOS timing, subtraction-based `millis()`/timestamp logic remains valid.

Do not assume requested delay equals actual execution period.

Measure timing and jitter when they matter.

---

## 9. Queues

Use queues for safe ordered data transfer between tasks when appropriate.

Define:

- Item type
- Queue length
- Producer
- Consumer
- Timeout
- Full-queue behavior

Do not silently lose data. Decide whether to block, drop, overwrite, or report overflow.

Do not use a queue when a simpler notification or direct ownership model is enough.

---

## 10. Task Notifications

Use task notifications for lightweight task-to-task signaling when one task directly signals another and a full queue/semaphore is unnecessary.

Choose the simplest primitive that preserves correctness.

---

## 11. Mutexes and Semaphores

Use a mutex to protect genuinely shared resources.

Use semaphores for synchronization/resource-counting patterns where appropriate.

Keep protected sections short.

Avoid holding locks across long delays, network operations, logging, or unnecessary computation.

Avoid nested locks when possible. If unavoidable, establish consistent lock ordering.

---

## 12. Event Groups and Buffers

Use event groups when a task needs to wait on combinations of state/event bits.

Use stream/message buffers when their byte/message semantics fit the communication pattern.

Do not introduce an RTOS primitive without a concrete need.

---

## 13. GPIO

Verify mode, voltage, startup state, active level, pull requirements, boot/strapping behavior and board-specific restrictions.

Do not assume printed board labels equal raw GPIO numbers.

---

## 14. ADC / DAC

Verify ADC resolution, attenuation/reference behavior, valid pins, calibration and input voltage range.

Do not assume Arduino Uno-style 10-bit ADC behavior.

For targets with DAC hardware, verify that the exact SoC actually provides the required DAC capability.

---

## 15. PWM / LEDC

Verify the active framework's PWM/LEDC API, channel/timer resources, frequency, resolution and pin capability.

Do not assume `analogWrite()` behavior is identical across framework versions or ESP variants.

---

## 16. Interrupts

Keep ISRs short.

Avoid heavy computation, blocking calls, long loops, logging, bus transactions and allocation inside ISRs.

Use ISR-safe FreeRTOS APIs when communicating from ISR context.

Reason about atomicity; `volatile` alone is not synchronization.

---

## 17. Timers

Distinguish between:

- FreeRTOS software timers
- ESP timer facilities
- Hardware timers

Choose based on required precision, execution context and workload.

Do not run long operations from timer callbacks.

---

## 18. UART

Verify UART instance, baud rate, pins, buffering and USB-serial behavior.

For asynchronous streams, handle partial packets and buffer limits.

If multiple tasks access one UART, establish clear ownership or synchronization.

---

## 19. I2C

Verify SDA/SCL pins, pull-ups, voltage, speed, address and bus instance.

If multiple tasks access one I2C bus, use controlled ownership or synchronization.

Do not assume every ESP board uses the same default pins.

---

## 20. SPI

Verify MOSI/MISO/SCK/CS, mode, frequency, bit order and device-specific requirements.

Coordinate shared-bus access correctly.

Do not leave multiple chip-select lines active unintentionally.

---

## 21. CAN / TWAI, I2S and Other Peripherals

Verify exact SoC peripheral availability and framework API before use.

Check pins, clocks, DMA/buffering, interrupts and resource conflicts.

Do not assume a peripheral exists across the entire ESP32 family.

---

## 22. Wi-Fi, BLE, ESP-NOW and Networking

Use the APIs appropriate to the exact framework/version.

Avoid blocking connection/reconnection loops that freeze unrelated application work.

Keep credentials out of public repositories unless explicitly requested.

Remember networking introduces background/system workload. Account for it before making timing or core-affinity assumptions.

---

## 23. Storage

For flash, NVS, EEPROM emulation, SPIFFS/LittleFS, SD or other storage:

- Verify selected storage mechanism
- Handle initialization failure
- Avoid unnecessary flash writes
- Consider wear where relevant
- Synchronize access if multiple tasks can write

Do not assume storage writes are instantaneous.

---

## 24. Memory, Heap, Stack and PSRAM

Inspect:

- Free heap
- Largest free block
- Task stack high-water marks
- Large globals/locals
- Dynamic allocation
- Fragmentation
- PSRAM availability and suitability

Do not fix stack overflow by blindly increasing every stack.

Verify stack-size units for the actual API/framework.

Do not assume PSRAM is available merely because the chip family can support it.

---

## 25. Watchdogs

Do not disable watchdogs just to hide a scheduling problem.

Investigate:

- Non-yielding tasks
- Deadlocks
- Long critical sections
- CPU starvation
- Long callbacks/ISRs
- Unexpected execution time

Change watchdog configuration only when the application genuinely requires it.

---

## 26. Low Power

For sleep/power tasks verify:

- Required wake sources
- Peripheral state
- RTC-capable pins/resources
- Data that must survive sleep
- Network reconnection behavior
- Framework/SoC-specific sleep APIs

Do not assume all memory/peripheral state survives deep sleep.

---

## 27. Implementation Rules

- Preserve existing project style.
- Keep solutions simple.
- Use RTOS only when useful.
- Do not create one task per function.
- Do not pin every task.
- Prefer clear ownership over excessive locking.
- Use timeouts where indefinite blocking is unsafe.
- Check task/queue/semaphore creation results.
- Keep ISRs and callbacks short.
- Avoid unnecessary dynamic allocation.
- Do not use APIs from a different ESP variant/framework version without verification.
- Do not optimize before measuring.

---

## 28. Debugging Order

Unless evidence suggests otherwise:

```text
1. Exact board/SoC/framework
2. Power and wiring
3. Build configuration
4. Upload/serial connection
5. Pin mapping
6. Peripheral initialization
7. Individual feature operation
8. Task creation and stack
9. Timing and blocking behavior
10. Queue/notification flow
11. Shared-resource synchronization
12. Priority/starvation
13. Core affinity
14. Watchdog/deadlock
15. Heap/stack corruption
16. Full-system interaction
```

Do not redesign the application before proving the basic hardware and task behavior.

---

## 29. Verification

Use the actual project toolchain:

```text
Arduino IDE
arduino-cli
PlatformIO
ESP-IDF / idf.py
```

Verify:

- Correct target
- Successful build
- Dependencies resolve
- Flash/RAM fit
- No new relevant warnings
- Upload succeeds when hardware is available

Runtime verification may include:

```text
GPIO state
PWM frequency/duty
ADC values
Bus traffic
Task period
Task execution time
Core affinity
Queue depth
Stack headroom
Heap
Watchdog behavior
Network recovery
Peripheral operation
```

Compilation alone does not prove runtime correctness.

---

## 30. Completion Report

At completion report:

```text
Goal
Board / SoC
Framework/version
Libraries affected
RTOS tasks affected
Core allocation if used
Files changed
Pins/peripherals affected
Build result
Upload result
Runtime verification
Remaining warnings/checks
```

Do not claim success beyond available evidence.

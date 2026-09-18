# ESP RTOS Programmer

A general-purpose AI agent skill for **ESP32-family programming with FreeRTOS and multicore support**.

It is essentially an ESP-focused embedded programming skill: GPIO, ADC, PWM, timers, interrupts, UART, I2C, SPI, CAN/TWAI, Wi-Fi, BLE, storage, libraries, debugging and hardware verification — plus FreeRTOS tasks, priorities, queues, synchronization and multicore execution.

It is **not tied to robotics or any specific sensor/application**.

## What it adds

Normal ESP programming:

```text
GPIO / ADC / PWM
UART / I2C / SPI
Timers / Interrupts
Wi-Fi / BLE / ESP-NOW
Storage
Libraries
Memory
Debugging
Hardware verification
```

FreeRTOS support:

```text
Tasks
Priorities
Queues
Task notifications
Mutexes
Semaphores
Event groups
Stream/message buffers
Watchdogs
Stack/heap monitoring
Deterministic timing
```

Multicore support:

```text
Workload analysis
Core availability detection
Core affinity when useful
Cross-task/core synchronization
System-task awareness
Timing and load verification
```

## Core idea

The skill does not automatically turn everything into RTOS tasks or pin everything to different cores.

It first checks the exact ESP target and application, then uses the simplest architecture that works.

```text
Application
   |
   +-- simple work ----------> normal code
   |
   +-- concurrent work ------> FreeRTOS tasks
   |
   +-- parallel workload ----> multicore when supported/useful
```

## ESP family support

The skill is designed to reason about ESP32-family targets without assuming that every chip has the same cores or peripherals.

It verifies the exact target before using core affinity or SoC-specific features.

## Development environments

- Arduino-ESP32
- Arduino IDE / Arduino CLI
- PlatformIO
- ESP-IDF
- C/C++

## Example

```text
Use ESP_RTOS_PROGRAMMER for this ESP32 project.

I need to read three sensors, control two outputs, handle Wi-Fi,
and process data continuously. Inspect the board and decide whether
FreeRTOS tasks or multiple cores are useful before implementing it.
```

## Repository

```text
ESP_RTOS_PROGRAMMER/
├── SKILL.md
└── README.md
```

## Related skill

For general Arduino-compatible boards:

https://github.com/Ashwin312007/Arduino_Programmer

## Author

Ashwin T E

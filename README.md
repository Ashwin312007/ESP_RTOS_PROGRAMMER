# ESP RTOS Programmer

An AI agent skill for designing and debugging **ESP32-family FreeRTOS firmware**, with emphasis on robotics, deterministic task scheduling, multicore processing, sensor pipelines, odometry, sensor fusion, and motor control.

## What it does

Instead of treating an ESP32 program like one large Arduino `loop()`, this skill makes the agent reason about the firmware as a real-time system:

```text
Sensors / Encoders
        |
        v
Acquisition Tasks
        |
        v
Queues / Notifications
        |
        v
Processing / Odometry / EKF
        |
        v
Control Task
        |
        v
Motor Output

Telemetry / Wi-Fi / BLE
        |
        +---- runs independently from critical control
```

## Main capabilities

- ESP32-family target detection
- FreeRTOS task architecture
- Dual-core/multicore task allocation when supported
- Task priorities and deterministic scheduling
- Queues, mutexes, semaphores, notifications, and event groups
- ISR-safe encoder acquisition
- IMU sampling
- Differential-drive wheel odometry
- Encoder + IMU sensor fusion architecture
- EKF/Kalman integration guidance
- PID and real-time motor-control scheduling
- I2C, SPI, UART, and CAN/TWAI ownership
- Wi-Fi/BLE isolation from time-critical work
- Watchdog debugging
- Stack, heap, timing, and jitter analysis
- Hardware/runtime verification

## Key principle

**Do not create tasks just because FreeRTOS allows it.**

The skill first determines the required data flow, timing, deadlines, and dependencies. Tasks and core affinity are introduced only when they solve an actual concurrency or real-time requirement.

## Example robotics architecture

```text
Encoder ISR/PCNT ----+
                     |
IMU Task ------------+--> Sensor Data --> Odometry / EKF --> Robot State
                                                        |
                                                        v
                                                 Control Task
                                                        |
                                                        v
                                                   Motor Driver

                                      Telemetry / Wi-Fi / BLE
                                      runs separately
```

The exact task/core split is selected from the actual ESP target and workload. The skill never assumes every ESP32-family chip is dual-core.

## Supported development styles

The skill is designed to reason about projects using:

- Arduino-ESP32
- PlatformIO
- ESP-IDF
- C/C++ FreeRTOS firmware

## Usage

Add `SKILL.md` to the AI agent/skill system you use, then ask it to inspect or build an ESP32 FreeRTOS project.

Example:

```text
Use ESP_RTOS_PROGRAMMER to design the FreeRTOS architecture for my
differential-drive ESP32 robot. I have two motor encoders, an IMU,
PID motor control, wheel odometry, and telemetry.
```

The agent should inspect the exact ESP target and project before choosing task priorities, core affinity, or synchronization.

## ESP32-family warning

The ESP32 family contains both single-core and multicore devices. Core pinning must only be used after the exact target and available cores are verified.

## Relationship to Arduino Programmer

This skill is focused specifically on **ESP32 + FreeRTOS + real-time/multicore architecture**.

For general Arduino firmware engineering, see the Arduino Programmer skill:

https://github.com/Ashwin312007/Arduino_Programmer

## Repository structure

```text
ESP_RTOS_PROGRAMMER/
├── SKILL.md
└── README.md
```

## Author

Ashwin T E

GitHub: https://github.com/Ashwin312007

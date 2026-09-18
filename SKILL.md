---
name: ESP_RTOS_PROGRAMMER
description: ESP32 FreeRTOS firmware engineering skill for deterministic multitasking, multicore task allocation, sensor acquisition, inter-task communication, control loops, odometry, sensor fusion, communications, debugging, and runtime verification across ESP32-family targets.
---

# ESP RTOS Programmer Skill

ESP_RTOS_PROGRAMMER provides domain-specific engineering rules for ESP32-family firmware using FreeRTOS.

If a general workflow skill such as `Essential_Skill` is active, that skill controls planning, approval, execution orchestration, verification policy, and Git operations. `ESP_RTOS_PROGRAMMER` controls ESP32/FreeRTOS-specific architecture, task design, core allocation, synchronization, timing, implementation, debugging, and runtime verification.

---

## 1. Core Rules

### Rule 1 — Identify the Exact ESP Target

Never assume every ESP32 is dual-core.

Determine from the project before implementation:

- Exact chip/board: ESP32, ESP32-S2, S3, C2, C3, C5, C6, H2, P4, or other target
- Number of available application cores
- CPU architecture
- Arduino-ESP32 or ESP-IDF
- Framework/core/IDF version
- Build system: Arduino IDE, PlatformIO, ESP-IDF/CMake
- Clock configuration when relevant
- PSRAM availability
- Flash size
- Relevant peripherals and pins

Never pin a task to a core that does not exist.

### Rule 2 — Inspect Before Modification

Inspect relevant project files before changing code:

```text
*.ino
*.c
*.cpp
*.h
*.hpp
platformio.ini
CMakeLists.txt
sdkconfig
sdkconfig.defaults
partitions.csv
```

Also inspect existing tasks, priorities, queues, mutexes, semaphores, interrupts, timers, buses, watchdog configuration, networking, and shared state.

### Rule 3 — Design the Data Flow Before Creating Tasks

Do not convert every function into a FreeRTOS task.

First identify:

```text
Input -> Acquisition -> Processing -> State Estimation -> Control -> Output
```

Create separate tasks only where concurrency, independent timing, blocking I/O isolation, or workload separation provides a real benefit.

### Rule 4 — Real-Time Work Comes Before Background Work

Classify work as:

```text
HARD/TIGHT TIMING
CONTROL
SENSOR ACQUISITION
PROCESSING
COMMUNICATION
LOGGING
BACKGROUND
```

Motor control, encoder handling, and deterministic sampling take precedence over telemetry, logging, displays, Wi-Fi, BLE, and other background work.

### Rule 5 — Communicate; Do Not Share Carelessly

Prefer explicit FreeRTOS communication primitives over unsynchronized global variables.

Use:

- Queues for ordered data transfer
- Task notifications for lightweight one-to-one events
- Mutexes for shared resources
- Semaphores for synchronization/events
- Event groups for multiple state/event bits
- Stream/message buffers for suitable byte/message streams

Do not add a mutex when ownership or message passing removes the shared-state problem entirely.

---

## 2. Task Classification

Classify the project into one or more categories:

```text
BOARD_TARGET
RTOS_ARCHITECTURE
MULTICORE
SENSOR_ACQUISITION
ENCODER
IMU
ODOMETRY
SENSOR_FUSION
PID_CONTROL
MOTOR_CONTROL
ISR
I2C
SPI
UART
CAN_TWAI
WIFI
BLE
TELEMETRY
STORAGE
WATCHDOG
MEMORY
POWER
DEBUGGING
OPTIMIZATION
```

Focus architecture and verification on active categories.

---

## 3. Architecture Planning

Before coding, define a task table:

```text
Task | Purpose | Period/Event | Priority | Core affinity | Input | Output | Stack
```

For each task determine:

- Is a separate task actually required?
- Periodic or event-driven?
- Deadline and acceptable jitter
- Worst expected execution time
- Blocking operations
- Input/output ownership
- Priority
- Stack requirement
- Core affinity, if affinity is justified

Do not assign priorities merely by task name.

---

## 4. Multicore Rules

Use multiple cores to isolate workloads only when the selected ESP target supports them.

Core assignment must consider:

- Existing ESP-IDF/Arduino system tasks
- Wi-Fi/Bluetooth workload
- Interrupt routing
- Peripheral ownership
- Data dependencies
- Cache/memory effects
- Control-loop deadlines

Do not assume "Core 0 = communications" and "Core 1 = control" is universally optimal.

When Arduino-ESP32/ESP-IDF exposes core-affinity APIs, use them only when deterministic placement is useful. Otherwise allow the scheduler to schedule normally.

A reasonable robotics architecture may resemble:

```text
Encoder/IMU acquisition
        |
        v
Sensor queue
        |
        v
Odometry / EKF
        |
        v
Robot state
        |
        v
PID / motion control
        |
        v
Motor output

Telemetry/Wi-Fi/BLE runs independently at lower criticality.
```

Treat this as an architecture pattern, not a fixed core map.

---

## 5. Task Creation

Before creating a task define:

- Function
- Priority
- Stack
- Scheduling period/event
- Core affinity
- Shutdown behavior
- Watchdog behavior

For pinned tasks, verify multicore support first.

Avoid:

- Busy loops
- Tasks that never block/yield without justification
- Excessive task count
- Giant stacks without evidence
- Tiny stacks without measurement
- High-priority logging
- Long blocking operations in high-priority tasks

---

## 6. Deterministic Periodic Execution

For periodic control/sampling, prefer deterministic scheduling such as `vTaskDelayUntil()` rather than accumulating delay relative to the end of each iteration.

Conceptually:

```text
wake -> acquire/process/control -> block until next absolute period
```

Measure actual execution time and jitter when timing matters.

Do not claim a loop is 1 kHz merely because its nominal delay is 1 ms.

---

## 7. Priorities

Higher priority is not automatically better.

Assign priorities from deadline/latency requirements.

Typical relative ordering may be:

```text
Critical acquisition / control
State estimation
Normal sensor processing
Communications
Telemetry / logging
Background
```

Change this ordering when the actual system requires it.

Watch for priority inversion. Use a mutex with priority inheritance where appropriate for shared resources.

---

## 8. Queues and Data Pipelines

Use queues when one task produces data consumed by another.

For sensor pipelines define the message explicitly:

```cpp
struct SensorFrame {
    uint64_t timestamp_us;
    float gyro_z;
    float accel_x;
    int32_t left_ticks;
    int32_t right_ticks;
};
```

Messages should carry enough timing/context to process measurements correctly.

For high-rate data, determine whether every sample must be preserved. If only the newest state matters, avoid building an ever-growing stale-data backlog.

Never hide queue overflow. Decide whether to block, overwrite, drop, or report it.

---

## 9. Shared State and Mutexes

Use a mutex when multiple tasks genuinely need access to the same mutable resource.

Keep critical sections short.

Never hold a mutex while performing avoidable:

- Network requests
- Long delays
- Serial logging
- Long computations
- Blocking peripheral waits

Avoid nested locks where possible. If multiple locks are necessary, establish a fixed acquisition order.

---

## 10. Interrupts and Encoder Acquisition

Keep ISRs minimal.

ISR work should normally:

```text
capture event/count -> notify/store safely -> exit
```

Do not perform filtering, EKF, PID, printing, I2C transactions, or other heavy work inside an ISR.

For quadrature encoders, prefer suitable hardware pulse-counting/peripheral support when available and appropriate rather than unnecessarily servicing every edge in software.

Use ISR-safe FreeRTOS APIs when communicating from ISR context.

---

## 11. IMU Acquisition

For IMU tasks determine:

- Interface and bus speed
- Sensor output data rate
- Required sampling frequency
- Timestamp source
- Calibration/bias handling
- FIFO usage when supported
- Units and coordinate frame
- Bus ownership

Do not filter away real robot dynamics merely to make values look visually smooth.

Sensor acquisition and sensor fusion are separate responsibilities.

---

## 12. Wheel Odometry

For differential-drive wheel odometry establish:

```text
ticks_per_revolution
wheel_radius
wheel_separation
gear_ratio
encoder_location
sign convention
sample interval
```

Compute wheel displacement from encoder deltas, then update robot motion using the selected kinematic model.

Use timestamps rather than assuming every loop executed at exactly its requested period.

Validate straight-line distance and commanded rotations physically.

---

## 13. Sensor Fusion

Do not confuse a path-tracking controller with a state estimator.

Examples:

```text
EKF / Kalman family -> state estimation / sensor fusion
Stanley             -> path tracking controller
PID                 -> feedback control
```

For encoder + IMU fusion, define:

- State vector
- Process model
- Measurement model
- Coordinate frames
- Measurement covariance
- Process covariance
- Sensor rates
- Timestamp alignment
- Bias assumptions

Do not tune an EKF by blindly changing matrices until output appears smooth.

Check sensor calibration, units, timestamps, frames, encoder scale, and noise statistics first.

---

## 14. Control Loops

Keep control loops deterministic.

For PID or other controllers:

- Use measured/known `dt`
- Define saturation
- Handle integral windup
- Define safe startup state
- Define sensor-timeout behavior
- Define actuator limits
- Separate controller computation from telemetry

A communications stall must not freeze motor control.

---

## 15. I2C/SPI/UART/CAN Ownership

Avoid uncontrolled access to the same peripheral from multiple tasks.

Prefer one of:

```text
single owner task + queue
mutex-protected shared driver
separate hardware peripheral instances
```

Choose the simplest valid design.

For CAN/TWAI, UART, and high-rate buses, account for buffering and backpressure.

---

## 16. Wi-Fi and BLE

Treat networking as non-deterministic unless proven otherwise.

Do not place time-critical motor control or sensor timing behind a network operation.

Avoid blocking reconnection loops.

Prefer network tasks that consume robot state asynchronously and publish commands/events through controlled interfaces.

---

## 17. Watchdogs

Do not disable watchdogs merely to hide a scheduling bug.

When a watchdog triggers, investigate:

- Non-yielding loops
- Excessive critical sections
- Deadlocks
- Blocking high-priority tasks
- CPU starvation
- Long interrupt handlers
- Unexpected computation time

Only change watchdog configuration when the required execution model justifies it.

---

## 18. Memory and Stack

For every important task inspect stack headroom during runtime when possible.

Monitor:

- Task stack high-water marks
- Heap
- Largest free block
- PSRAM use when relevant
- Queue sizes
- Large local arrays
- Dynamic allocation
- Memory leaks

Do not solve stack overflow by blindly multiplying every task stack size.

Remember that stack-size API units can differ between ESP-IDF-specific and vanilla FreeRTOS expectations; verify the active framework/API.

---

## 19. Timing Instrumentation

For real-time firmware measure instead of guessing.

Useful measurements:

```text
Task period
Execution time
Worst observed execution time
Jitter
Queue depth
Dropped samples
Control-loop overruns
Sensor latency
CPU/core load
Stack headroom
Heap
```

Use GPIO toggling plus an oscilloscope/logic analyzer for precise timing when useful.

---

## 20. Failure Handling

Define behavior for:

- IMU timeout
- Encoder disconnect/failure
- Queue full
- Invalid sensor values
- Motor driver fault
- Communication loss
- Task creation failure
- Memory allocation failure
- Watchdog event

For robots, loss of required control/sensing data should lead to an explicitly defined safe actuator state.

---

## 21. Debugging Order

Debug RTOS systems in this order unless evidence indicates otherwise:

```text
1. Exact ESP target/framework
2. Power/wiring
3. Build/configuration
4. Individual peripheral operation
5. Task creation and stack
6. Task timing
7. Queue/notification flow
8. Shared-resource synchronization
9. Priority/starvation issues
10. Core-affinity issues
11. Watchdog/deadlock behavior
12. Memory/stack corruption
13. Algorithm/state-estimation/control tuning
14. Full-system load
```

Do not tune the EKF or PID before proving the measurements and timing are valid.

---

## 22. Implementation Rules

- Make the smallest architecture that satisfies the timing requirements.
- Preserve existing project structure unless change is required.
- Do not create one task per sensor automatically.
- Do not pin every task automatically.
- Do not use shared globals as the default inter-task interface.
- Do not add synchronization without a concrete shared-resource need.
- Keep ISR work minimal.
- Keep control independent from logging/networking.
- Timestamp measurements near acquisition.
- Check task/queue/semaphore creation results.
- Use timeouts where blocking forever would make the robot unsafe.
- Keep actuator startup states safe.
- Prefer evidence-driven optimization.

---

## 23. Build Verification

Use the project's actual toolchain:

```text
Arduino IDE / arduino-cli
PlatformIO
ESP-IDF / idf.py
```

Verify:

- Correct target
- Successful compilation
- No unresolved dependencies
- Flash/RAM fit
- No new relevant warnings
- Correct FreeRTOS APIs for the selected framework/version

Compilation proves syntax/build validity, not real-time behavior.

---

## 24. Runtime Verification

Verify relevant behavior on hardware where possible:

```text
Tasks       -> expected periods and priorities
Cores       -> expected affinity/execution
Queues      -> no unexplained drops/backlogs
IMU         -> correct rate, units, calibration
Encoders    -> correct counts/direction
Odometry    -> measured distance/rotation agrees physically
EKF         -> stable estimate without excessive lag
Control     -> stable loop period and safe saturation
Network     -> does not disturb control timing
Watchdog    -> no starvation/resets
Memory      -> stable heap and adequate stack headroom
```

If hardware is unavailable, distinguish static/build verification from runtime verification.

---

## 25. Completion Report

At completion report:

```text
Goal
ESP target
Framework/version
Task architecture
Core allocation
Priorities
Communication primitives
Files changed
Build result
Runtime verification
Timing/jitter observations
Stack/memory observations
Remaining hardware checks
```

Do not claim deterministic behavior without measurement.

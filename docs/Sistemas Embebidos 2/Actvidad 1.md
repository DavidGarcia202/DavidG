# Session 2 - Class Activity 01

## 1) Exercise Goals

[x]  Train you to identify logical FreeRTOS tasks from system behavior, even when no RTOS code is shown.

## 2) Materials & Setup

- **Tools/Software** - Editors: VS Code, Python 3.12

## 3) Procedure 

### Exercise  1

| Task Name                 | Trigger (Time / Event) | Periodic or Event-Based |
|---------------------------|------------------------|-------------------------|
| Temperature Read Task     | Time (every 50 ms)     | Periodic                |
| Wi-Fi Send Task           | Time (every 2 s)       | Periodic                |
| Emergency Button Monitor  | Event (button press)   | Event-Based             |
| Status LED Blink Task     | Time (1 Hz / 1 s)      | Periodic                |
| Error Logging Task        | Event (error detected) | Event-Based             |

### Exercise  2

| Task Name                | Time-Critical | Can Block Safely | If Delayed… |
|--------------------------|---------------|------------------|-------------|
| Temperature Read Task    | Yes, the temperature should be measured at precise intervals to have a correct statistic            | Yes, it could be blocked, but it will lose the sampling period             | Sensor data becomes outdated; control decisions may be inaccurate. |
| Wi-Fi Send Task          | No, the data will arrive at some point            | Yes, it is not critical communication             | Data transmission is delayed; buffered data may accumulate. |
| Emergency Button Monitor | Yes, immediate action is required          | No, because blocking may delay the emergency handling               | Emergency event may not be detected in time, causing safety risk. |
| Status LED Blink Task    | No, because the logging is not timme  sensitive           | Yes, it can be surely blocked              | LED blink becomes irregular; no functional impact. |
| Error Logging Task       | No, because it is not time sensitive            | Yes, because it can be deffered              | Error information may be lost or recorded late. |

### Exercise  3

| Task Name                | Priority (H/M/L) | Justification |
|--------------------------|------------------|---------------|
| Emergency Button Monitor | High             | Requires immediate response to ensure system and user safety. |
| Temperature Read Task    | Medium           | Must meet a fixed sampling period but tolerates small timing jitter. |
| Wi-Fi Send Task          | Low              | Communication latency is acceptable and does not affect real-time behavior. |
| Status LED Blink Task    | Low              | Purely informative task with no impact on system functionality. |
| Error Logging Task       | Low              | Can be deferred without affecting real-time system operation. |

### Exercise  4

Which of the following should NOT necessarily be implemented as a FreeRTOS task?

- Emergency button monitoring
- Wi-Fi transmission
- Error logging
- Status LED blinking
- Explain why in 2–3 sentences.

Emergency button, no need to create a task there is no way to scale.

### Exercise 5 — Identifying Hidden Tasks in Pseudo-Code
Task 5.1 — Identify Hidden Tasks
| Hidden Task                | Trigger (Time / Event)        | Why it should be a Task |
|----------------------------|-------------------------------|--------------------------|
| Temperature Sampling       | Time (every loop / ~2 ms)    | Requires periodic measurements for it to have he correct statistics. |
| Emergency Button Monitoring| Event (button press)          | Safety-critical and requires immediate response. |
| Wi-Fi Data Transmission    | Time (every 2 seconds)        | The system will be fully resposive|
| Status LED Blinking        | Time (1 Hz)                   | Periodic behavior independent of main control flow. |

Task 5.2 — Blocking Analysis

- The function send_data_over_wifi() can block the CPU for 100–300 ms.

- While it blocks, button monitoring, temperature sampling, and LED timing are delayed.

- The emergency button monitoring task is most at risk because delayed response can cause a safety failure.

Task 5.3 — RTOS Refactoring Thought Experiment
- Temperature sampling and Wi-Fi transmission because the temperature sensor requires a periodic timing for it to operates properly and the wi-fi transmission is blocking other tasks to execute and is slower.
- Emergency button handling should be triggered by an interrupt to guarantee immediate response.
- The emergency button task should have the highest priority because it is safety-critical.
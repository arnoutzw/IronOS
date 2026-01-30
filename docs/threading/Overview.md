# Threading Overview

IronOS uses FreeRTOS to manage concurrent operations through a multi-threaded architecture. This document provides an overview of the threading model.

---

## Overview

The firmware uses four main threads (tasks) to handle different responsibilities:

| Thread | Priority | Purpose |
|--------|----------|---------|
| PID Task | Highest | Temperature control |
| GUI Task | Normal | User interface |
| MOV Task | Low | Motion detection |
| POW Task | Low | Power management |

---

## Thread Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         FreeRTOS Kernel                         │
└─────────────────────────────────────────────────────────────────┘
        │              │              │              │
        v              v              v              v
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  PID Task   │ │  GUI Task   │ │  MOV Task   │ │  POW Task   │
│  (Highest)  │ │  (Normal)   │ │   (Low)     │ │   (Low)     │
├─────────────┤ ├─────────────┤ ├─────────────┤ ├─────────────┤
│ • Temp ADC  │ │ • Display   │ │ • Accel I2C │ │ • PD Negot  │
│ • PID calc  │ │ • Buttons   │ │ • Orientation│ │ • QC Negot  │
│ • PWM out   │ │ • Menus     │ │ • Movement  │ │ • Voltage   │
│ • Safety    │ │ • State     │ │ • Sleep     │ │ • Power src │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

---

## Thread Communication

### Shared Variables

Threads communicate through shared variables with appropriate protection:

```c
// Temperature target (volatile for thread-safe access)
extern volatile TemperatureType_t currentTempTargetDegC;

// Movement detection timestamp
extern TickType_t lastMovementTime;

// Power supply information
extern int32_t powerSupplyWattageLimit;
```

### Task Notifications

FreeRTOS task notifications are used for efficient signaling:

```c
// PID task notification handle
extern TaskHandle_t pidTaskNotification;

// Notify PID task
xTaskNotifyGive(pidTaskNotification);
```

### Semaphores

I2C and other shared resources use semaphores:

```c
// I2C bus access
static SemaphoreHandle_t I2CSemaphore;

// Usage
if (xSemaphoreTake(I2CSemaphore, TICKS_100MS)) {
    // Access I2C
    xSemaphoreGive(I2CSemaphore);
}
```

---

## Thread Priorities

### Priority Levels

| Priority | Thread | Rationale |
|----------|--------|-----------|
| osPriorityRealtime | PID Task | Safety-critical temperature control |
| osPriorityNormal | GUI Task | User responsiveness |
| osPriorityLow | MOV Task | Background monitoring |
| osPriorityLow | POW Task | Non-critical negotiations |

### Priority Inversion Protection

FreeRTOS provides priority inheritance for mutexes to prevent priority inversion.

---

## Stack Allocation

Each thread has dedicated stack space:

| Thread | Stack Size | Notes |
|--------|------------|-------|
| PID Task | 512 words | Minimal for speed |
| GUI Task | 1024 words | Largest (rendering) |
| MOV Task | 256 words | Simple operations |
| POW Task | 512 words | PD protocol buffers |

All stacks are statically allocated to avoid heap fragmentation:

```c
static StaticTask_t pidTaskBuffer;
static StackType_t pidTaskStack[512];
```

---

## Timing and Scheduling

### PID Task Timing

The PID task runs at a fixed rate synchronized with ADC sampling:

```
ADC Complete IRQ ──> Notify PID Task ──> PID Calculation ──> PWM Update
                          │
                          └── ~10ms cycle time
```

### GUI Task Timing

The GUI task runs in a loop with adaptive timing:

```c
void guiTask() {
    while (1) {
        // Process one frame
        updateDisplay();

        // Adaptive delay (targeting ~60fps)
        osDelay(TICKS_10MS);
    }
}
```

### Background Tasks

MOV and POW tasks run at lower rates:

- MOV Task: ~100ms between accelerometer polls
- POW Task: ~10-100ms based on negotiation state

---

## Startup Sequence

```
main()
│
├── preRToSInit()         // Hardware init (no RTOS)
│
├── BSPInit()             // Board-specific init
│
├── osKernelInitialize()  // RTOS kernel init
│
├── Create Tasks:
│   ├── osThreadNew(startPIDTask, ...)
│   ├── osThreadNew(startGUITask, ...)
│   ├── osThreadNew(startMOVTask, ...)
│   └── osThreadNew(startPOWTask, ...)
│
├── osKernelStart()       // Start scheduler
│   │
│   ├── PID Task starts first (highest priority)
│   ├── GUI Task starts
│   ├── MOV Task starts
│   └── POW Task starts
│
└── (never returns)
```

---

## RTOS Configuration

Key FreeRTOS configuration options:

```c
// FreeRTOSConfig.h
#define configUSE_PREEMPTION            1
#define configUSE_TICKLESS_IDLE         0
#define configCPU_CLOCK_HZ              (SystemCoreClock)
#define configTICK_RATE_HZ              1000
#define configMAX_PRIORITIES            7
#define configMINIMAL_STACK_SIZE        128
#define configTOTAL_HEAP_SIZE           0  // Static allocation only
#define configUSE_MUTEXES               1
#define configUSE_COUNTING_SEMAPHORES   1
#define configUSE_TASK_NOTIFICATIONS    1
```

---

## Critical Sections

Critical sections disable interrupts to protect shared data:

```c
// Enter critical section
taskENTER_CRITICAL();

// Protected operations
sharedVariable = newValue;

// Exit critical section
taskEXIT_CRITICAL();
```

Use sparingly as they increase interrupt latency.

---

## Watchdog Integration

The hardware watchdog is reset from the PID task:

```c
void pidTask() {
    while (1) {
        // Wait for ADC notification
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        // Reset watchdog
        resetWatchdog();

        // PID control...
    }
}
```

If PID task hangs, watchdog triggers system reset.

---

## Debugging

### Stack Overflow Detection

FreeRTOS can detect stack overflows:

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    // Handle overflow - typically reboot
    reboot();
}
```

### Task Statistics

Runtime statistics can be enabled for debugging:

```c
#define configGENERATE_RUN_TIME_STATS   1
#define configUSE_STATS_FORMATTING_FUNCTIONS 1
```

---

## Thread-Specific Documentation

- [GUI Thread](GUIThread.md) - User interface rendering
- [PID Thread](PIDThread.md) - Temperature control
- [MOV Thread](MOVThread.md) - Movement detection
- [POW Thread](POWThread.md) - Power management

---

## See Also

- [Main API](../core/Main.md) - Thread entry points
- [OLED Driver](../drivers/OLED.md) - Display rendering
- [Settings API](../core/Settings.md) - Shared configuration

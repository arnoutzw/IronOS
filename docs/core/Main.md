# Main API

The Main module provides the application entry point, thread declarations, and global state management for IronOS.

**Header File:** `source/Core/Inc/main.hpp`
**Implementation:** `source/Core/Src/main.cpp`

---

## Overview

The main module is responsible for:
- Initializing hardware and FreeRTOS
- Declaring and starting system threads
- Managing global state variables
- Accelerometer type detection

---

## Global Variables

### currentTempTargetDegC

```c
extern volatile TemperatureType_t currentTempTargetDegC;
```

The current temperature target in degrees Celsius. This is the setpoint that the PID controller is trying to achieve.

**Type:** `TemperatureType_t` (int32_t)

**Notes:**
- Marked `volatile` for thread-safe access
- Set by the UI based on current mode
- Read by the PID thread for control

---

### settingsWereReset

```c
extern bool settingsWereReset;
```

Flag indicating whether settings were reset to defaults during startup.

**Values:**
- `true` - Settings were reset (version mismatch or corruption)
- `false` - Settings loaded successfully from flash

**Usage:**
- Used to display a notification to the user on first boot after firmware update

---

### usb_pd_available

```c
extern bool usb_pd_available;
```

Flag indicating whether USB Power Delivery is available and operational.

**Values:**
- `true` - USB-PD hardware detected and initialized
- `false` - No USB-PD available (using QC or DC)

---

### pidTaskNotification

```c
extern TaskHandle_t pidTaskNotification;
```

FreeRTOS task handle for the PID control thread. Used for inter-task notifications.

**Usage:**
```c
// Notify PID task of new temperature reading
xTaskNotifyGive(pidTaskNotification);
```

---

### powerSupplyWattageLimit

```c
extern int32_t powerSupplyWattageLimit;
```

The maximum power available from the current power source in tenths of watts (x10).

**Example:**
```c
// powerSupplyWattageLimit = 650 means 65.0W available
if (requestedPower > powerSupplyWattageLimit) {
    requestedPower = powerSupplyWattageLimit;
}
```

---

### accelInit

```c
extern uint8_t accelInit;
```

Accelerometer initialization status.

**Values:**
- `0` - Not initialized / scanning
- `1` - Initialized successfully
- Other values may indicate specific initialization states

---

### lastMovementTime

```c
extern TickType_t lastMovementTime;
```

FreeRTOS tick count when movement was last detected. Used for:
- Sleep timeout calculations
- Shutdown timeout calculations
- Activity detection

**Usage:**
```c
TickType_t idleTime = xTaskGetTickCount() - lastMovementTime;
if (idleTime > SLEEP_TIMEOUT_TICKS) {
    enterSleepMode();
}
```

---

### DetectedAccelerometerVersion

```c
extern AccelType DetectedAccelerometerVersion;
```

The type of accelerometer detected during hardware scanning.

**Type:** `AccelType` enum

---

## Enumerations

### AccelType

```c
enum class AccelType {
  Scanning  = 0,   // Currently scanning for accelerometer
  None      = 1,   // No accelerometer found
  MMA       = 2,   // MMA8652FC accelerometer
  LIS       = 3,   // LIS2DH12 accelerometer
  BMA       = 4,   // BMA223 accelerometer
  MSA       = 5,   // MSA301 accelerometer
  SC7       = 6,   // SC7A20 accelerometer
  GPIO      = 7,   // GPIO-based orientation (no accel)
  LIS_CLONE = 8    // LIS2DH12 clone variant
};
```

Identifies the type of motion sensor present in the device:

| Value | Accelerometer | I2C Address | Notes |
|-------|---------------|-------------|-------|
| `MMA` | Freescale MMA8652FC | 0x1D | Miniware original |
| `LIS` | STMicro LIS2DH12 | 0x19 | Common replacement |
| `BMA` | Bosch BMA223 | 0x18 | Pinecil V1 |
| `MSA` | MSA301 | 0x26 | Alternative |
| `SC7` | SC7A20 | 0x18/0x19 | Clone/imitation |
| `GPIO` | N/A | N/A | Button-based orientation |

---

## Thread Functions

### startGUITask

```c
void startGUITask(void const *argument);
```

Entry point for the GUI thread. Handles all user interface operations.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Responsibilities:**
- OLED display updates
- Button input processing
- Operating mode state machine
- Settings menu navigation
- Screen rendering

**Priority:** Normal (lower than PID)

---

### startPIDTask

```c
void startPIDTask(void const *argument);
```

Entry point for the PID temperature control thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Responsibilities:**
- Temperature sampling
- PID algorithm execution
- PWM output control
- Thermal safety checks
- Power management

**Priority:** High (highest priority task)

---

### startMOVTask

```c
void startMOVTask(void const *argument);
```

Entry point for the movement detection thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Responsibilities:**
- Accelerometer polling
- Movement detection
- Orientation updates
- Activity tracking
- Sleep/wake decisions

**Priority:** Low

---

### startPOWTask

```c
void startPOWTask(void const *argument);
```

Entry point for the power management thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Responsibilities:**
- Power source monitoring
- QC/PD negotiation
- Voltage monitoring
- Battery cell detection

**Priority:** Low

---

## FreeRTOS Hooks

### vApplicationStackOverflowHook

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName);
```

Called by FreeRTOS when a stack overflow is detected.

**Parameters:**
- `xTask` - Handle of the overflowing task
- `pcTaskName` - Name of the overflowing task

**Behavior:**
- Typically triggers a reboot or enters error state
- Should not return

---

## System Initialization Flow

```
main()
├── preRToSInit()           // Hardware initialization
├── BSPInit()               // Board-specific setup
├── loadSettings()          // Load from flash
├── OLED::initialize()      // Display init
├── osKernelInitialize()    // FreeRTOS init
├── Create threads:
│   ├── startGUITask
│   ├── startPIDTask
│   ├── startMOVTask
│   └── startPOWTask
├── osKernelStart()         // Start scheduler
└── (never returns)
```

---

## Thread Interaction Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  GUI Task   │     │  PID Task   │     │  MOV Task   │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ - Display   │     │ - Temp Ctrl │     │ - Accel     │
│ - Buttons   │────>│ - PWM       │<────│ - Orient    │
│ - Menus     │     │ - Safety    │     │ - Movement  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   ^                   │
       │                   │                   │
       v                   │                   v
┌─────────────┐     ┌──────┴───────┐    ┌─────────────┐
│  Settings   │     │ Temp Target  │    │ Last Move   │
│  (shared)   │     │ (volatile)   │    │ Time        │
└─────────────┘     └──────────────┘    └─────────────┘
```

---

## Memory Layout

### Stack Sizes

| Thread | Stack Size | Notes |
|--------|------------|-------|
| GUI Task | 1024 words | Largest due to rendering |
| PID Task | 512 words | Minimal for speed |
| MOV Task | 256 words | Simple polling |
| POW Task | 512 words | PD negotiation |

---

## Example Usage

### Checking Accelerometer

```c
if (DetectedAccelerometerVersion == AccelType::None) {
    // Show warning that motion detection is unavailable
    showNoAccelerometerWarning();
} else {
    // Motion-based features available
    enableAutoOrientation();
}
```

### Updating Temperature Target

```c
// From GUI thread
currentTempTargetDegC = 350;  // Set to 350°C

// PID task will automatically track this target
```

---

## See Also

- [Threading Overview](../threading/Overview.md) - Thread architecture
- [GUI Thread](../threading/GUIThread.md) - User interface details
- [PID Thread](../threading/PIDThread.md) - Temperature control
- [Accelerometers](../drivers/Accelerometers.md) - Motion sensor drivers

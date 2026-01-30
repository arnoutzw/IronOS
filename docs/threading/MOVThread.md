# MOV Thread

The MOV (Movement) thread handles motion detection, screen orientation, and activity tracking for sleep/wake functionality.

**Entry Point:** `startMOVTask()` in `source/Core/Inc/main.hpp`
**Implementation:** `source/Core/Threads/MOVThread.cpp`

---

## Overview

The MOV thread is responsible for:
- Accelerometer polling and motion detection
- Automatic screen orientation updates
- Activity tracking for sleep timeout
- Hall effect sensor monitoring (if equipped)
- Power source checks

---

## Thread Function

### startMOVTask

```c
void startMOVTask(void const *argument);
```

Entry point for the movement detection thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Main Loop:**
```c
void startMOVTask(void const *argument) {
    // Detect and initialize accelerometer
    detectAccelerometer();

    while (1) {
        // Poll accelerometer for motion
        if (checkForMotion()) {
            lastMovementTime = xTaskGetTickCount();
        }

        // Update screen orientation
        updateOrientation();

        // Check hall effect sensor
        if (getHallSensorFitted()) {
            checkHallSensor();
        }

        // Periodic power checks
        power_check();

        // Sleep between polls
        osDelay(TICKS_100MS);
    }
}
```

---

## Accelerometer Detection

On startup, the MOV thread detects which accelerometer is present:

```c
void detectAccelerometer() {
    // Try each accelerometer in order
    if (MMA8652FC::detect()) {
        DetectedAccelerometerVersion = AccelType::MMA;
        MMA8652FC::initalize();
    } else if (LIS2DH12::detect()) {
        DetectedAccelerometerVersion = AccelType::LIS;
        LIS2DH12::initalize();
    } else if (BMA223::detect()) {
        DetectedAccelerometerVersion = AccelType::BMA;
        BMA223::initalize();
    } else if (MSA301::detect()) {
        DetectedAccelerometerVersion = AccelType::MSA;
        MSA301::initalize();
    } else if (SC7A20::detect()) {
        DetectedAccelerometerVersion = AccelType::SC7;
        SC7A20::initalize();
    } else {
        DetectedAccelerometerVersion = AccelType::None;
    }
}
```

---

## Motion Detection

Motion is detected by comparing consecutive accelerometer readings:

```c
bool checkForMotion() {
    static int16_t lastX, lastY, lastZ;
    int16_t x, y, z;

    // Read current axis values
    getActiveAccelerometer().getAxisReadings(x, y, z);

    // Calculate deltas
    int16_t deltaX = abs(x - lastX);
    int16_t deltaY = abs(y - lastY);
    int16_t deltaZ = abs(z - lastZ);

    // Save current readings
    lastX = x; lastY = y; lastZ = z;

    // Get sensitivity threshold from settings
    uint16_t threshold = getMotionThreshold();

    // Motion detected if any axis exceeds threshold
    return (deltaX > threshold) ||
           (deltaY > threshold) ||
           (deltaZ > threshold);
}
```

### Sensitivity Levels

The `Sensitivity` setting controls motion threshold:

| Setting Value | Threshold | Sensitivity |
|---------------|-----------|-------------|
| 0 | Disabled | Motion detection off |
| 1 | High | Low sensitivity |
| 5 | Medium | Medium sensitivity |
| 9 | Low | High sensitivity |

---

## Orientation Detection

Screen orientation is determined by gravity vector:

```c
void updateOrientation() {
    Orientation newOrientation = getActiveAccelerometer().getOrientation();

    // Debounce orientation changes
    static Orientation lastOrientation = ORIENTATION_FLAT;
    static uint8_t sameCount = 0;

    if (newOrientation == lastOrientation) {
        sameCount++;
    } else {
        sameCount = 0;
        lastOrientation = newOrientation;
    }

    // Apply after stable for several readings
    if (sameCount >= ORIENTATION_DEBOUNCE_COUNT) {
        applyOrientation(newOrientation);
    }
}

void applyOrientation(Orientation orient) {
    if (getSettingValue(OrientationMode) == AUTO) {
        switch (orient) {
            case ORIENTATION_RIGHT_HAND:
                OLED::setRotation(false);
                break;
            case ORIENTATION_LEFT_HAND:
                OLED::setRotation(true);
                break;
            case ORIENTATION_FLAT:
                // Keep current orientation
                break;
        }
    }
}
```

---

## Sleep/Wake Logic

### Activity Tracking

```c
extern TickType_t lastMovementTime;

// Updated when motion detected
if (checkForMotion()) {
    lastMovementTime = xTaskGetTickCount();
}
```

### Sleep Timeout Calculation

```c
bool shouldDeviceSleep() {
    // Get sleep timeout from settings (minutes)
    uint16_t sleepMinutes = getSettingValue(SleepTime);
    if (sleepMinutes == 0) return false;  // Disabled

    TickType_t sleepTicks = sleepMinutes * 60 * TICKS_SECOND;
    TickType_t idleTime = xTaskGetTickCount() - lastMovementTime;

    return (idleTime > sleepTicks);
}
```

### Shutdown Timeout

```c
bool shouldDeviceShutdown() {
    uint16_t shutdownMinutes = getSettingValue(ShutdownTime);
    if (shutdownMinutes == 0) return false;

    TickType_t shutdownTicks = shutdownMinutes * 60 * TICKS_SECOND;
    TickType_t idleTime = xTaskGetTickCount() - lastMovementTime;

    return (idleTime > shutdownTicks);
}
```

---

## Hall Effect Sensor

When equipped, the Hall effect sensor enables magnetic stand detection:

```c
void checkHallSensor() {
    int16_t reading = getRawHallEffect();
    uint16_t threshold = lookupHallEffectThreshold();

    static TickType_t magnetDetectTime = 0;

    if (abs(reading) > threshold) {
        // Magnet detected (in stand)
        if (magnetDetectTime == 0) {
            magnetDetectTime = xTaskGetTickCount();
        }

        // Check if in stand long enough
        uint16_t sleepDelay = getSettingValue(HallEffectSleepTime) * 5;
        if (xTaskGetTickCount() - magnetDetectTime > sleepDelay * TICKS_SECOND) {
            triggerHallEffectSleep();
        }
    } else {
        // No magnet
        magnetDetectTime = 0;
        if (isSleepingDueToHall) {
            wakeFromHallSleep();
        }
    }
}
```

---

## Power Checks

The MOV thread also performs periodic power system checks:

```c
void power_check() {
    // Platform-specific power monitoring
    // May include:
    // - Battery voltage monitoring
    // - Power source change detection
    // - Undervoltage warnings
}
```

---

## Thread Timing

### Poll Rate

The MOV thread polls at approximately 10Hz (100ms interval):

```c
osDelay(TICKS_100MS);
```

This provides:
- Responsive motion detection
- Low power consumption
- Stable orientation readings

### I2C Bandwidth

Accelerometer communication uses I2C:
- Read frequency: 10 Hz
- Data size: 6 bytes per read
- I2C overhead: ~100µs per transaction

---

## Thread Priority

MOV is a low-priority background thread:

```c
#define MOV_TASK_PRIORITY  osPriorityLow
```

Rationale:
- Not time-critical
- Should not interrupt PID or GUI
- Can be delayed without safety impact

---

## Global Variables

### lastMovementTime

```c
extern TickType_t lastMovementTime;
```

Updated when motion is detected. Used by:
- GUI thread for sleep timeout display
- Operating modes for sleep decisions

### accelInit

```c
extern uint8_t accelInit;
```

Accelerometer initialization status:
- `0` - Not initialized
- `1` - Initialized successfully

### DetectedAccelerometerVersion

```c
extern AccelType DetectedAccelerometerVersion;
```

Identifies which accelerometer was found.

---

## State Flow

```
┌─────────────────┐
│ Startup         │ Detect accelerometer
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Initialize      │ Configure accelerometer
└────────┬────────┘
         │
         v
┌─────────────────┐ <─────────────────────┐
│ Poll Motion     │                       │
└────────┬────────┘                       │
         │                                │
         v                                │
┌─────────────────┐                       │
│ Update          │                       │
│ Orientation     │                       │
└────────┬────────┘                       │
         │                                │
         v                                │
┌─────────────────┐                       │
│ Check Hall      │                       │
│ Sensor          │                       │
└────────┬────────┘                       │
         │                                │
         v                                │
┌─────────────────┐                       │
│ Power Check     │                       │
└────────┬────────┘                       │
         │                                │
         v                                │
┌─────────────────┐                       │
│ Sleep 100ms     │ ──────────────────────┘
└─────────────────┘
```

---

## No Accelerometer Fallback

If no accelerometer is detected:
- Motion detection disabled
- Sleep timeout uses button activity instead
- Orientation locked to user setting
- Warning shown on first boot

---

## See Also

- [Accelerometers](../drivers/Accelerometers.md) - Driver details
- [Hall Effect Sensor](../drivers/Si7210.md) - Si7210 driver
- [Settings API](../core/Settings.md) - Sensitivity settings
- [Threading Overview](Overview.md) - Thread architecture

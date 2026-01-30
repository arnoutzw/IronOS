# Accelerometer Drivers

IronOS supports multiple accelerometer ICs for motion detection and automatic screen orientation. This document covers all supported accelerometer drivers.

**Header Files:**
- `source/Core/Drivers/LIS2DH12.hpp`
- `source/Core/Drivers/BMA223.hpp`
- `source/Core/Drivers/MMA8652FC.hpp`
- `source/Core/Drivers/MSA301.h`
- `source/Core/Drivers/SC7A20.hpp`

**Common Header:** `source/Core/Drivers/accelerometers_common.h`

---

## Overview

The accelerometer subsystem provides:
- Motion detection for sleep/wake functionality
- Automatic screen orientation based on tilt
- Sensitivity adjustment via settings

All accelerometer drivers share a common interface for seamless hardware abstraction.

---

## Common Types

### Orientation

Defined in the BSP, represents the detected device orientation:

```c
enum Orientation {
  ORIENTATION_RIGHT_HAND = 0,  // Right-hand grip (normal)
  ORIENTATION_LEFT_HAND  = 1,  // Left-hand grip (180° rotation)
  ORIENTATION_FLAT       = 2   // Laying flat (no preference)
};
```

---

## Common Interface

All accelerometer classes implement the same static interface:

### detect

```c
static bool detect();
```

Probes the I2C bus to detect if this accelerometer is present.

**Returns:**
- `true` - Accelerometer found at expected I2C address
- `false` - Not detected

**Usage:**
```c
if (LIS2DH12::detect()) {
    DetectedAccelerometerVersion = AccelType::LIS;
}
```

---

### initalize

```c
static bool initalize();
```

Initializes the accelerometer with appropriate settings.

**Returns:**
- `true` - Initialization successful
- `false` - Initialization failed

**Notes:**
- Configures sample rate
- Sets up interrupt for orientation change
- Enables motion detection

---

### getOrientation

```c
static Orientation getOrientation();
```

Reads the current device orientation.

**Returns:**
- `ORIENTATION_RIGHT_HAND` - Normal right-hand position
- `ORIENTATION_LEFT_HAND` - Inverted for left-hand use
- `ORIENTATION_FLAT` - Device is laying flat

**Implementation varies:**
- Some accelerometers use interrupt source registers
- Others compute orientation from raw axis data

---

### getAxisReadings

```c
static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
```

Reads raw axis acceleration values.

**Parameters (output):**
- `x` - X-axis acceleration (signed)
- `y` - Y-axis acceleration (signed)
- `z` - Z-axis acceleration (signed)

**Notes:**
- Values are in accelerometer-specific units
- Used for detailed motion detection and debugging

---

## Supported Accelerometers

### LIS2DH12

**Manufacturer:** STMicroelectronics
**I2C Address:** 0x19
**Type:** `AccelType::LIS`

The LIS2DH12 is a 3-axis MEMS accelerometer commonly used in newer soldering irons.

```c
class LIS2DH12 {
public:
  static bool detect();
  static bool isClone();      // Detects clone/imitation chips
  static bool initalize();
  static Orientation getOrientation();
  static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
};
```

**Special Functions:**

#### isClone

```c
static bool isClone();
```

Detects if the chip is a genuine STMicro LIS2DH12 or a clone variant.

**Returns:**
- `true` - Clone/imitation detected
- `false` - Genuine part

---

### BMA223

**Manufacturer:** Bosch Sensortec
**I2C Address:** 0x18
**Type:** `AccelType::BMA`

The BMA223 is used in Pinecil V1 devices.

```c
class BMA223 {
public:
  static bool detect();
  static bool initalize();
  static Orientation getOrientation();
  static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
};
```

**Orientation Logic:**
```c
static Orientation getOrientation() {
    uint8_t val = I2C_RegisterRead(BMA223_INT_STATUS_3);
    val >>= 4;
    val &= 0b11;
    if (val & 0b10) {
        return ORIENTATION_FLAT;
    } else {
        return static_cast<Orientation>(!val);
    }
}
```

---

### MMA8652FC

**Manufacturer:** NXP/Freescale
**I2C Address:** 0x1D
**Type:** `AccelType::MMA`

The MMA8652FC is the original accelerometer used in Miniware TS100.

```c
class MMA8652FC {
public:
  static bool detect();
  static bool initalize();
  static Orientation getOrientation();
  static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
};
```

---

### MSA301

**Manufacturer:** MEMSensing Microsystems
**I2C Address:** 0x26
**Type:** `AccelType::MSA`

Alternative accelerometer used in some device variants.

```c
class MSA301 {
public:
  static bool detect();
  static bool initalize();
  static Orientation getOrientation();
  static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
};
```

---

### SC7A20

**Manufacturer:** Silan Microelectronics
**I2C Address:** 0x18 or 0x19
**Type:** `AccelType::SC7`

The SC7A20 and its variants are often used as LIS2DH12 replacements.

```c
class SC7A20 {
public:
  static bool detect();
  static bool initalize();
  static Orientation getOrientation();
  static void getAxisReadings(int16_t &x, int16_t &y, int16_t &z);
private:
  static bool isInImitationMode;  // Tracks address variant
};
```

**Notes:**
- Supports multiple I2C addresses
- `isInImitationMode` tracks which address is in use

---

## I2C Bus Configuration

The accelerometer I2C bus is configured per-platform:

```c
// From accelerometers_common.h
#if defined(ACCEL_I2CBB2)
#define ACCEL_I2C_CLASS I2CBB2        // Software I2C bus 2
#elif defined(ACCEL_I2CBB1)
#define ACCEL_I2C_CLASS I2CBB1        // Software I2C bus 1
#else
#define ACCEL_I2C_CLASS FRToSI2C      // Hardware I2C
#endif
```

---

## Detection Sequence

During startup, accelerometers are detected in order:

```c
void detectAccelerometer() {
    if (MMA8652FC::detect()) {
        DetectedAccelerometerVersion = AccelType::MMA;
        MMA8652FC::initalize();
    } else if (LIS2DH12::detect()) {
        if (LIS2DH12::isClone()) {
            DetectedAccelerometerVersion = AccelType::LIS_CLONE;
        } else {
            DetectedAccelerometerVersion = AccelType::LIS;
        }
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

Motion is detected by comparing consecutive readings:

```c
// Simplified motion detection
static int16_t lastX, lastY, lastZ;

bool hasMotion() {
    int16_t x, y, z;
    getAxisReadings(x, y, z);

    int16_t deltaX = abs(x - lastX);
    int16_t deltaY = abs(y - lastY);
    int16_t deltaZ = abs(z - lastZ);

    lastX = x; lastY = y; lastZ = z;

    uint16_t threshold = getSensitivityThreshold();
    return (deltaX > threshold) ||
           (deltaY > threshold) ||
           (deltaZ > threshold);
}
```

---

## Sensitivity Settings

Motion sensitivity is configured via the `Sensitivity` setting:

| Value | Sensitivity | Motion Threshold |
|-------|-------------|------------------|
| 0 | Off | Motion detection disabled |
| 1 | Low | High threshold (less sensitive) |
| 5 | Medium | Moderate threshold |
| 9 | High | Low threshold (very sensitive) |

---

## Power Consumption

Accelerometers are configured for low-power operation:

| Chip | Active Current | Low-Power Mode |
|------|----------------|----------------|
| LIS2DH12 | 11 µA @ 1Hz | 2 µA |
| BMA223 | 14 µA @ 7.81Hz | 3.5 µA |
| MMA8652FC | 24 µA @ 1.56Hz | 6 µA |
| MSA301 | 7 µA @ 1Hz | 2 µA |
| SC7A20 | 10 µA @ 1Hz | 2 µA |

---

## Debugging

Accelerometer data can be viewed in the debug menu:

```
Debug Menu:
  Accel: LIS
  X: 1234
  Y: -567
  Z: 16000
```

---

## Example: Orientation Check

```c
void updateScreenOrientation() {
    Orientation orient = getActiveAccelerometer().getOrientation();

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
```

---

## See Also

- [Main API](../core/Main.md) - AccelType enumeration
- [MOV Thread](../threading/MOVThread.md) - Motion detection thread
- [Settings API](../core/Settings.md) - Sensitivity settings
- [I2C Interfaces](../communication/I2C_Wrapper.md) - I2C communication

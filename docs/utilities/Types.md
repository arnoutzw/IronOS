# Common Types

The Types module defines common type definitions used throughout IronOS.

**Header File:** `source/Core/Inc/Types.h`

---

## Overview

This header provides standardized type definitions for consistent usage across the codebase.

---

## Type Definitions

### TemperatureType_t

```c
typedef int32_t TemperatureType_t;
```

Type used for temperature values throughout the system.

**Characteristics:**
- Signed 32-bit integer
- Can represent negative temperatures (cold junction, errors)
- Large range for safety margin
- Units: degrees (Celsius or Fahrenheit depending on context)

**Usage:**
```c
TemperatureType_t currentTemp = TipThermoModel::getTipInC();
TemperatureType_t targetTemp = 350;

if (currentTemp < targetTemp) {
    // Heat up
}
```

**Common Values:**

| Value | Meaning |
|-------|---------|
| 0-50 | Room temperature range |
| 100-200 | Sleep temperature range |
| 300-450 | Soldering temperature range |
| < 0 | Error or invalid reading |
| > 500 | Likely disconnected tip |

---

## Why These Types?

### int32_t for Temperature

1. **Range**: -2,147,483,648 to 2,147,483,647
   - Far exceeds any practical temperature needs
   - Provides headroom for calculations

2. **Signed**: Allows negative values
   - Cold junction can be below zero
   - Error codes can use negative values
   - Temperature differentials can be negative

3. **Performance**: Native word size on 32-bit MCUs
   - No extension needed for operations
   - Efficient arithmetic

---

## Temperature Conventions

### Celsius vs Fahrenheit

Temperature is stored and calculated in Celsius internally:
- Settings stored in Celsius
- PID calculations in Celsius
- Only converted to Fahrenheit for display

```c
TemperatureType_t tempC = TipThermoModel::getTipInC();

// Convert for display if needed
if (getSettingValue(TemperatureInF)) {
    TemperatureType_t tempF = TipThermoModel::convertCtoF(tempC);
    displayTemp(tempF);
} else {
    displayTemp(tempC);
}
```

### X10 Values

Some values use "X10" convention (tenths):
- `getInputVoltageX10()` returns voltage × 10
- `availableW10()` returns power × 10
- Allows decimal precision with integers

```c
uint16_t voltageX10 = getInputVoltageX10(div, 1);
// voltageX10 = 240 means 24.0V

uint32_t powerX10 = availableW10(1);
// powerX10 = 650 means 65.0W
```

---

## Related Types

### Orientation

Defined in BSP, represents device orientation:

```c
enum Orientation {
    ORIENTATION_RIGHT_HAND = 0,
    ORIENTATION_LEFT_HAND  = 1,
    ORIENTATION_FLAT       = 2
};
```

### AccelType

Defined in main.hpp, identifies accelerometer:

```c
enum class AccelType {
    Scanning  = 0,
    None      = 1,
    MMA       = 2,
    LIS       = 3,
    BMA       = 4,
    MSA       = 5,
    SC7       = 6,
    GPIO      = 7,
    LIS_CLONE = 8,
};
```

### ButtonState

Defined in Buttons.hpp:

```c
enum ButtonState {
    BUTTON_NONE      = 0,
    BUTTON_F_SHORT   = 1,
    BUTTON_B_SHORT   = 2,
    BUTTON_F_LONG    = 4,
    BUTTON_B_LONG    = 8,
    BUTTON_BOTH      = 16,
    BUTTON_BOTH_LONG = 32,
};
```

### OperatingMode

Defined in OperatingModes.h:

```c
enum class OperatingMode {
    StartupLogo       = 10,
    CJCCalibration    = 11,
    // ... etc
};
```

---

## Standard Integer Types

IronOS uses standard C99 integer types:

| Type | Size | Range |
|------|------|-------|
| `int8_t` | 1 byte | -128 to 127 |
| `uint8_t` | 1 byte | 0 to 255 |
| `int16_t` | 2 bytes | -32,768 to 32,767 |
| `uint16_t` | 2 bytes | 0 to 65,535 |
| `int32_t` | 4 bytes | -2B to 2B |
| `uint32_t` | 4 bytes | 0 to 4B |
| `int64_t` | 8 bytes | Very large |
| `uint64_t` | 8 bytes | Very large |

Include `<stdint.h>` for these types.

---

## Boolean Type

```c
#include <stdbool.h>

bool flag = true;
bool result = (temp > threshold);
```

---

## FreeRTOS Types

Common FreeRTOS types used in IronOS:

```c
#include "FreeRTOS.h"

TickType_t ticks;           // Timer ticks
TaskHandle_t taskHandle;     // Task reference
SemaphoreHandle_t sem;       // Semaphore reference
BaseType_t result;           // Return type for RTOS functions
```

---

## See Also

- [Main API](../core/Main.md) - Global variable types
- [Settings API](../core/Settings.md) - Setting value types
- [Tip Thermo Model](../drivers/TipThermoModel.md) - Temperature conversions

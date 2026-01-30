# Power API

The Power module handles power calculations, PWM control for the soldering tip heater, and power history tracking for PID control.

**Header File:** `source/Core/Inc/power.hpp`
**Implementation:** `source/Core/Src/power.cpp`

---

## Overview

The power system is responsible for:
- Calculating available power from the power source
- Converting desired wattage to PWM duty cycle
- Tracking power history for PID integral term
- Managing power limits based on supply capabilities

---

## Constants

### wattHistoryFilter

```c
const uint8_t wattHistoryFilter = 24;
```

Weighting factor for the exponential moving average of watt history. Used by the PID controller's integral term to smooth power delivery.

---

## Global Variables

### x10WattHistory

```c
extern expMovingAverage<uint32_t, wattHistoryFilter> x10WattHistory;
```

Exponential moving average filter tracking recent power consumption in tenths of watts (x10). This history is used by the PID controller to:
- Calculate the integral term
- Provide smooth power transitions
- Prevent overshoot

**Type:** `expMovingAverage<uint32_t, 24>`

---

## Functions

### availableW10

```c
uint32_t availableW10(uint8_t sample);
```

Calculates the available power from the current power source in tenths of watts (x10).

**Parameters:**
- `sample` - If non-zero, triggers a new ADC sample; otherwise uses cached values

**Returns:**
- Available power in tenths of watts (e.g., 650 = 65.0W)

**Notes:**
- Takes into account:
  - Input voltage
  - Tip resistance
  - Power source type (QC, PD, DC)
  - User-configured power limit
  - Power delivery contract limits

**Example:**
```c
uint32_t maxPower = availableW10(1);  // Sample and get max power
// maxPower = 650 means 65.0 watts available
```

---

### setTipX10Watts

```c
void setTipX10Watts(int32_t mw);
```

Sets the power output to the soldering tip in tenths of watts.

**Parameters:**
- `mw` - Desired power in tenths of watts (x10), e.g., 350 = 35.0W

**Notes:**
- Automatically clips to available power
- Converts to appropriate PWM duty cycle
- Updates power history for PID tracking
- Power of 0 turns off the heater

**Example:**
```c
setTipX10Watts(350);  // Request 35.0 watts
setTipX10Watts(0);    // Turn off heater
```

---

### X10WattsToPWM

```c
uint8_t X10WattsToPWM(int32_t milliWatts, uint8_t sample = 0);
```

Converts a power value in tenths of watts to a PWM duty cycle value.

**Parameters:**
- `milliWatts` - Desired power in tenths of watts (x10)
- `sample` - If non-zero, triggers new ADC sampling for voltage

**Returns:**
- PWM duty cycle value (0-255 typically, platform-dependent)

**Notes:**
- Calculation is based on:
  - Current input voltage
  - Tip resistance
  - PWM period configuration

**Formula:**
```
P = V² / R
PWM = (P_desired / P_max) * PWM_max

Where:
  P = Power in watts
  V = Input voltage
  R = Tip resistance
  PWM_max = Maximum PWM value (powerPWM constant)
```

**Example:**
```c
uint8_t pwm = X10WattsToPWM(300, 1);  // Get PWM for 30.0W
setTipPWM(pwm, false);                 // Apply to tip
```

---

## Power Calculation Details

### Thermal Mass Compensation

The tip has a thermal mass of approximately 1690 milliJ/°C. This means:
- 1 Watt × 10 seconds raises temperature ~5.9°C
- The PID integral term uses this for feed-forward control

### Power History and PID

The `x10WattHistory` moving average serves the PID controller:

```c
// In PID control loop:
x10WattHistory.update(currentPowerX10);
int32_t integralTerm = x10WattHistory.average();
```

The weighting factor of 24 provides:
- ~94% weight on new values (24/256)
- Smooth transitions over approximately 10 samples
- Stable integral response near setpoint

---

## Power Sources

The power module works with multiple power sources:

| Source | Typical Power | Notes |
|--------|---------------|-------|
| DC Jack (5V-24V) | 30-100W | Based on input voltage |
| USB QC 3.0 (9V/12V/20V) | 18-45W | Negotiated voltage |
| USB PD (5V-20V PPS) | 18-100W | Power delivery contract |
| USB PD (EPR up to 28V) | Up to 140W | Extended power range |

---

## Integration with PID

The power module is tightly integrated with the PID temperature control:

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ Temperature │───>│ PID          │───>│ Power       │
│ Sensor      │    │ Controller   │    │ Module      │
└─────────────┘    └──────────────┘    └─────────────┘
                          │                   │
                          │                   v
                          │            ┌─────────────┐
                          └───────────>│ PWM Output  │
                                       │ (Tip)       │
                                       └─────────────┘
```

The PID controller:
1. Calculates temperature error
2. Determines required power
3. Calls `setTipX10Watts()` or `X10WattsToPWM()`
4. Power module calculates appropriate PWM

---

## Example Usage

### Basic Power Control

```c
// Get available power
uint32_t available = availableW10(1);

// Set power to 50% of available
setTipX10Watts(available / 2);
```

### Power-Limited Operation

```c
// Check if power limiting is active
uint16_t powerLimit = getSettingValue(PowerLimit);
if (powerLimit > 0) {
    // User has set a power limit
    uint32_t limitX10 = powerLimit * 10;
    uint32_t available = availableW10(0);
    if (available > limitX10) {
        available = limitX10;
    }
}
```

---

## See Also

- [BSP Core](../bsp/BSP.md) - `setTipPWM()` function
- [BSP Power](../bsp/BSP_Power.md) - Tip resistance and thermal mass
- [PID Thread](../threading/PIDThread.md) - Temperature control loop
- [Exponential Moving Average](../utilities/ExpMovingAverage.md) - Filter implementation

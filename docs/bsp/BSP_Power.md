# BSP Power Interface

The BSP Power module provides platform-specific power control and tip characterization functions.

**Header File:** `source/Core/BSP/BSP_Power.h`
**Implementation:** Platform-specific in `source/Core/BSP/<platform>/Power.cpp`

---

## Overview

The Power BSP provides:
- Periodic power system checks
- Tip resistance measurement
- Thermal characteristics
- Maximum temperature limits

---

## Functions

### power_check

```c
void power_check();
```

Periodic power system check callback.

**Notes:**
- Called from the movement handling thread
- Can be used for:
  - Power source monitoring
  - Voltage supervision
  - Battery state updates

**Example implementation:**
```c
void power_check() {
    // Check for power source changes
    if (powerSourceChanged()) {
        recalculatePowerLimits();
    }

    // Monitor for undervoltage
    if (getInputVoltageX10(div, 0) < minVoltage) {
        triggerUndervoltageWarning();
    }
}
```

---

### getTipResistanceX10

```c
uint8_t getTipResistanceX10();
```

Returns the tip resistance in tenths of an ohm.

**Returns:**
- Resistance × 10 (e.g., 80 = 8.0Ω)

**Notes:**
- May be fixed value or measured
- Used for power calculations
- Affects PWM duty cycle calculation

**Tip Resistance Values:**

| Tip Type | Resistance | Return Value |
|----------|------------|--------------|
| T12 Standard | 8.0Ω | 80 |
| T12 Short (Pine64) | 6.2Ω | 62 |
| T12 Low-R (PTS200) | 4.0Ω | 40 |
| TS80 | 4.5Ω | 45 |

---

### getTipThermalMass

```c
uint16_t getTipThermalMass();
```

Returns the tip's thermal mass characteristic.

**Returns:**
- Thermal mass value in milliJoules per degree C

**Notes:**
- Used by PID controller for tuning
- Affects heating response
- Typical value: ~1690 mJ/°C

**Usage:**
```c
// Calculate energy needed to heat tip
uint16_t mass = getTipThermalMass();
int32_t deltaT = targetTemp - currentTemp;
int32_t energyNeeded = mass * deltaT;  // in mJ
```

---

### getTipInertia

```c
uint16_t getTipInertia();
```

Returns the tip's thermal inertia value.

**Returns:**
- Thermal inertia characteristic

**Notes:**
- Related to thermal mass but accounts for lag
- Used for PID derivative term tuning
- Helps prevent overshoot

---

### getCustomTipMaxInC

```c
TemperatureType_t getCustomTipMaxInC();
```

Returns the maximum safe temperature for the current tip type.

**Returns:**
- Maximum temperature in degrees Celsius

**Notes:**
- Prevents damage to tip
- May vary by tip type
- Typical values: 450-500°C

---

## Power Calculation

The tip power is calculated using:

```
P = V² / R

Where:
  P = Power in watts
  V = Applied voltage (from PWM duty cycle)
  R = Tip resistance
```

Example:
```c
uint8_t resistance = getTipResistanceX10();  // e.g., 80 (8.0Ω)
uint16_t voltage = getInputVoltageX10(div, 0);  // e.g., 200 (20.0V)

// Power at 100% duty cycle
// P = (20.0)² / 8.0 = 50W
uint32_t maxPowerW = (voltage * voltage) / (resistance * 10);
```

---

## Thermal Model Integration

The thermal mass and inertia values feed into the PID controller:

```
┌─────────────────┐
│ Temperature     │
│ Setpoint        │
└────────┬────────┘
         │
         v
┌─────────────────┐     ┌──────────────────┐
│ PID Controller  │<────│ Thermal Mass     │
│                 │     │ (getTipThermalMass)
└────────┬────────┘     └──────────────────┘
         │                      ^
         v                      │
┌─────────────────┐     ┌───────┴──────────┐
│ Power Output    │     │ Thermal Inertia  │
│ (setTipPWM)     │     │ (getTipInertia)  │
└─────────────────┘     └──────────────────┘
```

---

## Platform-Specific Implementation

### Miniware (TS100/TS80)

```c
uint8_t getTipResistanceX10() {
    #ifdef MODEL_TS100
    return 80;  // 8.0 ohm T12 tips
    #else
    return 45;  // 4.5 ohm TS80 tips
    #endif
}

uint16_t getTipThermalMass() {
    return 1690;  // mJ/°C
}
```

### Pinecil (with tip type support)

```c
uint8_t getTipResistanceX10() {
    uint8_t userSelected = getUserSelectedTipResistance();
    if (userSelected != 0) {
        return userSelected;  // User override
    }
    return autoDetectedResistance;
}
```

---

## Resistance Detection

Some platforms can auto-detect tip resistance:

```c
// During pre-start checks
uint8_t measureTipResistance() {
    // Apply known voltage
    setTipPWM(testPWM, false);
    delay_ms(10);

    // Measure current (via shunt resistor)
    uint16_t current = measureCurrent();

    // Calculate resistance
    // R = V / I
    return (testVoltage * 10) / current;
}
```

---

## Safety Limits

Power is limited based on tip characteristics:

| Limit | Source | Purpose |
|-------|--------|---------|
| Max Temperature | `getCustomTipMaxInC()` | Prevent tip damage |
| Max Power | Calculated from R | Prevent overcurrent |
| Min Voltage | Settings | Prevent deep discharge |

---

## Example: Power Budget

```c
void calculatePowerBudget() {
    // Get tip characteristics
    uint8_t tipR = getTipResistanceX10();
    uint16_t inputV = getInputVoltageX10(div, 1);

    // Calculate maximum theoretical power
    // P = V²/R, but in our units:
    // P(W) = (V/10)² / (R/10) = V² / (R * 10)
    uint32_t maxPowerW = (inputV * inputV) / (tipR * 10);

    // Apply power limit if set
    uint16_t userLimit = getSettingValue(PowerLimit);
    if (userLimit > 0 && maxPowerW > userLimit) {
        maxPowerW = userLimit;
    }

    // Apply power supply limit
    if (maxPowerW > powerSupplyWattageLimit / 10) {
        maxPowerW = powerSupplyWattageLimit / 10;
    }

    actualPowerLimit = maxPowerW;
}
```

---

## See Also

- [Power API](../core/Power.md) - Power calculations
- [Tip Thermo Model](../drivers/TipThermoModel.md) - Temperature conversion
- [PID Thread](../threading/PIDThread.md) - Temperature control
- [Settings API](../core/Settings.md) - Power limit settings

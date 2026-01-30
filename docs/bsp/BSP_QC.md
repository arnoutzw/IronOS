# BSP Quick Charge Interface

The BSP QC module provides platform-specific GPIO control for Quick Charge 3.0 voltage negotiation.

**Header File:** `source/Core/BSP/BSP_QC.h`
**Implementation:** Platform-specific in `source/Core/BSP/<platform>/QC_GPIO.cpp`

---

## Overview

Quick Charge uses the D+ and D- USB data lines for voltage negotiation. The QC BSP provides:
- D+/D- voltage level control
- Pull-up/pull-down resistor management
- Signal detection

---

## Functions

### QC_Init_GPIO

```c
void QC_Init_GPIO();
```

Initializes the GPIO pins used for QC negotiation.

**Actions:**
- Configures D+ pin as output
- Configures D- pin as output with input capability
- Sets initial voltage levels
- Disables USB data mode

**Notes:**
- Must be called before any QC negotiation
- Disables normal USB communication

---

### QC_DPlusZero_Six

```c
void QC_DPlusZero_Six();
```

Sets the D+ line to 0.6V.

**Implementation:**
- Uses voltage divider or DAC
- Typically: GPIO low with 10kΩ pull-up to 3.3V through resistor divider

**QC Protocol Use:**
- Part of initial handshake
- Voltage decrement signal (with D- at 0.6V)

---

### QC_DNegZero_Six

```c
void QC_DNegZero_Six();
```

Sets the D- line to 0.6V.

**Implementation:**
- Similar to D+ control
- May use separate resistor network

**QC Protocol Use:**
- Part of QC2.0/3.0 mode entry
- Voltage control when combined with D+

---

### QC_DPlusThree_Three

```c
void QC_DPlusThree_Three();
```

Sets the D+ line to 3.3V.

**Implementation:**
- GPIO high (3.3V logic level)
- Or controlled voltage source

**QC Protocol Use:**
- Voltage increment signal (with D- at 0.6V)
- QC3.0 mode entry signal

---

### QC_DNegThree_Three

```c
void QC_DNegThree_Three();
```

Sets the D- line to 3.3V.

**Implementation:**
- GPIO high (3.3V logic level)

**QC Protocol Use:**
- Continuous mode control
- Combined with D+ for specific modes

---

### QC_DM_PullDown

```c
void QC_DM_PullDown();
```

Enables a weak pull-down resistor on D-.

**Purpose:**
- Helps some power banks detect device
- Part of charger detection protocol
- May improve compatibility

---

### QC_DM_No_PullDown

```c
void QC_DM_No_PullDown();
```

Disables the D- pull-down resistor.

**Purpose:**
- Returns to normal QC operation
- Called after initial detection phase

---

### QC_Post_Probe_En

```c
void QC_Post_Probe_En();
```

Enables output drivers after initial probe phase.

**Purpose:**
- Prevents voltage spikes during mode detection
- Called after charger type is determined
- Enables stronger drive on D+/D-

---

### QC_DM_PulledDown

```c
uint8_t QC_DM_PulledDown();
```

Reads the current state of the D- line.

**Returns:**
- `1` - D- is pulled low (grounded)
- `0` - D- is pulled high

**Purpose:**
- Detects charger response
- Verifies QC mode entry

---

### QC_resync

```c
void QC_resync();
```

Re-synchronizes the QC voltage negotiation.

**Actions:**
- Resets the QC state machine
- Sends fresh handshake sequence
- Used after power source changes

**Usage:**
```c
// After detecting voltage instability
QC_resync();
startQC(divisor);  // Re-negotiate
```

---

## QC Voltage Levels

The D+/D- voltage combinations control the output voltage:

### QC 2.0 Class A

| D+ | D- | Output Voltage |
|----|----|----- |
| 0.6V | 0V | 5V (default) |
| 3.3V | 0.6V | 9V |
| 0.6V | 0.6V | 12V |
| 3.3V | 3.3V | 20V |

### QC 3.0 (Continuous Mode)

| D+ | D- | Action |
|----|----|----- |
| 3.3V | 0.6V | Increment (+200mV) |
| 0.6V | 0.6V | Decrement (-200mV) |
| 0.6V | 3.3V | Hold (exit increment) |
| 3.3V | 3.3V | Hold (exit decrement) |

---

## Hardware Circuit

Typical QC control circuit:

```
                     ┌─────────────┐
 GPIO_DP_HIGH ──────>│             │
                     │   Voltage   │──────> D+
 GPIO_DP_LOW ───────>│   Control   │
                     │   Circuit   │
                     └─────────────┘

                     ┌─────────────┐
 GPIO_DM_HIGH ──────>│             │
                     │   Voltage   │──────> D-
 GPIO_DM_LOW ───────>│   Control   │
                     │   Circuit   │
                     └─────────────┘
```

Voltage divider for 0.6V:
```
  3.3V ────┬──[4.7kΩ]──┬── D+/D-
           │           │
      GPIO_LOW    GPIO_HIGH
           │           │
           └───[1.5kΩ]─┴── GND

When GPIO_LOW=High, GPIO_HIGH=Low:
  Output ≈ 3.3V * 1.5k / (4.7k + 1.5k) ≈ 0.8V ≈ 0.6V
```

---

## Timing Requirements

QC protocol has strict timing requirements:

| Phase | Duration | Tolerance |
|-------|----------|-----------|
| Initial connection | 1.25s | +0.25s |
| D+ 0.6V hold | >100ms | - |
| D- 0.6V hold | >100ms | - |
| Voltage step | 200µs min | - |
| Step-to-step | 200µs min | - |

---

## Platform Implementation Example

### STM32F103 (Miniware)

```c
// Pin definitions
#define QC_DP_HIGH_PIN  GPIO_PIN_6
#define QC_DP_LOW_PIN   GPIO_PIN_7
#define QC_DM_HIGH_PIN  GPIO_PIN_8
#define QC_DM_LOW_PIN   GPIO_PIN_9
#define QC_PORT         GPIOB

void QC_DPlusZero_Six() {
    // Set 0.6V: LOW high, HIGH low
    HAL_GPIO_WritePin(QC_PORT, QC_DP_LOW_PIN, GPIO_PIN_SET);
    HAL_GPIO_WritePin(QC_PORT, QC_DP_HIGH_PIN, GPIO_PIN_RESET);
}

void QC_DPlusThree_Three() {
    // Set 3.3V: LOW low, HIGH high
    HAL_GPIO_WritePin(QC_PORT, QC_DP_LOW_PIN, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(QC_PORT, QC_DP_HIGH_PIN, GPIO_PIN_SET);
}
```

---

## Error Handling

Common issues and recovery:

| Issue | Cause | Solution |
|-------|-------|----------|
| No voltage change | Charger not QC | Fall back to 5V |
| Unstable voltage | Poor cable | Add settling delay |
| Charger disconnect | Overcurrent | Reduce power, retry |
| Voltage too low | Wrong mode | Call QC_resync() |

---

## See Also

- [QC3.0 Protocol Driver](../drivers/QC3.md) - Protocol implementation
- [USB Power Delivery](../drivers/USBPD.md) - Alternative negotiation
- [POW Thread](../threading/POWThread.md) - Power management
- [Power API](../core/Power.md) - Power calculations

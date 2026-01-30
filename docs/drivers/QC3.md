# QC3.0 Protocol Driver

The QC3.0 (Quick Charge 3.0) driver handles negotiation with QC-compatible power supplies to obtain higher voltages for improved performance.

**Header File:** `source/Core/Inc/QC3.h`
**Implementation:** `source/Core/Src/QC3.cpp`

---

## Overview

Quick Charge 3.0 allows the soldering iron to:
- Negotiate voltages from 3.6V to 20V
- Operate at higher power than standard USB
- Provide voltage adjustment in 200mV steps

---

## Functions

### startQC

```c
void startQC(uint16_t divisor);
```

Initiates QC3.0 negotiation to obtain the highest supported voltage.

**Parameters:**
- `divisor` - Voltage divider value for ADC calibration

**Behavior:**
1. Sends QC3.0 handshake sequence
2. Requests maximum voltage (typically 20V)
3. Falls back to lower voltages if rejected
4. Sets `QCIdealVoltage` setting value

**Notes:**
- Must be called after USB connection detected
- Should not be called while drawing high current
- Negotiation takes ~1-2 seconds

**Example:**
```c
if (!USBPowerDelivery::negotiationComplete()) {
    startQC(getSettingValue(VoltageDiv));
}
```

---

### seekQC

```c
void seekQC(int16_t Vx10, uint16_t divisor);
```

Adjusts the QC3.0 voltage to a specific target.

**Parameters:**
- `Vx10` - Target voltage in tenths of volts (e.g., 120 = 12.0V)
- `divisor` - Voltage divider value for ADC calibration

**Behavior:**
- Steps voltage up or down in 200mV increments
- Monitors actual voltage during adjustment
- Stops when target is reached (±200mV)

**Example:**
```c
// Request 12V
seekQC(120, getSettingValue(VoltageDiv));

// Request 9V
seekQC(90, getSettingValue(VoltageDiv));
```

---

### hasQCNegotiated

```c
bool hasQCNegotiated();
```

Checks if QC negotiation has successfully completed.

**Returns:**
- `true` - QC negotiation worked, operating above 5V
- `false` - No QC negotiation, at default USB voltage

**Example:**
```c
if (hasQCNegotiated()) {
    // Can operate at higher power
    maxPower = calculateQCPower();
} else {
    // Limited to 5V USB power
    maxPower = 10;  // ~10W at 5V
}
```

---

## QC3.0 Protocol Details

### Handshake Sequence

```
1. D+ = 0.6V, D- = 0V         Initial state
2. Wait 1.25 seconds          Charger detection time
3. D+ = 0.6V, D- = 0.6V       QC2.0 mode request
4. D+ = 3.3V, D- = 0.6V       QC3.0 mode request
5. Charger enters QC3.0 mode
```

### Voltage Control

In QC3.0 mode, voltage is controlled by D+/D- pulses:

| D+ | D- | Action |
|----|----|----- |
| 3.3V | 0.6V | Increment voltage (+200mV) |
| 0.6V | 0.6V | Decrement voltage (-200mV) |
| 0.6V | 3.3V | Continuous mode (20V) |
| 3.3V | 3.3V | Continuous mode (5V) |

---

## Voltage Levels

| Setting | Voltage | Max Power (2A limit) |
|---------|---------|---------------------|
| 5V | 5.0V | 10W |
| 9V | 9.0V | 18W |
| 12V | 12.0V | 24W |
| 20V | 20.0V | 40W |

**Note:** QC3.0 chargers may have different current limits at each voltage.

---

## BSP Interface

The QC driver uses BSP functions for GPIO control:

```c
// From BSP_QC.h
void QC_Init_GPIO();           // Initialize D+/D- pins
void QC_DPlusZero_Six();       // Set D+ to 0.6V
void QC_DNegZero_Six();        // Set D- to 0.6V
void QC_DPlusThree_Three();    // Set D+ to 3.3V
void QC_DNegThree_Three();     // Set D- to 3.3V
void QC_DM_PullDown();         // Enable D- pulldown
void QC_DM_No_PullDown();      // Disable D- pulldown
void QC_Post_Probe_En();       // Enable output drivers
uint8_t QC_DM_PulledDown();    // Check D- state
void QC_resync();              // Re-synchronize QC
```

---

## Negotiation State Machine

```
┌─────────────┐
│   Idle      │
└──────┬──────┘
       │ Start negotiation
       v
┌─────────────┐
│ Set D+/D-   │
│ 0.6V/0V     │
└──────┬──────┘
       │ Wait 1.25s
       v
┌─────────────┐
│ QC2.0 Mode  │
│ 0.6V/0.6V   │
└──────┬──────┘
       │ Wait
       v
┌─────────────┐
│ QC3.0 Mode  │
│ 3.3V/0.6V   │
└──────┬──────┘
       │
       v
┌─────────────────────┐
│ Voltage Adjustment  │<──┐
│ (seek target)       │   │
└──────┬──────────────┘   │
       │                  │
       v                  │
┌─────────────────────┐   │
│ Target reached?     │───┘ No
└──────┬──────────────┘
       │ Yes
       v
┌─────────────┐
│ Negotiated  │
└─────────────┘
```

---

## Power Bank Compatibility

Some power banks require special handling:

```c
void QC_DM_PullDown() {
    // Turn on weak pulldown on D-
    // Helps with power banks that need
    // the pulldown to detect the device
}
```

---

## Settings Integration

The `QCIdealVoltage` setting stores the user's preferred QC voltage:

| Value | Voltage |
|-------|---------|
| 0 | 9V |
| 1 | 12V |
| 2 | 20V |

```c
uint8_t qcSetting = getSettingValue(QCIdealVoltage);
switch (qcSetting) {
    case 0: targetVoltage = 90; break;   // 9V
    case 1: targetVoltage = 120; break;  // 12V
    case 2: targetVoltage = 200; break;  // 20V
}
seekQC(targetVoltage, divisor);
```

---

## Example Usage

### Initialization

```c
void initPowerNegotiation() {
    // Try USB-PD first
    if (USBPowerDelivery::start()) {
        usb_pd_available = true;
        return;
    }

    // Fall back to QC3.0
    startQC(getSettingValue(VoltageDiv));
}
```

### Power Source Detection

```c
int8_t getPowerSourceNumber() {
    if (USBPowerDelivery::negotiationComplete()) {
        return 2;  // USB-PD
    } else if (hasQCNegotiated()) {
        return 1;  // QC3.0
    } else if (getIsPoweredByDCIN()) {
        return 0;  // DC jack
    }
    return -1;  // Unknown/5V USB
}
```

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| No negotiation | Charger not QC compatible | Use different charger |
| Voltage unstable | Poor cable | Use shorter/better cable |
| Falls back to 5V | Current spike | Reduce power during negotiation |
| Power bank disconnect | No keep-alive | Enable power pulse |

---

## See Also

- [BSP QC](../bsp/BSP_QC.md) - GPIO interface for QC
- [USB Power Delivery](USBPD.md) - Alternative power negotiation
- [Power API](../core/Power.md) - Power calculations
- [POW Thread](../threading/POWThread.md) - Power management

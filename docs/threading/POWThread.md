# POW Thread

The POW (Power) thread handles power source management, USB Power Delivery negotiation, and Quick Charge 3.0 negotiation.

**Entry Point:** `startPOWTask()` in `source/Core/Inc/main.hpp`
**Implementation:** `source/Core/Threads/POWThread.cpp`

---

## Overview

The POW thread is responsible for:
- USB Power Delivery (PD) protocol handling
- Quick Charge 3.0 negotiation
- Power source detection and monitoring
- Voltage and power limit management
- Power supply capability tracking

---

## Thread Function

### startPOWTask

```c
void startPOWTask(void const *argument);
```

Entry point for the power management thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Main Loop:**
```c
void startPOWTask(void const *argument) {
    // Initialize power negotiation
    initPowerNegotiation();

    while (1) {
        // Handle USB-PD if available
        if (usb_pd_available) {
            USBPowerDelivery::step();

            // Handle PPS timer
            if (isPPSContract) {
                USBPowerDelivery::PPSTimerCallback();
            }
        }

        // Fallback to QC if no PD
        if (!USBPowerDelivery::negotiationComplete()) {
            handleQCNegotiation();
        }

        // Update power limits
        updatePowerLimits();

        // Thread timing
        osDelay(TICKS_10MS);
    }
}
```

---

## Power Source Priority

Power negotiation follows this priority:

```
┌─────────────────┐
│ Check USB-PD    │ Try PD first (highest power)
└────────┬────────┘
         │
    ┌────┴────┐
    │ PD OK?  │
    └────┬────┘
    Yes  │   No
    │    │    │
    v    │    v
┌───────┐│ ┌─────────────┐
│ Use   ││ │ Try QC 3.0  │
│ PD    ││ └──────┬──────┘
└───────┘│        │
         │   ┌────┴────┐
         │   │ QC OK?  │
         │   └────┬────┘
         │   Yes  │   No
         │   │    │    │
         │   v    │    v
         │ ┌───────┐ ┌─────────┐
         │ │ Use   │ │ Use     │
         │ │ QC    │ │ 5V/DC   │
         │ └───────┘ └─────────┘
         │
         └──> Continue monitoring
```

---

## USB Power Delivery Handling

### PD State Machine

```c
void handlePD() {
    // Step the PD state machine
    USBPowerDelivery::step();

    // Check for completion
    if (USBPowerDelivery::negotiationComplete()) {
        // Successfully negotiated higher voltage
        pdNegotiatedVoltage = getCurrentPDVoltage();
        updatePowerLimits();
    }
}
```

### PPS (Programmable Power Supply)

For USB-PD 3.0 PPS contracts:

```c
void handlePPS() {
    static TickType_t lastPPSRequest = 0;

    // PPS requires periodic re-request (every 10 seconds)
    if (xTaskGetTickCount() - lastPPSRequest > TICKS_SECOND * 8) {
        USBPowerDelivery::PPSTimerCallback();
        lastPPSRequest = xTaskGetTickCount();
    }
}
```

### IRQ Handling

```c
void handlePDIRQ() {
    if (getFUS302IRQLow()) {
        USBPowerDelivery::IRQOccured();
    }
}
```

---

## Quick Charge Handling

### QC Negotiation

```c
void handleQCNegotiation() {
    static bool qcAttempted = false;

    if (!qcAttempted) {
        // Try QC negotiation
        startQC(getSettingValue(VoltageDiv));
        qcAttempted = true;
    }

    // Monitor QC status
    if (hasQCNegotiated()) {
        // QC active, adjust voltage as needed
        uint16_t targetVoltage = getQCTargetVoltage();
        seekQC(targetVoltage, getSettingValue(VoltageDiv));
    }
}

uint16_t getQCTargetVoltage() {
    uint8_t qcSetting = getSettingValue(QCIdealVoltage);
    switch (qcSetting) {
        case 0: return 90;   // 9V
        case 1: return 120;  // 12V
        case 2: return 200;  // 20V
        default: return 90;
    }
}
```

---

## Power Limit Management

### Updating Power Limits

```c
void updatePowerLimits() {
    int32_t maxPower = 0;

    if (USBPowerDelivery::negotiationComplete()) {
        // Get PD contract power
        maxPower = getPDContractPower();
    } else if (hasQCNegotiated()) {
        // Calculate QC power from voltage
        uint16_t voltage = getInputVoltageX10(div, 0);
        // Assume 2A max for QC
        maxPower = (voltage * 20) / 10;  // x10 watts
    } else if (getIsPoweredByDCIN()) {
        // DC input - estimate from voltage
        uint16_t voltage = getInputVoltageX10(div, 0);
        maxPower = estimateDCPower(voltage);
    } else {
        // Default 5V USB
        maxPower = 100;  // 10W
    }

    // Apply user power limit
    uint16_t userLimit = getSettingValue(PowerLimit);
    if (userLimit > 0 && maxPower > userLimit * 10) {
        maxPower = userLimit * 10;
    }

    powerSupplyWattageLimit = maxPower;
}
```

### Power Budget

| Source | Typical Max | Notes |
|--------|-------------|-------|
| 5V USB | 10W | 5V × 2A |
| QC 9V | 18W | 9V × 2A |
| QC 12V | 24W | 12V × 2A |
| QC 20V | 40W | 20V × 2A |
| PD 20V 3A | 60W | Fixed PDO |
| PD 20V 5A | 100W | With E-marker cable |
| PD EPR 28V | 140W | USB-PD 3.1 |

---

## Voltage Monitoring

The POW thread monitors input voltage:

```c
void monitorVoltage() {
    uint16_t voltage = getInputVoltageX10(getSettingValue(VoltageDiv), 1);

    // Check for undervoltage
    uint16_t minVoltage = lookupVoltageLevel();
    if (voltage < minVoltage) {
        triggerUndervoltageWarning();
    }

    // Check for overvoltage (safety)
    if (voltage > MAX_SAFE_VOLTAGE) {
        triggerOvervoltageShutdown();
    }
}
```

---

## Thread Timing

### Poll Rate

The POW thread timing varies based on state:

```c
// During active negotiation
osDelay(TICKS_10MS);  // Fast polling

// After negotiation complete
osDelay(TICKS_100MS);  // Slower monitoring
```

### PD Timing Requirements

| Event | Timeout |
|-------|---------|
| Source capability response | 150ms |
| Request response | 30ms |
| PS_RDY after accept | 500ms |
| PPS keep-alive | 10s |

---

## Thread Priority

POW is a low-priority background thread:

```c
#define POW_TASK_PRIORITY  osPriorityLow
```

Rationale:
- Negotiation is not time-critical after initial setup
- Should not interrupt temperature control
- PD IRQs are handled separately

---

## Global Variables

### powerSupplyWattageLimit

```c
extern int32_t powerSupplyWattageLimit;
```

Maximum available power in tenths of watts. Used by:
- PID controller for power limiting
- GUI for power display

### usb_pd_available

```c
extern bool usb_pd_available;
```

Indicates whether USB-PD is available:
- `true` - FUSB302 detected and initialized
- `false` - No PD capability

---

## Error Handling

### Negotiation Failures

```c
void handleNegotiationFailure() {
    static uint8_t failureCount = 0;

    failureCount++;

    if (failureCount > MAX_RETRY_COUNT) {
        // Give up on PD/QC, use default power
        powerSupplyWattageLimit = DEFAULT_5V_POWER;
        return;
    }

    // Retry with delay
    osDelay(TICKS_SECOND);
    retryNegotiation();
}
```

### Cable Disconnect

```c
void handleCableDisconnect() {
    // Reset negotiation state
    qcAttempted = false;

    // Re-detect power source
    detectPowerSource();
}
```

---

## Power Source Identification

```c
int8_t getPowerSourceNumber() {
    if (USBPowerDelivery::negotiationComplete()) {
        return 2;  // USB-PD
    } else if (hasQCNegotiated()) {
        return 1;  // QC3.0
    } else if (getIsPoweredByDCIN()) {
        return 0;  // DC input
    }
    return -1;  // Unknown/5V USB
}
```

---

## State Flow

```
┌─────────────────┐
│ Startup         │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Detect FUSB302  │
└────────┬────────┘
         │
    ┌────┴────┐
    │ Found?  │
    └────┬────┘
    Yes  │   No
    │    │    │
    v    │    v
┌────────┐│ ┌────────────┐
│ Start  ││ │ Try QC3.0  │
│ PD     ││ │            │
└───┬────┘│ └─────┬──────┘
    │     │       │
    v     │       v
┌─────────┴───────────────┐
│ Monitor & Adjust        │ <──┐
│ Power Limits            │    │
└────────────┬────────────┘    │
             │                 │
             └─────────────────┘
```

---

## See Also

- [USB Power Delivery](../drivers/USBPD.md) - PD protocol details
- [QC3.0 Protocol](../drivers/QC3.md) - QC implementation
- [BSP PD](../bsp/BSP_PD.md) - FUSB302 interface
- [BSP QC](../bsp/BSP_QC.md) - QC GPIO control
- [Power API](../core/Power.md) - Power calculations

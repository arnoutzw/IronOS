# USB Power Delivery Driver

The USB Power Delivery (USB-PD) driver handles negotiation with USB-PD power sources to obtain optimal voltage and power for the soldering iron.

**Header File:** `source/Core/Drivers/USBPD.h`
**Implementation:** `source/Core/Drivers/USBPD.cpp`

---

## Overview

The USB-PD driver provides:
- USB Power Delivery protocol implementation
- FUSB302 controller interface
- Power contract negotiation
- PPS (Programmable Power Supply) support
- EPR (Extended Power Range) support for USB-PD 3.1

---

## Conditional Compilation

The USB-PD driver is only available when `POW_PD` is defined:

```c
#ifdef POW_PD
class USBPowerDelivery {
    // ...
};
#endif
```

---

## Class: USBPowerDelivery

The `USBPowerDelivery` class provides static methods for all power delivery operations.

---

## Functions

### start

```c
static bool start();
```

Starts the USB-PD negotiation stack.

**Returns:**
- `true` - PD stack started successfully
- `false` - Failed to start (hardware not present)

**Notes:**
- Called during system initialization
- Begins scanning for source capabilities

**Example:**
```c
if (USBPowerDelivery::start()) {
    usb_pd_available = true;
}
```

---

### step

```c
static void step();
```

Advances the USB-PD state machine by one iteration.

**Notes:**
- Called periodically from POW thread
- Processes incoming messages
- Handles protocol timeouts
- Manages negotiation flow

**Usage:**
```c
void powerThread() {
    while (1) {
        USBPowerDelivery::step();
        osDelay(TICKS_10MS);
    }
}
```

---

### negotiationComplete

```c
static bool negotiationComplete();
```

Checks if power negotiation has completed with a voltage above 5V.

**Returns:**
- `true` - Successfully negotiated a higher voltage (9V, 12V, 15V, 20V, etc.)
- `false` - Still at 5V or negotiation ongoing

**Example:**
```c
if (USBPowerDelivery::negotiationComplete()) {
    // Can operate at full power
    enableFullPowerMode();
}
```

---

### negotiationInProgress

```c
static bool negotiationInProgress();
```

Checks if negotiation is currently ongoing.

**Returns:**
- `true` - Negotiation in progress
- `false` - Negotiation complete or not started

**Notes:**
- Used to avoid power-intensive operations during negotiation
- UI may show "Negotiating..." status

---

### negotiationHasWorked

```c
static bool negotiationHasWorked();
```

Checks if any USB-PD negotiation has successfully completed.

**Returns:**
- `true` - In a valid PD contract
- `false` - No PD contract established

**Notes:**
- Returns true even if at 5V PD contract
- Different from `negotiationComplete()` which requires >5V

---

### fusbPresent

```c
static bool fusbPresent();
```

Checks if the FUSB302 PD controller IC is present.

**Returns:**
- `true` - FUSB302 detected on I2C bus
- `false` - FUSB302 not found

**Notes:**
- Called during hardware detection
- FUSB302 is the PD PHY/controller IC

---

### isVBUSConnected

```c
static bool isVBUSConnected();
```

Checks if VBUS power is present.

**Returns:**
- `true` - VBUS voltage detected
- `false` - No VBUS connection

**Notes:**
- Used to detect cable connection
- Independent of PD negotiation state

---

### IRQOccured

```c
static void IRQOccured();
```

Callback notification that an IRQ from the FUSB302 has occurred.

**Notes:**
- Called from interrupt context or dedicated handler
- Signals that the PD controller needs attention
- Triggers state machine processing

**Usage:**
```c
void FUSB302_IRQ_Handler() {
    USBPowerDelivery::IRQOccured();
}
```

---

### PPSTimerCallback

```c
static void PPSTimerCallback();
```

Timer callback for PPS (Programmable Power Supply) management.

**Notes:**
- PPS requires periodic re-negotiation (every 10 seconds)
- Maintains PPS contract active
- Updates power parameters if needed

---

### getStateNumber

```c
static uint8_t getStateNumber();
```

Returns the current internal state number for debugging.

**Returns:**
- Current state machine state (0-255)

**Usage:**
- Displayed in PD debug menu
- Helps diagnose negotiation issues

---

### getLastSeenCapabilities

```c
static uint32_t *getLastSeenCapabilities();
```

Returns pointer to the last received source capability PDOs.

**Returns:**
- Pointer to array of PDO (Power Data Object) values
- Up to 7 PDOs as per USB-PD specification

**PDO Format (simplified):**
```
Fixed Supply PDO:
  Bits 31-30: PDO type (00 = Fixed)
  Bits 29-20: Voltage (in 50mV units)
  Bits 9-0:   Max current (in 10mA units)
```

---

## USB-PD Modes

The `usbpdMode_t` setting controls PD behavior:

### DEFAULT (1)

```c
usbpdMode_t::DEFAULT
```

- Enables PPS (Programmable Power Supply)
- Enables EPR (Extended Power Range) if available
- Requests additional power to compensate for cable losses
- Best performance but may stress marginal adapters

### SAFE (2)

```c
usbpdMode_t::SAFE
```

- Enables PPS and EPR
- Does not request power compensation
- Better compatibility with all adapters

### NO_DYNAMIC (0)

```c
usbpdMode_t::NO_DYNAMIC
```

- Uses only fixed PDOs (5V, 9V, 12V, 15V, 20V)
- No PPS or dynamic voltage adjustment
- Maximum compatibility mode

---

## Negotiation Flow

```
┌─────────────┐
│   Start     │
└─────┬───────┘
      │
      v
┌─────────────────┐
│ Wait for VBUS   │
└─────┬───────────┘
      │
      v
┌─────────────────┐
│ Receive Source  │
│ Capabilities    │
└─────┬───────────┘
      │
      v
┌─────────────────┐
│ Select Best PDO │
│ (voltage/power) │
└─────┬───────────┘
      │
      v
┌─────────────────┐
│ Send Request    │
└─────┬───────────┘
      │
      v
┌─────────────────┐     ┌─────────────┐
│ Wait for Accept │────>│   Reject    │
└─────┬───────────┘     └─────────────┘
      │
      v
┌─────────────────┐
│ Wait for PS_RDY │
└─────┬───────────┘
      │
      v
┌─────────────────┐
│ Contract Active │
└─────────────────┘
```

---

## Power Capabilities

| Protocol | Voltage Range | Max Power | Notes |
|----------|---------------|-----------|-------|
| USB-PD 2.0 | 5V-20V | 100W | Fixed voltages |
| USB-PD 3.0 PPS | 3.3V-21V | 100W | Adjustable in 20mV steps |
| USB-PD 3.1 EPR | 5V-48V | 240W | Extended Power Range |

---

## FUSB302 Registers

Key registers used by the driver:

| Register | Address | Description |
|----------|---------|-------------|
| DEVICE_ID | 0x01 | Chip identification |
| SWITCHES0 | 0x02 | Port configuration |
| SWITCHES1 | 0x03 | Auto-response config |
| MEASURE | 0x04 | Voltage measurement |
| CONTROL0 | 0x06 | Host control |
| STATUS0 | 0x40 | Interrupt status |
| STATUS1 | 0x41 | Connection status |
| FIFO | 0x43 | TX/RX FIFO access |

---

## Example: Power Information Display

```c
void displayPDInfo() {
    if (USBPowerDelivery::fusbPresent()) {
        if (USBPowerDelivery::negotiationComplete()) {
            // Show negotiated voltage and power
            OLED::print("PD: ", FontStyle::SMALL);
            OLED::printNumber(negotiatedVoltage / 10, 2, FontStyle::SMALL);
            OLED::print("V", FontStyle::SMALL);
        } else if (USBPowerDelivery::negotiationInProgress()) {
            OLED::print("PD: Neg...", FontStyle::SMALL);
        } else {
            OLED::print("PD: 5V", FontStyle::SMALL);
        }
    } else {
        OLED::print("No PD", FontStyle::SMALL);
    }
}
```

---

## Error Handling

Common issues and their handling:

| Issue | Detection | Response |
|-------|-----------|----------|
| No FUSB302 | `!fusbPresent()` | Fall back to QC/DC |
| Negotiation timeout | State timeout | Retry or fall back |
| Contract rejected | Reject message | Request lower power |
| Cable disconnect | VBUS loss | Re-init on reconnect |

---

## See Also

- [BSP PD](../bsp/BSP_PD.md) - Board-level PD interface
- [POW Thread](../threading/POWThread.md) - Power management thread
- [Power API](../core/Power.md) - Power calculations
- [Settings API](../core/Settings.md) - PD mode settings

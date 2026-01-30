# BSP Power Delivery Interface

The BSP PD module provides platform-specific interface functions for USB Power Delivery communication with the FUSB302 controller.

**Header File:** `source/Core/BSP/BSP_PD.h`
**Implementation:** Platform-specific in `source/Core/BSP/<platform>/BSP_PD.cpp`

---

## Overview

The PD BSP provides:
- I2C communication with FUSB302 PD controller
- IRQ handling for PD events
- Low-level register access

---

## Functions

### fusb_write_buf

```c
bool fusb_write_buf(const uint8_t deviceAddr, const uint8_t registerAdd, const uint8_t size, uint8_t *buf);
```

Writes a buffer of data to FUSB302 registers.

**Parameters:**
- `deviceAddr` - I2C device address (typically 0x22)
- `registerAdd` - Starting register address
- `size` - Number of bytes to write
- `buf` - Pointer to data buffer

**Returns:**
- `true` - Write successful
- `false` - Write failed (NAK or bus error)

**Example:**
```c
uint8_t data[4] = {0x01, 0x02, 0x03, 0x04};
if (!fusb_write_buf(FUSB302_ADDR, FIFO_REG, 4, data)) {
    // Handle error
}
```

---

### fusb_read_buf

```c
bool fusb_read_buf(const uint8_t deviceAddr, const uint8_t registerAdd, const uint8_t size, uint8_t *buf);
```

Reads a buffer of data from FUSB302 registers.

**Parameters:**
- `deviceAddr` - I2C device address
- `registerAdd` - Starting register address
- `size` - Number of bytes to read
- `buf` - Pointer to buffer for received data

**Returns:**
- `true` - Read successful
- `false` - Read failed

**Example:**
```c
uint8_t status[2];
if (fusb_read_buf(FUSB302_ADDR, STATUS0, 2, status)) {
    // Process status registers
}
```

---

### setupFUSBIRQ

```c
void setupFUSBIRQ();
```

Configures the interrupt line for FUSB302 events.

**Actions:**
- Configures GPIO pin as input
- Sets up falling-edge interrupt
- Enables interrupt in NVIC

**Notes:**
- FUSB302 IRQ is active-low
- Interrupt triggers on pending PD events

---

### getFUS302IRQLow

```c
bool getFUS302IRQLow();
```

Checks if the FUSB302 interrupt line is asserted.

**Returns:**
- `true` - IRQ line is low (interrupt pending)
- `false` - IRQ line is high (no interrupt)

**Usage:**
```c
void pdPollingTask() {
    if (getFUS302IRQLow()) {
        USBPowerDelivery::IRQOccured();
    }
}
```

---

## FUSB302 Overview

The FUSB302 is a USB Type-C and PD controller that handles:
- Cable detection and orientation
- PD protocol communication
- VBUS voltage monitoring
- CC line communication

### Register Map (Key Registers)

| Register | Address | Description |
|----------|---------|-------------|
| DEVICE_ID | 0x01 | Chip identification |
| SWITCHES0 | 0x02 | Port measurement switches |
| SWITCHES1 | 0x03 | Auto-response configuration |
| MEASURE | 0x04 | Measurement configuration |
| SLICE | 0x05 | BMC decoder threshold |
| CONTROL0 | 0x06 | Host control |
| CONTROL1 | 0x07 | Control register 1 |
| CONTROL2 | 0x08 | Control register 2 |
| CONTROL3 | 0x09 | Control register 3 |
| MASK | 0x0A | Interrupt mask |
| POWER | 0x0B | Power control |
| RESET | 0x0C | Reset control |
| STATUS0A | 0x3C | Status register 0A |
| STATUS1A | 0x3D | Status register 1A |
| INTERRUPTA | 0x3E | Interrupt register A |
| INTERRUPTB | 0x3F | Interrupt register B |
| STATUS0 | 0x40 | Status register 0 |
| STATUS1 | 0x41 | Status register 1 |
| INTERRUPT | 0x42 | Main interrupt register |
| FIFOS | 0x43 | FIFO access |

---

## I2C Timing

FUSB302 I2C specifications:
- Standard mode: 100 kHz
- Fast mode: 400 kHz
- Address: 0x22 (7-bit) / 0x44 (8-bit with R/W)

```
I2C Write Transaction:
┌─────┬─────────┬─────┬──────────┬─────┬──────┬─────┬──────┬─────┐
│START│ADDR+W(0)│ ACK │ REG ADDR │ ACK │ DATA │ ACK │ ... │STOP │
└─────┴─────────┴─────┴──────────┴─────┴──────┴─────┴──────┴─────┘

I2C Read Transaction:
┌─────┬─────────┬─────┬──────────┬─────┬──────┬─────────┬─────┬──────┬─────┐
│START│ADDR+W(0)│ ACK │ REG ADDR │ ACK │RSTART│ADDR+R(1)│ ACK │ DATA │STOP │
└─────┴─────────┴─────┴──────────┴─────┴──────┴─────────┴─────┴──────┴─────┘
```

---

## IRQ Handling

The FUSB302 generates interrupts for:
- Message received
- Message sent
- Cable attach/detach
- VBUS changes
- Collision detection

### Interrupt Flow

```
┌──────────────┐
│ FUSB302 IRQ  │ Hardware interrupt line goes low
│ asserted     │
└──────┬───────┘
       │
       v
┌──────────────┐
│ GPIO ISR     │ Optionally notify task
└──────┬───────┘
       │
       v
┌──────────────────────┐
│ getFUS302IRQLow()    │ Check IRQ state
│ returns true         │
└──────┬───────────────┘
       │
       v
┌──────────────────────┐
│ USBPowerDelivery::   │ Process PD messages
│ IRQOccured()         │
└──────┬───────────────┘
       │
       v
┌──────────────────────┐
│ Read INTERRUPT reg   │ Clears interrupt
└──────────────────────┘
```

---

## Platform Implementations

### Miniware (STM32F103)

```c
bool fusb_write_buf(const uint8_t deviceAddr, const uint8_t registerAdd,
                    const uint8_t size, uint8_t *buf) {
    return FRToSI2C::Mem_Write(deviceAddr, registerAdd, buf, size);
}

bool fusb_read_buf(const uint8_t deviceAddr, const uint8_t registerAdd,
                   const uint8_t size, uint8_t *buf) {
    return FRToSI2C::Mem_Read(deviceAddr, registerAdd, buf, size);
}
```

### Pinecil V2 (BL706)

Uses platform-specific I2C driver but same interface.

---

## Error Handling

Common failure modes:

| Issue | Detection | Recovery |
|-------|-----------|----------|
| I2C NAK | Function returns false | Retry or reset |
| Bus stuck | Timeout | Call `unstick_I2C()` |
| FUSB302 not responding | Probe fails | Disable PD features |
| IRQ stuck low | Continuous interrupts | Read/clear registers |

---

## Example: FUSB302 Detection

```c
bool detectFUSB302() {
    uint8_t deviceId;

    // Try to read device ID register
    if (!fusb_read_buf(FUSB302_ADDR, DEVICE_ID_REG, 1, &deviceId)) {
        return false;
    }

    // Check for valid FUSB302 ID
    // Device ID format: 0b1001_VVVV where VVVV is version
    if ((deviceId & 0xF0) != 0x90) {
        return false;
    }

    return true;
}
```

---

## See Also

- [USB Power Delivery Driver](../drivers/USBPD.md) - PD protocol implementation
- [I2C Wrapper](../communication/I2C_Wrapper.md) - I2C communication
- [POW Thread](../threading/POWThread.md) - Power management thread

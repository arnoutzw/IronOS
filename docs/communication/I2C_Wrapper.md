# I2C Wrapper (FRToSI2C)

The I2C Wrapper provides a FreeRTOS-aware I2C communication interface with mutex protection and DMA support.

**Header File:** `source/Core/Drivers/I2C_Wrapper.hpp`
**Implementation:** `source/Core/Drivers/I2C_Wrapper.cpp`

---

## Overview

The `FRToSI2C` class provides:
- Thread-safe I2C communication via mutex
- DMA transfers for efficient data movement
- Register read/write helpers
- Bulk register write support
- Bus recovery functions

---

## Class: FRToSI2C

```c
class FRToSI2C {
public:
    static void FRToSInit();
    static void CpltCallback();

    // Memory operations
    static bool Mem_Read(uint16_t DevAddress, uint16_t MemAddress,
                         uint8_t *pData, uint16_t Size);
    static bool Mem_Write(uint16_t DevAddress, uint16_t MemAddress,
                          uint8_t *pData, uint16_t Size);

    // Basic operations
    static bool probe(uint16_t DevAddress);
    static bool wakePart(uint16_t DevAddress);
    static bool Transmit(uint16_t DevAddress, uint8_t *pData, uint16_t Size);
    static void Receive(uint16_t DevAddress, uint8_t *pData, uint16_t Size);
    static void TransmitReceive(uint16_t DevAddress,
                                uint8_t *pData_tx, uint16_t Size_tx,
                                uint8_t *pData_rx, uint16_t Size_rx);

    // Register operations
    static bool    I2C_RegisterWrite(uint8_t address, uint8_t reg, uint8_t data);
    static uint8_t I2C_RegisterRead(uint8_t address, uint8_t reg);
    static bool    writeRegistersBulk(const uint8_t address,
                                      const I2C_REG *registers,
                                      const uint8_t registersLength);
};
```

---

## Types

### I2C_REG

Structure for bulk register writes:

```c
typedef struct {
    const uint8_t reg;       // Register address
    uint8_t       val;       // Value to write
    const uint8_t pause_ms;  // Delay after write (ms)
} I2C_REG;
```

---

## Initialization

### FRToSInit

```c
static void FRToSInit();
```

Initializes the I2C wrapper with FreeRTOS semaphore.

**Actions:**
- Creates binary semaphore for mutex
- Releases semaphore (unlocked state)

**Must be called:**
- After FreeRTOS scheduler starts
- Before any I2C operations

```c
void setup() {
    FRToSI2C::FRToSInit();
}
```

---

### CpltCallback

```c
static void CpltCallback();
```

DMA completion callback.

**Called from:**
- I2C DMA interrupt handler

**Actions:**
- Releases semaphore after DMA transfer
- Signals waiting task

---

## Memory Operations

### Mem_Read

```c
static bool Mem_Read(uint16_t DevAddress, uint16_t MemAddress,
                     uint8_t *pData, uint16_t Size);
```

Reads data from a device register (memory-mapped).

**Parameters:**
- `DevAddress` - I2C device address (8-bit format)
- `MemAddress` - Register/memory address to read from
- `pData` - Buffer to receive data
- `Size` - Number of bytes to read

**Returns:**
- `true` - Read successful
- `false` - Read failed (NAK, timeout, bus error)

**Example:**
```c
uint8_t buffer[6];
if (FRToSI2C::Mem_Read(0x32, 0x28, buffer, 6)) {
    // Process 6 bytes of accelerometer data
}
```

---

### Mem_Write

```c
static bool Mem_Write(uint16_t DevAddress, uint16_t MemAddress,
                      uint8_t *pData, uint16_t Size);
```

Writes data to a device register (memory-mapped).

**Parameters:**
- `DevAddress` - I2C device address
- `MemAddress` - Register/memory address to write to
- `pData` - Buffer containing data to write
- `Size` - Number of bytes to write

**Returns:**
- `true` - Write successful
- `false` - Write failed

**Example:**
```c
uint8_t config = 0x47;
FRToSI2C::Mem_Write(0x32, 0x20, &config, 1);
```

---

## Basic Operations

### probe

```c
static bool probe(uint16_t DevAddress);
```

Checks if a device responds at the given address.

**Parameters:**
- `DevAddress` - I2C device address to probe

**Returns:**
- `true` - Device ACKed the address
- `false` - No response (NAK)

**Example:**
```c
if (FRToSI2C::probe(0x3C)) {
    // OLED display found
}
```

---

### wakePart

```c
static bool wakePart(uint16_t DevAddress);
```

Sends a wake-up sequence to a device.

**Parameters:**
- `DevAddress` - I2C device address

**Returns:**
- `true` - Wake sequence completed
- `false` - Failed

**Usage:**
- Some devices require wake-up after sleep
- Typically sends address with no data

---

### Transmit

```c
static bool Transmit(uint16_t DevAddress, uint8_t *pData, uint16_t Size);
```

Transmits raw data to a device.

**Parameters:**
- `DevAddress` - I2C device address
- `pData` - Data buffer to transmit
- `Size` - Number of bytes

**Returns:**
- `true` - Transmission successful
- `false` - Failed

---

### Receive

```c
static void Receive(uint16_t DevAddress, uint8_t *pData, uint16_t Size);
```

Receives raw data from a device.

**Parameters:**
- `DevAddress` - I2C device address
- `pData` - Buffer for received data
- `Size` - Number of bytes to receive

---

### TransmitReceive

```c
static void TransmitReceive(uint16_t DevAddress,
                            uint8_t *pData_tx, uint16_t Size_tx,
                            uint8_t *pData_rx, uint16_t Size_rx);
```

Combined transmit and receive operation (restart condition).

**Parameters:**
- `DevAddress` - I2C device address
- `pData_tx` - Data to transmit
- `Size_tx` - Transmit size
- `pData_rx` - Receive buffer
- `Size_rx` - Receive size

---

## Register Operations

### I2C_RegisterWrite

```c
static bool I2C_RegisterWrite(uint8_t address, uint8_t reg, uint8_t data);
```

Writes a single byte to a device register.

**Parameters:**
- `address` - I2C device address
- `reg` - Register address
- `data` - Byte to write

**Returns:**
- `true` - Write successful
- `false` - Failed

**Example:**
```c
// Set accelerometer to active mode
FRToSI2C::I2C_RegisterWrite(ACCEL_ADDR, CTRL_REG1, 0x01);
```

---

### I2C_RegisterRead

```c
static uint8_t I2C_RegisterRead(uint8_t address, uint8_t reg);
```

Reads a single byte from a device register.

**Parameters:**
- `address` - I2C device address
- `reg` - Register address

**Returns:**
- Register value (byte)

**Example:**
```c
uint8_t whoami = FRToSI2C::I2C_RegisterRead(ACCEL_ADDR, WHO_AM_I_REG);
```

---

### writeRegistersBulk

```c
static bool writeRegistersBulk(const uint8_t address,
                               const I2C_REG *registers,
                               const uint8_t registersLength);
```

Writes multiple registers with optional delays.

**Parameters:**
- `address` - I2C device address
- `registers` - Array of register definitions
- `registersLength` - Number of registers

**Returns:**
- `true` - All writes successful
- `false` - A write failed

**Example:**
```c
const FRToSI2C::I2C_REG initSequence[] = {
    {0x20, 0x47, 0},   // CTRL_REG1 = 0x47
    {0x23, 0x88, 0},   // CTRL_REG4 = 0x88
    {0x22, 0x40, 10},  // CTRL_REG3 = 0x40, wait 10ms
};

FRToSI2C::writeRegistersBulk(ACCEL_ADDR, initSequence, 3);
```

---

## Thread Safety

### Mutex Protection

All operations are protected by a binary semaphore:

```c
static bool lock() {
    return xSemaphoreTake(I2CSemaphore, TICKS_100MS) == pdTRUE;
}

static void unlock() {
    xSemaphoreGive(I2CSemaphore);
}
```

### Usage Pattern

```c
bool Mem_Read(uint16_t DevAddress, uint16_t MemAddress,
              uint8_t *pData, uint16_t Size) {
    if (!lock()) return false;

    // Perform I2C operation
    HAL_I2C_Mem_Read_DMA(...);

    // Wait for completion (semaphore given in callback)
    ulTaskNotifyTake(pdTRUE, TICKS_100MS);

    unlock();
    return true;
}
```

---

## Bus Recovery

### I2C_Unstick

```c
static void I2C_Unstick();
```

Attempts to recover a stuck I2C bus.

**Actions:**
1. Set SCL as output
2. Toggle SCL until SDA releases
3. Send STOP condition
4. Reinitialize I2C peripheral

**Usage:**
```c
if (I2C_operation_failed()) {
    I2C_Unstick();
    // Retry operation
}
```

---

## Error Handling

### Common Failures

| Error | Cause | Recovery |
|-------|-------|----------|
| NAK | Device not present | Probe first |
| Timeout | Device hung | I2C_Unstick() |
| Bus busy | Previous op incomplete | Wait and retry |

### Example with Error Handling

```c
bool readAccelerometer(uint8_t *data) {
    int retries = 3;

    while (retries--) {
        if (FRToSI2C::Mem_Read(ACCEL_ADDR, DATA_REG, data, 6)) {
            return true;
        }

        // Recovery attempt
        I2C_Unstick();
        osDelay(TICKS_10MS);
    }

    return false;  // Failed after retries
}
```

---

## See Also

- [I2C Bit-Bang](I2CBB.md) - Software I2C implementation
- [BSP Core](../bsp/BSP.md) - `unstick_I2C()` function
- [Accelerometers](../drivers/Accelerometers.md) - I2C device example

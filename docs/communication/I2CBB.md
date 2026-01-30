# I2C Bit-Bang Drivers

The I2C Bit-Bang modules provide software-based I2C communication for platforms that require additional I2C buses or have hardware I2C limitations.

**Header Files:**
- `source/Core/Drivers/I2CBB1.hpp`
- `source/Core/Drivers/I2CBB2.hpp`

---

## Overview

The bit-bang I2C implementations provide:
- Software-controlled I2C communication
- Multiple independent I2C buses
- Same interface as hardware I2C wrapper
- FreeRTOS semaphore protection

---

## Available Buses

### I2CBB1

First software I2C bus, enabled by `I2C_SOFT_BUS_1` define.

### I2CBB2

Second software I2C bus, enabled by `I2C_SOFT_BUS_2` define.

Both provide identical interfaces.

---

## Class: I2CBB1 / I2CBB2

```c
class I2CBB1 {  // or I2CBB2
public:
    static void init();
    static bool probe(uint8_t address);

    // Memory operations
    static bool Mem_Read(uint16_t DevAddress, uint16_t MemAddress,
                         uint8_t *pData, uint16_t Size);
    static bool Mem_Write(uint16_t DevAddress, uint16_t MemAddress,
                          const uint8_t *pData, uint16_t Size);

    // Basic operations
    static void Transmit(uint16_t DevAddress, uint8_t *pData, uint16_t Size);
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
    static bool    wakePart(uint16_t DevAddress);
};
```

---

## Initialization

### init

```c
static void init();
```

Initializes the bit-bang I2C bus.

**Actions:**
- Configures GPIO pins for SDA and SCL
- Sets pins to open-drain mode
- Creates FreeRTOS semaphore
- Sets bus to idle state

**GPIO Configuration:**
```c
// Pins are configured as:
// - Open-drain output
// - With external pull-up resistors
// - High = released (pulled up externally)
// - Low = driven low
```

---

## Public Functions

All public functions match the `FRToSI2C` interface for compatibility:

### probe

```c
static bool probe(uint8_t address);
```

Checks if a device responds at the given address.

---

### Mem_Read

```c
static bool Mem_Read(uint16_t DevAddress, uint16_t MemAddress,
                     uint8_t *pData, uint16_t Size);
```

Reads from a device register.

---

### Mem_Write

```c
static bool Mem_Write(uint16_t DevAddress, uint16_t MemAddress,
                      const uint8_t *pData, uint16_t Size);
```

Writes to a device register.

---

### I2C_RegisterWrite / I2C_RegisterRead

Single-byte register access functions.

---

### writeRegistersBulk

Bulk register write with delays.

---

## Private Functions

The bit-bang implementation uses these low-level functions:

### start

```c
static void start();
```

Generates I2C START condition.

```
SDA: ────┐
         └────
SCL: ────────┐
             └──
```

---

### stop

```c
static void stop();
```

Generates I2C STOP condition.

```
SDA:      ┌────
    ──────┘
SCL:   ┌──────
    ───┘
```

---

### send

```c
static bool send(uint8_t value);
```

Sends a byte and receives ACK/NAK.

**Parameters:**
- `value` - Byte to send

**Returns:**
- `true` - ACK received
- `false` - NAK received

---

### read

```c
static uint8_t read(bool ack);
```

Receives a byte and sends ACK/NAK.

**Parameters:**
- `ack` - `true` to send ACK, `false` for NAK

**Returns:**
- Received byte

---

### read_bit / write_bit

```c
static uint8_t read_bit();
static void write_bit(uint8_t val);
```

Single-bit operations for the I2C protocol.

---

## Timing

### Bit Timing

Software I2C timing is controlled by delays:

```c
void write_bit(uint8_t val) {
    // Set SDA
    if (val) {
        SDA_HIGH();
    } else {
        SDA_LOW();
    }

    // Clock pulse
    delay_us(I2C_DELAY);
    SCL_HIGH();
    delay_us(I2C_DELAY);
    SCL_LOW();
}
```

### Speed

Typical speeds achieved:
- ~100 kHz (standard mode)
- Limited by GPIO toggle speed and delays

---

## Thread Safety

### Semaphore Protection

```c
static SemaphoreHandle_t I2CSemaphore;
static StaticSemaphore_t xSemaphoreBuffer;

static bool lock() {
    return xSemaphoreTake(I2CSemaphore, TICKS_100MS) == pdTRUE;
}

static void unlock() {
    xSemaphoreGive(I2CSemaphore);
}
```

All public functions acquire the semaphore before accessing the bus.

---

## Pin Configuration

Pin definitions are platform-specific:

```c
// From Pins.h (example)
#define I2C_SOFT_SDA_PIN    GPIO_PIN_7
#define I2C_SOFT_SCL_PIN    GPIO_PIN_6
#define I2C_SOFT_GPIO_PORT  GPIOB
```

### GPIO Macros

```c
#define SDA_HIGH()  HAL_GPIO_WritePin(I2C_SOFT_GPIO_PORT, I2C_SOFT_SDA_PIN, GPIO_PIN_SET)
#define SDA_LOW()   HAL_GPIO_WritePin(I2C_SOFT_GPIO_PORT, I2C_SOFT_SDA_PIN, GPIO_PIN_RESET)
#define SCL_HIGH()  HAL_GPIO_WritePin(I2C_SOFT_GPIO_PORT, I2C_SOFT_SCL_PIN, GPIO_PIN_SET)
#define SCL_LOW()   HAL_GPIO_WritePin(I2C_SOFT_GPIO_PORT, I2C_SOFT_SCL_PIN, GPIO_PIN_RESET)
#define SDA_READ()  HAL_GPIO_ReadPin(I2C_SOFT_GPIO_PORT, I2C_SOFT_SDA_PIN)
```

---

## Usage Selection

The appropriate I2C class is selected at compile time:

### For Accelerometers

```c
// accelerometers_common.h
#if defined(ACCEL_I2CBB2)
#define ACCEL_I2C_CLASS I2CBB2
#elif defined(ACCEL_I2CBB1)
#define ACCEL_I2C_CLASS I2CBB1
#else
#define ACCEL_I2C_CLASS FRToSI2C
#endif
```

### For OLED Display

```c
// OLED.hpp
#if defined(OLED_I2CBB2)
#define I2C_CLASS I2CBB2
#elif defined(OLED_I2CBB1)
#define I2C_CLASS I2CBB1
#else
#define I2C_CLASS FRToSI2C
#endif
```

---

## Advantages and Limitations

### Advantages

- Flexible pin assignment
- Multiple independent buses
- Works on any GPIO pins
- No DMA required

### Limitations

- CPU-intensive (blocks during transfer)
- Slower than hardware I2C
- No clock stretching support (usually)
- Sensitive to interrupt latency

---

## Example: Bit-Bang I2C Transaction

```c
bool readSensor(uint8_t *data, uint8_t len) {
    if (!lock()) return false;

    // START
    start();

    // Send device address (write)
    if (!send(SENSOR_ADDR << 1 | 0)) {
        stop();
        unlock();
        return false;
    }

    // Send register address
    send(DATA_REG);

    // Repeated START
    start();

    // Send device address (read)
    send(SENSOR_ADDR << 1 | 1);

    // Read data bytes
    for (uint8_t i = 0; i < len; i++) {
        data[i] = read(i < len - 1);  // NAK on last byte
    }

    // STOP
    stop();

    unlock();
    return true;
}
```

---

## See Also

- [I2C Wrapper](I2C_Wrapper.md) - Hardware I2C interface
- [Accelerometers](../drivers/Accelerometers.md) - I2C device example
- [OLED Driver](../drivers/OLED.md) - I2C display interface

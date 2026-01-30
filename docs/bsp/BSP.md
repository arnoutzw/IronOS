# Board Support Package (BSP) Core

The BSP (Board Support Package) provides a hardware abstraction layer that allows IronOS to run on multiple hardware platforms with a unified API.

**Header File:** `source/Core/BSP/BSP.h`
**Implementation:** Platform-specific in `source/Core/BSP/<platform>/BSP.cpp`

---

## Overview

The BSP layer abstracts:
- GPIO and peripheral control
- ADC readings for temperature and voltage
- PWM output for heater control
- Hardware watchdog management
- Platform-specific initialization

---

## Supported Platforms

| Platform | Directory | MCU | Notes |
|----------|-----------|-----|-------|
| Miniware | `BSP/Miniware/` | STM32F103 | TS100, TS80 |
| Pinecil | `BSP/Pinecil/` | GD32VF103 | Original Pinecil |
| Pinecil V2 | `BSP/Pinecilv2/` | BL706 | Newer Pinecil |
| MHP30 | `BSP/MHP30/` | STM32F103 | Hot plate variant |
| Sequre | `BSP/Sequre/` | Varies | Sequre irons |

---

## Global Variables

### powerPWM

```c
extern const uint16_t powerPWM;
```

Maximum PWM value for tip power control. This is the value that corresponds to 100% duty cycle.

---

### totalPWM

```c
extern uint16_t totalPWM;
```

Full PWM cycle period. Used for PWM configuration.

---

## Initialization Functions

### preRToSInit

```c
void preRToSInit();
```

Called first in `main()` before FreeRTOS is started.

**Actions:**
- Initializes system clocks
- Configures GPIO pins
- Sets up peripherals (I2C, SPI, UART, ADC, Timer)
- Initializes hardware watchdog

**Notes:**
- No FreeRTOS functions available
- Must complete before RTOS starts

---

### postRToSInit

```c
void postRToSInit();
```

Called after FreeRTOS scheduler has started.

**Actions:**
- Completes initialization that requires RTOS
- Starts DMA operations
- Enables interrupts that need task context

---

### BSPInit

```c
void BSPInit(void);
```

Called once from `preRToSInit()` to initialize board-specific hardware.

**Actions:**
- Configures board-specific GPIO
- Initializes platform-specific peripherals
- Sets up power management

---

## Watchdog Functions

### resetWatchdog

```c
void resetWatchdog();
```

Resets the hardware watchdog timer to prevent system reset.

**Notes:**
- Must be called periodically (typically every 100ms)
- Called from PID thread during normal operation
- Failure to call will trigger hardware reset

---

## PWM and Heater Control

### setTipPWM

```c
void setTipPWM(const uint8_t pulse, const bool shouldUseFastModePWM);
```

Sets the PWM duty cycle for the soldering tip heater.

**Parameters:**
- `pulse` - PWM value (0 to `powerPWM`)
- `shouldUseFastModePWM` - If `true`, use faster PWM frequency for reduced noise

**Example:**
```c
// 50% power
setTipPWM(powerPWM / 2, false);

// Full power
setTipPWM(powerPWM, false);

// Off
setTipPWM(0, false);
```

---

## Temperature and Voltage Sensing

### getTipRawTemp

```c
uint16_t getTipRawTemp(uint8_t refresh);
```

Returns the raw ADC reading from the tip temperature sensor.

**Parameters:**
- `refresh` - If non-zero, triggers a new ADC sample

**Returns:**
- Raw 12/16-bit ADC value

**Notes:**
- Use `TipThermoModel` to convert to actual temperature
- Reading occurs during PWM off-time for accuracy

---

### getHandleTemperature

```c
uint16_t getHandleTemperature(uint8_t sample);
```

Returns the handle/cold-junction temperature.

**Parameters:**
- `sample` - If non-zero, triggers a new ADC sample

**Returns:**
- Temperature in Celsius × 10 (e.g., 250 = 25.0°C)

**Notes:**
- Used for cold junction compensation
- Typically uses NTC thermistor

---

### getInputVoltageX10

```c
uint16_t getInputVoltageX10(uint16_t divisor, uint8_t sample);
```

Returns the main DC input voltage.

**Parameters:**
- `divisor` - Voltage divider calibration value
- `sample` - If non-zero, triggers a new ADC sample

**Returns:**
- Voltage × 10 (e.g., 240 = 24.0V)

**Example:**
```c
uint16_t voltage = getInputVoltageX10(getSettingValue(VoltageDiv), 1);
// voltage = 240 means 24.0V
```

---

## Button Input

### getButtonA

```c
uint8_t getButtonA();
```

Reads the state of button A (front button).

**Returns:**
- `1` - Button pressed/held
- `0` - Button released

---

### getButtonB

```c
uint8_t getButtonB();
```

Reads the state of button B (back button).

**Returns:**
- `1` - Button pressed/held
- `0` - Button released

---

## I2C Recovery

### unstick_I2C

```c
void unstick_I2C();
```

Attempts to recover a stuck I2C bus.

**Behavior:**
- Toggles SCL line until SDA goes high
- Ends any ongoing I2C transaction
- Resets I2C peripheral

**Notes:**
- Called when I2C communication fails
- May be needed after noise-induced lockup

---

## System Control

### reboot

```c
void reboot();
```

Performs a software reset of the microcontroller.

**Notes:**
- Used for fatal error recovery
- Also used after settings reset

---

### delay_ms

```c
void delay_ms(uint16_t count);
```

Blocking delay using hardware timer.

**Parameters:**
- `count` - Delay duration in milliseconds

**Notes:**
- Used before RTOS is started
- After RTOS starts, use `osDelay()` instead

---

## Hall Effect Sensor

### getHallSensorFitted

```c
bool getHallSensorFitted();
```

Checks if a Hall effect sensor is installed.

**Returns:**
- `true` - Hall sensor present
- `false` - No Hall sensor

---

### getRawHallEffect

```c
int16_t getRawHallEffect();
```

Reads the Hall effect sensor value.

**Returns:**
- Signed 16-bit magnetic field reading
- 0 to 32767 for single-polarity sensors

**Notes:**
- Used for magnetic stand detection
- Returns 0 if no sensor fitted

---

## Power Source Detection

### getIsPoweredByDCIN

```c
bool getIsPoweredByDCIN();
```

Checks if power is from DC input rather than USB.

**Returns:**
- `true` - Powered by DC jack (no negotiation)
- `false` - Powered by USB (QC/PD may be active)

---

## Tip Detection

### isTipDisconnected

```c
bool isTipDisconnected();
```

Checks if the soldering tip is disconnected.

**Returns:**
- `true` - No tip detected (open circuit)
- `false` - Tip is connected

**Notes:**
- Detected by ADC reading at maximum
- Used to disable heating

---

### isTipShorted

```c
bool isTipShorted();
```

Checks if the tip or output MOSFET is shorted.

**Returns:**
- `true` - Short circuit detected
- `false` - Normal operation

---

## Device Identification

### getDeviceID

```c
uint64_t getDeviceID();
```

Returns a hardware-unique identifier.

**Returns:**
- 64-bit unique ID from MCU

**Notes:**
- Used for device identification
- May be used for licensing

---

### getDeviceValidation

```c
uint32_t getDeviceValidation();
```

Returns the device validation code if present.

**Returns:**
- Validation code from OTP memory
- 0 if not programmed

---

### getDeviceValidationStatus

```c
uint8_t getDeviceValidationStatus();
```

Checks if device validation passes.

**Returns:**
- `0` - Validation passed
- Non-zero - Validation failed

---

## Status LED

### StatusLED Enum

```c
enum StatusLED {
  LED_OFF = 0,           // LED off
  LED_STANDBY,           // Unit in sleep/standby
  LED_HEATING,           // Heating up to temperature
  LED_HOT,               // At operating temperature
  LED_COOLING_STILL_HOT, // Off but still hot
  LED_UNKNOWN            // Unknown state
};
```

### setStatusLED

```c
void setStatusLED(const enum StatusLED state);
```

Sets the status LED color/pattern.

**Parameters:**
- `state` - Status LED state from enum

**Notes:**
- Implementation varies by platform
- Some platforms use RGB LED
- Some platforms have no status LED

---

## Buzzer

### setBuzzer

```c
void setBuzzer(bool on);
```

Controls the piezo buzzer (if present).

**Parameters:**
- `on` - `true` to turn buzzer on, `false` to turn off

---

## Boot Logo

### showBootLogo

```c
void showBootLogo(void);
```

Displays the boot logo on the OLED screen.

---

### showBootLogoIfavailable

```c
void showBootLogoIfavailable();
```

Displays a custom boot logo from flash if programmed.

**Returns:**
- (via behavior) Waits for timeout or button press if logo shown

---

## Pre-Start Checks

### preStartChecks

```c
uint8_t preStartChecks();
```

Runs hardware checks before normal operation.

**Returns:**
- Non-zero while checks are in progress
- `0` when checks complete

**Notes:**
- Called by PID task during ADC cycles
- Example: MHP30 uses this for resistance detection

---

### preStartChecksDone

```c
uint8_t preStartChecksDone();
```

Checks if pre-start checks have completed.

**Returns:**
- Non-zero if checks complete
- `0` if still running

---

## Logging

### log_system_state

```c
void log_system_state(int32_t PWMWattsx10);
```

Logs system state to debug interface if available.

**Parameters:**
- `PWMWattsx10` - Current power output in tenths of watts

**Notes:**
- Used for debugging and diagnostics
- May output to UART, USB CDC, or BLE

---

## See Also

- [BSP Flash](BSP_Flash.md) - Flash memory interface
- [BSP Power](BSP_Power.md) - Power control interface
- [BSP PD](BSP_PD.md) - USB Power Delivery interface
- [BSP QC](BSP_QC.md) - Quick Charge interface

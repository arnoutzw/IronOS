# Tip Thermo Model

The TipThermoModel class provides temperature sensing and conversion for the soldering tip, including thermocouple compensation and calibration.

**Header File:** `source/Core/Drivers/TipThermoModel.h`
**Implementation:** `source/Core/Drivers/TipThermoModel.cpp`

---

## Overview

The tip temperature model handles:
- Converting raw ADC readings to temperature
- Thermocouple voltage calculations
- Cold junction compensation (CJC)
- Celsius/Fahrenheit conversions
- Calibration offset application

---

## Class: TipThermoModel

The `TipThermoModel` class provides static methods for all temperature operations.

---

## Primary Functions

### getTipInC

```c
static TemperatureType_t getTipInC(bool sampleNow = false);
```

Returns the current tip temperature in degrees Celsius.

**Parameters:**
- `sampleNow` - If `true`, triggers a new ADC reading; otherwise uses cached value

**Returns:**
- Temperature in degrees Celsius (as `TemperatureType_t` / int32_t)

**Example:**
```c
TemperatureType_t temp = TipThermoModel::getTipInC(true);
// temp = 350 means 350°C
```

---

### getTipInF

```c
static TemperatureType_t getTipInF(bool sampleNow = false);
```

Returns the current tip temperature in degrees Fahrenheit.

**Parameters:**
- `sampleNow` - If `true`, triggers a new ADC reading

**Returns:**
- Temperature in degrees Fahrenheit

**Example:**
```c
TemperatureType_t temp = TipThermoModel::getTipInF(true);
// temp = 662 means 662°F (350°C)
```

---

### getTipMaxInC

```c
static TemperatureType_t getTipMaxInC();
```

Calculates the maximum temperature that can be read by the ADC.

**Returns:**
- Maximum readable temperature in Celsius

**Notes:**
- Limited by ADC resolution and op-amp gain
- Used to detect "tip disconnected" condition
- Typically around 450-500°C

---

## Conversion Functions

### convertTipRawADCToDegC

```c
static TemperatureType_t convertTipRawADCToDegC(uint16_t rawADC);
```

Converts a raw ADC reading to temperature in Celsius.

**Parameters:**
- `rawADC` - Raw 12-bit or 16-bit ADC value

**Returns:**
- Temperature in degrees Celsius

**Notes:**
- Applies calibration offset
- Compensates for cold junction temperature
- Uses tip-specific thermocouple curve

---

### convertTipRawADCToDegF

```c
static TemperatureType_t convertTipRawADCToDegF(uint16_t rawADC);
```

Converts a raw ADC reading to temperature in Fahrenheit.

**Parameters:**
- `rawADC` - Raw ADC value

**Returns:**
- Temperature in degrees Fahrenheit

---

### convertTipRawADCTouV

```c
static uint32_t convertTipRawADCTouV(uint16_t rawADC, bool skipCalOffset = false);
```

Converts a raw ADC reading to thermocouple voltage in microvolts.

**Parameters:**
- `rawADC` - Raw ADC value
- `skipCalOffset` - If `true`, skip calibration offset (for calibration use)

**Returns:**
- Thermocouple voltage in microvolts (µV)

**Notes:**
- Accounts for op-amp gain
- Compensates for pullup resistors
- Used for detailed diagnostics

---

### convertCtoF

```c
static TemperatureType_t convertCtoF(TemperatureType_t degC);
```

Converts Celsius to Fahrenheit.

**Parameters:**
- `degC` - Temperature in Celsius

**Returns:**
- Temperature in Fahrenheit

**Formula:**
```
°F = (°C × 9/5) + 32
```

---

### convertFtoC

```c
static TemperatureType_t convertFtoC(TemperatureType_t degF);
```

Converts Fahrenheit to Celsius.

**Parameters:**
- `degF` - Temperature in Fahrenheit

**Returns:**
- Temperature in Celsius

**Formula:**
```
°C = (°F - 32) × 5/9
```

---

## Private Functions

### convertuVToDegC

```c
static TemperatureType_t convertuVToDegC(uint32_t tipuVDelta);
```

Converts thermocouple voltage delta to temperature in Celsius.

**Parameters:**
- `tipuVDelta` - Thermocouple voltage relative to cold junction (µV)

**Returns:**
- Temperature in Celsius

**Notes:**
- Uses linearized thermocouple lookup table
- Tip-type specific coefficients

---

### convertuVToDegF

```c
static TemperatureType_t convertuVToDegF(uint32_t tipuVDelta);
```

Converts thermocouple voltage delta to temperature in Fahrenheit.

---

## Temperature Sensing Chain

```
┌─────────────────┐
│ Thermocouple    │ Tip temperature creates voltage
│ (in tip)        │
└────────┬────────┘
         │ ~40µV/°C (Type K)
         v
┌─────────────────┐
│ Op-Amp          │ Amplifies thermocouple voltage
│ (on PCB)        │ Gain: ~100-200x
└────────┬────────┘
         │
         v
┌─────────────────┐
│ ADC             │ Converts to digital value
│ (in MCU)        │ 12-bit resolution
└────────┬────────┘
         │
         v
┌─────────────────┐
│ TipThermoModel  │ Converts ADC to temperature
│                 │ - Op-amp gain compensation
│                 │ - Cold junction compensation
│                 │ - Calibration offset
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Temperature     │ Final value in °C or °F
│ (output)        │
└─────────────────┘
```

---

## Cold Junction Compensation

Thermocouple voltage is relative to the cold junction (where thermocouple meets PCB). CJC accounts for ambient PCB temperature:

```c
// Conceptual CJC calculation
uint32_t tipVoltage = convertTipRawADCTouV(rawADC);
uint16_t handleTemp = getHandleTemperature();
uint32_t cjcVoltage = tempToThermocoupleVoltage(handleTemp);
uint32_t actualVoltage = tipVoltage + cjcVoltage;
TemperatureType_t tipTemp = convertuVToDegC(actualVoltage);
```

---

## Calibration

The `CalibrationOffset` setting adjusts for manufacturing variations:

```c
// Applied during conversion
int16_t offset = (int16_t)getSettingValue(CalibrationOffset);
TemperatureType_t calibratedTemp = rawTemp + offset;
```

### Calibration Procedure

1. Heat tip to known reference temperature (e.g., melting point of solder)
2. Compare displayed temperature to reference
3. Adjust `CalibrationOffset` setting
4. Save settings

---

## Tip Types

Different tip types have different thermal characteristics:

| Tip Type | Resistance | Thermocouple | Notes |
|----------|------------|--------------|-------|
| T12 (8Ω) | 8.0Ω | Type K equiv | TS100/TS80 style |
| T12 Short (6.2Ω) | 6.2Ω | Type K equiv | Pine64 short tips |
| T12 Low-R (4Ω) | 4.0Ω | Type K equiv | PTS200 tips |

---

## Example Usage

### Basic Temperature Reading

```c
// Get current temperature
TemperatureType_t currentTemp = TipThermoModel::getTipInC(true);

// Check if at target
if (currentTemp >= targetTemp) {
    // At temperature
}
```

### Temperature Display

```c
void displayTemperature() {
    TemperatureType_t temp;

    if (getSettingValue(TemperatureInF)) {
        temp = TipThermoModel::getTipInF(false);
    } else {
        temp = TipThermoModel::getTipInC(false);
    }

    OLED::printNumber(temp, 3, FontStyle::LARGE);
    OLED::printSymbolDeg();
}
```

### Detecting Tip Disconnect

```c
bool isTipConnected() {
    uint16_t rawADC = getTipRawTemp(1);
    TemperatureType_t temp = TipThermoModel::convertTipRawADCToDegC(rawADC);
    TemperatureType_t maxTemp = TipThermoModel::getTipMaxInC();

    // If reading is at max, tip is likely disconnected
    return (temp < maxTemp - 10);
}
```

---

## Accuracy Considerations

| Factor | Typical Error | Notes |
|--------|---------------|-------|
| ADC resolution | ±1°C | 12-bit ADC |
| Thermocouple linearity | ±2°C | Linearization applied |
| CJC accuracy | ±2°C | Handle temperature sensor |
| Op-amp offset | ±3°C | Calibration compensates |
| **Total (calibrated)** | **±3-5°C** | After user calibration |

---

## See Also

- [BSP Core](../bsp/BSP.md) - `getTipRawTemp()`, `getHandleTemperature()`
- [BSP Power](../bsp/BSP_Power.md) - Tip resistance and thermal mass
- [PID Thread](../threading/PIDThread.md) - Temperature control loop
- [Settings API](../core/Settings.md) - Calibration settings

# Settings API

The Settings module manages all user-configurable settings for IronOS, including persistence to flash memory, value access, and enumeration of available options.

**Header File:** `source/Core/Inc/Settings.h`
**Implementation:** `source/Core/Src/Settings.cpp`

---

## Constants

### SETTINGSVERSION

```c
#define SETTINGSVERSION (0x55AA)  // Standard platforms
#define SETTINGSVERSION (0x55AB)  // Pinecil V2
```

Version identifier used to detect settings format changes. When the firmware is updated and this value changes, settings are reset to defaults.

---

## Enumerations

### SettingsOptions

Enumeration of all configurable settings in the system.

```c
enum SettingsOptions {
  SolderingTemp                  = 0,   // Current set point for the iron
  SleepTemp                      = 1,   // Temp to drop to in sleep mode
  SleepTime                      = 2,   // Minutes timeout to sleep
  MinDCVoltageCells              = 3,   // DC jack undervoltage cutoff
  MinVoltageCells                = 4,   // Minimum voltage per cell
  QCIdealVoltage                 = 5,   // Desired QC3.0 voltage (9,12,20V)
  OrientationMode                = 6,   // Screen orientation (Auto/Right/Left)
  Sensitivity                    = 7,   // Accelerometer sensitivity (5 bits)
  AnimationLoop                  = 8,   // Animation loop switch
  AnimationSpeed                 = 9,   // Animation speed (ms)
  AutoStartMode                  = 10,  // Auto-start behavior on power
  ShutdownTime                   = 11,  // Shutdown timeout if idle
  CoolingTempBlink               = 12,  // Blink temp until <50C
  DetailedIDLE                   = 13,  // Detailed idle screen
  DetailedSoldering              = 14,  // Detailed soldering screen
  TemperatureInF                 = 15,  // Temperature in Fahrenheit
  DescriptionScrollSpeed         = 16,  // Menu description scroll speed
  LockingMode                    = 17,  // Button locking mode
  KeepAwakePulse                 = 18,  // Keep awake pulse power (0.1W)
  KeepAwakePulseWait             = 19,  // Time between pulses (2.5s)
  KeepAwakePulseDuration         = 20,  // Pulse duration (250ms)
  VoltageDiv                     = 21,  // Voltage divisor calibration
  BoostTemp                      = 22,  // Boost mode temperature
  CalibrationOffset              = 23,  // Tip temperature offset
  PowerLimit                     = 24,  // Maximum power output
  ReverseButtonTempChangeEnabled = 25,  // Swap +/- buttons
  TempChangeLongStep             = 26,  // Long press temp increment
  TempChangeShortStep            = 27,  // Short press temp increment
  HallEffectSensitivity          = 28,  // Hall sensor mode
  AccelMissingWarningCounter     = 29,  // Accel warning counter
  PDMissingWarningCounter        = 30,  // PD warning counter
  UILanguage                     = 31,  // Language code (8 chars)
  PDNegTimeout                   = 32,  // PD timeout (100ms steps)
  OLEDInversion                  = 33,  // Invert display colors
  OLEDBrightness                 = 34,  // OLED brightness
  LOGOTime                       = 35,  // Logo display duration
  CalibrateCJC                   = 36,  // CJC calibration flag
  BluetoothLE                    = 37,  // BLE toggle
  USBPDMode                      = 38,  // PPS & EPR mode
  ProfilePhases                  = 39,  // Profile mode phase count
  ProfilePreheatTemp             = 40,  // Preheat temperature
  ProfilePreheatSpeed            = 41,  // Max preheat speed (C/s)
  ProfilePhase1Temp              = 42,  // Phase 1 target temp
  ProfilePhase1Duration          = 43,  // Phase 1 duration
  ProfilePhase2Temp              = 44,  // Phase 2 target temp
  ProfilePhase2Duration          = 45,  // Phase 2 duration
  ProfilePhase3Temp              = 46,  // Phase 3 target temp
  ProfilePhase3Duration          = 47,  // Phase 3 duration
  ProfilePhase4Temp              = 48,  // Phase 4 target temp
  ProfilePhase4Duration          = 49,  // Phase 4 duration
  ProfilePhase5Temp              = 50,  // Phase 5 target temp
  ProfilePhase5Duration          = 51,  // Phase 5 duration
  ProfileCooldownSpeed           = 52,  // Max cooldown speed (C/s)
  HallEffectSleepTime            = 53,  // Hall effect sleep timeout
  SolderingTipType               = 54,  // Soldering tip type
  ReverseButtonSettings          = 55,  // Swap A/B in settings
  SettingsOptionsLength          = 56   // Total setting count
};
```

### settingOffSpeed_t

Speed settings used for animations and scrolling.

```c
typedef enum {
  OFF       = 0,  // Disabled
  SLOW      = 1,  // Slow speed
  MEDIUM    = 2,  // Medium speed
  FAST      = 3,  // Fast speed
  MAX_VALUE = 4   // Maximum value marker
} settingOffSpeed_t;
```

### autoStartMode_t

Auto-start behavior when power is connected.

```c
typedef enum {
  NO     = 0,  // Disabled - start at home screen
  SOLDER = 1,  // Start at soldering temperature
  SLEEP  = 2,  // Start at sleep temperature
  ZERO   = 3   // Power on only, no heat
} autoStartMode_t;
```

### orientationMode_t

Screen orientation mode.

```c
typedef enum {
  RIGHT = 0,  // Right-hand orientation
  LEFT  = 1,  // Left-hand orientation
  AUTO  = 2   // Automatic based on accelerometer
} orientationMode_t;
```

### logoMode_t

Boot logo display mode.

```c
typedef enum {
  SKIP     = 0,  // Skip boot logo
  ONETIME  = 5,  // Show once, wait for button
  INFINITY = 6   // Loop animation until button
} logoMode_t;
```

### usbpdMode_t

USB Power Delivery negotiation mode.

```c
typedef enum {
  DEFAULT    = 1,  // PPS + EPR + power compensation
  SAFE       = 2,  // PPS + EPR without extra power
  NO_DYNAMIC = 0   // Fixed PDO only
} usbpdMode_t;
```

### lockingMode_t

Button locking behavior.

```c
typedef enum {
  DISABLED = 0,  // No button locking
  BOOST    = 1,  // Lock in boost mode only
  FULL     = 2   // Lock in boost and soldering
} lockingMode_t;
```

### tipType_t

Soldering tip type selection (when `TIP_TYPE_SUPPORT` is defined).

```c
typedef enum {
  TIP_TYPE_AUTO,  // Automatic detection
  T12_8_OHM,      // TS100 style / Hakko T12 (8Ω)
  T12_6_2_OHM,    // Pine64 short tips (6.2Ω)
  T12_4_OHM,      // PTS200 low resistance (4Ω)
  TIP_TYPE_MAX    // Maximum value marker
} tipType_t;
```

---

## Functions

### Settings Persistence

#### saveSettings

```c
void saveSettings();
```

Saves all current settings to flash memory. Settings are written to a dedicated flash sector.

**Usage:**
```c
setSettingValue(SolderingTemp, 350);
saveSettings();  // Persist the change
```

---

#### loadSettings

```c
bool loadSettings();
```

Loads settings from flash memory into RAM.

**Returns:**
- `true` - Settings loaded successfully
- `false` - Settings failed to load or were invalid (defaults applied)

**Notes:**
- Called during system initialization
- If version mismatch is detected, settings are reset to defaults

---

#### resetSettings

```c
void resetSettings();
```

Resets all settings to factory defaults. Does not automatically save to flash.

**Usage:**
```c
resetSettings();
saveSettings();  // Make reset permanent
```

---

### Settings Access

#### getSettingValue

```c
uint16_t getSettingValue(const enum SettingsOptions option);
```

Retrieves the current value of a setting.

**Parameters:**
- `option` - The setting to retrieve (from `SettingsOptions` enum)

**Returns:**
- The 16-bit value of the setting

**Example:**
```c
uint16_t temp = getSettingValue(SolderingTemp);
bool isF = getSettingValue(TemperatureInF) != 0;
```

---

#### setSettingValue

```c
void setSettingValue(const enum SettingsOptions option, const uint16_t newValue);
```

Sets a setting to a specific value.

**Parameters:**
- `option` - The setting to modify
- `newValue` - The new value to assign

**Notes:**
- Value is validated against min/max bounds
- Does not automatically persist to flash

**Example:**
```c
setSettingValue(SolderingTemp, 320);
setSettingValue(TemperatureInF, 0);  // Use Celsius
```

---

#### nextSettingValue

```c
void nextSettingValue(const enum SettingsOptions option);
```

Increments a setting to its next valid value, wrapping around if at maximum.

**Parameters:**
- `option` - The setting to increment

**Usage:**
- Used in settings menu for cycling through options

---

#### prevSettingValue

```c
void prevSettingValue(const enum SettingsOptions option);
```

Decrements a setting to its previous valid value, wrapping around if at minimum.

**Parameters:**
- `option` - The setting to decrement

---

#### isLastSettingValue

```c
bool isLastSettingValue(const enum SettingsOptions option);
```

Checks if a setting is at its maximum valid value.

**Parameters:**
- `option` - The setting to check

**Returns:**
- `true` - Setting is at maximum value (next increment will wrap)
- `false` - Setting has higher values available

---

### Helper Functions

#### lookupVoltageLevel

```c
uint8_t lookupVoltageLevel();
```

Calculates the minimum voltage level based on the current DC voltage settings.

**Returns:**
- Voltage level in tenths of a volt (e.g., 100 = 10.0V)

**Notes:**
- Used for undervoltage protection calculations
- Takes into account cell count settings

---

#### lookupHallEffectThreshold

```c
uint16_t lookupHallEffectThreshold();
```

Returns the hall effect sensor threshold based on sensitivity setting.

**Returns:**
- Threshold value for hall effect sleep detection

---

#### lookupTipName

```c
const char *lookupTipName();  // Only when TIP_TYPE_SUPPORT defined
```

Gets the display name for the currently selected tip type.

**Returns:**
- Pointer to null-terminated string with tip name

---

#### getUserSelectedTipResistance

```c
uint8_t getUserSelectedTipResistance();
```

Returns the resistance value for the user-selected tip type.

**Returns:**
- Resistance in x10 ohms (e.g., 80 = 8.0Ω)
- Returns 0 for auto-detection or when not supported

---

## Settings Ranges

| Setting | Min | Max | Default | Notes |
|---------|-----|-----|---------|-------|
| SolderingTemp | 10 | 450 | 320 | Celsius |
| SleepTemp | 10 | 300 | 150 | Celsius |
| SleepTime | 0 | 15 | 1 | Minutes |
| ShutdownTime | 0 | 60 | 30 | Minutes |
| BoostTemp | 0 | 450 | 420 | Celsius |
| PowerLimit | 0 | 120 | 0 | Watts (0=disabled) |
| OLEDBrightness | 1 | 101 | 13 | Contrast level |
| AnimationSpeed | 0 | 3 | 2 | Off/Slow/Med/Fast |

---

## Thread Safety

Settings access is generally thread-safe for reading. However, when modifying settings:
- Always modify from a single thread (typically GUI thread)
- Call `saveSettings()` only from the GUI thread
- Settings changes take effect immediately in RAM

---

## See Also

- [BSP Flash](../bsp/BSP_Flash.md) - Flash memory operations
- [Translation API](Translation.md) - Settings menu text
- [Operating Modes](../ui/OperatingModes.md) - Settings menu integration

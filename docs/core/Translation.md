# Translation API

The Translation module provides internationalization (i18n) support for IronOS, including multi-language text strings, symbols, and settings descriptions.

**Header File:** `source/Core/Inc/Translation.h`
**Implementation:** `source/Core/Src/Translation.cpp`

---

## Overview

The translation system provides:
- Multi-language UI text support
- Symbol definitions for temperature units, power displays
- Settings menu descriptions
- Font section management for different character sets

---

## Constants

### HasFahrenheit

```c
extern const bool HasFahrenheit;
```

Indicates whether the current language supports Fahrenheit display.

**Values:**
- `true` - Language includes Fahrenheit symbols
- `false` - Celsius only

---

## Symbol Strings

### Temperature Symbols

```c
extern const char *SmallSymbolDegC;   // Small Celsius symbol
extern const char *LargeSymbolDegC;   // Large Celsius symbol
extern const char *SmallSymbolDegF;   // Small Fahrenheit symbol
extern const char *LargeSymbolDegF;   // Large Fahrenheit symbol
```

### Numeric Symbols

```c
extern const char *SmallSymbolPlus;   // Small plus sign
extern const char *LargeSymbolPlus;   // Large plus sign
extern const char *SmallSymbolMinus;  // Small minus sign
extern const char *LargeSymbolMinus;  // Large minus sign
extern const char *SmallSymbolSpace;  // Small space character
extern const char *LargeSymbolSpace;  // Large space character
extern const char *SmallSymbolDot;    // Small decimal point
extern const char *LargeSymbolDot;    // Large decimal point
```

### Unit Symbols

```c
extern const char *SmallSymbolAmps;      // Small ampere symbol
extern const char *LargeSymbolAmps;      // Large ampere symbol
extern const char *SmallSymbolWatts;     // Small watt symbol
extern const char *LargeSymbolWatts;     // Large watt symbol
extern const char *SmallSymbolVolts;     // Small volt symbol
extern const char *LargeSymbolVolts;     // Large volt symbol
extern const char *LargeSymbolMinutes;   // Large minutes symbol
extern const char *SmallSymbolMinutes;   // Small minutes symbol
extern const char *LargeSymbolSeconds;   // Large seconds symbol
extern const char *SmallSymbolSeconds;   // Small seconds symbol
```

### Power Source Symbols

```c
extern const char *LargeSymbolDC;          // DC power symbol
extern const char *SmallSymbolDC;          // Small DC symbol
extern const char *LargeSymbolCellCount;   // Battery cell count
extern const char *SmallSymbolCellCount;   // Small cell count
```

### Special Symbols

```c
extern const char *SmallSymbolSlash;          // Slash separator
extern const char *SmallSymbolColon;          // Colon separator
extern const char *SmallSymbolVersionNumber;  // Version display
extern const char *SmallSymbolPDDebug;        // PD debug label
extern const char *SmallSymbolState;          // State label
extern const char *SmallSymbolNoVBus;         // No VBUS indicator
extern const char *SmallSymbolVBus;           // VBUS indicator
extern const char *LargeSymbolSleep;          // Sleep mode text
```

---

## Menu Arrays

### DebugMenu

```c
extern const char *DebugMenu[];
```

Array of debug menu item labels.

### AccelTypeNames

```c
extern const char *AccelTypeNames[];
```

Array of accelerometer type names for display.

### PowerSourceNames

```c
extern const char *PowerSourceNames[];
```

Array of power source type names (DC, QC, PD, etc.).

---

## Enumerations

### SettingsItemIndex

```c
enum class SettingsItemIndex : uint8_t {
  DCInCutoff,                    // DC input cutoff voltage
  MinVolCell,                    // Minimum cell voltage
  QCMaxVoltage,                  // QC maximum voltage
  PDNegTimeout,                  // PD negotiation timeout
  USBPDMode,                     // USB PD mode selection
  BoostTemperature,              // Boost temperature
  AutoStart,                     // Auto-start mode
  TempChangeShortStep,           // Short press temp step
  TempChangeLongStep,            // Long press temp step
  LockingMode,                   // Button locking mode
  ProfilePhases,                 // Profile phase count
  ProfilePreheatTemp,            // Preheat temperature
  ProfilePreheatSpeed,           // Preheat speed
  ProfilePhase1Temp,             // Phase 1 temperature
  ProfilePhase1Duration,         // Phase 1 duration
  ProfilePhase2Temp,             // Phase 2 temperature
  ProfilePhase2Duration,         // Phase 2 duration
  ProfilePhase3Temp,             // Phase 3 temperature
  ProfilePhase3Duration,         // Phase 3 duration
  ProfilePhase4Temp,             // Phase 4 temperature
  ProfilePhase4Duration,         // Phase 4 duration
  ProfilePhase5Temp,             // Phase 5 temperature
  ProfilePhase5Duration,         // Phase 5 duration
  ProfileCooldownSpeed,          // Cooldown speed
  MotionSensitivity,             // Motion sensitivity
  SleepTemperature,              // Sleep temperature
  SleepTimeout,                  // Sleep timeout
  ShutdownTimeout,               // Shutdown timeout
  HallEffSensitivity,            // Hall effect sensitivity
  HallEffSleepTimeout,           // Hall effect sleep timeout
  TemperatureUnit,               // Celsius/Fahrenheit
  DisplayRotation,               // Screen orientation
  CooldownBlink,                 // Cooldown temp blink
  ScrollingSpeed,                // Menu scroll speed
  ReverseButtonTempChange,       // Reverse +/- buttons
  ReverseButtonSettings,         // Reverse A/B in settings
  AnimSpeed,                     // Animation speed
  AnimLoop,                      // Animation loop
  Brightness,                    // Display brightness
  ColourInversion,               // Display inversion
  LOGOTime,                      // Logo display time
  AdvancedIdle,                  // Advanced idle screen
  AdvancedSoldering,             // Advanced soldering screen
  BluetoothLE,                   // Bluetooth enable
  PowerLimit,                    // Power limit
  CalibrateCJC,                  // CJC calibration
  VoltageCalibration,            // Voltage calibration
  PowerPulsePower,               // Keep-alive pulse power
  PowerPulseWait,                // Keep-alive pulse wait
  PowerPulseDuration,            // Keep-alive pulse duration
  SettingsReset,                 // Reset settings
  LanguageSwitch,                // Language selection
  SolderingTipType,              // Tip type selection
  NUM_ITEMS                      // Total item count
};
```

---

## Structures

### TranslationIndexTable

```c
struct TranslationIndexTable {
  uint16_t CalibrationDone;
  uint16_t ResetOKMessage;
  uint16_t SettingsResetMessage;
  uint16_t NoAccelerometerMessage;
  uint16_t NoPowerDeliveryMessage;
  uint16_t LockingKeysString;
  uint16_t UnlockingKeysString;
  uint16_t WarningKeysLockedString;
  uint16_t WarningThermalRunaway;
  uint16_t WarningTipShorted;
  uint16_t SettingsCalibrationWarning;
  uint16_t CJCCalibrating;
  uint16_t SettingsResetWarning;
  uint16_t UVLOWarningString;
  uint16_t UndervoltageString;
  uint16_t InputVoltageString;
  uint16_t ProfilePreheatString;
  uint16_t ProfileCooldownString;
  uint16_t SleepingAdvancedString;
  uint16_t SleepingTipAdvancedString;
  uint16_t DeviceFailedValidationWarning;
  uint16_t TooHotToStartProfileWarning;

  // Character strings for settings values
  uint16_t SettingRightChar;
  uint16_t SettingLeftChar;
  uint16_t SettingAutoChar;
  uint16_t SettingSlowChar;
  uint16_t SettingMediumChar;
  uint16_t SettingFastChar;
  uint16_t SettingStartSolderingChar;
  uint16_t SettingStartSleepChar;
  uint16_t SettingStartSleepOffChar;
  uint16_t SettingLockBoostChar;
  uint16_t SettingLockFullChar;

  // USB-PD mode strings
  uint16_t USBPDModeDefault;
  uint16_t USBPDModeNoDynamic;
  uint16_t USBPDModeSafe;

  // Tip type strings
  uint16_t TipTypeAuto;
  uint16_t TipTypeT12Long;
  uint16_t TipTypeT12Short;
  uint16_t TipTypeT12PTS;
  uint16_t TipTypeTS80;
  uint16_t TipTypeJBCC210;

  // Settings arrays
  uint16_t SettingsDescriptions[NUM_ITEMS];
  uint16_t SettingsShortNames[NUM_ITEMS];
  uint16_t SettingsMenuEntriesDescriptions[5];
  uint16_t SettingsMenuEntries[5];
};
```

Holds indices into the translation string table for all translatable UI elements.

---

### TranslationData

```c
struct TranslationData {
  TranslationIndexTable indices;
  char strings[1];  // Flexible array of strings
};
```

Complete translation data including index table and string storage.

---

### FontSection

```c
struct FontSection {
  const uint8_t *font12_start_ptr;         // 12pt font data
  const uint8_t *font06_start_ptr;         // 6pt font data
  uint16_t       font12_decompressed_size; // 12pt font size
  uint16_t       font06_decompressed_size; // 6pt font size
  const uint8_t *font12_compressed_source; // Compressed 12pt data
  const uint8_t *font06_compressed_source; // Compressed 6pt data
};
```

Font data section information for the current language.

---

## Global Variables

### Tr

```c
extern const TranslationIndexTable *Tr;
```

Pointer to the current translation index table. Used to look up translated strings.

---

### TranslationStrings

```c
extern const char *TranslationStrings;
```

Pointer to the base of the translation string table.

---

### FontSectionInfo

```c
extern const FontSection FontSectionInfo;
```

Font section information for the current language.

---

## Functions

### translatedString

```c
const char *translatedString(uint16_t index);
```

Retrieves a translated string by its index.

**Parameters:**
- `index` - Index into the translation string table (from `TranslationIndexTable`)

**Returns:**
- Pointer to null-terminated translated string

**Example:**
```c
const char *msg = translatedString(Tr->CalibrationDone);
OLED::print(msg, FontStyle::SMALL);
```

---

### prepareTranslations

```c
void prepareTranslations();
```

Initializes the translation system and decompresses fonts if needed. Called during system startup.

**Actions:**
- Loads selected language data
- Decompresses font data if stored compressed
- Sets up `Tr` and `TranslationStrings` pointers

---

### settings_displayLanguageSwitch

```c
void settings_displayLanguageSwitch(void);
```

Renders the language selection UI in the settings menu.

---

### settings_showLanguageSwitch

```c
bool settings_showLanguageSwitch(void);
```

Checks if language switch option should be displayed.

**Returns:**
- `true` - Show language selection option
- `false` - Hide (single language build)

---

### settings_setLanguageSwitch

```c
void settings_setLanguageSwitch(void);
```

Applies the selected language change. Called when user confirms language selection.

---

### isLastLanguageOption

```c
bool isLastLanguageOption(void);
```

Checks if current selection is the last available language.

**Returns:**
- `true` - No more languages after current
- `false` - More languages available

---

## Macros

### SETTINGS_DESC

```c
#define SETTINGS_DESC(i) (settings_item_index(i) + 1)
```

Macro to get the description index for a settings item.

**Parameters:**
- `i` - `SettingsItemIndex` enum value

**Returns:**
- Index for use with `translatedString()`

**Example:**
```c
const char *desc = translatedString(Tr->SettingsDescriptions[SETTINGS_DESC(SettingsItemIndex::SleepTimeout)]);
```

---

### settings_item_index

```c
constexpr uint8_t settings_item_index(const SettingsItemIndex i);
```

Converts `SettingsItemIndex` enum to numeric index.

---

## Language Support

IronOS supports multiple languages with different font requirements:

| Language | Font Type | Notes |
|----------|-----------|-------|
| English | ASCII | Base language |
| Chinese (Simplified) | Unicode | Compressed fonts |
| Chinese (Traditional) | Unicode | Compressed fonts |
| Japanese | Unicode | Compressed fonts |
| Korean | Unicode | Compressed fonts |
| Russian | Cyrillic | Extended ASCII |
| European languages | Latin-1 | Extended ASCII |

---

## Memory Layout

Translations are stored in flash memory:

```
┌─────────────────────┐
│ TranslationIndexTable│ <- Tr pointer
├─────────────────────┤
│ String 0            │ <- TranslationStrings
├─────────────────────┤
│ String 1            │
├─────────────────────┤
│ ...                 │
├─────────────────────┤
│ String N            │
├─────────────────────┤
│ Font Data (12pt)    │ <- FontSectionInfo
├─────────────────────┤
│ Font Data (6pt)     │
└─────────────────────┘
```

---

## Example Usage

### Displaying Translated Text

```c
// Get and display a message
const char *msg = translatedString(Tr->CalibrationDone);
OLED::print(msg, FontStyle::SMALL);

// Display with symbols
OLED::print("300", FontStyle::LARGE);
OLED::print(SmallSymbolDegC, FontStyle::SMALL);
```

### Settings Menu Integration

```c
// Get setting name and description
SettingsItemIndex item = SettingsItemIndex::SleepTimeout;
const char *name = translatedString(Tr->SettingsShortNames[static_cast<uint8_t>(item)]);
const char *desc = translatedString(Tr->SettingsDescriptions[static_cast<uint8_t>(item)]);
```

---

## See Also

- [OLED Display](../drivers/OLED.md) - Display rendering
- [Settings API](Settings.md) - Settings management
- [Operating Modes](../ui/OperatingModes.md) - UI text display

# Operating Modes

The Operating Modes module defines the UI state machine and mode handlers for IronOS.

**Header File:** `source/Core/Threads/UI/logic/OperatingModes.h`
**Implementation:** `source/Core/Threads/UI/logic/*.cpp`

---

## Overview

IronOS uses a state machine to manage different operating modes:
- Each mode has a dedicated handler function
- Mode transitions are triggered by buttons or timeouts
- The GUI thread runs the current mode handler each frame

---

## OperatingMode Enum

```c
enum class OperatingMode {
    StartupLogo       = 10,  // Boot logo display
    CJCCalibration    = 11,  // Cold junction calibration
    StartupWarnings   = 12,  // Startup warnings
    InitialisationDone = 13, // Transition to home screen
    HomeScreen        = 0,   // Idle/home screen
    Soldering         = 1,   // Active soldering
    SolderingProfile  = 6,   // Profile/reflow mode
    Sleeping          = 3,   // Sleep mode
    Hibernating       = 14,  // Deep sleep (heater off)
    SettingsMenu      = 4,   // Settings navigation
    DebugMenuReadout  = 5,   // Debug information
    TemperatureAdjust = 7,   // Temperature adjustment
    UsbPDDebug        = 8,   // USB-PD debug info
    ThermalRunaway    = 9,   // Thermal runaway error
};
```

---

## TransitionAnimation Enum

```c
enum class TransitionAnimation {
    None  = 0,  // Instant transition
    Right = 1,  // Slide from left to right
    Left  = 2,  // Slide from right to left
    Down  = 3,  // Slide from top to bottom
    Up    = 4,  // Slide from bottom to top
};
```

---

## GUI Context

The `guiContext` structure maintains state across frames:

```c
struct guiContext {
    TickType_t viewEnterTime;           // When current mode started
    OperatingMode previousMode;          // For back navigation
    TransitionAnimation transitionMode;  // Active animation

    struct scratch {
        uint16_t state1;  // Mode-specific scratch space
        uint16_t state2;
        uint32_t state3;
        uint32_t state4;
        uint16_t state5;
        uint16_t state6;
        uint32_t state7;
    } scratch_state;
};
```

---

## Mode Handlers

Each mode has a handler function with this signature:

```c
OperatingMode handler(const ButtonState buttons, guiContext *cxt);
```

**Parameters:**
- `buttons` - Current button state
- `cxt` - GUI context for timing and scratch state

**Returns:**
- Next operating mode (same mode to stay, different to transition)

---

### HomeScreen (Idle)

```c
OperatingMode drawHomeScreen(const ButtonState buttons, guiContext *cxt);
```

The main idle screen displayed when not soldering.

**Display:**
- Current tip temperature
- Input voltage
- Power source indicator
- Sleep/shutdown status

**Transitions:**
| Button | Next Mode |
|--------|-----------|
| Front Short | Soldering |
| Front Long | SolderingProfile |
| Back Short | SettingsMenu |
| Both | DebugMenuReadout |

---

### Soldering

```c
OperatingMode gui_solderingMode(const ButtonState buttons, guiContext *cxt);
```

Active soldering mode with temperature control.

**Display:**
- Target and current temperature
- Power output
- Heating indicator

**Transitions:**
| Condition | Next Mode |
|-----------|-----------|
| Back Short | HomeScreen |
| Sleep timeout | Sleeping |
| Both buttons | Boost (temporary) |
| Front Long | TemperatureAdjust |

---

### Sleeping

```c
OperatingMode gui_SolderingSleepingMode(const ButtonState buttons, guiContext *cxt);
```

Low-power sleep mode with reduced temperature.

**Display:**
- Sleep indicator
- Current temperature
- "Zzz" animation

**Transitions:**
| Condition | Next Mode |
|-----------|-----------|
| Any button | Soldering |
| Motion detected | Soldering |
| Shutdown timeout | Hibernating |

---

### TemperatureAdjust

```c
OperatingMode gui_solderingTempAdjust(const ButtonState buttons, guiContext *cxt);
```

Overlay for adjusting target temperature.

**Display:**
- Large temperature display
- Up/down indicators
- Current setpoint

**Button Actions:**
| Button | Action |
|--------|--------|
| Front Short | +1 step |
| Front Long | +10 step |
| Back Short | -1 step |
| Back Long | -10 step |
| Timeout | Return to previous |

---

### SettingsMenu

```c
OperatingMode gui_SettingsMenu(const ButtonState buttons, guiContext *cxt);
```

Settings menu navigation.

**Display:**
- Menu item name
- Current value
- Scroll indicator

**Button Actions:**
| Button | Action |
|--------|--------|
| Front Short | Next item |
| Back Short | Change value |
| Front Long | Enter submenu |
| Back Long | Exit menu |

---

### SolderingProfile

```c
OperatingMode gui_solderingProfileMode(const ButtonState buttons, guiContext *cxt);
```

Profile mode for reflow soldering.

**Phases:**
1. Preheat - Ramp to preheat temperature
2. Phase 1-5 - Follow temperature profile
3. Cooldown - Controlled cooling

**Display:**
- Current phase indicator
- Temperature graph/progress
- Time remaining

---

### DebugMenuReadout

```c
OperatingMode showDebugMenu(const ButtonState buttons, guiContext *cxt);
```

Debug information display.

**Pages:**
- Temperature readings (raw ADC, calibrated)
- Voltage and power data
- Accelerometer readings
- Hardware version info

---

### UsbPDDebug

```c
OperatingMode showPDDebug(const ButtonState buttons, guiContext *cxt);
```

USB Power Delivery debug information.

**Display:**
- PD state number
- Negotiated voltage/current
- Source capabilities
- Contract details

---

### ThermalRunaway

```c
OperatingMode showThermalRunaway(const ButtonState buttons, guiContext *cxt);
```

Thermal runaway error screen.

**Display:**
- Error message
- Current temperature
- Heater disabled indicator

**Recovery:**
- Requires power cycle
- Heater remains disabled

---

### CJC Calibration

```c
OperatingMode performCJCC(const ButtonState buttons, guiContext *cxt);
```

Cold Junction Compensation calibration.

**Process:**
1. Display instructions
2. Wait for stable temperature
3. Calculate offset
4. Save calibration

---

### Startup Warnings

```c
OperatingMode showWarnings(const ButtonState buttons, guiContext *cxt);
```

Displays startup warnings if needed.

**Warnings:**
- No accelerometer detected
- No PD detected (if expected)
- Settings were reset
- Device validation failure

---

## Helper Functions

### getPowerSourceNumber

```c
int8_t getPowerSourceNumber(void);
```

Returns the current power source identifier.

**Returns:**
| Value | Source |
|-------|--------|
| -1 | Unknown/5V USB |
| 0 | DC jack |
| 1 | QC 3.0 |
| 2 | USB-PD |

---

## Global Variables

### heaterThermalRunawayCounter

```c
extern uint8_t heaterThermalRunawayCounter;
```

Counts thermal runaway events for detection.

---

## Mode Transition Diagram

```
                    ┌────────────────┐
                    │  StartupLogo   │
                    └───────┬────────┘
                            │
                            v
                    ┌────────────────┐
              ┌─────│   Warnings     │─────┐
              │     └───────┬────────┘     │
              │             │              │
              v             v              v
        ┌──────────┐  ┌──────────┐  ┌──────────────┐
        │   CJC    │  │  Home    │  │    Debug     │
        │  Calib   │  │  Screen  │  │    Menu      │
        └────┬─────┘  └────┬─────┘  └──────────────┘
             │             │
             │             │
             v             │
        ┌──────────┐       │
        │ Home     │<──────┤
        │ Screen   │       │
        └────┬─────┘       │
             │             │
     ┌───────┼───────┐     │
     │       │       │     │
     v       v       v     v
┌────────┐ ┌────────┐ ┌────────┐
│Soldering│ │Settings│ │Profile │
└────┬───┘ └────────┘ └────┬───┘
     │                     │
     v                     v
┌────────┐           ┌────────┐
│Sleeping│           │Cooldown│
└────┬───┘           └────────┘
     │
     v
┌────────────┐
│Hibernating │
└────────────┘
```

---

## Example: Custom Mode

```c
OperatingMode myCustomMode(const ButtonState buttons, guiContext *cxt) {
    // Initialize on first entry
    if (cxt->scratch_state.state1 == 0) {
        cxt->scratch_state.state1 = 1;  // Mark initialized
        // Additional initialization...
    }

    // Clear and draw
    OLED::clearScreen();
    OLED::setCursor(0, 0);
    OLED::print("Custom Mode", FontStyle::SMALL);

    // Handle buttons
    switch (buttons) {
        case BUTTON_B_SHORT:
            return OperatingMode::HomeScreen;
        case BUTTON_F_SHORT:
            // Do something
            break;
    }

    return OperatingMode::MyCustomMode;  // Stay in this mode
}
```

---

## See Also

- [GUI Thread](../threading/GUIThread.md) - Thread implementation
- [Button Handling](Buttons.md) - Button states
- [Settings API](../core/Settings.md) - Settings integration
- [OLED Driver](../drivers/OLED.md) - Display functions

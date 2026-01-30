# GUI Thread

The GUI thread handles all user interface operations including display rendering, button input processing, and operating mode management.

**Entry Point:** `startGUITask()` in `source/Core/Inc/main.hpp`
**Implementation:** `source/Core/Threads/GUIThread.cpp`

---

## Overview

The GUI thread is responsible for:
- OLED display updates
- Button input detection and processing
- Operating mode state machine
- Settings menu navigation
- Animation and transitions
- User feedback (messages, warnings)

---

## Thread Function

### startGUITask

```c
void startGUITask(void const *argument);
```

Entry point for the GUI thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Main Loop:**
```c
void startGUITask(void const *argument) {
    // Initialization
    OLED::initialize();
    prepareTranslations();

    while (1) {
        // Get button state
        ButtonState buttons = getButtonState();

        // Run current mode handler
        OperatingMode nextMode = runCurrentMode(buttons);

        // Handle mode transitions
        if (nextMode != currentMode) {
            transitionToMode(nextMode);
        }

        // Refresh display
        OLED::refresh();

        // Delay for ~60fps
        osDelay(TICKS_10MS);
    }
}
```

---

## Operating Mode State Machine

The GUI uses a state machine to manage operating modes:

```
┌──────────────┐
│ StartupLogo  │ ──────────────────┐
└──────┬───────┘                   │
       │ Logo complete             │
       v                           │
┌──────────────┐                   │
│ CJC Calib    │ (if enabled)      │
└──────┬───────┘                   │
       │                           │
       v                           │
┌──────────────┐                   │
│ Warnings     │ (if any)          │
└──────┬───────┘                   │
       │                           │
       v                           │
┌──────────────┐  Front btn  ┌─────────────────┐
│  HomeScreen  │ ──────────> │   Soldering     │
│   (Idle)     │ <────────── │                 │
└──────┬───────┘  Timeout    └─────────────────┘
       │                           │
       │ Back btn                  │ Sleep timeout
       v                           v
┌──────────────┐             ┌─────────────────┐
│ SettingsMenu │             │   Sleeping      │
└──────────────┘             └─────────────────┘
```

---

## Mode Handlers

Each operating mode has a dedicated handler function:

### drawHomeScreen

```c
OperatingMode drawHomeScreen(const ButtonState buttons, guiContext *cxt);
```

Handles the idle/home screen display and transitions.

**Inputs:**
- `buttons` - Current button state
- `cxt` - GUI context (timing, scratch state)

**Returns:**
- Next operating mode

**Behavior:**
- Displays current temperature, input voltage
- Handles button navigation
- Starts soldering on front button
- Opens settings on back button

---

### gui_solderingMode

```c
OperatingMode gui_solderingMode(const ButtonState buttons, guiContext *cxt);
```

Main soldering mode handler.

**Features:**
- Temperature display (current and target)
- Power and voltage indicators
- Boost mode on long button press
- Temperature adjustment
- Sleep timeout detection

---

### gui_SolderingSleepingMode

```c
OperatingMode gui_SolderingSleepingMode(const ButtonState buttons, guiContext *cxt);
```

Sleep mode handler.

**Features:**
- Reduced temperature display
- Motion-based wake detection
- Button-based wake detection
- Shutdown timeout management

---

### gui_solderingTempAdjust

```c
OperatingMode gui_solderingTempAdjust(const ButtonState buttons, guiContext *cxt);
```

Temperature adjustment overlay.

**Features:**
- Live temperature adjustment
- Short/long press step sizes
- Visual feedback
- Save on exit

---

### gui_SettingsMenu

```c
OperatingMode gui_SettingsMenu(const ButtonState buttons, guiContext *cxt);
```

Settings menu navigation.

**Features:**
- Scrollable menu list
- Category organization
- Value adjustment
- Save/cancel handling

---

### gui_solderingProfileMode

```c
OperatingMode gui_solderingProfileMode(const ButtonState buttons, guiContext *cxt);
```

Profile (reflow) mode handler.

**Features:**
- Multi-phase temperature profiles
- Progress indication
- Preheat and cooldown phases
- Manual override capability

---

### showDebugMenu

```c
OperatingMode showDebugMenu(const ButtonState buttons, guiContext *cxt);
```

Debug information display.

**Features:**
- Temperature readings
- Voltage and power data
- Accelerometer data
- Hardware information

---

### showPDDebug

```c
OperatingMode showPDDebug(const ButtonState buttons, guiContext *cxt);
```

USB Power Delivery debug display.

**Features:**
- PD negotiation state
- Source capabilities
- Current contract details
- Voltage and current readings

---

## GUI Context

The `guiContext` structure maintains state across frames:

```c
struct guiContext {
    TickType_t viewEnterTime;      // When current view started
    OperatingMode previousMode;     // For back navigation
    TransitionAnimation transitionMode; // Active animation

    struct scratch {
        uint16_t state1;  // Mode-specific scratch
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

## Button Processing

Button states are processed each frame:

```c
ButtonState getButtonState();
```

### Button States

| State | Meaning |
|-------|---------|
| `BUTTON_NONE` | No button pressed |
| `BUTTON_F_SHORT` | Front button short press |
| `BUTTON_B_SHORT` | Back button short press |
| `BUTTON_F_LONG` | Front button held |
| `BUTTON_B_LONG` | Back button held |
| `BUTTON_BOTH` | Both buttons pressed |
| `BUTTON_BOTH_LONG` | Both buttons held |

### Button Timing

```c
extern TickType_t lastButtonTime;  // Last button activity

// Check for button timeout
if (xTaskGetTickCount() - lastButtonTime > BUTTON_TIMEOUT) {
    // No recent button activity
}
```

---

## Display Updates

### Frame Rate

The GUI targets approximately 60 frames per second:

```c
// Main loop timing
osDelay(TICKS_10MS);  // ~100Hz check rate

// Display refresh only when changed
OLED::refresh();  // Checksums buffer, sends only if changed
```

### Animations

Animations use the `viewEnterTime` for timing:

```c
void drawAnimation(guiContext *cxt) {
    TickType_t elapsed = xTaskGetTickCount() - cxt->viewEnterTime;
    uint8_t frame = (elapsed / TICKS_50MS) % FRAME_COUNT;
    drawAnimationFrame(frame);
}
```

### Transitions

Mode transitions can include animations:

```c
enum class TransitionAnimation {
    None  = 0,  // Instant switch
    Right = 1,  // Slide right
    Left  = 2,  // Slide left
    Down  = 3,  // Slide down
    Up    = 4   // Slide up
};
```

---

## Thread Safety

The GUI thread owns:
- OLED display (exclusive access)
- Button input (primary handler)
- Settings modification

Other threads should not:
- Call OLED functions directly
- Modify settings during GUI operations

---

## Resource Usage

### Stack Size

GUI thread has the largest stack (1024 words) due to:
- String formatting buffers
- Menu rendering
- Animation calculations

### CPU Usage

Typical: 5-15% depending on display activity
Peak: 30% during complex animations

---

## Example: Custom Screen

```c
OperatingMode customScreen(const ButtonState buttons, guiContext *cxt) {
    // Clear display
    OLED::clearScreen();

    // Draw content
    OLED::setCursor(0, 0);
    OLED::print("Custom Screen", FontStyle::SMALL);

    OLED::setCursor(0, 12);
    OLED::printNumber(someValue, 4, FontStyle::LARGE);

    // Handle buttons
    if (buttons == BUTTON_B_SHORT) {
        return OperatingMode::HomeScreen;
    }

    return OperatingMode::CustomScreen;
}
```

---

## See Also

- [Operating Modes](../ui/OperatingModes.md) - Mode definitions
- [Button Handling](../ui/Buttons.md) - Button state machine
- [OLED Driver](../drivers/OLED.md) - Display functions
- [Threading Overview](Overview.md) - Thread architecture

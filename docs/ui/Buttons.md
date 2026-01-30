# Button Handling

The Button handling module provides button state detection and input processing for IronOS.

**Header File:** `source/Core/Drivers/Buttons.hpp`
**Implementation:** `source/Core/Drivers/Buttons.cpp`

---

## Overview

IronOS uses two physical buttons for all user input:
- **Front Button (A)** - Primary action, typically "select" or "increase"
- **Back Button (B)** - Secondary action, typically "back" or "decrease"

The button module provides:
- Debounced button state detection
- Short press vs long press differentiation
- Dual-button press detection
- Activity tracking for sleep timeout

---

## ButtonState Enum

```c
enum ButtonState {
    BUTTON_NONE      = 0,   // No button pressed
    BUTTON_F_SHORT   = 1,   // Front button short press
    BUTTON_B_SHORT   = 2,   // Back button short press
    BUTTON_F_LONG    = 4,   // Front button held
    BUTTON_B_LONG    = 8,   // Back button held
    BUTTON_BOTH      = 16,  // Both buttons pressed
    BUTTON_BOTH_LONG = 32,  // Both buttons held
};
```

### State Definitions

| State | Physical Action | Timing |
|-------|-----------------|--------|
| `BUTTON_NONE` | No button activity | - |
| `BUTTON_F_SHORT` | Front pressed and released | < hold threshold |
| `BUTTON_B_SHORT` | Back pressed and released | < hold threshold |
| `BUTTON_F_LONG` | Front held down | > hold threshold |
| `BUTTON_B_LONG` | Back held down | > hold threshold |
| `BUTTON_BOTH` | Both pressed together | Short press |
| `BUTTON_BOTH_LONG` | Both held together | > hold threshold |

---

## Functions

### getButtonState

```c
ButtonState getButtonState();
```

Returns the current button state after debouncing and timing analysis.

**Returns:**
- Current `ButtonState` value

**Notes:**
- Called once per GUI frame
- Implements debouncing internally
- Tracks press/release timing for short vs long detection

**Example:**
```c
ButtonState buttons = getButtonState();

switch (buttons) {
    case BUTTON_F_SHORT:
        // Handle front button short press
        break;
    case BUTTON_B_LONG:
        // Handle back button long hold
        break;
}
```

---

### waitForButtonPressOrTimeout

```c
void waitForButtonPressOrTimeout(TickType_t timeout);
```

Blocks until a button is pressed or timeout expires.

**Parameters:**
- `timeout` - Maximum wait time in FreeRTOS ticks

**Usage:**
```c
// Wait up to 5 seconds for button
waitForButtonPressOrTimeout(TICKS_SECOND * 5);
```

---

### waitForButtonPress

```c
void waitForButtonPress();
```

Blocks until any button is pressed (no timeout).

**Usage:**
```c
// Show message and wait for acknowledgment
OLED::print("Press any button", FontStyle::SMALL);
OLED::refresh();
waitForButtonPress();
```

---

## Global Variables

### lastButtonTime

```c
extern TickType_t lastButtonTime;
```

FreeRTOS tick count of the last button activity. Used for:
- Sleep timeout calculation
- Activity detection

**Example:**
```c
// Check if buttons inactive for 1 minute
if (xTaskGetTickCount() - lastButtonTime > TICKS_SECOND * 60) {
    // No button activity for 1 minute
}
```

---

## Timing Parameters

### Short Press

A short press is detected when:
1. Button goes from released to pressed
2. Button goes from pressed to released
3. Total press duration < long press threshold

```
          ┌──────┐
Press:  ──┘      └──────
          <SHORT>

Returns BUTTON_x_SHORT on release
```

### Long Press

A long press is detected when:
1. Button goes from released to pressed
2. Button remains pressed > long press threshold
3. Continues returning LONG state while held

```
          ┌──────────────────
Press:  ──┘
          <──THRESH──>
                     ↑
                     Returns BUTTON_x_LONG
```

### Typical Thresholds

| Threshold | Value | Purpose |
|-----------|-------|---------|
| Debounce | 20ms | Filter mechanical bounce |
| Long Press | 500ms | Differentiate short/long |
| Repeat | 100ms | Auto-repeat while held |

---

## Button Detection Flow

```
┌─────────────────┐
│ Read Raw GPIO   │ getButtonA(), getButtonB()
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Debounce Filter │ Require stable state
└────────┬────────┘
         │
         v
┌─────────────────┐
│ State Machine   │ Track press/release
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Timing Analysis │ Short vs Long
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Return State    │ ButtonState enum
└─────────────────┘
```

---

## State Machine

```
                IDLE
                 │
     ┌───────────┼───────────┐
     │           │           │
     v           v           v
  A_DOWN      B_DOWN     BOTH_DOWN
     │           │           │
     │           │           │
┌────┴────┐ ┌────┴────┐ ┌────┴────┐
│ Timer   │ │ Timer   │ │ Timer   │
│ Running │ │ Running │ │ Running │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
┌────┴────┐ ┌────┴────┐ ┌────┴────┐
│ Release?│ │ Release?│ │ Release?│
│ Timeout?│ │ Timeout?│ │ Timeout?│
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     v           v           v
A_SHORT/LONG B_SHORT/LONG BOTH_SHORT/LONG
```

---

## Usage Patterns

### Simple Navigation

```c
OperatingMode handleButtons(ButtonState buttons) {
    switch (buttons) {
        case BUTTON_F_SHORT:
            return nextScreen();
        case BUTTON_B_SHORT:
            return previousScreen();
        case BUTTON_F_LONG:
            return enterSubmenu();
        case BUTTON_B_LONG:
            return exitToHome();
    }
    return currentMode;
}
```

### Value Adjustment

```c
void adjustValue(ButtonState buttons, int16_t *value) {
    int16_t step = getSettingValue(TempChangeShortStep);
    int16_t longStep = getSettingValue(TempChangeLongStep);

    switch (buttons) {
        case BUTTON_F_SHORT:
            *value += step;
            break;
        case BUTTON_F_LONG:
            *value += longStep;
            break;
        case BUTTON_B_SHORT:
            *value -= step;
            break;
        case BUTTON_B_LONG:
            *value -= longStep;
            break;
    }
}
```

### Confirmation Dialog

```c
bool confirmAction() {
    OLED::clearScreen();
    OLED::print("Confirm?", FontStyle::LARGE);
    OLED::print("A=Yes B=No", FontStyle::SMALL);
    OLED::refresh();

    while (1) {
        ButtonState buttons = getButtonState();
        if (buttons == BUTTON_F_SHORT) {
            return true;   // Confirmed
        }
        if (buttons == BUTTON_B_SHORT) {
            return false;  // Cancelled
        }
        osDelay(TICKS_10MS);
    }
}
```

---

## Button Locking

The `LockingMode` setting controls button behavior:

```c
typedef enum {
    DISABLED = 0,  // No locking
    BOOST    = 1,  // Lock only affects boost
    FULL     = 2,  // Lock in soldering mode too
} lockingMode_t;
```

### Lock/Unlock Sequence

```c
// Lock: Hold both buttons
if (buttons == BUTTON_BOTH_LONG) {
    buttonsLocked = true;
    showMessage("Locked");
}

// Unlock: Hold both buttons again
if (buttonsLocked && buttons == BUTTON_BOTH_LONG) {
    buttonsLocked = false;
    showMessage("Unlocked");
}
```

---

## Reversed Button Settings

Two settings allow button reversal:

### ReverseButtonTempChangeEnabled

Swaps +/- buttons for temperature adjustment:
- Normal: Front = +, Back = -
- Reversed: Front = -, Back = +

### ReverseButtonSettings

Swaps A/B buttons in settings menu:
- Normal: Front = next, Back = change value
- Reversed: Front = change value, Back = next

---

## See Also

- [BSP Core](../bsp/BSP.md) - `getButtonA()`, `getButtonB()`
- [Operating Modes](OperatingModes.md) - Button handling in modes
- [GUI Thread](../threading/GUIThread.md) - Button processing loop
- [Settings API](../core/Settings.md) - Button-related settings

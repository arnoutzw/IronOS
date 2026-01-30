# UI Drawing

The Drawing module provides screen rendering functions for different display sizes and UI elements.

**Header File:** `source/Core/Threads/UI/drawing/ui_drawing.hpp`
**Implementation:** `source/Core/Threads/UI/drawing/mono_*/*.cpp`

---

## Overview

IronOS supports two OLED display sizes:
- **128x32 pixels** - Used in Pinecil V2, MHP30
- **96x16 pixels** - Used in TS100, TS80, Pinecil V1

Each display size has dedicated drawing modules optimized for the available resolution.

---

## Display Configurations

### 128x32 Display

```
┌────────────────────────────────────────────────────────────┐
│ Row 0-7 (8 pixels)                                         │
├────────────────────────────────────────────────────────────┤
│ Row 8-15 (8 pixels)                                        │
├────────────────────────────────────────────────────────────┤
│ Row 16-23 (8 pixels)                                       │
├────────────────────────────────────────────────────────────┤
│ Row 24-31 (8 pixels)                                       │
└────────────────────────────────────────────────────────────┘
  <────────────── 128 pixels ──────────────>
```

### 96x16 Display

```
┌──────────────────────────────────────────────────┐
│ Row 0-7 (8 pixels)                               │
├──────────────────────────────────────────────────┤
│ Row 8-15 (8 pixels)                              │
└──────────────────────────────────────────────────┘
  <────────────── 96 pixels ──────────────>
```

---

## Drawing Modules

### Directory Structure

```
drawing/
├── ui_drawing.hpp           # Common interface
├── mono_128x32/             # 128x32 specific
│   ├── draw_battery.cpp
│   ├── draw_debug_menu.cpp
│   ├── draw_home_screen.cpp
│   ├── draw_power.cpp
│   ├── draw_profile.cpp
│   ├── draw_settings.cpp
│   ├── draw_soldering.cpp
│   ├── draw_temp_adjust.cpp
│   ├── draw_tip_temp.cpp
│   ├── draw_usb_pd_debug.cpp
│   ├── draw_voltage.cpp
│   ├── draw_warnings.cpp
│   └── prerender_assets.cpp
└── mono_96x16/              # 96x16 specific
    └── (same files)
```

---

## Common Drawing Functions

### Temperature Display

```c
void drawTipTemperature(TemperatureType_t temp, FontStyle style);
```

Draws the tip temperature with appropriate units.

**Parameters:**
- `temp` - Temperature value
- `style` - Font size

**Example:**
```c
drawTipTemperature(350, FontStyle::LARGE);
// Displays "350°C" or "662°F"
```

---

### Power Display

```c
void drawPowerStatus(uint16_t watts);
```

Draws the current power output.

**Parameters:**
- `watts` - Power in tenths of watts

---

### Voltage Display

```c
void drawInputVoltage(uint16_t volts);
```

Draws the input voltage.

**Parameters:**
- `volts` - Voltage in tenths of volts

---

### Battery/Power Source

```c
void drawPowerSourceIcon();
```

Draws an icon indicating the current power source:
- Battery icon for DC
- QC icon for Quick Charge
- PD icon for USB Power Delivery

---

## Screen Layouts

### Home Screen (128x32)

```
┌────────────────────────────────────────────────┐
│ [PWR] Temperature              [BAT/PD]        │
│       ███  350°C                               │
│       ███                                      │
│ 24.0V     65W                Sleep: 2:30      │
└────────────────────────────────────────────────┘
```

### Home Screen (96x16)

```
┌────────────────────────────────────────┐
│ 350°C              24.0V  [PD]        │
│ 65W                       Sleep       │
└────────────────────────────────────────┘
```

### Soldering Screen

```
┌────────────────────────────────────────────────┐
│ [HEAT]  Target: 350°C                          │
│         Current: 348°C  ████████░░  65W       │
└────────────────────────────────────────────────┘
```

### Settings Menu

```
┌────────────────────────────────────────────────┐
│ Sleep Temp                            ▲        │
│ 150°C                                 █        │
│ Temperature the tip drops to in      ░        │
│ sleep mode                            ▼        │
└────────────────────────────────────────────────┘
```

---

## Drawing Primitives

### Text Drawing

```c
OLED::setCursor(x, y);
OLED::print("Text", FontStyle::SMALL);
OLED::printNumber(value, digits, FontStyle::LARGE);
```

### Rectangle Drawing

```c
OLED::drawFilledRect(x0, y0, x1, y1, fill);
```

### Image/Icon Drawing

```c
OLED::drawArea(x, y, width, height, imageData);
OLED::drawSymbol(symbolId);
```

---

## Animation System

### Frame Timing

```c
void drawAnimation(guiContext *cxt) {
    TickType_t elapsed = xTaskGetTickCount() - cxt->viewEnterTime;
    uint8_t frame = (elapsed / TICKS_50MS) % ANIMATION_FRAMES;

    drawAnimationFrame(frame);
}
```

### Heat Indicator

```c
void drawHeatIndicator(bool heating) {
    static uint8_t frame = 0;

    if (heating) {
        OLED::drawHeatSymbol(frame);
        frame = (frame + 1) % 4;
    }
}
```

---

## Pre-rendered Assets

Large or complex graphics are pre-rendered at compile time:

```c
// prerender_assets.cpp
const uint8_t BootLogoData[] = {
    // Bitmap data
};

const uint8_t BatteryIcons[][ICON_SIZE] = {
    // Battery level 0
    { ... },
    // Battery level 1
    { ... },
    // etc.
};
```

---

## Detailed vs Simple Display

Settings control display detail level:

### DetailedIDLE

When enabled, home screen shows:
- Input voltage
- Tip voltage (µV)
- Handle temperature
- Power graph

### DetailedSoldering

When enabled, soldering screen shows:
- PWM duty cycle
- Power consumption
- PID state information

---

## Color Inversion

The `OLEDInversion` setting inverts display colors:

```c
if (getSettingValue(OLEDInversion)) {
    OLED::setInverseDisplay(true);
}
```

---

## Scroll Indicators

For menus with more items than screen height:

```c
void drawScrollIndicator(uint8_t position, uint8_t total) {
    // Calculate indicator position
    uint8_t barHeight = OLED_HEIGHT / total;
    uint8_t barY = position * barHeight;

    // Draw on right edge
    OLED::drawFilledRect(
        OLED_WIDTH - 2, barY,
        OLED_WIDTH - 1, barY + barHeight,
        true
    );
}
```

---

## Screen Transitions

### Slide Transitions

```c
void transitionSlideLeft(guiContext *cxt) {
    TickType_t elapsed = xTaskGetTickCount() - cxt->viewEnterTime;
    int16_t offset = OLED_WIDTH - (elapsed * SLIDE_SPEED);

    if (offset > 0) {
        // Draw old screen shifted right
        // Draw new screen at normal position
    }
}
```

### Double Buffering

For smooth transitions:

```c
// Draw to secondary buffer
OLED::useSecondaryFramebuffer(true);
drawNewScreen();

// Animate transition
OLED::transitionSecondaryFramebuffer(true, cxt->viewEnterTime);
```

---

## Example: Custom Screen Drawing

```c
void drawMyCustomScreen(guiContext *cxt, int32_t value) {
    OLED::clearScreen();

    // Title bar
    OLED::setCursor(0, 0);
    OLED::print("Custom View", FontStyle::SMALL);

    // Horizontal line
    OLED::fillArea(0, 10, OLED_WIDTH, 1, 0xFF);

    // Main content
    OLED::setCursor(10, 14);
    OLED::printNumber(value, 5, FontStyle::LARGE);

    // Status icon
    if (value > 100) {
        OLED::drawSymbol(SYMBOL_CHECK);
    } else {
        OLED::drawSymbol(SYMBOL_WARNING);
    }
}
```

---

## See Also

- [OLED Driver](../drivers/OLED.md) - Low-level display functions
- [Operating Modes](OperatingModes.md) - Screen usage
- [Translation API](../core/Translation.md) - Text and symbols
- [GUI Thread](../threading/GUIThread.md) - Rendering loop

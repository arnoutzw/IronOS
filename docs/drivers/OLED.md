# OLED Display Driver

The OLED driver provides a complete interface for controlling SSD1307-based OLED displays, including rendering, text output, and graphics primitives.

**Header File:** `source/Core/Drivers/OLED.hpp`
**Implementation:** `source/Core/Drivers/OLED.cpp`

---

## Overview

The OLED driver supports two display sizes:
- **128x32 pixels** - Used in Pinecil V2, MHP30
- **96x16 pixels** - Used in TS100, TS80, Pinecil V1

All drawing operations work on an internal framebuffer, which is transmitted to the display via I2C when `refresh()` is called.

---

## Constants

### Display Configuration

```c
#define DEVICEADDR_OLED (0x3c << 1)  // I2C address (0x78)
#define FRAMEBUFFER_START 17          // Offset in buffer for pixel data

// For 128x32 displays
#define OLED_WIDTH  128
#define OLED_HEIGHT 32

// For 96x16 displays
#define OLED_WIDTH  96
#define OLED_HEIGHT 16
```

### Display Commands

```c
#define OLED_ON  0xAF  // Turn display on
#define OLED_OFF 0xAE  // Turn display off
```

---

## Enumerations

### FontStyle

```c
enum class FontStyle {
  SMALL,   // 6-pixel height font
  LARGE,   // 12-pixel height font
  EXTRAS   // Special symbol font
};
```

Selects the font size for text rendering operations.

### DisplayState

```c
enum DisplayState : bool {
  OFF = false,  // Display powered off
  ON = true     // Display powered on
};
```

---

## Class: OLED

The `OLED` class provides static methods for all display operations. No instantiation is required.

---

## Initialization Functions

### initialize

```c
static void initialize();
```

Initializes the OLED display hardware and I2C communication.

**Actions:**
- Configures I2C communication
- Sends display initialization sequence
- Clears framebuffer
- Powers on display

**Usage:**
```c
OLED::initialize();
// Display is now ready for use
```

---

### isInitDone

```c
static bool isInitDone();
```

Checks if display initialization is complete.

**Returns:**
- `true` - Display is initialized and ready
- `false` - Still initializing

---

## Display Control

### refresh

```c
static void refresh();
```

Transmits the framebuffer to the display if changes were made.

**Behavior:**
- Computes checksum of framebuffer
- Only transmits if data changed since last refresh
- Uses DMA for efficient transfer (~20ms)

**Important:**
- Allow at least 25ms between refresh calls
- Call after all drawing operations complete

**Example:**
```c
OLED::clearScreen();
OLED::print("Hello", FontStyle::LARGE);
OLED::refresh();  // Send to display
```

---

### setDisplayState

```c
static void setDisplayState(DisplayState state);
```

Powers the display on or off.

**Parameters:**
- `state` - `DisplayState::ON` or `DisplayState::OFF`

**Example:**
```c
OLED::setDisplayState(DisplayState::OFF);  // Power save
osDelay(TICKS_SECOND * 30);
OLED::setDisplayState(DisplayState::ON);   // Wake up
```

---

### setBrightness

```c
static void setBrightness(uint8_t contrast);
```

Sets the display contrast/brightness level.

**Parameters:**
- `contrast` - Brightness level (0-255, higher = brighter)

**Notes:**
- Typical range is 0-127 for normal use
- Higher values may reduce display lifespan

---

### setInverseDisplay

```c
static void setInverseDisplay(bool inverted);
```

Inverts the display colors.

**Parameters:**
- `inverted` - `true` for inverted (white on black becomes black on white)

---

### setRotation

```c
static void setRotation(bool leftHanded);
```

Sets the screen orientation for left or right-handed use.

**Parameters:**
- `leftHanded` - `true` for left-handed orientation (180° rotation)

---

### getRotation

```c
static bool getRotation();
```

Gets the current screen orientation.

**Returns:**
- `true` - Left-handed mode
- `false` - Right-handed mode

---

## Drawing Functions

### clearScreen

```c
static void clearScreen();
```

Clears the entire framebuffer to black (all pixels off).

**Example:**
```c
OLED::clearScreen();
// Framebuffer is now empty
```

---

### setCursor

```c
static void setCursor(int16_t x, int16_t y);
```

Sets the cursor position for text rendering.

**Parameters:**
- `x` - Horizontal position in pixels (0 = left edge)
- `y` - Vertical position in pixels (0 = top)

**Example:**
```c
OLED::setCursor(10, 0);
OLED::print("Text at x=10", FontStyle::SMALL);
```

---

### getCursorX

```c
static int16_t getCursorX();
```

Returns the current horizontal cursor position.

**Returns:**
- X coordinate of cursor in pixels

---

## Text Rendering

### print

```c
static void print(const char *string, FontStyle fontStyle, uint8_t length = 255, const uint8_t soft_x_limit = 0);
```

Renders a text string at the current cursor position.

**Parameters:**
- `string` - Null-terminated string to print
- `fontStyle` - Font size (`SMALL`, `LARGE`, or `EXTRAS`)
- `length` - Maximum characters to print (default: 255)
- `soft_x_limit` - Soft boundary to stop printing (0 = disabled)

**Notes:**
- Cursor advances after each character
- Does not wrap text automatically

**Example:**
```c
OLED::setCursor(0, 0);
OLED::print("Temperature:", FontStyle::SMALL);
OLED::print("350", FontStyle::LARGE);
```

---

### printWholeScreen

```c
static void printWholeScreen(const char *string);
```

Prints a string that fills the entire screen, suitable for messages.

**Parameters:**
- `string` - Text to display

---

### printNumber

```c
static void printNumber(uint16_t number, uint8_t places, FontStyle fontStyle, bool noLeaderZeros = true);
```

Renders a numeric value at the current cursor position.

**Parameters:**
- `number` - Value to print (0-65535)
- `places` - Number of digits to display
- `fontStyle` - Font size
- `noLeaderZeros` - If true, suppress leading zeros

**Example:**
```c
OLED::printNumber(350, 3, FontStyle::LARGE);  // Prints "350"
OLED::printNumber(42, 4, FontStyle::LARGE, false);  // Prints "0042"
```

---

### printSymbolDeg

```c
static void printSymbolDeg(FontStyle fontStyle = FontStyle::LARGE);
```

Prints the degree symbol (°C or °F based on settings).

**Parameters:**
- `fontStyle` - Size of degree symbol

**Example:**
```c
OLED::printNumber(350, 3, FontStyle::LARGE);
OLED::printSymbolDeg();  // Prints "350°C" or "350°F"
```

---

### debugNumber

```c
static void debugNumber(int32_t val, FontStyle fontStyle);
```

Prints a signed debug value (including negative numbers).

**Parameters:**
- `val` - Signed value to print
- `fontStyle` - Font size

---

### drawHex

```c
static void drawHex(uint32_t x, FontStyle fontStyle, uint8_t digits);
```

Prints a value in hexadecimal format.

**Parameters:**
- `x` - Value to print
- `fontStyle` - Font size
- `digits` - Number of hex digits to display

---

## Graphics Primitives

### drawArea

```c
static void drawArea(int16_t x, int8_t y, uint8_t wide, uint8_t height, const uint8_t *ptr);
```

Draws a bitmap image to the framebuffer.

**Parameters:**
- `x` - X coordinate (can be negative for partial draw)
- `y` - Y coordinate (must be 0 or 8-aligned)
- `wide` - Width in pixels
- `height` - Height in pixels (should be 8 or 16)
- `ptr` - Pointer to bitmap data

**Notes:**
- Y must be aligned to 8-pixel boundaries (0, 8, 16, 24)
- Bitmap format is column-major, LSB at top

---

### drawAreaSwapped

```c
static void drawAreaSwapped(int16_t x, int8_t y, uint8_t wide, uint8_t height, const uint8_t *ptr);
```

Draws a bitmap with bytes swapped (for different endianness).

---

### fillArea

```c
static void fillArea(int16_t x, int8_t y, uint8_t wide, uint8_t height, const uint8_t value);
```

Fills a rectangular area with a pattern byte.

**Parameters:**
- `x`, `y` - Top-left corner
- `wide`, `height` - Dimensions
- `value` - Byte pattern to fill (0x00 = black, 0xFF = white)

---

### drawFilledRect

```c
static void drawFilledRect(uint8_t x0, uint8_t y0, uint8_t x1, uint8_t y1, bool clear);
```

Draws a filled rectangle.

**Parameters:**
- `x0`, `y0` - Top-left corner
- `x1`, `y1` - Bottom-right corner
- `clear` - If true, clear the area; if false, fill with white

---

### drawImage

```c
static void drawImage(const uint8_t *buffer, uint8_t x, uint8_t width);
```

Draws an image at the specified X position, full height.

**Parameters:**
- `buffer` - Image data
- `x` - X coordinate
- `width` - Image width

---

## Symbol Drawing

### drawSymbol

```c
static void drawSymbol(uint8_t symbolID);
```

Draws a predefined symbol from the symbol font.

**Parameters:**
- `symbolID` - Symbol identifier

---

### drawBattery

```c
static void drawBattery(uint8_t state);
```

Draws a battery indicator icon.

**Parameters:**
- `state` - Battery level (0-10)

**Example:**
```c
OLED::drawBattery(7);  // 70% battery icon
```

---

### drawCheckbox

```c
static void drawCheckbox(bool state);
```

Draws a checkbox (checked or unchecked).

**Parameters:**
- `state` - `true` for checked, `false` for unchecked

---

### drawHeatSymbol

```c
static void drawHeatSymbol(uint8_t state);
```

Draws the heating indicator animation frame.

**Parameters:**
- `state` - Animation frame number

---

### drawScrollIndicator

```c
static void drawScrollIndicator(uint8_t p, uint8_t h);
```

Draws a vertical scroll position indicator.

**Parameters:**
- `p` - Current position
- `h` - Total height/items

---

### drawUnavailableIcon

```c
static void drawUnavailableIcon();
```

Draws the "feature unavailable" icon.

---

### maskScrollIndicatorOnOLED

```c
static void maskScrollIndicatorOnOLED();
```

Masks/hides the scroll indicator area.

---

## Animation and Transitions

### transitionSecondaryFramebuffer

```c
static void transitionSecondaryFramebuffer(const bool forwardNavigation, const TickType_t viewEnterTime);
```

Performs an animated transition between framebuffers.

**Parameters:**
- `forwardNavigation` - Direction of transition
- `viewEnterTime` - When the transition started

---

### useSecondaryFramebuffer

```c
static void useSecondaryFramebuffer(bool useSecondary);
```

Switches between primary and secondary framebuffer for double-buffering.

**Parameters:**
- `useSecondary` - `true` to draw to secondary buffer

---

### transitionScrollDown

```c
static void transitionScrollDown(const TickType_t viewEnterTime);
```

Performs a downward scroll transition animation.

---

### transitionScrollUp

```c
static void transitionScrollUp(const TickType_t viewEnterTime);
```

Performs an upward scroll transition animation.

---

## Framebuffer Details

### Memory Layout

```
screenBuffer[]:
┌──────────────────────────────────────────┐
│ Command bytes (17 bytes)                 │ Bytes 0-16
├──────────────────────────────────────────┤
│ Pixel data (WIDTH * HEIGHT/8 bytes)      │ Bytes 17+
│ - Column 0, rows 0-7 (1 byte)            │
│ - Column 0, rows 8-15 (1 byte)           │
│ - ...                                    │
│ - Column N, rows 0-7                     │
│ - Column N, rows 8-15                    │
└──────────────────────────────────────────┘
```

### Pixel Format

Each byte represents 8 vertical pixels:
- Bit 0 = top pixel
- Bit 7 = bottom pixel

```
Byte value 0x81 = ■□□□□□□■ (top and bottom pixels lit)
```

---

## Example: Custom Screen

```c
void drawCustomScreen() {
    // Clear and set up
    OLED::clearScreen();
    OLED::setCursor(0, 0);

    // Draw header
    OLED::print("TEMP", FontStyle::SMALL);

    // Draw large temperature
    OLED::setCursor(0, 8);
    OLED::printNumber(350, 3, FontStyle::LARGE);
    OLED::printSymbolDeg();

    // Draw status icon
    OLED::drawHeatSymbol(animationFrame);

    // Update display
    OLED::refresh();
}
```

---

## Thread Safety

The OLED driver is designed for single-thread access (GUI thread). Key considerations:
- All drawing should occur in the GUI thread
- `refresh()` uses DMA and may block briefly
- I2C access is protected by FreeRTOS mutex

---

## See Also

- [I2C Wrapper](../communication/I2C_Wrapper.md) - I2C communication
- [GUI Thread](../threading/GUIThread.md) - Display update loop
- [Translation API](../core/Translation.md) - Font and symbol data

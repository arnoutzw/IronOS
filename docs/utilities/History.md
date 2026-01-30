# History Buffer

The History template provides a circular buffer with automatic sum tracking for efficient averaging.

**Header File:** `source/Core/Inc/history.hpp`

---

## Overview

The `history` template provides:
- Fixed-size circular buffer
- Running sum for O(1) average calculation
- Indexed access (0 = newest, size-1 = oldest)

---

## Template Definition

```c
template <class T, uint8_t SIZE>
struct history {
    static const uint8_t size = SIZE;  // Buffer size
    T        buf[size];                 // Data buffer
    uint32_t sum;                       // Running sum
    uint8_t  loc;                       // Current write position

    void update(T const val);           // Add new value
    T operator[](uint8_t i) const;      // Access by index
    T average() const;                  // Get average
};
```

---

## Template Parameters

### T

Data type stored in the buffer. Typically:
- `uint16_t` for temperature readings
- `int16_t` for signed values
- `uint32_t` for power values

### SIZE

Number of elements in the buffer. Maximum 127 due to `uint8_t` indexing.

---

## Methods

### update

```c
void update(T const val);
```

Adds a new value to the buffer.

**Parameters:**
- `val` - Value to add

**Behavior:**
- Subtracts oldest value from sum
- Adds new value to sum
- Overwrites oldest value
- Advances write position

**Implementation:**
```c
void update(T const val) {
    sum -= buf[loc];    // Remove oldest
    sum += val;         // Add new
    buf[loc] = val;     // Store
    loc = (loc + 1) % size;  // Advance
}
```

**Example:**
```c
history<uint16_t, 10> tempHistory;

// Add temperature readings
tempHistory.update(350);
tempHistory.update(352);
tempHistory.update(351);
```

---

### operator[]

```c
T operator[](uint8_t i) const;
```

Accesses a value by index.

**Parameters:**
- `i` - Index (0 = newest, size-1 = oldest)

**Returns:**
- Value at index

**Implementation:**
```c
T operator[](uint8_t i) const {
    i = (i + loc) % size;
    return buf[i];
}
```

**Example:**
```c
T newest = tempHistory[0];
T oldest = tempHistory[tempHistory.size - 1];
```

---

### average

```c
T average() const;
```

Returns the average of all values in the buffer.

**Returns:**
- Sum divided by size

**Implementation:**
```c
T average() const {
    return sum / size;
}
```

**Complexity:** O(1) due to running sum

**Example:**
```c
uint16_t avgTemp = tempHistory.average();
```

---

## Usage Examples

### Temperature Smoothing

```c
history<uint16_t, 8> tempFilter;

void sampleTemperature() {
    uint16_t rawTemp = getTipRawTemp(1);
    tempFilter.update(rawTemp);
}

uint16_t getSmoothedTemperature() {
    return tempFilter.average();
}
```

### Power History for PID

```c
history<uint32_t, 16> powerHistory;

void recordPower(uint32_t watts) {
    powerHistory.update(watts);
}

uint32_t getAveragePower() {
    return powerHistory.average();
}
```

### Detecting Trends

```c
history<int16_t, 10> readings;

bool isIncreasing() {
    // Compare newest to oldest
    int16_t newest = readings[0];
    int16_t oldest = readings[readings.size - 1];
    return newest > oldest;
}

int16_t getChangeRate() {
    // Change over buffer duration
    return readings[0] - readings[readings.size - 1];
}
```

---

## Memory Layout

```
buf[]:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 5 │ 6 │ 7 │ 8 │ 1 │ 2 │ 3 │ 4 │  (values added in order 1-8)
└───┴───┴───┴───┴───┴───┴───┴───┘
              ↑
             loc (next write position)

Logical view (index order):
  [0] = 8 (newest)
  [1] = 7
  [2] = 6
  [3] = 5
  [4] = 4
  [5] = 3
  [6] = 2
  [7] = 1 (oldest)
```

---

## Performance

| Operation | Complexity | Notes |
|-----------|------------|-------|
| update() | O(1) | Single write, two additions |
| average() | O(1) | Division of running sum |
| operator[] | O(1) | Index calculation |

### Memory Usage

```
sizeof(history<T, SIZE>) = SIZE * sizeof(T) + sizeof(uint32_t) + sizeof(uint8_t) + padding
```

Example: `history<uint16_t, 8>` = 8×2 + 4 + 1 + 1 = 22 bytes

---

## Initialization

The buffer should be initialized before use:

```c
history<uint16_t, 8> buffer;

void init() {
    // Fill with initial values
    for (int i = 0; i < buffer.size; i++) {
        buffer.update(initialValue);
    }
}
```

Or initialize sum to match initial buffer state:
```c
buffer.sum = initialValue * buffer.size;
```

---

## Thread Safety

The `history` struct is **not** thread-safe. If accessed from multiple threads:

```c
// Option 1: Critical section
taskENTER_CRITICAL();
buffer.update(value);
taskEXIT_CRITICAL();

// Option 2: Access only from one thread
// (typical pattern in IronOS)
```

---

## See Also

- [Exponential Moving Average](ExpMovingAverage.md) - Alternative filter
- [Power API](../core/Power.md) - Uses history for power tracking
- [PID Thread](../threading/PIDThread.md) - Temperature history usage

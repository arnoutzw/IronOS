# Exponential Moving Average

The `expMovingAverage` template provides an efficient exponential moving average filter with configurable weighting.

**Header File:** `source/Core/Inc/expMovingAverage.h`

---

## Overview

The exponential moving average (EMA) provides:
- Single-value signal smoothing
- Configurable response speed via weighting
- Minimal memory usage (just one sum value)
- O(1) update and read operations

---

## Template Definition

```c
template <class T, uint8_t weighting>
struct expMovingAverage {
    int32_t sum;                    // Accumulated weighted value

    void update(T const val);       // Add new sample
    T average() const;              // Get current average
};
```

---

## Template Parameters

### T

Data type for input/output values. Typically:
- `uint16_t` for temperature
- `uint32_t` for power values
- `int16_t` for signed measurements

### weighting

Weight given to new values (0-255):
- Higher = faster response to changes
- Lower = more smoothing
- Represents the numerator in `weighting/256`

---

## Methods

### update

```c
void update(T const val);
```

Adds a new sample to the filter.

**Parameters:**
- `val` - New value to incorporate

**Implementation:**
```c
void update(T const val) {
    sum = ((val * weighting) + (sum * (256 - weighting))) / 256;
}
```

**Formula:**
```
new_sum = (new_value × weight + old_sum × (256 - weight)) / 256
        = new_value × (weight/256) + old_sum × (1 - weight/256)
```

**Example:**
```c
expMovingAverage<uint32_t, 24> powerAvg;

// Update with new power reading
powerAvg.update(currentPower);
```

---

### average

```c
T average() const;
```

Returns the current filtered value.

**Returns:**
- The accumulated sum (which represents the filtered average)

**Implementation:**
```c
T average() const {
    return sum;
}
```

**Note:** The `sum` already represents the smoothed value due to the EMA formula.

---

## Weighting Effects

| Weighting | Response | Time Constant* | Use Case |
|-----------|----------|----------------|----------|
| 8 | Very slow | ~32 samples | Heavy noise filtering |
| 24 | Slow | ~10 samples | PID integral term |
| 64 | Medium | ~4 samples | General smoothing |
| 128 | Fast | ~2 samples | Quick response needed |
| 192 | Very fast | ~1.3 samples | Minimal filtering |

*Approximate samples to reach 63% of step change

---

## Filter Response

For a step change from 0 to 100 with weighting = 64:

```
Sample:  0    1    2    3    4    5    6    7    8
Value:   0   25   44   58   68   76   82   87   90
```

```
100 |                                    ────────
    |                              ─────
    |                        ─────
    |                  ─────
 50 |            ─────
    |      ─────
    | ────
  0 |────
    └─────────────────────────────────────────────
      0    1    2    3    4    5    6    7    8
                      Sample Number
```

---

## Usage Examples

### Power Tracking

```c
// From power.hpp
const uint8_t wattHistoryFilter = 24;
extern expMovingAverage<uint32_t, wattHistoryFilter> x10WattHistory;

// Usage in PID controller
x10WattHistory.update(currentPowerX10);
int32_t smoothedPower = x10WattHistory.average();
```

### Temperature Filtering

```c
expMovingAverage<uint16_t, 64> tempFilter;

void sampleTemperature() {
    uint16_t rawTemp = getTipRawTemp(1);
    tempFilter.update(rawTemp);
}

uint16_t getFilteredTemp() {
    return tempFilter.average();
}
```

### Signal Conditioning

```c
expMovingAverage<int16_t, 32> accelFilter;

void processAccelerometer() {
    int16_t raw = readRawAccel();
    accelFilter.update(raw);
    int16_t filtered = accelFilter.average();

    // Use filtered value for motion detection
    if (abs(filtered) > threshold) {
        motionDetected = true;
    }
}
```

---

## Comparison with History Buffer

| Feature | expMovingAverage | history |
|---------|------------------|---------|
| Memory | 4 bytes (sum only) | N × sizeof(T) + 5 |
| Oldest value access | No | Yes |
| Response shape | Exponential | Rectangular |
| Initialization | Set sum directly | Fill buffer |
| All values access | No | Yes (indexed) |

---

## Initialization

Before first use, initialize the sum:

```c
expMovingAverage<uint32_t, 24> filter;

void init(uint32_t initialValue) {
    filter.sum = initialValue;
}
```

Or let it settle from zero:
```c
// First several updates will ramp toward actual value
for (int i = 0; i < 20; i++) {
    filter.update(sensorRead());
}
// Now filter is settled
```

---

## Mathematical Background

### EMA Formula

The exponential moving average is defined as:

```
EMAₙ = α × Xₙ + (1 - α) × EMAₙ₋₁

Where:
  EMAₙ = current filtered value
  Xₙ = current input value
  α = weighting factor (0 to 1)
  EMAₙ₋₁ = previous filtered value
```

### Implementation Optimization

The template uses integer math:
```
α = weighting / 256
(1 - α) = (256 - weighting) / 256
```

All calculations use integer multiplication and division by 256 (shift by 8 bits), avoiding floating-point operations.

---

## Choosing Weighting

### Guidelines

1. **Noise level**: More noise → lower weighting
2. **Response speed**: Need quick response → higher weighting
3. **Sample rate**: Higher rate can use lower weighting

### Calculation

To achieve a time constant of `τ` samples:
```
weighting ≈ 256 / τ
```

Example: For τ = 10 samples: weighting = 256/10 ≈ 25

---

## Thread Safety

The `expMovingAverage` struct is **not** thread-safe. For multi-thread access:

```c
// Protect with critical section
taskENTER_CRITICAL();
filter.update(value);
result = filter.average();
taskEXIT_CRITICAL();
```

Or ensure single-thread access (typical in IronOS - updated in PID thread only).

---

## See Also

- [History Buffer](History.md) - Alternative with full history access
- [Power API](../core/Power.md) - EMA usage in power tracking
- [PID Thread](../threading/PIDThread.md) - Temperature control filtering

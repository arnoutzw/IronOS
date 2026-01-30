# PID Thread

The PID thread handles temperature control using a Proportional-Integral-Derivative (PID) controller to maintain the soldering tip at the target temperature.

**Entry Point:** `startPIDTask()` in `source/Core/Inc/main.hpp`
**Implementation:** `source/Core/Threads/PIDThread.cpp`

---

## Overview

The PID thread is the most critical thread, responsible for:
- Temperature measurement and sampling
- PID control algorithm execution
- PWM output to heater
- Thermal safety monitoring
- Watchdog management

---

## Thread Function

### startPIDTask

```c
void startPIDTask(void const *argument);
```

Entry point for the PID temperature control thread.

**Parameters:**
- `argument` - FreeRTOS task argument (unused)

**Main Loop:**
```c
void startPIDTask(void const *argument) {
    // Initialize
    initPIDController();

    while (1) {
        // Wait for ADC sample notification
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        // Reset watchdog
        resetWatchdog();

        // Run pre-start checks if needed
        if (!preStartChecksDone()) {
            preStartChecks();
            continue;
        }

        // Read temperature
        TemperatureType_t currentTemp = TipThermoModel::getTipInC(true);

        // Get target temperature
        TemperatureType_t targetTemp = currentTempTargetDegC;

        // Calculate PID output
        int32_t power = calculatePID(currentTemp, targetTemp);

        // Apply power to tip
        setTipX10Watts(power);

        // Safety checks
        checkThermalRunaway(currentTemp, targetTemp);
    }
}
```

---

## PID Algorithm

### Control Equation

The PID controller uses the standard form:

```
output = Kp * error + Ki * integral + Kd * derivative

Where:
  error = targetTemp - currentTemp
  integral = sum of errors over time
  derivative = rate of change of error
```

### Implementation

```c
int32_t calculatePID(TemperatureType_t current, TemperatureType_t target) {
    // Calculate error
    int32_t error = target - current;

    // Proportional term
    int32_t pTerm = error * Kp;

    // Integral term (using exponential moving average)
    x10WattHistory.update(lastPower);
    int32_t iTerm = x10WattHistory.average() * Ki;

    // Derivative term (rate of temperature change)
    int32_t dTerm = (current - lastTemp) * Kd;
    lastTemp = current;

    // Calculate total output
    int32_t output = pTerm + iTerm - dTerm;

    // Clamp to available power
    if (output < 0) output = 0;
    if (output > availableW10(0)) output = availableW10(0);

    return output;
}
```

---

## Tuning Parameters

### Proportional Gain (Kp)

Controls immediate response to temperature error:
- Higher Kp = faster response, more overshoot
- Lower Kp = slower response, less overshoot

### Integral Gain (Ki)

Eliminates steady-state error:
- Higher Ki = eliminates error faster, may oscillate
- Lower Ki = slow error correction, stable

### Derivative Gain (Kd)

Dampens oscillation:
- Higher Kd = more damping, may amplify noise
- Lower Kd = less damping, allows oscillation

### Thermal Mass Factor

```c
const uint8_t wattHistoryFilter = 24;
```

The exponential moving average weighting provides feed-forward compensation based on tip thermal mass.

---

## Timing

### ADC Synchronization

The PID loop is synchronized with ADC sampling:

```
┌─────────────┐
│ PWM Off     │ ADC samples during PWM off-time
│ Period      │ for accurate temperature reading
└─────┬───────┘
      │
      v
┌─────────────┐
│ ADC Sample  │
│ Complete    │
└─────┬───────┘
      │
      v
┌─────────────┐
│ Notify PID  │ xTaskNotifyGive(pidTaskNotification)
│ Task        │
└─────┬───────┘
      │
      v
┌─────────────┐
│ PID         │ Calculate new power output
│ Calculation │
└─────┬───────┘
      │
      v
┌─────────────┐
│ Set PWM     │ Apply new duty cycle
│             │
└─────────────┘
```

### Sample Rate

- ADC sampling: ~100Hz (10ms period)
- PID calculation: Same rate as ADC
- PWM update: Immediate after calculation

---

## Safety Features

### Thermal Runaway Detection

```c
void checkThermalRunaway(TemperatureType_t current, TemperatureType_t target) {
    static TickType_t lastCheck = 0;
    static TemperatureType_t lastTemp = 0;

    // Only check when heating
    if (currentPower == 0) return;

    // Check every second
    if (xTaskGetTickCount() - lastCheck < TICKS_SECOND) return;

    // If applying power but temperature not rising
    if (current < lastTemp + RUNAWAY_THRESHOLD) {
        runawayCounter++;
        if (runawayCounter > RUNAWAY_LIMIT) {
            // Thermal runaway detected!
            enterThermalRunawayMode();
        }
    } else {
        runawayCounter = 0;
    }

    lastTemp = current;
    lastCheck = xTaskGetTickCount();
}
```

### Tip Disconnection

```c
if (isTipDisconnected()) {
    setTipPWM(0, false);  // Disable heater
    return;
}
```

### Short Circuit Detection

```c
if (isTipShorted()) {
    setTipPWM(0, false);  // Disable heater
    enterErrorMode();
    return;
}
```

### Watchdog Reset

```c
// Reset watchdog every PID cycle
resetWatchdog();
```

If PID thread hangs, watchdog triggers system reset.

---

## Power Output

### Power Calculation

```c
void setTipX10Watts(int32_t requestedPower) {
    // Limit to available power
    int32_t maxPower = availableW10(0);
    if (requestedPower > maxPower) {
        requestedPower = maxPower;
    }

    // Limit to user setting
    uint16_t powerLimit = getSettingValue(PowerLimit);
    if (powerLimit > 0 && requestedPower > powerLimit * 10) {
        requestedPower = powerLimit * 10;
    }

    // Convert to PWM
    uint8_t pwm = X10WattsToPWM(requestedPower, 0);

    // Apply to tip
    setTipPWM(pwm, useFastPWM);

    // Update history
    x10WattHistory.update(requestedPower);
}
```

### PWM Modes

| Mode | Frequency | Use Case |
|------|-----------|----------|
| Normal | ~1 kHz | Standard operation |
| Fast | ~10 kHz | Reduced audible noise |

---

## State Machine

```
┌─────────────────┐
│ Pre-Start       │ Run initialization checks
│ Checks          │
└────────┬────────┘
         │ Checks complete
         v
┌─────────────────┐
│ Idle            │ Target = 0, heater off
│ (Standby)       │
└────────┬────────┘
         │ Target > 0
         v
┌─────────────────┐
│ Heating         │ Target > Current
│                 │
└────────┬────────┘
         │ At target
         v
┌─────────────────┐
│ Maintaining     │ PID holding temperature
│                 │
└────────┬────────┘
         │ Target = 0 or error
         v
┌─────────────────┐
│ Cooling /       │
│ Error           │
└─────────────────┘
```

---

## Thread Priority

PID is the highest priority thread because:
- Safety-critical temperature control
- Time-sensitive PWM updates
- Must not be preempted during control

```c
#define PID_TASK_PRIORITY  osPriorityRealtime
```

---

## Stack Usage

Minimal stack (512 words) because:
- No complex data structures
- No recursive algorithms
- Predictable execution path

---

## Example: Temperature Profile

```c
void executeTempProfile(TemperatureType_t temps[], uint32_t durations[], int phases) {
    for (int i = 0; i < phases; i++) {
        // Set target for this phase
        currentTempTargetDegC = temps[i];

        // Wait for duration
        osDelay(durations[i]);

        // PID thread automatically tracks target
    }

    // Cooldown
    currentTempTargetDegC = 0;
}
```

---

## Debugging

### Debug Values

Available in debug menu:
- Current temperature
- Target temperature
- Current power (watts)
- PWM duty cycle
- Tip voltage (µV)

### PID Tuning Tips

1. Start with Ki = 0, Kd = 0
2. Increase Kp until oscillation
3. Reduce Kp by 50%
4. Slowly increase Ki until steady-state error eliminated
5. Add Kd if overshoot is problematic

---

## See Also

- [Power API](../core/Power.md) - Power calculations
- [Tip Thermo Model](../drivers/TipThermoModel.md) - Temperature conversion
- [BSP Core](../bsp/BSP.md) - PWM and ADC functions
- [Threading Overview](Overview.md) - Thread architecture

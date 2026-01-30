# BSP Flash Interface

The BSP Flash module provides an interface for reading and writing settings data to non-volatile flash memory.

**Header File:** `source/Core/BSP/BSP_Flash.h`
**Implementation:** Platform-specific in `source/Core/BSP/<platform>/Flash.cpp`

---

## Overview

The Flash interface allows:
- Persistent storage of user settings
- Safe erase and program operations
- Platform-independent flash access

Flash memory is used because:
- Retains data without power
- Limited write cycles (typically 10,000-100,000)
- Must be erased before writing

---

## Functions

### flash_save_buffer

```c
void flash_save_buffer(const uint8_t *buffer, const uint16_t length);
```

Erases the settings flash sector and saves the provided buffer.

**Parameters:**
- `buffer` - Pointer to data to save
- `length` - Number of bytes to save

**Behavior:**
1. Erases the designated flash sector
2. Programs the new data
3. Verifies the write (platform-dependent)

**Notes:**
- This is a blocking operation
- May take 10-100ms depending on platform
- Interrupts may be disabled during write
- Called from `saveSettings()` in Settings module

**Example:**
```c
uint8_t settingsData[256];
// ... populate settingsData ...
flash_save_buffer(settingsData, sizeof(settingsData));
```

---

### flash_read_buffer

```c
void flash_read_buffer(uint8_t *buffer, const uint16_t length);
```

Reads data from the settings flash sector.

**Parameters:**
- `buffer` - Pointer to buffer for read data
- `length` - Number of bytes to read

**Notes:**
- Non-destructive read operation
- Fast operation (direct memory access on most platforms)
- Called from `loadSettings()` in Settings module

**Example:**
```c
uint8_t settingsData[256];
flash_read_buffer(settingsData, sizeof(settingsData));
// ... process settingsData ...
```

---

## Flash Memory Layout

Typical flash organization for settings:

```
┌─────────────────────────────┐ High Address
│ Application Code            │
├─────────────────────────────┤
│ ...                         │
├─────────────────────────────┤
│ Settings Sector             │ <- flash_save_buffer/flash_read_buffer
│ (typically 1-4 KB)          │
├─────────────────────────────┤
│ Bootloader (if present)     │
└─────────────────────────────┘ Low Address (0x0800_0000 on STM32)
```

---

## Platform-Specific Details

### STM32F103 (Miniware)

- Flash page size: 1024 bytes
- Settings location: Last page(s) of flash
- Erase time: ~40ms per page
- Write time: ~70µs per half-word

### GD32VF103 (Pinecil)

- Flash page size: 1024 bytes
- Similar to STM32F103
- May require different unlock sequence

### BL706 (Pinecil V2)

- Flash page size: 4096 bytes
- Settings location: Designated data area
- XIP (execute in place) flash

---

## Write Cycle Considerations

Flash memory has limited write endurance:

| Platform | Guaranteed Cycles | Typical Cycles |
|----------|-------------------|----------------|
| STM32F103 | 10,000 | 100,000+ |
| GD32VF103 | 10,000 | 50,000+ |
| BL706 | 10,000 | 100,000+ |

To maximize lifespan:
- Settings are only saved on explicit user action
- Automatic saves are batched/debounced
- Settings format minimizes unnecessary writes

---

## Data Integrity

### Version Checking

The settings format includes a version number:

```c
#define SETTINGSVERSION (0x55AA)
```

On load:
1. Read version from flash
2. If mismatch, reset to defaults
3. Save defaults to flash

### Validation

Settings are validated on load:
- Checksum verification (if implemented)
- Range checking for each setting
- Invalid values replaced with defaults

---

## Thread Safety

Flash operations are **not** thread-safe:

- All flash operations should occur from one thread (typically GUI)
- Interrupts may be disabled during flash writes
- Other threads should not access settings during save

---

## Example: Settings Storage

```c
// Settings structure (simplified)
struct SettingsStruct {
    uint16_t version;
    uint16_t solderingTemp;
    uint16_t sleepTemp;
    uint8_t  orientation;
    // ... more settings ...
    uint16_t checksum;
};

void saveSettings() {
    SettingsStruct settings;
    // ... populate settings ...

    settings.version = SETTINGSVERSION;
    settings.checksum = calculateChecksum(&settings);

    flash_save_buffer((uint8_t*)&settings, sizeof(settings));
}

bool loadSettings() {
    SettingsStruct settings;

    flash_read_buffer((uint8_t*)&settings, sizeof(settings));

    if (settings.version != SETTINGSVERSION) {
        return false;  // Version mismatch
    }

    if (!validateChecksum(&settings)) {
        return false;  // Corruption detected
    }

    // Apply settings...
    return true;
}
```

---

## Error Handling

Flash operations can fail due to:
- Power loss during write
- Flash wear-out
- Incorrect unlock sequence

Most platforms silently fail but settings module handles:
- Invalid data detection via version/checksum
- Automatic reset to defaults on corruption

---

## See Also

- [Settings API](../core/Settings.md) - Settings management
- [BSP Core](BSP.md) - Board support overview

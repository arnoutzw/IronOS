# IronOS API Documentation

This documentation provides comprehensive information about the IronOS firmware's functions, interfaces, and APIs. IronOS is an open-source firmware for soldering irons supporting multiple hardware platforms including Miniware, Pinecil, Pinecil V2, MHP30, and Sequre devices.

## Table of Contents

### Core APIs
- [Settings API](core/Settings.md) - System settings management and persistence
- [Power API](core/Power.md) - Power calculation and PWM control
- [Main API](core/Main.md) - Application entry points and global state
- [Translation API](core/Translation.md) - Internationalization and localization

### Hardware Drivers
- [OLED Display](drivers/OLED.md) - Display driver for SSD1307 OLED screens
- [Accelerometers](drivers/Accelerometers.md) - Motion sensing drivers (LIS2DH12, BMA223, MMA8652FC, MSA301, SC7A20)
- [USB Power Delivery](drivers/USBPD.md) - USB-PD protocol stack
- [Tip Thermo Model](drivers/TipThermoModel.md) - Temperature sensing and conversion
- [Hall Effect Sensor](drivers/Si7210.md) - Magnetic field sensing
- [QC3.0 Protocol](drivers/QC3.md) - Quick Charge 3.0 negotiation

### Board Support Package (BSP)
- [BSP Core](bsp/BSP.md) - Hardware abstraction layer
- [BSP Flash](bsp/BSP_Flash.md) - Flash memory operations
- [BSP Power](bsp/BSP_Power.md) - Power control interface
- [BSP PD](bsp/BSP_PD.md) - Power Delivery board interface
- [BSP QC](bsp/BSP_QC.md) - Quick Charge board interface

### Communication Interfaces
- [I2C Wrapper](communication/I2C_Wrapper.md) - FreeRTOS I2C interface
- [I2C Bit-Bang](communication/I2CBB.md) - Software I2C implementations

### Threading System
- [Threading Overview](threading/Overview.md) - FreeRTOS thread architecture
- [GUI Thread](threading/GUIThread.md) - User interface rendering
- [PID Thread](threading/PIDThread.md) - Temperature control loop
- [MOV Thread](threading/MOVThread.md) - Movement/acceleration detection
- [POW Thread](threading/POWThread.md) - Power management

### User Interface
- [Operating Modes](ui/OperatingModes.md) - UI state machine and modes
- [Button Handling](ui/Buttons.md) - Button state management
- [UI Drawing](ui/Drawing.md) - Screen rendering functions

### Utilities
- [History Buffer](utilities/History.md) - Circular buffer with averaging
- [Exponential Moving Average](utilities/ExpMovingAverage.md) - Signal filtering
- [Types](utilities/Types.md) - Common type definitions

---

## Architecture Overview

```
IronOS/
├── Core/
│   ├── Inc/              # Core header files
│   ├── Src/              # Core implementation
│   ├── Drivers/          # Hardware drivers
│   │   └── usb-pd/       # USB Power Delivery stack
│   ├── BSP/              # Board Support Packages
│   │   ├── Miniware/     # STM32F103 platform
│   │   ├── Pinecil/      # Pinecil (GD32VF103)
│   │   ├── Pinecilv2/    # Pinecil V2 (BL706)
│   │   ├── MHP30/        # Hot plate variant
│   │   └── Sequre/       # Sequre platform
│   └── Threads/          # FreeRTOS tasks
│       └── UI/           # User interface
│           ├── logic/    # Mode logic
│           └── drawing/  # Screen rendering
└── Middlewares/          # FreeRTOS and libraries
```

## Supported Hardware

| Platform | MCU | Display | Features |
|----------|-----|---------|----------|
| Miniware (TS100/TS80) | STM32F103 | 96x16 OLED | QC3.0, USB-PD |
| Pinecil | GD32VF103 | 96x16 OLED | USB-PD, QC3.0 |
| Pinecil V2 | BL706 | 128x32 OLED | USB-PD, BLE |
| MHP30 | STM32F103 | 128x32 OLED | Hot plate mode |
| Sequre | Varies | 96x16 OLED | QC3.0 |

## Build Configuration

The firmware uses `configuration.h` to define platform-specific features. Key defines include:

- `OLED_128x32` / `OLED_96x16` - Display size selection
- `POW_PD` - Enable USB Power Delivery
- `POW_QC` - Enable Quick Charge 3.0
- `MAG_SLEEP_SUPPORT` - Enable hall effect sensor support
- `TIP_TYPE_SUPPORT` - Enable multiple tip type selection

## License

IronOS is released under the GPL-3.0 license. See the main repository for details.

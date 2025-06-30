---
title: "Simplifying I²C on ESP32: Meet the i2c_bus Component"
date: "2025-07-01"
showAuthor: false
summary: "This article..."
authors:
  - "pedro-minatel" 
tags: ["ESP32C3", "HTTP", "Connectivity"]
---

## Introduction

Managing I²C communication on the ESP32 can quickly become repetitive, especially when juggling multiple sensors or displays. Espressif’s [`i2c_bus`](https://components.espressif.com/components/espressif/i2c_bus/versions/1.4.0) component addresses this pain point by wrapping the low-level ESP-IDF I²C driver into a cleaner, more modular, and thread-safe interface—making your I²C code **cleaner, safer, and easier to scale**.

> Version **1.4.0** brings improved stability, CMake-based dependency management, and optional software I²C support.

## ESP‑IDF 5.2 Migration Guide for the I²C Driver Redesign

Espressif revamped the I²C driver architecture in ESP‑IDF v5.2. Here's what’s changed—and why `i2c_bus` aligns well with the new model:

### Major Conceptual Updates

- **Separate bus/device initialization**: Replaces the old ambiguous `i2c_config_t`, introducing explicit `i2c_master_bus_config_t` and `i2c_slave_config_t`.  
- **Clear separation of master/slave**: The deprecated `i2c_mode_t` is removed; you now call distinct initialization APIs for master or slave.  
- **Simplified address handling**: `i2c_addr_mode_t` is renamed to `i2c_addr_bit_len_t`.  
- **Command chaining hidden**: Previous `i2c_cmd_link_*` APIs have been removed—the driver now handles command lists internally.  
- **Renamed convenience functions**:  
  - `i2c_master_write_to_device` → `i2c_master_transmit`  
  - `i2c_master_read_from_device` → `i2c_master_receive`  
  - `i2c_master_write_read_device` → `i2c_master_transmit_receive`  
  - Slave-side equivalents renamed similarly :contentReference[oaicite:1]{index=1}.

### Impacts & Benefits for `i2c_bus`

- **Aligned with modern APIs**  
  `i2c_bus` was designed atop the new, cleaner bus–device model requiring you to create a bus (`i2c_bus_create`) and then add devices (`i2c_bus_add_device`), so it's fully compatible with ESP‑IDF 5.2+.

- **Removes boilerplate**  
  With command creation and chaining handled internally by the new driver, `i2c_bus` maps naturally to the renamed transmit/receive functions, simplifying your application code even more.

- **Easier migration**  
  Existing projects moving from ESP-IDF v4.x to v5.2+ can switch to `i2c_bus` as a bridge—leveraging the new driver paradigm without rewriting low-level transfer logic or worrying about deprecated APIs.

## Comparing the Traditional I²C API vs i2c_bus

If you’ve used `driver/i2c.h` in the past, you’re probably familiar with how verbose and error-prone it can get. Here’s how the new `i2c_bus` API compares to the traditional ESP-IDF approach:

### Traditional I²C API (ESP-IDF driver/i2c.h)

```c
// Install and configure the I2C driver manually
i2c_config_t conf = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = 21,
    .scl_io_num = 22,
    .sda_pullup_en = GPIO_PULLUP_ENABLE,
    .scl_pullup_en = GPIO_PULLUP_ENABLE,
    .master.clk_speed = 400000,
};
i2c_param_config(I2C_NUM_0, &conf);
i2c_driver_install(I2C_NUM_0, I2C_MODE_MASTER, 0, 0, 0);

// Create command link and send manually
i2c_cmd_handle_t cmd = i2c_cmd_link_create();
i2c_master_start(cmd);
i2c_master_write_byte(cmd, (0x40 << 1) | I2C_MASTER_WRITE, true);
i2c_master_write_byte(cmd, 0xF3, true);
i2c_master_stop(cmd);
i2c_master_cmd_begin(I2C_NUM_0, cmd, pdMS_TO_TICKS(1000));
i2c_cmd_link_delete(cmd);
```

### Using i2c_bus

```c
i2c_bus_config_t config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = 22,
    .sda_io_num = 21,
    .i2c_mode = I2C_MODE_MASTER,
    .master = {.clk_speed = 400000}
};

i2c_bus_handle_t bus = i2c_bus_create(&config);

i2c_bus_device_config_t dev_cfg = {
    .dev_addr_length = I2C_BUS_ADDR_LEN_7,
    .device_address = 0x40,
};

i2c_bus_device_handle_t dev;
i2c_bus_add_device(bus, &dev_cfg, &dev);

uint8_t cmd = 0xF3;
i2c_bus_write_bytes(dev, NULL, 0, &cmd, 1);
```

### Summary

| Feature                         | Traditional API            |.   i2c_bus Component     |
|---------------------------------|----------------------------|--------------------------|
| Manual driver setup             | ✅ Required                | ❌ Abstracted            |
| Thread safety                   | ❌ Manual (if at all)      | ✅ Built-in              |
| Resource cleanup                | ❌ Manual                  | ✅ Automatic             |
| Device abstraction              | ❌ Raw address operations  | ✅ Handle-based devices  |
| Software I²C support            | ❌ Not built-in            | ✅ Supported via config  |
| Code clarity & maintainability  | ❌ Verbose & repetitive    | ✅ Modular & scalable    |

The `i2c_bus` component significantly reduces boilerplate and risk of error, making it the preferred approach for modern ESP-IDF projects.

## Why Use i2c_bus Component?

Espressif’s `i2c_bus` component offers several practical benefits over manually using the lower-level `driver/i2c` APIs:

- Thread-Safe Device Access
  - Automatically wraps I²C operations with internal locks.
  - Great for FreeRTOS-based systems with multiple tasks accessing the same I²C bus.
- Simplified Device Management
  - Devices are created with config structs and automatically registered to the bus.
  - You can reuse device handles easily in your app.
- Automatic Resource Cleanup
  - Handles memory allocation/deallocation for bus and devices.
  - Prevents common memory leaks or double-init errors.
- Modular and Scalable
  - Clean abstraction for managing multiple I²C buses and devices.
  - Ideal for complex projects with several sensors, displays, and peripherals.
- Hardware & Software I²C Support
  - Use software I²C just by setting `i2c_port = -1`.
  - Perfect for prototyping or when hardware pins are unavailable.
- Smarter Defaults and Initialization
  - Avoids common pitfalls (like forgetting to install the driver).
  - Handles one-time initialization of I²C hardware internally.
- Safer API Design
  - Uses handle-based access to I²C devices (like `dev_handle_t`), avoiding magic constants.
  - Encourages clear separation between bus and device logic.
- Seamless Integration with ESP-IDF
  - Fully compatible with CMake and `idf_component.yml`.
  - Pulls dependencies from [components.espressif.com](https://components.espressif.com) automatically.

## Installing the Component

### Option 1: Add via CLI

```bash
idf.py add-dependency "espressif/i2c_bus^1.4.0"
```

### Option 2: Add to `idf_component.yml`

```yaml
dependencies:
  espressif/i2c_bus: "^1.4.0"
```

Then run:

```bash
idf.py reconfigure
```

---

## Using the Component

### 1. Include the Header

```c
#include "i2c_bus.h"
```

### 2. Create and Initialize an I²C Bus

```c
i2c_bus_handle_t i2c_bus;

i2c_bus_config_t config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = 22,
    .sda_io_num = 21,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .i2c_mode = I2C_MODE_MASTER,
    .master = {
        .clk_speed = 400000,
    }
};

i2c_bus = i2c_bus_create(&config);
```

### 3. Add an I²C Device to the Bus

```c
i2c_bus_device_handle_t dev;

i2c_bus_device_config_t dev_cfg = {
    .dev_addr_length = I2C_BUS_ADDR_LEN_7,
    .device_address = 0x40,  // Replace with your device's I2C address
};

i2c_bus_add_device(i2c_bus, &dev_cfg, &dev);
```

### 4. Read or Write to the Device

#### Write one byte:

```c
uint8_t cmd = 0xF3;
i2c_bus_write_bytes(dev, NULL, 0, &cmd, 1);
```

#### Read response:

```c
uint8_t data[2];
i2c_bus_read_bytes(dev, NULL, 0, data, 2);
```

### 5. Cleanup

```c
i2c_bus_rm_device(dev);
i2c_bus_delete(i2c_bus);
```

## Optional: Use Software I²C

To use software I²C instead of hardware, set:

```c
config.i2c_port = -1;  // Special value for software I²C
```

This is useful when all hardware I²C ports are occupied, or when precision timing is not critical.

## Notes

- If you see this message:

  ```
  E (xxx) i2c.master: this port has not been initialized...
  ```

  It's **safe to ignore**—the component handles bus initialization automatically and still functions correctly.

- Works with ESP-IDF **v4.0 and newer**.

## Documentation & Source

- [Component Registry](https://components.espressif.com/components/espressif/i2c_bus/versions/1.4.0)
- [GitHub Source](https://github.com/espressif/esp-iot-solution/tree/master/components/i2c_bus)
- [Tested in examples](https://github.com/espressif/esp-iot-solution/tree/master/examples)

## Conclusion

The `i2c_bus` component abstracts away boilerplate and pitfalls from raw I²C handling. Whether you’re reading a temperature sensor or chaining several I²C devices, this component offers a clean, scalable way to manage communication—without compromising performance or flexibility.

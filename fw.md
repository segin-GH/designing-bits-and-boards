# Embedded Firmware Development Guide

This document outlines the best practices, tools, and workflows for writing robust, maintainable, and testable embedded firmware.
Whether you're starting a new project or maintaining an existing one, this serves as a comprehensive reference.

---

## 1. Coding Style and Guidelines

#### General Principles

- [CMU C Coding Style Guide](https://users.ece.cmu.edu/~eno/coding/CCodingStandard.html)
- [Uncle Bob - Clean Code Series](https://www.youtube.com/watch?v=7EmboKQH8lM&list=PLUxszVpqZTNShoypLQW9a4dEcffsoZT4k&index=1)
- Function and Variable Naming
- No Global Variables
- Use Guard Clauses

```c
    // Gaurd clause pattern
    int sensor_write(uint8_t * data, size_t len)
    {
        if (!data || len <= 0)
        {
            return -EINVAL;
        }

        if (!sensor_is_ready())
        {
            return -ENODEV;
        }

        return i2c_write(SENSOR_ADDR, data, len);
    }


    // Without Guard Clause
    int sensor_write(uint8_t * data, size_t len)
    {
        if (data != NULL && len > 0)
        {
            if (sensor_is_ready())
            {
                return i2c_write(SENSOR_ADDR, data, len);
            }
            else
            {
                return -ENODEV;
            }
        }
        else
        {
            return -EINVAL;
        }
    }
```
---
## 2. Abstraction & Modularity

### Abstraction
  Separate hardware-dependent code from business logic using layers or interfaces.
```c
// led_driver.h
#ifndef LED_DRIVER_H
#define LED_DRIVER_H

#include <stdint.h>

typedef enum {
    LED_OFF = 0,
    LED_ON  = 1
} led_state_t;

int led_driver_init(void);
int led_driver_set_state(uint8_t led_id, led_state_t state);

#endif // LED_DRIVER_H

-----------------------------------------------------


// led_driver.c
#include "led_driver.h"

// Example stub for hardware LED pins
#define LED1_PIN 0

int led_driver_init(void) {
    // Configure GPIO for LED1
    return gpio_configure(LED1_PIN, GPIO_OUTPUT);
}

int led_driver_set_state(uint8_t led_id, led_state_t state) {
        return gpio_write(LED1_PIN, state);
}

-----------------------------------------------------

// Business Logic
// led_interface.h

#ifndef LED_INTERFACE
#define LED_INTERFACE

int led_interface_set_wifi_connected(void);
int led_interface_set_wifi_disconnected(void);
int led_interface_sensor_error();
int led_interface_syncing();
int led_interface_syncing_complete();

#endif // LED_INTERFACE


// led_interface.c

#include "led_interface.h"
#include "led_driver.h"

// Define LED IDs for different states
#define LED_WIFI     1
#define LED_SENSOR   2
#define LED_SYNC     3

int led_interface_set_wifi_connected(void) {
    return led_driver_set_state(LED_WIFI, LED_STATE_ON);
}

int led_interface_set_wifi_disconnected(void) {
    return led_driver_set_state(LED_WIFI, LED_STATE_OFF);
}

int led_interface_sensor_error(void) {
    return led_driver_set_state(LED_SENSOR, LED_STATE_ON);  // Error = LED ON
}

int led_interface_sync_fail(void) {
    return led_driver_set_state(LED_SYNC, LED_STATE_BLINK); // Blinking = fail
}

int led_interface_sync_success(void) {
    return led_driver_set_state(LED_SYNC, LED_STATE_ON); // Solid ON = success
}

  ```

####  **Module Structure:**
  Organize files into clear modules:

```sh
/src
│
├── /core
│   └── led_interface.c         # Business logic: event-driven LED control
│   └── led_interface.h
│
├── /sensor
│   └── sensor_handler.c        # Calls into led_interface on error
│   └── sensor_handler.h
│
├── /drivers
│   └── led_driver.c            # Talks to GPIO
│   └── led_driver.h
│   └── gpio.c                  # Platform GPIO interface
│   └── gpio.h
│
├── /utils
│   └── logger.c                # Logging or helper functions
│   └── logger.h

```

## 3. Code Formatting with `clang-format`

Use `.clang-format` file in the root of the repository to standardize formatting.

```bash
clang-format -i src/*.c include/*.h
```

Use git hooks to enforce formatting on commit.

---

## 4. Static Analysis

Integrate static analyzers in your CI.

### Tools

- **Cppcheck**
  ```bash
  cppcheck --enable=all --inconclusive src/
  ```
---


## 5. Testing

Testing is critical in embedded firmware where debugging is limited.

### Test Types

- **Black-box Testing:**
  Focus on inputs and expected outputs without considering internal code structure.

- **Regression Testing:**
  Ensure old features continue working after updates.

- **SIL/HIL Testing:**
  - SIL: Software in Loop, Test if the device will work for long time.
  - HIL: Hardware in Loop, Test if the both hw and sw will work for long time without any issues.

---

## 6. Versioning

Use [Semantic Versioning 2.0.0](https://semver.org/) (`MAJOR.MINOR.PATCH`) to track changes.

### Embedding Version in Firmware

Use tools like [VTrace](https://github.com/segin-GH/VTrace) to embed version info at build time.

```c
/* This file was auto generated from VTrace. */

#pragma once

#define TIMESTAMP         "2025-05-01 14:32:11.230176"
#define GIT_BRANCH        "main"
#define GIT_DESCRIBE      "v1.0.3-5-g3f8a2cd-dirty"
#define GIT_SHA           "3f8a2cd123456789abcdefabcdef123456789abc"
#define GIT_SHA_SHORT     "3f8a2cd"
#define GIT_TAG           "v1.0.3-dirty"
```
---

## 7. GitHub Workflow

### Commits

Follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/#summary):

```
feat(sensor): add temperature calibration
fix(comm): resolve UART framing bug
```

### Branching

#### Main Branches

##### `main`
- Contains stable code currently running in production.
- Protected: Only accepts PRs from `release/` and `hotfix/` branches.
- Tagged with stable production releases (e.g., `v1.0.0`).
##### `dev/v<version>`
- Main development branch for a specific upcoming version.
- All feature branches should branch off from here.
- Example: `dev/v1.1.0`
---

#### Feature Branches
##### `feat/v<version>_<feature-name>`
- Used to develop a specific feature for a targeted version.
- **Branch from:** `dev/v<version>`  
- **Merge into:** `dev/v<version>`
- Naming Example: `feat/v1.1.0_ble-support`

---

#### Release Branches
##### `release/v<version>`
- For final stabilization and testing before going to production.
- Only bug fixes, documentation updates, and version bumps are allowed.
- **Branch from:** `dev/v<version>`
- **Merge into:** `main` and `dev/v<next-version>`
- Naming Example: `release/v1.1.0`

---

#### Hotfix Branches
##### `hotfix/v<version>_<hotfix-name>`
- For urgent fixes directly in production.
- **Branch from:** `main`
- **Merge into:** `main` and `dev/v<version>`
- Naming Example: `hotfix/v1.0.0_uart-fix`

---

#### Version Tagging

- Tags use [Semantic Versioning](https://semver.org/)
- Format: `v<MAJOR>.<MINOR>.<PATCH>`
- Examples:
  - Initial release: `v1.0.0`
  - Patch fix: `v1.0.1`
  - Minor feature release: `v1.1.0`
  - Breaking changes: `v2.0.0`
---

#### Workflow Summary

| Task             | Start From      | New Branch                      | Merge Target(s)          |
|------------------|------------------|----------------------------------|---------------------------|
| New Feature       | `dev/v1.x.0`     | `feature/v1.x.0_<feature>`      | `dev/v1.x.0`              |
| Start Release     | `dev/v1.x.0`     | `release/v1.x.0`                | `main`, `dev/v1.(x+1).0`  |
| Urgent Hotfix     | `main`           | `hotfix/v1.x.0_<hotfix>`        | `main`, `dev/v1.x.0`      |
| Final Release     | `release/v1.x.0` | _Tag as_ `v1.x.0`               | -                         |
---

## 8. Development Phase

### Architecture Diagram
  - Document the high-level architecture of the system.
  - Ensure shared understanding of system components, data flow, and responsibilities.
  - Update the diagram as the system evolves to reflect the current state.
  - Helps identify potential risks or overlaps early in the development cycle.

### Code Reviews
- Enforce coding standards and architectural guidelines.
- Detect side effects, blind spots, and regressions before they enter the main codebase.
- Encourage peer learning and shared ownership of the code.
- Ensure each Pull Request (PR) has:
    - Clear title and description.
    - Linked issue or the feature request.
    - Tests (What tests you have done and the results).

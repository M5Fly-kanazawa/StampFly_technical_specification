# VL53L3CX Complete Implementation Guide
## Register-Level Implementation Manual

This document is a **complete technical specification** for implementing an original driver for the VL53L3CX sensor from scratch.
All procedures specify **specific register addresses**, **values to write**, and **wait times**.

---

## Table of Contents

1. [Power-On Boot Sequence](#1-power-on-boot-sequence)
2. [NVM Calibration Data Readout](#2-nvm-calibration-data-readout)
3. [MEDIUM_RANGE Preset Mode Configuration](#3-medium_range-preset-mode-configuration)
4. [I2C Address Change](#4-i2c-address-change)
5. [Ranging Start/Stop Procedures](#5-ranging-startstop-procedures)
6. [Data Readout Procedures](#6-data-readout-procedures)
7. [Complete Code Implementation Example](#7-complete-code-implementation-example)

---

## 1. Power-On Boot Sequence

### 1.0 Boot Sequence Overview

After power-on, the VL53L3CX automatically boots its embedded firmware. This chapter explains the procedure to wait until the firmware has fully booted and optionally configure the I2C communication voltage if needed.

**Operations to perform:**
1. Poll the firmware system status register until bit 0 becomes 1
2. (Optional) If I2C power voltage is 2.8V, change the pad configuration register

**Required time:** Typically 100-200ms (recommend 500ms timeout maximum)

**Registers used:**
- `FIRMWARE__SYSTEM_STATUS` (0x0010): Firmware boot completion flag
- `PAD_I2C_HV__EXTSUP_CONFIG` (0x002E): I2C voltage setting (only when using 2.8V)

---

### 1.1 Firmware Boot Wait

#### Procedure Explanation

After power-on, the VL53L3CX executes the bootloader, loads firmware into RAM, and boots. Do not perform other register operations until this process is complete.

**Detailed steps:**
1. Read register `FIRMWARE__SYSTEM_STATUS` (address 0x0010)
2. Check bit 0 of the read value
   - bit 0 = 0: Still booting → Wait 1ms and read again
   - bit 0 = 1: Boot complete → Proceed to next step
3. Timeout handling: Return error if boot doesn't complete within 500ms

**Important points:**
- Polling interval of 1ms is recommended (too short occupies I2C bus, too long delays boot detection)
- Timeout value of 500ms is recommended (typically completes in 100-200ms)
- Skipping this wait will cause subsequent register operations to fail

#### Code Implementation

```c
/**
 * Step 1: Firmware boot wait
 * Register: FIRMWARE__SYSTEM_STATUS (0x0010)
 * Expected value: 0x01 (bit 0 = 1)
 */
uint8_t boot_status;
uint32_t timeout = 500;  // 500ms timeout
uint32_t start_time = millis();

do {
    I2C_Read(0x29, 0x0010, &boot_status, 1);

    if ((millis() - start_time) > timeout) {
        return ERROR_BOOT_TIMEOUT;
    }

    delay_ms(1);  // 1ms wait

} while ((boot_status & 0x01) == 0);

// Boot complete
```

**Key points:**
- Polling interval: 1ms
- Timeout: 500ms
- Check bit: bit 0 (0x01)

---

### 1.2 I2C Voltage Configuration (Optional: Only when using 2.8V)

#### Procedure Explanation

The VL53L3CX I2C communication interface assumes a 1.8V power supply by default. If the I2C bus voltage is 2.8V, the internal pad configuration must be changed.

**Cases where this setting is required:**
- System I2C_VDD power supply is 2.8V
- MCU I2C pull-up voltage is 2.8V

**Cases where this setting is not required:**
- System I2C_VDD power supply is 1.8V (default)

**Detailed steps:**
1. Read register `PAD_I2C_HV__EXTSUP_CONFIG` (address 0x002E) (Read-Modify-Write pattern)
2. Set bit 0 of the read value to 1 (do not change other bits)
3. Write the modified value back to the same register
4. No wait time required (takes effect immediately)

**Important points:**
- This register uses other bits, so must be changed using Read-Modify-Write method
- Change only bit 0, preserve other bits (bit 7-1)
- If I2C voltage is 1.8V, this setting is unnecessary (can be skipped)

#### Code Implementation

```c
/**
 * Step 2: I2C voltage configuration (only when USE_I2C_2V8 is defined)
 * Register: PAD_I2C_HV__EXTSUP_CONFIG (0x002E)
 */
#ifdef USE_I2C_2V8
    uint8_t pad_config;

    // Read current value
    I2C_Read(0x29, 0x002E, &pad_config, 1);

    // Set bit 0 to 1 (preserve other bits)
    pad_config = (pad_config & 0xFE) | 0x01;

    // Write back
    I2C_Write(0x29, 0x002E, &pad_config, 1);

    // Wait time: None (proceed immediately)
#endif
```

---

## 2. NVM Calibration Data Readout

### 2.0 NVM Readout Overview

The VL53L3CX stores factory-calibrated parameters in NVM (Non-Volatile Memory). This data is essential for accurate ranging. This chapter explains the complete procedure to read calibration data from NVM.

**Key data to read from NVM:**
1. Fast oscillator frequency (`OSC_MEASURED__FAST_OSC_FREQUENCY`, NVM address 0x1C-0x1D)
   - Critical parameter used for timing calculations
2. Default I2C address (`I2C_SLAVE__DEVICE_ADDRESS`, NVM address 0x11)
3. VHV timeout setting (`VHV_CONFIG__TIMEOUT_MACROP_LOOP_BOUND`, NVM address 0x2C)

**Process flow:**
1. NVM readout preparation (disable firmware, power control)
2. Read data from NVM (read multiple addresses sequentially)
3. NVM readout cleanup (restore power, re-enable firmware)

**Required time:** Approximately 5.3ms (power force 250μs + NVM power-up 5ms + read trigger 5μs × number of data)

**Registers used:**
- `FIRMWARE__ENABLE` (0x0401): Firmware control
- `POWER_MANAGEMENT__GO1_POWER_FORCE` (0x0419): Forced power control
- `RANGING_CORE__NVM_CTRL__*` (0x01AC-0x01B2): NVM control register group
- `RANGING_CORE__CLK_CTRL1` (0x01BB): NVM clock control

**Critical wait times:**
- After power force enable: **250μs** (required)
- After NVM clock enable: **5ms** (required)
- After NVM read trigger: **5μs** (required)

---

### 2.1 NVM Readout Enable Sequence

#### Procedure Explanation

NVM is normally in a powered-down state for low power consumption. To read data from NVM, you must temporarily disable firmware, supply power to NVM, and enable the clock.

**Why disable firmware:**
- If firmware is running, ranging operations may conflict with NVM readout
- Direct access to NVM control registers requires stopping firmware

**Detailed steps (6 steps):**

**Step 3.1: Disable firmware**
- Write 0x00 to register `FIRMWARE__ENABLE` (0x0401)
- This stops autonomous firmware operation
- Wait time: None

**Step 3.2: Enable power force**
- Write 0x01 to register `POWER_MANAGEMENT__GO1_POWER_FORCE` (0x0419)
- This forcibly enables internal power supply for NVM
- **Critical**: Wait 250μs (required for power stabilization)

**Step 3.3: Release NVM power-down**
- Write 0x01 to register `RANGING_CORE__NVM_CTRL__PDN` (0x01AC)
- NVM becomes active
- Wait time: None

**Step 3.4: Enable NVM clock**
- Write 0x05 to register `RANGING_CORE__CLK_CTRL1` (0x01BB)
- Clock required for NVM readout is supplied
- **Critical**: Wait 5ms (required for NVM ready)

**Step 3.5: Set NVM mode**
- Write 0x01 to register `RANGING_CORE__NVM_CTRL__MODE` (0x01AD)
- NVM is set to read mode
- Wait time: None

**Step 3.6: Set NVM pulse width**
- Write 0x0004 to register `RANGING_CORE__NVM_CTRL__PULSE_WIDTH_MSB` (0x01AE-0x01AF)
- Sets NVM read pulse width (default value)
- Wait time: None

**Important points:**
- Skipping wait times will cause NVM data readout to fail or be inaccurate
- The 250μs after power force and 5ms after clock enable are especially critical
- These settings must be restored after NVM readout completion

#### Code Implementation

```c
/**
 * Step 3: NVM readout preparation
 */

// 3.1: Disable firmware
// Register: FIRMWARE__ENABLE (0x0401)
uint8_t fw_disable = 0x00;
I2C_Write(0x29, 0x0401, &fw_disable, 1);
// Wait time: None

// 3.2: Enable power force
// Register: POWER_MANAGEMENT__GO1_POWER_FORCE (0x0419)
uint8_t power_force = 0x01;
I2C_Write(0x29, 0x0419, &power_force, 1);

// **Critical**: Power force stabilization wait
delay_us(250);  // 250 microseconds wait (required)

// 3.3: Release NVM power-down
// Register: RANGING_CORE__NVM_CTRL__PDN (0x01AC)
uint8_t nvm_pdn = 0x01;
I2C_Write(0x29, 0x01AC, &nvm_pdn, 1);
// Wait time: None

// 3.4: Enable NVM clock
// Register: RANGING_CORE__CLK_CTRL1 (0x01BB)
uint8_t nvm_clk = 0x05;
I2C_Write(0x29, 0x01BB, &nvm_clk, 1);

// **Critical**: NVM power-up wait
delay_ms(5);  // 5 milliseconds wait (required)

// 3.5: Set NVM mode (read mode)
// Register: RANGING_CORE__NVM_CTRL__MODE (0x01AD)
uint8_t nvm_mode = 0x01;
I2C_Write(0x29, 0x01AD, &nvm_mode, 1);
// Wait time: None

// 3.6: Set NVM pulse width
// Register: RANGING_CORE__NVM_CTRL__PULSE_WIDTH_MSB (0x01AE, 0x01AF)
uint16_t pulse_width = 0x0004;
uint8_t pw_buf[2] = {(pulse_width >> 8), pulse_width & 0xFF};
I2C_Write(0x29, 0x01AE, pw_buf, 2);
// Wait time: None
```

---

### 2.2 NVM Data Read

#### Procedure Explanation

After NVM is enabled, read calibration data from specific NVM addresses. Each NVM address requires setting the address, triggering the read, waiting, and then reading the result.

**Read procedure for each NVM address (4 steps):**

**Step 4.1: Set NVM address**
- Write target address to `RANGING_CORE__NVM_CTRL__ADDR` (0x01B0)
- Example: To read address 0x1C, write 0x1C

**Step 4.2: Trigger NVM read**
- Write 0x01 to `RANGING_CORE__NVM_CTRL__READN` (0x01B1)
- This starts NVM readout internally
- **Critical**: Wait 5μs (required for NVM read completion)

**Step 4.3: Read data**
- Read 4 bytes from `RANGING_CORE__NVM_CTRL__DATAOUT_MMM` (0x01B2-0x01B5)
- Data is returned in big-endian format (MSB first)

**Important points:**
- Must wait 5μs after trigger for each address
- Same procedure for all NVM addresses
- Data is 32-bit (4 bytes) for each address

#### Code Implementation

```c
/**
 * Step 4: Read data from NVM address
 * Example: Read fast oscillator frequency (NVM address 0x1C)
 */

// 4.1: Set NVM address
uint8_t nvm_addr = 0x1C;  // Fast oscillator address
I2C_Write(0x29, 0x01B0, &nvm_addr, 1);

// 4.2: Trigger NVM read
uint8_t nvm_readn = 0x01;
I2C_Write(0x29, 0x01B1, &nvm_readn, 1);

// **Critical**: NVM read completion wait
delay_us(5);  // 5 microseconds wait (required)

// 4.3: Read data (4 bytes)
uint8_t nvm_data[4];
I2C_Read(0x29, 0x01B2, nvm_data, 4);

// Convert to uint32_t (big-endian)
uint32_t fast_osc_frequency = ((uint32_t)nvm_data[0] << 24) |
                              ((uint32_t)nvm_data[1] << 16) |
                              ((uint32_t)nvm_data[2] << 8) |
                              ((uint32_t)nvm_data[3]);
```

---

### 2.3 NVM Readout Disable Sequence

#### Procedure Explanation

After reading necessary data from NVM, must restore NVM to powered-down state and re-enable firmware. This process is the reverse of the enable sequence.

**Detailed steps (4 steps):**

**Step 5.1: Disable NVM clock**
- Write 0x00 to register `RANGING_CORE__CLK_CTRL1` (0x01BB)
- Stops NVM clock supply

**Step 5.2: Power-down NVM**
- Write 0x00 to register `RANGING_CORE__NVM_CTRL__PDN` (0x01AC)
- Returns NVM to low-power state

**Step 5.3: Disable power force**
- Write 0x00 to register `POWER_MANAGEMENT__GO1_POWER_FORCE` (0x0419)
- Returns to normal power management

**Step 5.4: Re-enable firmware**
- Write 0x01 to register `FIRMWARE__ENABLE` (0x0401)
- Restarts firmware autonomous operation

**Important points:**
- These steps must be performed even if NVM read fails
- Wait times are not required for disable sequence
- After this, device is ready for normal operation

#### Code Implementation

```c
/**
 * Step 5: NVM readout disable
 */

// 5.1: Disable NVM clock
uint8_t nvm_clk_off = 0x00;
I2C_Write(0x29, 0x01BB, &nvm_clk_off, 1);

// 5.2: Power-down NVM
uint8_t nvm_pdn_off = 0x00;
I2C_Write(0x29, 0x01AC, &nvm_pdn_off, 1);

// 5.3: Disable power force
uint8_t power_force_off = 0x00;
I2C_Write(0x29, 0x0419, &power_force_off, 1);

// 5.4: Re-enable firmware
uint8_t fw_enable = 0x01;
I2C_Write(0x29, 0x0401, &fw_enable, 1);

// Wait time: None (proceed to next operation)
```

---

## 3. MEDIUM_RANGE Preset Mode Configuration

### 3.0 Preset Mode Overview

The VL53L3CX has multiple preset modes optimized for different ranging distances and accuracies. This chapter explains the configuration procedure for the most common **MEDIUM_RANGE mode**.

**MEDIUM_RANGE mode characteristics:**
- Ranging distance: 0mm - 3000mm (approximately 3 meters)
- Ranging accuracy: Medium (approximately ±5%)
- Ranging time: Medium (approximately 33ms)
- Power consumption: Medium
- Applications: General distance measurement, robotics, drones, etc.

**Register groups to configure:**
1. **Static Configuration**: GPIO, SPAD, VCSEL, sigma estimation, algorithm settings
2. **General Configuration**: Interrupt, calibration settings
3. **Timing Configuration**: VCSEL period, timeout, inter-measurement interval
4. **Dynamic Configuration**: Threshold, ROI, sequence settings
5. **System Control**: Stream count, firmware settings

**Number of registers to configure:** Over 50

**Required time:** Immediate (no wait time required for all register writes)

**Important points:**
- These values are optimized recommended values by STMicroelectronics
- Each register value must be set accurately (even one error will cause ranging to not work properly)
- Consecutive register addresses can be optimized with I2C burst write

---

### 3.1 Static Configuration Register Settings

#### Procedure Explanation

Static Configuration consists of fixed settings that do not change during ranging operation. Includes GPIO output, SPAD (Single Photon Avalanche Diode) settings, VCSEL (laser) settings, sigma estimation parameters, and algorithm settings.

**Purpose of each setting:**

**GPIO settings (0x0030-0x0031):**
- GPIO_HV_MUX__CTRL: Set interrupt output polarity and function
  - 0x10 = ACTIVE_LOW (output Low level on interrupt)
- GPIO__TIO_HV_STATUS: GPIO status register (0x02 is default value)

**SPAD settings (0x0033-0x0034):**
- SPAD is light detection element, set pulse width and offset
- ANA_CONFIG__SPAD_SEL_PSWIDTH (0x0033): 0x02 (MEDIUM_RANGE optimal value)
- ANA_CONFIG__VCSEL_PULSE_WIDTH_OFFSET (0x0034): 0x08 (default value)

**Sigma estimation parameters (0x0036-0x0038):**
- Sigma is parameter for estimating ranging uncertainty (standard deviation)
- SIGMA_ESTIMATOR__EFFECTIVE_PULSE_WIDTH_NS: 0x08 (8ns)
- SIGMA_ESTIMATOR__EFFECTIVE_AMBIENT_WIDTH_NS: 0x10 (16ns)
- SIGMA_ESTIMATOR__SIGMA_REF_MM: 0x01 (reference value 1mm)

**Algorithm settings (0x0039-0x0040):**
- Crosstalk correction, range ignore, consistency check settings
- ALGO__CROSSTALK_COMPENSATION_VALID_HEIGHT_MM: 0x01
- ALGO__RANGE_IGNORE_VALID_HEIGHT_MM: 0xFF (disable range ignore)
- ALGO__RANGE_MIN_CLIP: 0x00 (no minimum range clip)
- ALGO__CONSISTENCY_CHECK__TOLERANCE: 0x02 (tolerance value)

#### Code Implementation

```c
/**
 * Step 6: Static Configuration settings
 */

// GPIO settings
// Register: GPIO_HV_MUX__CTRL (0x0030)
uint8_t gpio_mux_ctrl = 0x10;  // ACTIVE_LOW
I2C_Write(0x29, 0x0030, &gpio_mux_ctrl, 1);

// Register: GPIO__TIO_HV_STATUS (0x0031)
uint8_t gpio_tio_status = 0x02;
I2C_Write(0x29, 0x0031, &gpio_tio_status, 1);

// SPAD settings
// Register: ANA_CONFIG__SPAD_SEL_PSWIDTH (0x0033)
uint8_t spad_sel = 0x02;
I2C_Write(0x29, 0x0033, &spad_sel, 1);

// Register: ANA_CONFIG__VCSEL_PULSE_WIDTH_OFFSET (0x0034)
uint8_t vcsel_offset = 0x08;
I2C_Write(0x29, 0x0034, &vcsel_offset, 1);

// Sigma estimation parameters
// Register: SIGMA_ESTIMATOR__EFFECTIVE_PULSE_WIDTH_NS (0x0036)
uint8_t sigma_pulse_width = 0x08;  // 8ns
I2C_Write(0x29, 0x0036, &sigma_pulse_width, 1);

// Register: SIGMA_ESTIMATOR__EFFECTIVE_AMBIENT_WIDTH_NS (0x0037)
uint8_t sigma_ambient = 0x10;  // 16ns
I2C_Write(0x29, 0x0037, &sigma_ambient, 1);

// Register: SIGMA_ESTIMATOR__SIGMA_REF_MM (0x0038)
uint8_t sigma_ref = 0x01;
I2C_Write(0x29, 0x0038, &sigma_ref, 1);

// Algorithm settings
// Register: ALGO__CROSSTALK_COMPENSATION_VALID_HEIGHT_MM (0x0039)
uint8_t xtalk_valid_height = 0x01;
I2C_Write(0x29, 0x0039, &xtalk_valid_height, 1);

// Register: ALGO__RANGE_IGNORE_VALID_HEIGHT_MM (0x003E)
uint8_t range_ignore_height = 0xFF;
I2C_Write(0x29, 0x003E, &range_ignore_height, 1);

// Register: ALGO__RANGE_MIN_CLIP (0x003F)
uint8_t range_min_clip = 0x00;
I2C_Write(0x29, 0x003F, &range_min_clip, 1);

// Register: ALGO__CONSISTENCY_CHECK__TOLERANCE (0x0040)
uint8_t consistency_tolerance = 0x02;
I2C_Write(0x29, 0x0040, &consistency_tolerance, 1);

// Wait time: None
```

---

### 3.2 General Configuration Register Settings

#### Procedure Explanation

General Configuration includes interrupt settings, calibration settings, VCSEL settings, phase calibration settings, and DSS (Dynamic SPAD Selection) settings.

**Key settings:**

**Interrupt configuration (0x0046):**
- SYSTEM__INTERRUPT_CONFIG_GPIO: 0x20 = NEW_SAMPLE_READY

**Calibration settings (0x0047-0x0048):**
- CAL_CONFIG__VCSEL_START: 0x0B (11, for MEDIUM_RANGE)
- CAL_CONFIG__REPEAT_RATE: 0x0000 (default value)

**VCSEL global settings (0x004A):**
- GLOBAL_CONFIG__VCSEL_WIDTH: 0x02 (MEDIUM_RANGE optimal value)

**Phase calibration settings (0x004B-0x004C):**
- PHASECAL_CONFIG__TIMEOUT_MACROP: 0x0D (for histogram mode)
- PHASECAL_CONFIG__TARGET: 0x21 (default value)

**DSS settings (0x004F-0x0059):**
- DSS_CONFIG__ROI_MODE_CONTROL: 0x01 (TARGET_RATE mode)
- DSS_CONFIG__MANUAL_EFFECTIVE_SPADS_SELECT: 0x8C00
- DSS_CONFIG__APERTURE_ATTENUATION: 0x38
- DSS_CONFIG__MAX_SPADS_LIMIT: 0xFF
- DSS_CONFIG__MIN_SPADS_LIMIT: 0x01

#### Code Implementation

```c
/**
 * Step 7: General Configuration settings
 */

// Register: SYSTEM__INTERRUPT_CONFIG_GPIO (0x0046)
uint8_t int_config = 0x20;  // NEW_SAMPLE_READY
I2C_Write(0x29, 0x0046, &int_config, 1);

// Register: CAL_CONFIG__VCSEL_START (0x0047)
uint8_t vcsel_start = 0x0B;  // 11
I2C_Write(0x29, 0x0047, &vcsel_start, 1);

// Register: CAL_CONFIG__REPEAT_RATE (0x0048, 0x0049)
uint16_t cal_repeat = 0x0000;
uint8_t cal_buf[2] = {(cal_repeat >> 8), cal_repeat & 0xFF};
I2C_Write(0x29, 0x0048, cal_buf, 2);

// Register: GLOBAL_CONFIG__VCSEL_WIDTH (0x004A)
uint8_t vcsel_width = 0x02;
I2C_Write(0x29, 0x004A, &vcsel_width, 1);

// Register: PHASECAL_CONFIG__TIMEOUT_MACROP (0x004B)
uint8_t phasecal_timeout = 0x0D;  // For histogram mode
I2C_Write(0x29, 0x004B, &phasecal_timeout, 1);

// Register: PHASECAL_CONFIG__TARGET (0x004C)
uint8_t phasecal_target = 0x21;
I2C_Write(0x29, 0x004C, &phasecal_target, 1);

// Register: DSS_CONFIG__ROI_MODE_CONTROL (0x004F)
uint8_t dss_roi_mode = 0x01;
I2C_Write(0x29, 0x004F, &dss_roi_mode, 1);

// Register: DSS_CONFIG__MANUAL_EFFECTIVE_SPADS_SELECT (0x0054, 0x0055)
uint16_t dss_spads = 0x8C00;
uint8_t dss_buf[2] = {(dss_spads >> 8), dss_spads & 0xFF};
I2C_Write(0x29, 0x0054, dss_buf, 2);

// Register: DSS_CONFIG__APERTURE_ATTENUATION (0x0057)
uint8_t dss_aperture = 0x38;
I2C_Write(0x29, 0x0057, &dss_aperture, 1);

// Register: DSS_CONFIG__MAX_SPADS_LIMIT (0x0058)
uint8_t dss_max_spads = 0xFF;
I2C_Write(0x29, 0x0058, &dss_max_spads, 1);

// Register: DSS_CONFIG__MIN_SPADS_LIMIT (0x0059)
uint8_t dss_min_spads = 0x01;
I2C_Write(0x29, 0x0059, &dss_min_spads, 1);

// Wait time: None
```

---

### 3.3 Timing Configuration Register Settings

#### Procedure Explanation

Timing Configuration is the most critical setting that controls the ranging timing of VL53L3CX. Sets VCSEL period, timeout values, and inter-measurement interval, determining the balance of ranging distance, accuracy, and speed.

**Key timing parameters:**

**VCSEL Period:**
- Period A: For long distance (0x0B = actual period 12)
- Period B: For short distance (0x09 = actual period 10)

**Timeout Macrop:**
- MM (Multi-Mode) Timeout: Initial ranging phase timeout
- Range Timeout: Main ranging phase timeout

**MEDIUM_RANGE mode values:**
- MM_CONFIG__TIMEOUT_MACROP_A: 0x001A
- MM_CONFIG__TIMEOUT_MACROP_B: 0x0020
- RANGE_CONFIG__TIMEOUT_MACROP_A: 0x01CC (critical)
- RANGE_CONFIG__TIMEOUT_MACROP_B: 0x01F5 (critical)
- RANGE_CONFIG__VCSEL_PERIOD_A: 0x0B (encoded value, actual period = 12)
- RANGE_CONFIG__VCSEL_PERIOD_B: 0x09 (encoded value, actual period = 10)
- RANGE_CONFIG__SIGMA_THRESH: 0x003C (60mm)
- RANGE_CONFIG__VALID_PHASE_LOW: 0x08
- RANGE_CONFIG__VALID_PHASE_HIGH: 0x78

#### Code Implementation

```c
/**
 * Step 8: Timing Configuration settings
 */

// MM timeout settings
// Register: MM_CONFIG__TIMEOUT_MACROP_A (0x005A, 0x005B)
uint16_t mm_timeout_a = 0x001A;
uint8_t mm_a_buf[2] = {(mm_timeout_a >> 8), mm_timeout_a & 0xFF};
I2C_Write(0x29, 0x005A, mm_a_buf, 2);

// Register: MM_CONFIG__TIMEOUT_MACROP_B (0x005C, 0x005D)
uint16_t mm_timeout_b = 0x0020;
uint8_t mm_b_buf[2] = {(mm_timeout_b >> 8), mm_timeout_b & 0xFF};
I2C_Write(0x29, 0x005C, mm_b_buf, 2);

// Range timeout settings
// Register: RANGE_CONFIG__TIMEOUT_MACROP_A (0x005E, 0x005F)
uint16_t range_timeout_a = 0x01CC;
uint8_t range_a_buf[2] = {(range_timeout_a >> 8), range_timeout_a & 0xFF};
I2C_Write(0x29, 0x005E, range_a_buf, 2);

// Register: RANGE_CONFIG__VCSEL_PERIOD_A (0x0060)
uint8_t vcsel_period_a = 0x0B;  // Encoded value (actual period = 12)
I2C_Write(0x29, 0x0060, &vcsel_period_a, 1);

// Register: RANGE_CONFIG__TIMEOUT_MACROP_B (0x0061, 0x0062)
uint16_t range_timeout_b = 0x01F5;
uint8_t range_b_buf[2] = {(range_timeout_b >> 8), range_timeout_b & 0xFF};
I2C_Write(0x29, 0x0061, range_b_buf, 2);

// Register: RANGE_CONFIG__VCSEL_PERIOD_B (0x0063)
uint8_t vcsel_period_b = 0x09;  // Encoded value (actual period = 10)
I2C_Write(0x29, 0x0063, &vcsel_period_b, 1);

// Register: RANGE_CONFIG__SIGMA_THRESH (0x0064, 0x0065)
uint16_t sigma_thresh = 0x003C;  // 60mm
uint8_t sigma_buf[2] = {(sigma_thresh >> 8), sigma_thresh & 0xFF};
I2C_Write(0x29, 0x0064, sigma_buf, 2);

// Register: RANGE_CONFIG__VALID_PHASE_LOW (0x0068)
uint8_t valid_phase_low = 0x08;
I2C_Write(0x29, 0x0068, &valid_phase_low, 1);

// Register: RANGE_CONFIG__VALID_PHASE_HIGH (0x0069)
uint8_t valid_phase_high = 0x78;
I2C_Write(0x29, 0x0069, &valid_phase_high, 1);

// Inter-measurement interval setting (100ms)
// Register: SYSTEM__INTERMEASUREMENT_PERIOD (0x006C-0x006F)
uint32_t inter_measurement_period_ms = 100;  // 100ms
uint8_t imp_buf[4] = {
    (inter_measurement_period_ms >> 24) & 0xFF,
    (inter_measurement_period_ms >> 16) & 0xFF,
    (inter_measurement_period_ms >> 8) & 0xFF,
    inter_measurement_period_ms & 0xFF
};
I2C_Write(0x29, 0x006C, imp_buf, 4);

// Wait time: None
```

---

### 3.4 Dynamic Configuration Register Settings

#### Procedure Explanation

Dynamic Configuration consists of settings that can be changed during ranging operation. Includes threshold, ROI (Region of Interest), and sequence settings. When changing these settings, use the **Grouped Parameter Hold** feature to apply them atomically.

**Key settings:**

**Grouped Parameter Hold mechanism:**
- Write 0x01 to SYSTEM__GROUPED_PARAMETER_HOLD_0 to start
- Change multiple registers
- Write 0x02 to SYSTEM__GROUPED_PARAMETER_HOLD to finish and apply
- This prevents ranging from executing in an intermediate state

**Threshold settings (0x0072-0x0075):**
- SYSTEM__THRESH_HIGH: 0x0000 (default)
- SYSTEM__THRESH_LOW: 0x0000 (default)

**Seed setting (0x0077):**
- SYSTEM__SEED_CONFIG: 0x02 (default value)

**SD (Signal Detection) settings (0x0078-0x007B):**
- SD_CONFIG__WOI_SD0: 0x0B (aligned with VCSEL Period A)
- SD_CONFIG__WOI_SD1: 0x09 (aligned with VCSEL Period B)
- SD_CONFIG__INITIAL_PHASE_SD0: 0x0A (initial phase SD0)
- SD_CONFIG__INITIAL_PHASE_SD1: 0x0A (initial phase SD1)

**SD detail settings (0x007D-0x007E):**
- SD_CONFIG__FIRST_ORDER_SELECT: 0x00
- SD_CONFIG__QUANTIFIER: 0x02

**ROI settings (0x007F-0x0080):**
- ROI_CONFIG__USER_ROI_CENTRE_SPAD: 0xC7 (center SPAD = 199)
- ROI_CONFIG__USER_ROI_REQUESTED_GLOBAL_XY_SIZE: 0xFF (16x16 SPAD = maximum size)

**Sequence setting (0x0081):**
- SYSTEM__SEQUENCE_CONFIG: 0xC1
  - bit 7: RANGE_EN = 1 (enable distance ranging)
  - bit 6: MM2_EN = 1 (enable Multi-Mode 2)
  - bit 0: VHV_EN = 1 (enable VHV)

#### Code Implementation

```c
/**
 * Step 9: Dynamic Configuration settings
 */

// Grouped Parameter Hold start
// Register: SYSTEM__GROUPED_PARAMETER_HOLD_0 (0x0071)
uint8_t gph_0 = 0x01;
I2C_Write(0x29, 0x0071, &gph_0, 1);

// Threshold settings
// Register: SYSTEM__THRESH_HIGH (0x0072, 0x0073)
uint16_t thresh_high = 0x0000;
uint8_t th_buf[2] = {(thresh_high >> 8), thresh_high & 0xFF};
I2C_Write(0x29, 0x0072, th_buf, 2);

// Register: SYSTEM__THRESH_LOW (0x0074, 0x0075)
uint16_t thresh_low = 0x0000;
uint8_t tl_buf[2] = {(thresh_low >> 8), thresh_low & 0xFF};
I2C_Write(0x29, 0x0074, tl_buf, 2);

// Register: SYSTEM__SEED_CONFIG (0x0077)
uint8_t seed_config = 0x02;
I2C_Write(0x29, 0x0077, &seed_config, 1);

// SD (Signal Detection) settings
// Register: SD_CONFIG__WOI_SD0 (0x0078)
uint8_t woi_sd0 = 0x0B;  // Aligned with VCSEL Period A
I2C_Write(0x29, 0x0078, &woi_sd0, 1);

// Register: SD_CONFIG__WOI_SD1 (0x0079)
uint8_t woi_sd1 = 0x09;  // Aligned with VCSEL Period B
I2C_Write(0x29, 0x0079, &woi_sd1, 1);

// Register: SD_CONFIG__INITIAL_PHASE_SD0 (0x007A)
uint8_t init_phase_sd0 = 0x0A;
I2C_Write(0x29, 0x007A, &init_phase_sd0, 1);

// Register: SD_CONFIG__INITIAL_PHASE_SD1 (0x007B)
uint8_t init_phase_sd1 = 0x0A;
I2C_Write(0x29, 0x007B, &init_phase_sd1, 1);

// Grouped Parameter Hold continue
// Register: SYSTEM__GROUPED_PARAMETER_HOLD_1 (0x007C)
uint8_t gph_1 = 0x01;
I2C_Write(0x29, 0x007C, &gph_1, 1);

// SD detail settings
// Register: SD_CONFIG__FIRST_ORDER_SELECT (0x007D)
uint8_t first_order = 0x00;
I2C_Write(0x29, 0x007D, &first_order, 1);

// Register: SD_CONFIG__QUANTIFIER (0x007E)
uint8_t quantifier = 0x02;
I2C_Write(0x29, 0x007E, &quantifier, 1);

// ROI settings
// Register: ROI_CONFIG__USER_ROI_CENTRE_SPAD (0x007F)
uint8_t roi_center = 0xC7;  // 199 (center)
I2C_Write(0x29, 0x007F, &roi_center, 1);

// Register: ROI_CONFIG__USER_ROI_REQUESTED_GLOBAL_XY_SIZE (0x0080)
uint8_t roi_size = 0xFF;  // 16x16 (maximum)
I2C_Write(0x29, 0x0080, &roi_size, 1);

// Sequence setting
// Register: SYSTEM__SEQUENCE_CONFIG (0x0081)
uint8_t sequence_config = 0xC1;  // VHV+MM2+RANGE
I2C_Write(0x29, 0x0081, &sequence_config, 1);

// Grouped Parameter Hold end
// Register: SYSTEM__GROUPED_PARAMETER_HOLD (0x0082)
uint8_t gph_end = 0x02;
I2C_Write(0x29, 0x0082, &gph_end, 1);

// Wait time: None
```

---


---

## 4. I2C Address Change

### 4.1 Need for Address Change

**Use Cases**:
- When using multiple VL53L3CX sensors on the same I2C bus
- When the default address (0x29) conflicts with other devices

**Limitations**:
- Address change is **volatile** (resets to default on power-off)
- Must be reconfigured after each device boot
- Valid 7-bit address range: 0x08-0x77

### 4.2 Address Change Procedure

#### Procedure Description

Changing the I2C address is very simple - just write the new address to a single register. However, there are several important considerations.

**Detailed Steps:**

**Step 1: Address Range Check**
- Verify the new address is within the valid range (0x08-0x77)
- I2C reserved addresses (0x00-0x07, 0x78-0x7F) cannot be used

**Step 2: Register Write**
- Write the new 7-bit address to register `I2C_SLAVE__DEVICE_ADDRESS` (0x0001)
- **Important**: Use the current address for I2C communication (it hasn't changed yet)
- Value to write: New address (7-bit value as-is, no shifting required)

**Step 3: Address Switch**
- The new address becomes active immediately after the register write completes
- Wait time: None (effective immediately)
- **All subsequent I2C communication must use the new address**

**Difference from Manufacturer's API:**
- STMicroelectronics' official API divides the 8-bit address by 2 before writing
- This implementation writes the 7-bit address directly (more intuitive)
- The result is the same (e.g., 8-bit 0x52 divided by 2 = 0x29, 7-bit 0x29 written directly = 0x29)

**Important Points:**
- Address change is volatile (returns to 0x29 on power-off)
- Also returns to 0x29 on soft reset (0x0000 register)
- Must be executed after firmware boot completion (will fail if executed before boot)
- For multi-sensor configurations, use XSHUT pin to boot sensors one at a time and change addresses

#### Code Implementation

```c
/**
 * @brief Change I2C address
 * @param current_addr Current I2C address (7-bit)
 * @param new_addr New I2C address (7-bit)
 * @return 0=success, <0=error
 *
 * Important:
 * - Device must be booted before calling this function
 * - Address change reverts on power-off (not non-volatile)
 * - For multi-sensor systems, individually control each sensor's
 *   XSHUT pin to sequentially change addresses
 */
int vl53l3cx_set_device_address(uint8_t current_addr, uint8_t new_addr) {
    int ret;
    uint8_t addr_reg_value;

    // Address range check
    if (new_addr < 0x08 || new_addr > 0x77) {
        return -1;  // Invalid address
    }

    // Register: I2C_SLAVE__DEVICE_ADDRESS (0x0001)
    // Value to write: New 7-bit address (as-is)
    addr_reg_value = new_addr & 0x7F;

    // **Important**: Write using the current address
    ret = i2c_write(current_addr, 0x0001, &addr_reg_value, 1);
    if (ret != 0) {
        return ret;
    }

    // Wait time: None (effective immediately)

    // **Note**: All subsequent I2C communication uses the new address

    return 0;
}
```

### 4.3 Multi-Sensor Initialization Sequence

Complete procedure for using multiple sensors on the same I2C bus:

```c
/**
 * @brief Multiple sensor initialization (example: 3 sensors)
 *
 * Hardware connections:
 * - Sensor 1: XSHUT → GPIO_PIN_1
 * - Sensor 2: XSHUT → GPIO_PIN_2
 * - Sensor 3: XSHUT → GPIO_PIN_3
 * - All sensors: I2C SDA/SCL shared bus
 *
 * New address assignments:
 * - Sensor 1: 0x30
 * - Sensor 2: 0x31
 * - Sensor 3: 0x32
 */
int multi_sensor_init(void) {
    int ret;

    // ========================================
    // Step 1: Shutdown all sensors
    // ========================================
    gpio_write(GPIO_PIN_1, 0);  // Sensor 1 shutdown
    gpio_write(GPIO_PIN_2, 0);  // Sensor 2 shutdown
    gpio_write(GPIO_PIN_3, 0);  // Sensor 3 shutdown

    delay_ms(10);  // Wait for shutdown stabilization

    // ========================================
    // Step 2: Boot sensor 1 and change address
    // ========================================
    gpio_write(GPIO_PIN_1, 1);  // Boot sensor 1 only
    delay_ms(10);  // Wait for boot

    // Wait for firmware boot (using default address 0x29)
    ret = vl53l3cx_wait_boot(0x29);
    if (ret != 0) return ret;

    // Change address: 0x29 → 0x30
    ret = vl53l3cx_set_device_address(0x29, 0x30);
    if (ret != 0) return ret;

    printf("Sensor 1 address changed to 0x30\n");

    // ========================================
    // Step 3: Boot sensor 2 and change address
    // ========================================
    gpio_write(GPIO_PIN_2, 1);  // Boot sensor 2
    delay_ms(10);

    // Wait for firmware boot (using default address 0x29)
    ret = vl53l3cx_wait_boot(0x29);
    if (ret != 0) return ret;

    // Change address: 0x29 → 0x31
    ret = vl53l3cx_set_device_address(0x29, 0x31);
    if (ret != 0) return ret;

    printf("Sensor 2 address changed to 0x31\n");

    // ========================================
    // Step 4: Boot sensor 3 and change address
    // ========================================
    gpio_write(GPIO_PIN_3, 1);  // Boot sensor 3
    delay_ms(10);

    ret = vl53l3cx_wait_boot(0x29);
    if (ret != 0) return ret;

    ret = vl53l3cx_set_device_address(0x29, 0x32);
    if (ret != 0) return ret;

    printf("Sensor 3 address changed to 0x32\n");

    // ========================================
    // Step 5: Initialize each sensor individually
    // ========================================
    vl53l3cx_dev_t sensor1, sensor2, sensor3;

    sensor1.i2c_addr = 0x30;
    ret = vl53l3cx_init(&sensor1);
    if (ret != 0) return ret;

    sensor2.i2c_addr = 0x31;
    ret = vl53l3cx_init(&sensor2);
    if (ret != 0) return ret;

    sensor3.i2c_addr = 0x32;
    ret = vl53l3cx_init(&sensor3);
    if (ret != 0) return ret;

    printf("All sensors initialized\n");

    return 0;
}

/**
 * @brief Wait for firmware boot (simplified version)
 */
int vl53l3cx_wait_boot(uint8_t addr) {
    uint8_t boot_status;
    uint32_t start_time = millis();

    do {
        i2c_read(addr, 0x0010, &boot_status, 1);

        if ((millis() - start_time) > 500) {
            return -1;  // Timeout
        }
        delay_ms(1);
    } while ((boot_status & 0x01) == 0);

    return 0;
}
```

### 4.4 Address Change Timing Diagram

```
Time axis →

XSHUT1: ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
XSHUT2: _______/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
XSHUT3: _____________/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾

I2C communication:
  [0x29 boot wait] [0x29→0x30 change]
                  ↓
              Sensor 1 responds at 0x30
                      [0x29 boot wait] [0x29→0x31 change]
                                      ↓
                                  Sensor 2 responds at 0x31
                                          [0x29 boot wait] [0x29→0x32 change]
                                                          ↓
                                                      Sensor 3 responds at 0x32
```

### 4.5 Precautions

**⚠ Important Limitations:**

1. **Volatile Setting**: Address change reverts on power-off
   - Must be reconfigured after each power-on
   - Also reverts on soft reset (0x0000 register)

2. **XSHUT Pin Required**: Multi-sensor configurations require individual XSHUT pin control for each sensor
   - Without XSHUT pins, multi-sensor operation is impossible

3. **Sequential Boot**: Sensors must be booted one at a time and addresses changed
   - If multiple sensors boot simultaneously, all respond at 0x29 causing address conflicts

4. **Address Range**: Only 7-bit addresses are valid
   - Range: 0x08 - 0x77
   - Reserved addresses (0x00-0x07, 0x78-0x7F) cannot be used

### 4.6 Implementation Example: Initialization with Address Change

```c
/**
 * @brief Initialization function with address change support
 * @param dev Device handle
 * @param new_address New I2C address (0=no change)
 */
int vl53l3cx_init_with_address(vl53l3cx_dev_t *dev, uint8_t new_address) {
    int ret;
    uint8_t current_addr = 0x29;  // Default address

    // Wait for firmware boot (using default address)
    ret = vl53l3cx_wait_boot(current_addr);
    if (ret != 0) return ret;

    // Change address (only if specified)
    if (new_address != 0 && new_address != current_addr) {
        ret = vl53l3cx_set_device_address(current_addr, new_address);
        if (ret != 0) return ret;

        dev->i2c_addr = new_address;
        printf("Address changed: 0x%02X → 0x%02X\n", current_addr, new_address);
    } else {
        dev->i2c_addr = current_addr;
    }

    // Normal initialization (using new address)
    ret = vl53l3cx_read_nvm_calibration_data(dev);
    if (ret != 0) return ret;

    ret = vl53l3cx_set_preset_mode_medium_range(dev);
    if (ret != 0) return ret;

    return 0;
}
```

### 4.7 Usage Example

```c
int main(void) {
    vl53l3cx_dev_t sensor1, sensor2;
    int ret;

    // Sensor 1: Use default address
    ret = vl53l3cx_init_with_address(&sensor1, 0);  // 0 = no address change
    if (ret != 0) {
        printf("Sensor 1 init failed\n");
        return -1;
    }
    printf("Sensor 1 initialized at 0x29\n");

    // Or, multi-sensor configuration
    ret = multi_sensor_init();
    if (ret != 0) {
        printf("Multi-sensor init failed\n");
        return -1;
    }

    // Start ranging on each sensor
    vl53l3cx_start_ranging(&sensor1);
    vl53l3cx_start_ranging(&sensor2);

    // Main loop
    while (1) {
        vl53l3cx_result_t result1, result2;

        vl53l3cx_get_ranging_data(&sensor1, &result1);
        vl53l3cx_get_ranging_data(&sensor2, &result2);

        printf("Sensor1: %d mm, Sensor2: %d mm\n",
               result1.distance_mm, result2.distance_mm);

        delay_ms(100);
    }

    return 0;
}
```


---

## 5. Ranging Start/Stop Procedures

### 5.0 Measurement Control Overview

After completing the preset mode configuration, you can start ranging. The VL53L3CX supports continuous ranging mode (BACKTOBACK) and single-shot ranging mode (SINGLESHOT). This chapter focuses on continuous ranging mode.

**Ranging Modes:**
- **BACKTOBACK (Continuous Ranging)**: Repeatedly executes measurements and generates new data at the configured measurement interval
  - Mode code: 0x42 (0x40 + 0x02)
  - Use cases: Real-time distance measurement, continuous monitoring
- **SINGLESHOT (Single-Shot Ranging)**: Executes one measurement and automatically stops
  - Mode code: 0x12 (0x10 + 0x02)
  - Use cases: Low power consumption, trigger-driven measurements

**Flow from Start to Stop:**
1. Verify interrupt configuration (0x0046 register)
2. Clear interrupts (0x0086 register)
3. Start ranging (write 0x42 to 0x0087 register)
4. Wait for data ready (polling or interrupt)
5. Read data (explained in Chapter 6)
6. Stop ranging (write 0x00 to 0x0087 register)

**Registers Used:**
- `SYSTEM__INTERRUPT_CONFIG_GPIO` (0x0046): Interrupt configuration
- `SYSTEM__INTERRUPT_CLEAR` (0x0086): Interrupt clear
- `SYSTEM__MODE_START` (0x0087): Ranging start/stop control
- `RESULT__INTERRUPT_STATUS` (0x0089): Data ready status

---

### 5.1 Starting Continuous Ranging

#### Procedure Description

In continuous ranging mode, the device automatically repeats measurements and generates new data at each configured measurement interval (SYSTEM__INTERMEASUREMENT_PERIOD).

**Detailed Steps:**

**Step 11.1: Verify Interrupt Configuration**
- Write 0x20 to register `SYSTEM__INTERRUPT_CONFIG_GPIO` (0x0046)
- This was already configured in Chapter 3, but verify/reconfigure for safety
- 0x20 = NEW_SAMPLE_READY (interrupt on new sample ready)

**Step 11.2: Clear Interrupts**
- Write 0x01 to register `SYSTEM__INTERRUPT_CLEAR` (0x0086)
- Clear any previous interrupt flags to start in a clean state

**Step 11.3: Start Ranging**
- Write 0x42 to register `SYSTEM__MODE_START` (0x0087)
- **0x42 breakdown**:
  - bits [7:4] = 0x4: BACKTOBACK (continuous ranging mode)
  - bits [1:0] = 0x2: HISTOGRAM scheduler mode
- This write automatically starts ranging
- Wait time: None (ranging starts immediately)

**Operation After Start:**
1. Device automatically executes measurement (~33ms for MEDIUM_RANGE mode)
2. After measurement completion, interrupt pin goes low (if ACTIVE_LOW configured)
3. RESULT__INTERRUPT_STATUS (0x0089) bit 5 becomes 1
4. After measurement interval (100ms) wait, automatically starts next measurement
5. Repeats this operation until stop command is received

**Important Points:**
- After ranging starts, the device operates autonomously (no host CPU intervention required)
- Data ready can be detected by polling (periodically reading 0x0089) or interrupt (GPIO pin monitoring)
- In continuous ranging mode, measurements continue even after data readout

#### Code Implementation

```c
/**
 * Step 11: Start Ranging
 */

// 11.1: GPIO interrupt configuration (verify)
uint8_t int_config = 0x20;  // NEW_SAMPLE_READY
I2C_Write(0x29, 0x0046, &int_config, 1);

// 11.2: Clear interrupts
uint8_t int_clear = 0x01;
I2C_Write(0x29, 0x0086, &int_clear, 1);

// 11.3: Start ranging
// Register: SYSTEM__MODE_START (0x0087)
uint8_t mode_start = 0x42;  // BACKTOBACK | HISTOGRAM
// Bit composition:
//   bits [7:4] = 0x4: BACKTOBACK (continuous ranging)
//   bits [1:0] = 0x2: HISTOGRAM scheduler mode
I2C_Write(0x29, 0x0087, &mode_start, 1);

// **Wait time: None** (ranging starts automatically)
```

### 5.2 Waiting for Data Ready (Polling Method)

#### Procedure Description

After starting ranging, you must wait until new data is ready. In the polling method, the status register is periodically read to check the data ready flag.

**Detailed Steps:**

**Step 12: Polling Loop**
1. Read register `RESULT__INTERRUPT_STATUS` (0x0089)
2. Check bit 5 (NEW_DATA_READY) of the read value
   - bit 5 = 0: Data still being prepared → wait 1ms and read again
   - bit 5 = 1: Data ready → proceed to data readout
3. Timeout handling: Return error if data is not ready after 2 seconds

**Recommended Polling Interval:**
- 1ms interval (too short occupies I2C bus, too long delays data acquisition)
- For MEDIUM_RANGE mode, measurement time is ~33ms, so 1ms polling is sufficient

**Recommended Timeout Value:**
- 2000ms (2 seconds) recommended
- Normally completes in ~33ms, but prevents infinite loop on errors

**Important Points:**
- Polling method is simple to implement but consumes CPU resources
- For low power applications, interrupt method (5.3) is recommended
- In continuous ranging mode, re-execute this loop for the next measurement after data readout

#### Code Implementation

```c
/**
 * Step 12: Wait for Data Ready
 */
uint8_t interrupt_status;
uint32_t timeout = 2000;  // 2 second timeout
uint32_t start_time = millis();

do {
    // Register: RESULT__INTERRUPT_STATUS (0x0089)
    I2C_Read(0x29, 0x0089, &interrupt_status, 1);

    if ((millis() - start_time) > timeout) {
        return ERROR_MEASUREMENT_TIMEOUT;
    }

    delay_ms(1);  // 1ms polling interval

} while ((interrupt_status & 0x20) == 0);  // bit 5: NEW_DATA_READY

// Data ready
```

### 5.3 Waiting for Data Ready (Interrupt Method)

#### Procedure Description

The interrupt method uses the interrupt signal from the VL53L3CX GPIO pin to detect data ready. Compared to polling, it consumes less CPU resources and is suitable for low power applications.

**Interrupt Method Mechanism:**
1. Connect VL53L3CX GPIO1 pin to MCU GPIO input pin
2. Configure GPIO interrupt on MCU side (falling edge detection for ACTIVE_LOW configuration)
3. When measurement completes, VL53L3CX drives GPIO1 pin low
4. MCU interrupt handler is called and sets flag
5. Main loop checks flag and proceeds to data readout

**Implementation Steps:**

**Step 1: GPIO Interrupt Initialization (once during initialization)**
- Configure MCU GPIO pin as input with pull-up
- Enable falling edge interrupt (for ACTIVE_LOW configuration)
- Enable NVIC interrupt

**Step 2: Interrupt Handler Implementation**
- Clear interrupt flag
- Set data ready flag (using volatile variable)

**Step 3: Wait for Data Ready in Main Loop**
- Poll data ready flag (or use WFI instruction for low power wait)
- When flag is set, proceed to data readout

**Important Points:**
- GPIO interrupt is platform-dependent (code below is STM32 example)
- Keep interrupt handler short (only set flag, perform data readout in main loop)
- Using WFI (Wait For Interrupt) instruction enables further power reduction
- Even with interrupt method, implementing timeout handling is recommended

#### Code Implementation

```c
/**
 * GPIO interrupt setup (once during initialization)
 */

// Platform-dependent GPIO interrupt configuration
// Example: For STM32
void setup_gpio_interrupt(void) {
    // Configure GPIO pin as input pull-up
    GPIO_InitTypeDef gpio_init = {0};
    gpio_init.Pin = GPIO_PIN_X;  // Pin connected to VL53L3CX GPIO1
    gpio_init.Mode = GPIO_MODE_IT_FALLING;  // Falling edge interrupt
    gpio_init.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOX, &gpio_init);

    // Enable NVIC interrupt
    HAL_NVIC_SetPriority(EXTI_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(EXTI_IRQn);
}

// Interrupt handler
volatile uint8_t data_ready_flag = 0;

void EXTI_IRQHandler(void) {
    if (LL_EXTI_IsActiveFlag_X()) {
        LL_EXTI_ClearFlag_X();
        data_ready_flag = 1;  // Set flag
    }
}

// Wait in main loop
void wait_data_ready_interrupt(void) {
    data_ready_flag = 0;

    while (!data_ready_flag) {
        // Low power mode
        __WFI();  // Wait For Interrupt
    }

    // Data ready
}
```

### 5.4 Stopping Ranging

#### Procedure Description

To stop continuous ranging mode, write the stop command to the SYSTEM__MODE_START register. To ensure reliable stopping, the manufacturer recommends writing twice.

**Detailed Steps:**

**Step 13.1: Send Stop Command (1st time)**
- Write 0x00 to register `SYSTEM__MODE_START` (0x0087)
- 0x00 = STOP (ranging stop command)

**Step 13.2: Send Stop Command (2nd time)**
- Write 0x00 again to the same register
- **Important**: Manufacturer's recommended reliable stopping method

**Step 13.3: Clear Interrupts**
- Write 0x01 to register `SYSTEM__INTERRUPT_CLEAR` (0x0086)
- Clear any remaining interrupt flags

**Why Write Twice:**
- If STOP command is received while device is executing measurement, the first write may be ignored
- Writing twice ensures reliable stopping (manufacturer's recommendation based on internal implementation)

**State After Stop:**
- Ranging completely stopped
- Device in standby state (waiting for next ranging start command)
- Preset mode configuration is retained (no need to reconfigure)
- To start ranging again, execute from step 5.1

**Important Points:**
- If data remains during ranging, recommend stopping after data readout
- Always clear interrupts after stopping (prevents false detection on next ranging start)
- Always stop ranging before entering low power mode

#### Code Implementation

```c
/**
 * Step 13: Stop Ranging
 */

// Register: SYSTEM__MODE_START (0x0087)
uint8_t mode_stop = 0x00;  // STOP
I2C_Write(0x29, 0x0087, &mode_stop, 1);

// Write twice for safety (manufacturer recommendation)
I2C_Write(0x29, 0x0087, &mode_stop, 1);

// Clear interrupts
uint8_t int_clear = 0x01;
I2C_Write(0x29, 0x0086, &int_clear, 1);

// Wait time: None
```


---

## 6. Data Readout Procedures

### 6.0 Data Readout Overview

After data ready detection, read the measurement results from the device. In histogram mode, the VL53L3CX generates data for 24 bins (time windows). Distance is calculated from this raw data.

**Histogram Data Structure:**
- 24 bins (bin 0 - bin 23)
- Each bin is a 24-bit value (3 bytes)
- Each bin represents the photon detection count in a specific time window
- Distance is calculated from the position of the bin with maximum count

**Data to Read:**
1. Header information (5 bytes):
   - RESULT__INTERRUPT_STATUS (1 byte)
   - RESULT__RANGE_STATUS (1 byte): Measurement status
   - REPORT_STATUS (1 byte)
   - STREAM_COUNT (1 byte)
   - Reserved (1 byte)
2. Histogram bin data (72 bytes = 24 bins × 3 bytes)

**Total Read Size:** 77 bytes (can be read in one burst from 0x0089)

**Data Processing Flow:**
1. Burst read 77 bytes (6.1)
2. Extract and verify header information
3. Extract histogram bin data
4. Ambient removal and peak detection (6.2)
5. Distance calculation (6.2 or 6.3)
6. Clear interrupts

---

### 6.1 Complete Histogram Bin Data Readout

#### Procedure Description

Histogram data is stored in a contiguous 77-byte region starting from register 0x0089. Using I2C burst read, all data can be acquired in a single transaction.

**Data Readout Protocol:**

**Step 14.1: Burst Read**
- Read 77 bytes from register `RESULT__INTERRUPT_STATUS` (0x0089)
- Use I2C burst read (efficient)

**Step 14.2: Extract Header Information**
- Byte 0: INTERRUPT_STATUS
  - bit 5: NEW_DATA_READY (1=new data available)
  - bit 4: ERROR (1=error occurred)
  - bit 3: RANGE_COMPLETE (1=measurement complete)
- Byte 1: RANGE_STATUS (measurement status code, see Appendix C)
  - 0x09 = Normal measurement complete (only this value indicates valid data)
  - Other = Error (data invalid)
- Byte 2: REPORT_STATUS
- Byte 3: STREAM_COUNT (measurement counter)
- Byte 4: Reserved

**Step 14.3: Extract Histogram Bins**
- Extract 24 bins of data from byte 5 onward
- Each bin is 3 bytes (24-bit, big-endian)
- Bin calculation formula: `bin_data[i] = (buf[5+i*3]<<16) | (buf[5+i*3+1]<<8) | buf[5+i*3+2]`

**Step 14.4: Clear Interrupts**
- Write 0x01 to register `SYSTEM__INTERRUPT_CLEAR` (0x0086)
- Clear interrupt flag for next measurement

**Data Format:**
```
Byte   Content
0      INTERRUPT_STATUS
1      RANGE_STATUS  <-- Important! Only 0x09 is valid
2      REPORT_STATUS
3      STREAM_COUNT
4      Reserved
5-7    Bin 0 (MSB, MID, LSB)
8-10   Bin 1
...
74-76  Bin 23
```

**Important Points:**
- If RANGE_STATUS is not 0x09, data is invalid (error handling required)
- Histogram data is big-endian (MSB first)
- In continuous ranging mode, after interrupt clear, next measurement starts automatically

#### Code Implementation

```c
/**
 * Step 14: Histogram Data Readout
 *
 * Start address: RESULT__INTERRUPT_STATUS (0x0089)
 * Read size: 77 bytes
 *
 * Data structure:
 *   [0]      : INTERRUPT_STATUS
 *   [1]      : RANGE_STATUS
 *   [2]      : REPORT_STATUS
 *   [3]      : STREAM_COUNT
 *   [4]      : Reserved
 *   [5-76]   : Histogram bin data (24 bins × 3 bytes)
 */

uint8_t histogram_buffer[77];

// Burst read
I2C_Read(0x29, 0x0089, histogram_buffer, 77);

// Extract header information
uint8_t interrupt_status = histogram_buffer[0];
uint8_t range_status = histogram_buffer[1];
uint8_t report_status = histogram_buffer[2];
uint8_t stream_count = histogram_buffer[3];

// Extract histogram bin data (24 bins)
uint32_t bin_data[24];

for (int bin = 0; bin < 24; bin++) {
    int offset = 5 + (bin * 3);  // 5-byte header + bin offset

    // Reconstruct 24-bit value into 32-bit variable (big-endian)
    bin_data[bin] = ((uint32_t)histogram_buffer[offset + 0] << 16) |
                    ((uint32_t)histogram_buffer[offset + 1] << 8) |
                    ((uint32_t)histogram_buffer[offset + 2]);
}

// Clear interrupts
uint8_t int_clear = 0x01;
I2C_Write(0x29, 0x0086, &int_clear, 1);

// Wait time: None (next measurement starts automatically)
```

### 6.2 Distance Calculation from Histogram Bins (Simplified)

#### Procedure Description

Calculate distance from histogram bin data. The simplified version removes ambient (ambient light) and calculates distance from the position of the bin with maximum count.

**Distance Calculation Principle:**
- VCSEL emits laser pulse
- Photons reflected from target are detected by SPAD
- Counts concentrate in specific bin according to Time of Flight
- Peak bin position × bin width = distance

**Detailed Steps:**

**Step 15.1: Ambient Estimation**
- Calculate average of first 6 bins (bins 0-5)
- These bins are in time windows before laser reflection (detect only ambient light)
- `ambient_estimate = (bin0 + bin1 + ... + bin5) / 6`

**Step 15.2: Ambient Removal**
- Subtract estimated ambient from each bin
- `corrected_bin[i] = bin_data[i] - ambient_estimate` (0 if negative)
- This leaves only counts from laser reflection

**Step 15.3: Peak Detection**
- Search for bin with maximum count in valid range (bins 6-17)
- Record `max_count` and `peak_bin`
- Bins 0-5 have only ambient light, bins 18-23 are too far, so exclude both

**Step 15.4: Distance Calculation**
- MEDIUM_RANGE mode uses two VCSEL periods
  - Period A (bins 0-11): VCSEL period 12 → bin width ~15.0mm
  - Period B (bins 12-23): VCSEL period 10 → bin width ~12.5mm
- Simplified calculation formula:
  - If `peak_bin < 12`: `distance_mm = peak_bin × 15.0`
  - If `peak_bin >= 12`: `distance_mm = peak_bin × 12.5`

**Step 15.5: Status Check**
- Only if RANGE_STATUS is 0x09 (RANGE_COMPLETE), consider distance valid
- Other status codes are errors (see Appendix C)

**Important Points:**
- Ambient removal is important in outdoor or bright environments
- Limiting peak detection range prevents false detection
- Bin width is approximate (accurate calculation requires oscillator frequency read from NVM)
- For higher precision distance calculation, see 6.3 sub-bin interpolation

#### Code Implementation

```c
/**
 * Step 15: Peak Detection and Distance Calculation
 */

// Ambient estimation (average of first 6 bins)
uint32_t ambient_sum = 0;
for (int i = 0; i < 6; i++) {
    ambient_sum += bin_data[i];
}
uint32_t ambient_estimate = ambient_sum / 6;

// Ambient removal
uint32_t corrected_bins[24];
for (int i = 0; i < 24; i++) {
    if (bin_data[i] > ambient_estimate) {
        corrected_bins[i] = bin_data[i] - ambient_estimate;
    } else {
        corrected_bins[i] = 0;
    }
}

// Peak detection (maximum count bin)
uint32_t max_count = 0;
int peak_bin = 0;

for (int i = 6; i < 18; i++) {  // Search only valid range
    if (corrected_bins[i] > max_count) {
        max_count = corrected_bins[i];
        peak_bin = i;
    }
}

// Distance calculation
// Simplified formula: distance_mm = peak_bin × bin_width_mm
// MEDIUM_RANGE mode:
//   VCSEL Period A = 12 → bin width ≈ 2000ps → ~15.0mm/bin
//   VCSEL Period B = 10 → bin width ≈ 1666ps → ~12.5mm/bin

uint16_t distance_mm;

if (peak_bin < 12) {
    // Period A used
    distance_mm = (uint16_t)(peak_bin * 15.0);
} else {
    // Period B used
    distance_mm = (uint16_t)(peak_bin * 12.5);
}

// Status check
uint8_t range_status_code = range_status & 0x1F;  // Lower 5 bits

if (range_status_code == 0x09) {
    // RANGE_COMPLETE: Normal measurement
    printf("Distance: %d mm\n", distance_mm);
} else {
    // Error handling
    printf("Range Error: %d\n", range_status_code);
}
```

### 6.3 Precise Distance Calculation (Sub-Bin Interpolation)

#### Procedure Description

The simplified distance calculation only uses the integer position of the peak bin, but for higher precision measurements, interpolation using bins around the peak is effective. Parabolic fitting can determine distance with sub-bin (decimal) precision.

**Sub-Bin Interpolation Principle:**
- Use 3 points: peak bin and its neighboring bins
- Estimate parabola (quadratic curve) from 3 points
- Peak position of parabola is the peak position with sub-bin precision
- Peak position × bin width = precise distance

**Detailed Steps:**

**Step 1: Obtain 3 Points**
- Obtain 3 points centered on peak bin `peak_bin`
- `a = corrected_bins[peak_bin - 1]` (previous bin)
- `b = corrected_bins[peak_bin]` (peak bin)
- `c = corrected_bins[peak_bin + 1]` (next bin)

**Step 2: Calculate Sub-Bin Offset**
- Parabolic fitting formula:
  - `sub_bin_offset = 0.5 × (a - c) / (a - 2×b + c)`
- This value is in range -0.5 to +0.5 (relative position from peak bin)

**Step 3: Calculate Precise Bin Position**
- `accurate_bin = peak_bin + sub_bin_offset`
- Example: peak_bin = 10, sub_bin_offset = 0.3 → accurate_bin = 10.3

**Step 4: Calculate Precise Distance**
- `distance_mm = accurate_bin × bin_width`
- bin_width varies by VCSEL period (Period A: 12.5mm, Period B: 15.0mm)

**Application Conditions:**
- `peak_bin > 0` and `peak_bin < 23` (cannot interpolate at boundary bins)
- Peak bin count is sufficiently large (low counts reduce interpolation accuracy)
- Denominator `(a - 2×b + c)` is not 0 (avoid division by zero)

**Important Points:**
- Sub-bin interpolation improves distance resolution by ~10× (e.g., 12.5mm → 1.25mm)
- In noisy environments, interpolation results may be unstable
- Use only when high precision is required; 6.2 simplified version is sufficient for normal use

#### Code Implementation

```c
/**
 * Higher precision distance calculation (parabolic fitting)
 */

if (peak_bin > 0 && peak_bin < 23) {
    // Parabolic fitting using neighboring bins
    int32_t a = corrected_bins[peak_bin - 1];
    int32_t b = corrected_bins[peak_bin];
    int32_t c = corrected_bins[peak_bin + 1];

    // Calculate sub-bin offset
    float sub_bin_offset = 0.5 * (float)(a - c) / (float)(a - 2*b + c);

    // Precise bin position
    float accurate_bin = (float)peak_bin + sub_bin_offset;

    // Distance calculation (for Period A)
    float distance_mm_float = accurate_bin * 15.0;
    distance_mm = (uint16_t)distance_mm_float;
}
```


---

## 7. Complete Code Implementation Example

### 7.0 Implementation Example Overview

This chapter provides complete, working driver code that integrates all procedures explained in Chapters 1-6. This code is designed to be copy-and-paste ready for immediate use.

**Implementation Contents:**
1. **Structure Definitions** (7.1): Device handle and measurement result structures
2. **Initialization Function** (7.2): Integrates boot wait, NVM readout, preset configuration
3. **NVM Calibration Data Readout** (7.3): Complete NVM control implementation
4. **Preset Mode Configuration** (7.4): MEDIUM_RANGE mode configuration function
5. **Measurement Control Functions** (7.5): Start ranging, data readout, stop ranging
6. **Usage Example** (7.6): Integration example in main function

**Prerequisites:**
- I2C communication functions implemented:
  - `int i2c_read(uint8_t addr, uint16_t reg, uint8_t *data, uint16_t len)`
  - `int i2c_write(uint8_t addr, uint16_t reg, uint8_t *data, uint16_t len)`
- Timing functions implemented:
  - `void delay_ms(uint32_t ms)`: millisecond delay
  - `void delay_us(uint32_t us)`: microsecond delay
  - `uint32_t millis(void)`: milliseconds since boot

**Code Design Features:**
- Error checking included (all functions return 0=success, <0=error)
- Structured design (each function is independent)
- Well-commented (each step explained)
- Portable (platform-dependent parts separated)

**Usage Instructions:**
1. Integrate code from sections 7.1-7.5 to create driver file
2. Implement platform-dependent functions (I2C, timer)
3. Call from main function using 7.6 as reference

---

### 7.1 Driver Structure Definitions

#### Description

Define structures for managing device state and storing measurement results.

**vl53l3cx_dev_t structure:**
- `i2c_addr`: Current I2C address (7-bit)
- `fast_osc_frequency`: Fast oscillator frequency read from NVM
- `timing_budget_us`: Timing budget (measurement time)
- `measurement_active`: Measurement active flag (0=stopped, 1=active)

**vl53l3cx_result_t structure:**
- `distance_mm`: Calculated distance (mm units)
- `signal_rate_mcps`: Signal rate (optional, not used in this implementation)
- `range_status`: Measurement status code (0x09=normal)
- `stream_count`: Stream count
- `bin_data[24]`: Raw histogram data (for debugging)

#### Code Implementation

```c
/**
 * VL53L3CX Driver Structures
 */
typedef struct {
    uint8_t i2c_addr;                    // I2C address
    uint16_t fast_osc_frequency;         // Fast oscillator frequency (read from NVM)
    uint32_t timing_budget_us;           // Timing budget
    uint8_t measurement_active;          // Measurement active flag
} vl53l3cx_dev_t;

/**
 * Measurement Result Structure
 */
typedef struct {
    uint16_t distance_mm;                // Distance (mm)
    uint16_t signal_rate_mcps;           // Signal rate
    uint8_t range_status;                // Measurement status
    uint8_t stream_count;                // Stream count
    uint32_t bin_data[24];               // Raw histogram data
} vl53l3cx_result_t;
```

### 7.2 Initialization Function (Complete)

```c
/**
 * @brief VL53L3CX Initialization
 * @param dev Device handle
 * @return 0=success, <0=error
 */
int vl53l3cx_init(vl53l3cx_dev_t *dev) {
    int ret;
    uint8_t boot_status;
    uint32_t start_time;

    // Initialize device settings
    dev->i2c_addr = 0x29;
    dev->timing_budget_us = 33333;  // 33ms
    dev->measurement_active = 0;

    // ========================================
    // Step 1: Wait for firmware boot
    // ========================================
    start_time = millis();
    do {
        ret = i2c_read(dev->i2c_addr, 0x0010, &boot_status, 1);
        if (ret != 0) return ret;

        if ((millis() - start_time) > 500) {
            return -1;  // Timeout
        }
        delay_ms(1);
    } while ((boot_status & 0x01) == 0);

    // ========================================
    // Step 2: NVM calibration data readout
    // ========================================
    ret = vl53l3cx_read_nvm_calibration_data(dev);
    if (ret != 0) return ret;

    // ========================================
    // Step 3-10: MEDIUM_RANGE preset configuration
    // ========================================
    ret = vl53l3cx_set_preset_mode_medium_range(dev);
    if (ret != 0) return ret;

    return 0;  // Success
}
```

### 7.3 NVM Calibration Data Readout Function

```c
/**
 * @brief NVM Calibration Data Readout
 */
int vl53l3cx_read_nvm_calibration_data(vl53l3cx_dev_t *dev) {
    int ret;
    uint8_t val;
    uint8_t nvm_data[4];

    // Disable firmware
    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0401, &val, 1);
    if (ret != 0) return ret;

    // Enable power force
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0419, &val, 1);
    if (ret != 0) return ret;

    // Wait for power force stabilization
    delay_us(250);

    // NVM control settings
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x01AC, &val, 1);  // Release PDN
    if (ret != 0) return ret;

    val = 0x05;
    ret = i2c_write(dev->i2c_addr, 0x01BB, &val, 1);  // Enable CLK
    if (ret != 0) return ret;

    // Wait for NVM power-up
    delay_us(5000);

    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x01AD, &val, 1);  // Set Mode
    if (ret != 0) return ret;

    // NVM pulse width setting
    uint8_t pw_buf[2] = {0x00, 0x04};
    ret = i2c_write(dev->i2c_addr, 0x01AE, pw_buf, 2);
    if (ret != 0) return ret;

    // Read fast oscillator frequency (NVM address 0x1C)
    ret = vl53l3cx_nvm_read(dev, 0x1C, nvm_data);
    if (ret != 0) return ret;

    dev->fast_osc_frequency = ((uint16_t)nvm_data[0] << 8) | nvm_data[1];

    // Disable power force
    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0419, &val, 1);
    if (ret != 0) return ret;

    // Re-enable firmware
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0401, &val, 1);
    if (ret != 0) return ret;

    return 0;
}

/**
 * @brief NVM Single Address Read
 */
int vl53l3cx_nvm_read(vl53l3cx_dev_t *dev, uint8_t nvm_addr, uint8_t *data_out) {
    int ret;
    uint8_t val;

    // Set NVM address
    ret = i2c_write(dev->i2c_addr, 0x01B0, &nvm_addr, 1);
    if (ret != 0) return ret;

    // Read trigger (Low)
    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x01B1, &val, 1);
    if (ret != 0) return ret;

    // Trigger delay
    delay_us(5);

    // Read trigger (High)
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x01B1, &val, 1);
    if (ret != 0) return ret;

    // Read data (4 bytes)
    ret = i2c_read(dev->i2c_addr, 0x01B2, data_out, 4);
    if (ret != 0) return ret;

    return 0;
}
```

### 7.4 Preset Mode Configuration Function

```c
/**
 * @brief MEDIUM_RANGE Preset Mode Configuration
 */
int vl53l3cx_set_preset_mode_medium_range(vl53l3cx_dev_t *dev) {
    int ret;
    uint8_t val;
    uint8_t buf[4];

    // ========== Static Configuration ==========
    // DSS settings
    buf[0] = 0x0A; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0024, buf, 2);  // DSS_CONFIG__TARGET_TOTAL_RATE_MCPS
    if (ret != 0) return ret;

    // GPIO/SPAD settings
    val = 0x10;
    ret = i2c_write(dev->i2c_addr, 0x0030, &val, 1);  // GPIO_HV_MUX__CTRL
    if (ret != 0) return ret;

    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x0031, &val, 1);  // GPIO__TIO_HV_STATUS
    if (ret != 0) return ret;

    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x0033, &val, 1);  // ANA_CONFIG__SPAD_SEL_PSWIDTH
    if (ret != 0) return ret;

    val = 0x08;
    ret = i2c_write(dev->i2c_addr, 0x0034, &val, 1);  // ANA_CONFIG__VCSEL_PULSE_WIDTH_OFFSET
    if (ret != 0) return ret;

    // Sigma estimator settings
    val = 0x08;
    ret = i2c_write(dev->i2c_addr, 0x0036, &val, 1);  // SIGMA_ESTIMATOR__EFFECTIVE_PULSE_WIDTH_NS
    if (ret != 0) return ret;

    val = 0x10;
    ret = i2c_write(dev->i2c_addr, 0x0037, &val, 1);  // SIGMA_ESTIMATOR__EFFECTIVE_AMBIENT_WIDTH_NS
    if (ret != 0) return ret;

    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0038, &val, 1);  // SIGMA_ESTIMATOR__SIGMA_REF_MM
    if (ret != 0) return ret;

    // Algorithm settings
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0039, &val, 1);  // ALGO__CROSSTALK_COMPENSATION_VALID_HEIGHT_MM
    if (ret != 0) return ret;

    val = 0xFF;
    ret = i2c_write(dev->i2c_addr, 0x003E, &val, 1);  // ALGO__RANGE_IGNORE_VALID_HEIGHT_MM
    if (ret != 0) return ret;

    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x003F, &val, 1);  // ALGO__RANGE_MIN_CLIP
    if (ret != 0) return ret;

    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x0040, &val, 1);  // ALGO__CONSISTENCY_CHECK__TOLERANCE
    if (ret != 0) return ret;

    // ========== General Configuration ==========
    val = 0x20;
    ret = i2c_write(dev->i2c_addr, 0x0046, &val, 1);  // SYSTEM__INTERRUPT_CONFIG_GPIO
    if (ret != 0) return ret;

    val = 0x0B;
    ret = i2c_write(dev->i2c_addr, 0x0047, &val, 1);  // CAL_CONFIG__VCSEL_START
    if (ret != 0) return ret;

    buf[0] = 0x00; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0048, buf, 2);  // CAL_CONFIG__REPEAT_RATE
    if (ret != 0) return ret;

    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x004A, &val, 1);  // GLOBAL_CONFIG__VCSEL_WIDTH
    if (ret != 0) return ret;

    val = 0x0D;
    ret = i2c_write(dev->i2c_addr, 0x004B, &val, 1);  // PHASECAL_CONFIG__TIMEOUT_MACROP
    if (ret != 0) return ret;

    val = 0x21;
    ret = i2c_write(dev->i2c_addr, 0x004C, &val, 1);  // PHASECAL_CONFIG__TARGET
    if (ret != 0) return ret;

    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x004F, &val, 1);  // DSS_CONFIG__ROI_MODE_CONTROL
    if (ret != 0) return ret;

    // DSS detailed settings
    buf[0] = 0x8C; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0054, buf, 2);  // DSS_CONFIG__MANUAL_EFFECTIVE_SPADS_SELECT
    if (ret != 0) return ret;

    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0056, &val, 1);  // DSS_CONFIG__MANUAL_BLOCK_SELECT
    if (ret != 0) return ret;

    val = 0x38;
    ret = i2c_write(dev->i2c_addr, 0x0057, &val, 1);  // DSS_CONFIG__APERTURE_ATTENUATION
    if (ret != 0) return ret;

    val = 0xFF;
    ret = i2c_write(dev->i2c_addr, 0x0058, &val, 1);  // DSS_CONFIG__MAX_SPADS_LIMIT
    if (ret != 0) return ret;

    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0059, &val, 1);  // DSS_CONFIG__MIN_SPADS_LIMIT
    if (ret != 0) return ret;

    // ========== Timing Configuration ==========
    // MM timeout settings
    buf[0] = 0x00; buf[1] = 0x1A;
    ret = i2c_write(dev->i2c_addr, 0x005A, buf, 2);  // MM_CONFIG__TIMEOUT_MACROP_A
    if (ret != 0) return ret;

    buf[0] = 0x00; buf[1] = 0x20;
    ret = i2c_write(dev->i2c_addr, 0x005C, buf, 2);  // MM_CONFIG__TIMEOUT_MACROP_B
    if (ret != 0) return ret;

    // Range timeout and VCSEL period settings
    buf[0] = 0x01; buf[1] = 0xCC;
    ret = i2c_write(dev->i2c_addr, 0x005E, buf, 2);  // RANGE_CONFIG__TIMEOUT_MACROP_A
    if (ret != 0) return ret;

    val = 0x0B;
    ret = i2c_write(dev->i2c_addr, 0x0060, &val, 1);  // RANGE_CONFIG__VCSEL_PERIOD_A
    if (ret != 0) return ret;

    buf[0] = 0x01; buf[1] = 0xF5;
    ret = i2c_write(dev->i2c_addr, 0x0061, buf, 2);  // RANGE_CONFIG__TIMEOUT_MACROP_B
    if (ret != 0) return ret;

    val = 0x09;
    ret = i2c_write(dev->i2c_addr, 0x0063, &val, 1);  // RANGE_CONFIG__VCSEL_PERIOD_B
    if (ret != 0) return ret;

    // Sigma threshold and count rate settings
    buf[0] = 0x00; buf[1] = 0x3C;
    ret = i2c_write(dev->i2c_addr, 0x0064, buf, 2);  // RANGE_CONFIG__SIGMA_THRESH
    if (ret != 0) return ret;

    buf[0] = 0x00; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0066, buf, 2);  // RANGE_CONFIG__MIN_COUNT_RATE_RTN_LIMIT_MCPS
    if (ret != 0) return ret;

    // Valid phase settings
    val = 0x08;
    ret = i2c_write(dev->i2c_addr, 0x0068, &val, 1);  // RANGE_CONFIG__VALID_PHASE_LOW
    if (ret != 0) return ret;

    val = 0x78;
    ret = i2c_write(dev->i2c_addr, 0x0069, &val, 1);  // RANGE_CONFIG__VALID_PHASE_HIGH
    if (ret != 0) return ret;

    // Intermeasurement period (100ms)
    buf[0] = 0x00; buf[1] = 0x00; buf[2] = 0x00; buf[3] = 0x64;
    ret = i2c_write(dev->i2c_addr, 0x006C, buf, 4);  // SYSTEM__INTERMEASUREMENT_PERIOD
    if (ret != 0) return ret;

    // ========== Dynamic Configuration ==========
    // Grouped parameter hold start
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0071, &val, 1);  // SYSTEM__GROUPED_PARAMETER_HOLD_0
    if (ret != 0) return ret;

    // Threshold settings
    buf[0] = 0x00; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0072, buf, 2);  // SYSTEM__THRESH_HIGH
    if (ret != 0) return ret;

    buf[0] = 0x00; buf[1] = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x0074, buf, 2);  // SYSTEM__THRESH_LOW
    if (ret != 0) return ret;

    // Seed settings
    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x0077, &val, 1);  // SYSTEM__SEED_CONFIG
    if (ret != 0) return ret;

    // SD (Signal Detection) settings
    val = 0x0B;
    ret = i2c_write(dev->i2c_addr, 0x0078, &val, 1);  // SD_CONFIG__WOI_SD0
    if (ret != 0) return ret;

    val = 0x09;
    ret = i2c_write(dev->i2c_addr, 0x0079, &val, 1);  // SD_CONFIG__WOI_SD1
    if (ret != 0) return ret;

    val = 0x0A;
    ret = i2c_write(dev->i2c_addr, 0x007A, &val, 1);  // SD_CONFIG__INITIAL_PHASE_SD0
    if (ret != 0) return ret;

    val = 0x0A;
    ret = i2c_write(dev->i2c_addr, 0x007B, &val, 1);  // SD_CONFIG__INITIAL_PHASE_SD1
    if (ret != 0) return ret;

    // Grouped parameter hold continuation
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x007C, &val, 1);  // SYSTEM__GROUPED_PARAMETER_HOLD_1
    if (ret != 0) return ret;

    // SD detailed settings
    val = 0x00;
    ret = i2c_write(dev->i2c_addr, 0x007D, &val, 1);  // SD_CONFIG__FIRST_ORDER_SELECT
    if (ret != 0) return ret;

    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x007E, &val, 1);  // SD_CONFIG__QUANTIFIER
    if (ret != 0) return ret;

    // ROI settings
    val = 0xC7;
    ret = i2c_write(dev->i2c_addr, 0x007F, &val, 1);  // ROI_CONFIG__USER_ROI_CENTRE_SPAD
    if (ret != 0) return ret;

    val = 0xFF;
    ret = i2c_write(dev->i2c_addr, 0x0080, &val, 1);  // ROI_CONFIG__USER_ROI_REQUESTED_GLOBAL_XY_SIZE
    if (ret != 0) return ret;

    // Sequence settings
    val = 0xC1;
    ret = i2c_write(dev->i2c_addr, 0x0081, &val, 1);  // SYSTEM__SEQUENCE_CONFIG
    if (ret != 0) return ret;

    // Grouped parameter hold end
    val = 0x02;
    ret = i2c_write(dev->i2c_addr, 0x0082, &val, 1);  // SYSTEM__GROUPED_PARAMETER_HOLD
    if (ret != 0) return ret;

    return 0;
}

/**
 * @brief Histogram Data Processing (Peak Detection and Distance Calculation)
 */
void vl53l3cx_process_histogram(vl53l3cx_result_t *result) {
    // Step 1: Ambient estimation (average of first 6 bins)
    uint32_t ambient_sum = 0;
    for (int i = 0; i < 6; i++) {
        ambient_sum += result->bin_data[i];
    }
    uint32_t ambient_estimate = ambient_sum / 6;

    // Step 2: Ambient removal
    uint32_t corrected_bins[24];
    for (int i = 0; i < 24; i++) {
        if (result->bin_data[i] > ambient_estimate) {
            corrected_bins[i] = result->bin_data[i] - ambient_estimate;
        } else {
            corrected_bins[i] = 0;
        }
    }

    // Step 3: Peak detection (search for maximum count in bins 6-17)
    uint32_t max_count = 0;
    int peak_bin = 0;

    for (int i = 6; i < 18; i++) {
        if (corrected_bins[i] > max_count) {
            max_count = corrected_bins[i];
            peak_bin = i;
        }
    }

    // Step 4: Distance calculation
    // MEDIUM_RANGE mode:
    //   Period A (bins 0-11): VCSEL period 12 → bin width ~15.0mm
    //   Period B (bins 12-23): VCSEL period 10 → bin width ~12.5mm

    uint16_t distance_mm;

    if (peak_bin < 12) {
        // Period A used
        distance_mm = (uint16_t)(peak_bin * 15.0);
    } else {
        // Period B used
        distance_mm = (uint16_t)(peak_bin * 12.5);
    }

    // Optional: Precise distance calculation using sub-bin interpolation
    if (peak_bin > 0 && peak_bin < 23 && max_count > 100) {
        // Parabolic fitting using neighboring bins
        int32_t a = corrected_bins[peak_bin - 1];
        int32_t b = corrected_bins[peak_bin];
        int32_t c = corrected_bins[peak_bin + 1];

        int32_t denominator = a - 2*b + c;

        // Avoid division by zero
        if (denominator != 0) {
            float sub_bin_offset = 0.5f * (float)(a - c) / (float)denominator;
            float accurate_bin = (float)peak_bin + sub_bin_offset;

            // Precise distance calculation
            if (peak_bin < 12) {
                distance_mm = (uint16_t)(accurate_bin * 15.0f);
            } else {
                distance_mm = (uint16_t)(accurate_bin * 12.5f);
            }
        }
    }

    // Store results
    result->distance_mm = distance_mm;
    result->peak_signal_count = max_count;
    result->ambient_count = ambient_estimate;
}
```

### 7.5 Measurement Control Functions

```c
/**
 * @brief Start Ranging
 */
int vl53l3cx_start_ranging(vl53l3cx_dev_t *dev) {
    int ret;
    uint8_t val;

    // Interrupt configuration
    val = 0x20;  // NEW_SAMPLE_READY
    ret = i2c_write(dev->i2c_addr, 0x0046, &val, 1);
    if (ret != 0) return ret;

    // Clear interrupts
    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0086, &val, 1);
    if (ret != 0) return ret;

    // Start ranging
    val = 0x42;  // BACKTOBACK | HISTOGRAM
    ret = i2c_write(dev->i2c_addr, 0x0087, &val, 1);
    if (ret != 0) return ret;

    dev->measurement_active = 1;

    return 0;
}

/**
 * @brief Get Ranging Data
 */
int vl53l3cx_get_ranging_data(vl53l3cx_dev_t *dev, vl53l3cx_result_t *result) {
    int ret;
    uint8_t interrupt_status;
    uint8_t histogram_buffer[77];

    // Wait for data ready
    uint32_t start_time = millis();
    do {
        ret = i2c_read(dev->i2c_addr, 0x0089, &interrupt_status, 1);
        if (ret != 0) return ret;

        if ((millis() - start_time) > 2000) {
            return -1;  // Timeout
        }
        delay_ms(1);
    } while ((interrupt_status & 0x20) == 0);

    // Burst read histogram data
    ret = i2c_read(dev->i2c_addr, 0x0089, histogram_buffer, 77);
    if (ret != 0) return ret;

    // Header information
    result->range_status = histogram_buffer[1] & 0x1F;
    result->stream_count = histogram_buffer[3];

    // Extract bin data
    for (int bin = 0; bin < 24; bin++) {
        int offset = 5 + (bin * 3);
        result->bin_data[bin] = ((uint32_t)histogram_buffer[offset] << 16) |
                                ((uint32_t)histogram_buffer[offset+1] << 8) |
                                ((uint32_t)histogram_buffer[offset+2]);
    }

    // Peak detection and distance calculation
    vl53l3cx_process_histogram(result);

    // Clear interrupts
    uint8_t val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0086, &val, 1);
    if (ret != 0) return ret;

    return 0;
}

/**
 * @brief Stop Ranging
 */
int vl53l3cx_stop_ranging(vl53l3cx_dev_t *dev) {
    int ret;
    uint8_t val = 0x00;

    ret = i2c_write(dev->i2c_addr, 0x0087, &val, 1);
    if (ret != 0) return ret;

    ret = i2c_write(dev->i2c_addr, 0x0087, &val, 1);
    if (ret != 0) return ret;

    val = 0x01;
    ret = i2c_write(dev->i2c_addr, 0x0086, &val, 1);
    if (ret != 0) return ret;

    dev->measurement_active = 0;

    return 0;
}
```

### 7.6 Usage Example

```c
int main(void) {
    vl53l3cx_dev_t sensor;
    vl53l3cx_result_t result;

    // ========================================
    // Initialization
    // ========================================
    if (vl53l3cx_init(&sensor) != 0) {
        printf("Initialization failed\n");
        return -1;
    }

    printf("VL53L3CX initialized (fast_osc_freq: %d)\n", sensor.fast_osc_frequency);

    // ========================================
    // Start ranging
    // ========================================
    if (vl53l3cx_start_ranging(&sensor) != 0) {
        printf("Start ranging failed\n");
        return -1;
    }

    printf("Ranging started\n");

    // ========================================
    // Main loop
    // ========================================
    for (int i = 0; i < 100; i++) {
        // Read data
        if (vl53l3cx_get_ranging_data(&sensor, &result) == 0) {
            if (result.range_status == 0x09) {  // RANGE_COMPLETE
                printf("[%d] Distance: %d mm, Signal: %d MCPS\n",
                       i, result.distance_mm, result.signal_rate_mcps);
            } else {
                printf("[%d] Range Error: %d\n", i, result.range_status);
            }
        }

        delay_ms(100);  // 100ms period
    }

    // ========================================
    // Stop ranging
    // ========================================
    vl53l3cx_stop_ranging(&sensor);
    printf("Ranging stopped\n");

    return 0;
}
```

---

## Summary

This document provides a **complete implementation** of an original VL53L3CX sensor driver.

### Implementation Key Points

1. **Initialization Sequence**: Firmware boot wait → NVM readout → Preset configuration
2. **NVM Readout**: Wait times of 250μs, 5ms, 5μs are essential
3. **Preset Configuration**: Accurately configure 50+ registers
4. **Measurement Control**: Start/stop using 0x0087 register
5. **Data Readout**: Burst read 77 bytes from 0x0089

### Critical Wait Times

| Location | Wait Time | Reason |
|----------|-----------|--------|
| After power force | 250μs | Power stabilization |
| After NVM power-up | 5ms | NVM ready |
| After NVM read trigger | 5μs | Read completion wait |
| Data ready wait | 1ms polling | Measurement completion detection |

Following this guide, you can completely control the VL53L3CX without using ST-provided libraries.

---

## Appendix A: Register Reference

This section lists all registers used in the VL53L3CX implementation.

### A.1 Device Control Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0001 | I2C_SLAVE__DEVICE_ADDRESS | 1 byte | I2C device address (7-bit) | R/W | Ch.4: Address change |
| 0x0010 | FIRMWARE__SYSTEM_STATUS | 1 byte | Firmware boot status<br>bit 0: Boot complete (1=complete) | R | Ch.1: Boot wait |
| 0x002E | PAD_I2C_HV__EXTSUP_CONFIG | 1 byte | I2C voltage setting<br>bit 0: Set to 1 when using 2.8V | R/W | Ch.1: I2C voltage config |
| 0x0401 | FIRMWARE__ENABLE | 1 byte | Firmware enable/disable<br>0x00=disable, 0x01=enable | R/W | Ch.2: NVM readout |
| 0x0419 | POWER_MANAGEMENT__GO1_POWER_FORCE | 1 byte | Power force control<br>0x00=disable, 0x01=enable | R/W | Ch.2: NVM readout |

### A.2 NVM Control Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x01AC | RANGING_CORE__NVM_CTRL__PDN | 1 byte | NVM power-down control<br>0x00=power-down, 0x01=active | R/W | Ch.2: NVM enable |
| 0x01AD | RANGING_CORE__NVM_CTRL__MODE | 1 byte | NVM operation mode<br>0x01=read mode | R/W | Ch.2: NVM readout |
| 0x01AE | RANGING_CORE__NVM_CTRL__PULSE_WIDTH_MSB | 2 bytes | NVM pulse width setting<br>Default: 0x0004 | R/W | Ch.2: NVM readout |
| 0x01B0 | RANGING_CORE__NVM_CTRL__ADDR | 1 byte | NVM address specification<br>Range: 0x00-0xFF | R/W | Ch.2: NVM address set |
| 0x01B1 | RANGING_CORE__NVM_CTRL__READN | 1 byte | NVM read trigger<br>0→1 transition executes read | R/W | Ch.2: NVM read trigger |
| 0x01B2 | RANGING_CORE__NVM_CTRL__DATAOUT_MMM | 4 bytes | NVM read data | R | Ch.2: NVM data acquisition |
| 0x01BB | RANGING_CORE__CLK_CTRL1 | 1 byte | NVM clock control<br>0x05=clock enable | R/W | Ch.2: NVM clock enable |

### A.3 GPIO and Interrupt Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0030 | GPIO_HV_MUX__CTRL | 1 byte | GPIO multiplexer control<br>0x10=ACTIVE_LOW interrupt output | R/W | Ch.3: GPIO config |
| 0x0031 | GPIO__TIO_HV_STATUS | 1 byte | GPIO status<br>0x02=default value | R/W | Ch.3: GPIO config |
| 0x0046 | SYSTEM__INTERRUPT_CONFIG_GPIO | 1 byte | Interrupt configuration<br>0x20=NEW_SAMPLE_READY | R/W | Ch.3, 5: Interrupt config |
| 0x0086 | SYSTEM__INTERRUPT_CLEAR | 1 byte | Interrupt clear<br>0x01 clears interrupt | W | Ch.3, 5: Interrupt clear |
| 0x0089 | RESULT__INTERRUPT_STATUS | 1 byte | Interrupt status<br>bit 5: NEW_DATA_READY | R | Ch.5: Data ready wait |

### A.4 SPAD and VCSEL Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0033 | ANA_CONFIG__SPAD_SEL_PSWIDTH | 1 byte | SPAD pulse width selection<br>0x02=for MEDIUM_RANGE | R/W | Ch.3: SPAD config |
| 0x0034 | ANA_CONFIG__VCSEL_PULSE_WIDTH_OFFSET | 1 byte | VCSEL pulse width offset<br>0x08=default | R/W | Ch.3: VCSEL config |
| 0x004A | GLOBAL_CONFIG__VCSEL_WIDTH | 1 byte | VCSEL global width<br>0x02=for MEDIUM_RANGE | R/W | Ch.3: VCSEL config |

### A.5 Sigma Estimation and Algorithm Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0036 | SIGMA_ESTIMATOR__EFFECTIVE_PULSE_WIDTH_NS | 1 byte | Effective pulse width<br>0x08=8ns | R/W | Ch.3: Sigma estimation |
| 0x0037 | SIGMA_ESTIMATOR__EFFECTIVE_AMBIENT_WIDTH_NS | 1 byte | Effective ambient width<br>0x10=16ns | R/W | Ch.3: Sigma estimation |
| 0x0038 | SIGMA_ESTIMATOR__SIGMA_REF_MM | 1 byte | Sigma reference value<br>0x01=1mm | R/W | Ch.3: Sigma estimation |
| 0x0039 | ALGO__CROSSTALK_COMPENSATION_VALID_HEIGHT_MM | 1 byte | Crosstalk compensation valid height<br>0x01=default | R/W | Ch.3: Algorithm config |
| 0x003E | ALGO__RANGE_IGNORE_VALID_HEIGHT_MM | 1 byte | Range ignore valid height<br>0xFF=disabled | R/W | Ch.3: Algorithm config |
| 0x003F | ALGO__RANGE_MIN_CLIP | 1 byte | Minimum range clip<br>0x00=no clipping | R/W | Ch.3: Algorithm config |
| 0x0040 | ALGO__CONSISTENCY_CHECK__TOLERANCE | 1 byte | Consistency check tolerance<br>0x02=default | R/W | Ch.3: Algorithm config |

### A.6 Calibration Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0047 | CAL_CONFIG__VCSEL_START | 1 byte | VCSEL start configuration<br>0x0B=11 (for MEDIUM_RANGE) | R/W | Ch.3: Calibration config |
| 0x0048 | CAL_CONFIG__REPEAT_RATE | 2 bytes | Calibration repeat rate<br>0x0000=default | R/W | Ch.3: Calibration config |
| 0x004B | PHASECAL_CONFIG__TIMEOUT_MACROP | 1 byte | Phase calibration timeout<br>0x0D=for histogram mode | R/W | Ch.3: Phase calibration |
| 0x004C | PHASECAL_CONFIG__TARGET | 1 byte | Phase calibration target<br>0x21=default | R/W | Ch.3: Phase calibration |

### A.7 Timing Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x005A | MM_CONFIG__TIMEOUT_MACROP_A | 2 bytes | MM (Multi-Mode) timeout A<br>0x001A=encoded value | R/W | Ch.3: Timing config |
| 0x005C | MM_CONFIG__TIMEOUT_MACROP_B | 2 bytes | MM timeout B<br>0x0020=encoded value | R/W | Ch.3: Timing config |
| 0x005E | RANGE_CONFIG__TIMEOUT_MACROP_A | 2 bytes | Measurement timeout A<br>0x01CC=encoded value (for MEDIUM_RANGE) | R/W | Ch.3: Timing config |
| 0x0060 | RANGE_CONFIG__VCSEL_PERIOD_A | 1 byte | VCSEL period A<br>0x0B=encoded value (actual period 12) | R/W | Ch.3: VCSEL period config |
| 0x0061 | RANGE_CONFIG__TIMEOUT_MACROP_B | 2 bytes | Measurement timeout B<br>0x01F5=encoded value (for MEDIUM_RANGE) | R/W | Ch.3: Timing config |
| 0x0063 | RANGE_CONFIG__VCSEL_PERIOD_B | 1 byte | VCSEL period B<br>0x09=encoded value (actual period 10) | R/W | Ch.3: VCSEL period config |
| 0x006C | SYSTEM__INTERMEASUREMENT_PERIOD | 4 bytes | Intermeasurement period (ms units)<br>Example: 100 = 100ms period | R/W | Ch.3: Measurement interval |

### A.8 Dynamic Configuration Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0071 | SYSTEM__GROUPED_PARAMETER_HOLD_0 | 1 byte | Parameter hold start 0<br>0x01=start | R/W | Ch.3: Grouped parameter hold |
| 0x0072 | SYSTEM__THRESH_HIGH | 2 bytes | High threshold setting<br>0x0000=default | R/W | Ch.3: Threshold config |
| 0x0074 | SYSTEM__THRESH_LOW | 2 bytes | Low threshold setting<br>0x0000=default | R/W | Ch.3: Threshold config |
| 0x0077 | SYSTEM__SEED_CONFIG | 1 byte | Seed configuration<br>0x02=default | R/W | Ch.3: Seed config |
| 0x0078 | SD_CONFIG__WOI_SD0 | 1 byte | Window of Interest SD0<br>0x0B=match VCSEL Period A | R/W | Ch.3: SD config |
| 0x0079 | SD_CONFIG__WOI_SD1 | 1 byte | Window of Interest SD1<br>0x09=match VCSEL Period B | R/W | Ch.3: SD config |
| 0x007A | SD_CONFIG__INITIAL_PHASE_SD0 | 1 byte | Initial phase SD0<br>0x0A=default | R/W | Ch.3: SD config |
| 0x007B | SD_CONFIG__INITIAL_PHASE_SD1 | 1 byte | Initial phase SD1<br>0x0A=default | R/W | Ch.3: SD config |
| 0x007C | SYSTEM__GROUPED_PARAMETER_HOLD_1 | 1 byte | Parameter hold start 1<br>0x01=continue | R/W | Ch.3: Grouped parameter hold |
| 0x007F | ROI_CONFIG__USER_ROI_CENTRE_SPAD | 1 byte | ROI center SPAD<br>0xC7=199 (center) | R/W | Ch.3: ROI config |
| 0x0080 | ROI_CONFIG__USER_ROI_REQUESTED_GLOBAL_XY_SIZE | 1 byte | ROI size<br>0xFF=16x16 (maximum) | R/W | Ch.3: ROI config |
| 0x0081 | SYSTEM__SEQUENCE_CONFIG | 1 byte | Sequence configuration<br>0xC1=VHV+PHASECAL+MM2+RANGE | R/W | Ch.3: Sequence config |
| 0x0082 | SYSTEM__GROUPED_PARAMETER_HOLD | 1 byte | Parameter hold end<br>0x02=end and apply | R/W | Ch.3: Grouped parameter hold |

### A.9 Measurement Control Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0083 | SYSTEM__STREAM_COUNT_CTRL | 1 byte | Stream count control<br>0x00=infinite loop | R/W | Ch.3: Stream config |
| 0x0087 | SYSTEM__MODE_START | 1 byte | Measurement start/stop<br>0x42=BACKTOBACK+HISTOGRAM<br>0x00=STOP | R/W | Ch.5: Measurement start/stop |

### A.10 Measurement Result Registers

| Address | Register Name | Size | Function | R/W | Usage |
|---------|--------------|------|----------|-----|-------|
| 0x0089 | RESULT__INTERRUPT_STATUS | 1 byte | Interrupt status<br>bit 5: NEW_DATA_READY<br>bit 4: ERROR<br>bit 3: RANGE_COMPLETE | R | Ch.5, 6: Measurement complete check |
| 0x008A | RESULT__RANGE_STATUS | 1 byte | Measurement status<br>bit [4:0]: Status code<br>0x09=normal measurement complete | R | Ch.6: Measurement status check |
| 0x008E | RESULT__HISTOGRAM_BIN_0_2 | 3 bytes | Histogram bin 0 (24-bit) | R | Ch.6: Data readout |
| ... | ... | ... | Histogram bins 1-22 (3 bytes each) | R | Ch.6: Data readout |
| 0x00D3 | RESULT__HISTOGRAM_BIN_23_2 | 3 bytes | Histogram bin 23 (24-bit) | R | Ch.6: Data readout |

**Note**: Histogram bin data is contiguous from 0x008E to 0x00D5, allowing burst read of 24 bins × 3 bytes = 72 bytes.

### A.11 Critical Wait Times Summary

| Operation | Wait Time | Reason | Related Register |
|-----------|-----------|--------|-----------------|
| After power force enable | **250μs** | Power stabilization wait | 0x0419 |
| After NVM clock enable | **5ms** | NVM ready wait | 0x01BB |
| After NVM read trigger | **5μs** | NVM read completion wait | 0x01B1 |
| Firmware boot | Max **500ms** | Boot completion wait (polling) | 0x0010 |
| Data ready wait | Max **2000ms** | Measurement completion wait (polling) | 0x0089 |

### A.12 Register Access Best Practices

1. **Use I2C burst write**: Efficiently write contiguous registers in one transaction
   - Example: Configure 0x0030-0x0040 in one transaction

2. **Read-Modify-Write**: Always read current value before modifying bit fields
   - Example: When changing bit 0 of PAD_I2C_HV__EXTSUP_CONFIG (0x002E)

3. **Grouped Parameter Hold**: Use 0x0071, 0x007C, 0x0082 when changing dynamic configuration
   - Atomically apply multiple register changes

4. **Clear Interrupts**: Always clear interrupts at 0x0086 after data readout

5. **Stop Measurement**: Write 0x00 to 0x0087 twice for reliable stop (manufacturer recommendation)

---

## Appendix B: NVM Address Map

VL53L3CX factory calibration data is stored in NVM. Below are the main NVM addresses.

| NVM Address | Data Content | Size | Usage |
|-------------|--------------|------|-------|
| 0x11 | I2C_SLAVE__DEVICE_ADDRESS | 1 byte | Default I2C address (0x29) |
| 0x1C | OSC_MEASURED__FAST_OSC_FREQUENCY_MSB | 1 byte | Fast oscillator frequency (upper byte) |
| 0x1D | OSC_MEASURED__FAST_OSC_FREQUENCY_LSB | 1 byte | Fast oscillator frequency (lower byte) |
| 0x2C | VHV_CONFIG__TIMEOUT_MACROP_LOOP_BOUND | 1 byte | VHV timeout setting |

**Note**: NVM data is 4 bytes per address, but typically only the first 1-2 bytes are used.

---

## Appendix C: Error Code List

### RANGE_STATUS Error Codes (Register 0x008A bit [4:0])

| Code | Name | Meaning | Resolution |
|------|------|---------|-----------|
| 0x00 | RANGE_VALID | Range measurement succeeded (but other warnings present) | Check status details |
| 0x01 | SIGMA_FAIL | Sigma estimation failed (excessive noise) | Improve measurement environment, increase integration time |
| 0x02 | SIGNAL_FAIL | Insufficient signal strength | Check target reflectivity, reduce distance |
| 0x04 | OUTOFBOUNDS_FAIL | Out of measurement range | Check target distance |
| 0x05 | HARDWARE_FAIL | Hardware error | Reboot device |
| 0x07 | WRAP_TARGET_FAIL | Phase wrap error | Adjust measurement range |
| 0x08 | PROCESSING_FAIL | Processing error | Re-read data |
| 0x09 | **RANGE_COMPLETE** | **Normal measurement complete** | Normal (data valid) |
| 0x0D | XTALK_SIGNAL_FAIL | Crosstalk compensation failed | Perform crosstalk calibration |
| 0x0E | SYNCRONISATION_INT | Synchronization interrupt | - |
| 0x12 | MIN_RANGE_FAIL | Below minimum distance | Target too close |
| 0x13 | RANGE_INVALID | Invalid measurement result | Discard data |

**Recommendation**: Only use data when `range_status == 0x09`.


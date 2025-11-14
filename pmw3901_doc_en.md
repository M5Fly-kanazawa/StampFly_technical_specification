# PMW3901MB Optical Motion Sensor Implementation Guide

## 1.0 Introduction: PMW3901MB Sensor Overview

### 1.1 Purpose of This Document

This guide is a practical technical reference for efficiently and reliably integrating PixArt's high-performance optical motion sensor "PMW3901MB" into your system. The PMW3901MB plays an essential role in drones and navigation systems, particularly for self-localization in environments where GPS is unavailable. This guide extracts initialization sequences, motion data readout protocols, and advanced frame capture procedures directly needed for implementation from information scattered throughout the official datasheet, providing procedure-based explanations to solve challenges engineers face.

### 1.2 Core Functions and Specification Evaluation of PMW3901MB Sensor

The PMW3901MB is the latest optical navigation chip based on PixArt's far field optics technology. This technology enables aerial navigation without requiring lens focusing. The sensor continuously acquires image elements (pixels) and mathematically analyzes their changes to calculate the direction and magnitude of movement. This core function is realized by the built-in Pixel Array and Evaluation System (PEAS) and Digital Signal Processing System (DSPS).

The sensor's key features and specifications provide the following strategic advantages for applications in mobile platforms such as drones:

- **Wide operating range (80 mm to infinity)**: Enables stable motion detection even on drones and mobile robots where distance from the ground varies.
- **Simple interface (4-wire SPI @ 2 MHz)**: Standard support on many microcontrollers (MCU) makes system integration easy.
- **Flexible supply voltage (VDD: 1.8–2.1V, VDDIO: 1.8–3.6V)**: Compatible with low-voltage systems and flexibly adapts to various power designs.
- **High-precision motion data (16-bit)**: Acquires movement in X-Y directions with high resolution, contributing to precise position control.
- **Low power consumption (< 9mA in run mode)**: A critical element for achieving long-term operation in battery-powered applications.

To maximize these functions, accurate communication with the sensor is essential. The next chapter provides detailed explanations of the SPI interface specifications and configuration that form this foundation.

---

## 2.0 SPI Interface Specifications and Configuration

### 2.1 SPI Communication Basics

All control command transmission and data reception with the PMW3901MB occur via 4-wire SPI (Serial Peripheral Interface). Accurately understanding and correctly implementing this communication protocol is the first step toward maximizing sensor performance. In particular, chip select (NCS) operation is extremely important for defining transaction boundaries and ensuring communication reliability.

The following table shows the roles of the main pins used for SPI communication.

| Pin Name | Type | Description |
|----------|------|-------------|
| SCLK | Input | Serial clock signal supplied from the master (MCU). |
| MOSI | Input | Data input line from master to slave (sensor) (Master Out / Slave In). |
| MISO | Output | Data output line from slave (sensor) to master (Master In / Slave Out). High impedance (Hi-Z) state when NCS is High. |
| NCS | Input | Chip select signal (active low). Lowering this pin enables SPI communication with the sensor. |

#### Notes on Chip Select (NCS) Operation

NCS not only signals the start of communication but also guarantees transaction integrity. If NCS goes High during communication, the transaction is immediately interrupted and the serial port is reset. Even if a communication error occurs, the port can be reset and returned to a stable state by setting NCS High once and then back to Low.

### 2.2 Write and Read Operation Sequence Analysis

#### Write Operation

The operation to write configuration values to sensor registers consists of a 2-byte (16-bit) transaction.

1. The MCU sets NCS to Low to begin communication.
2. Sends the address byte. The most significant bit (MSB) of this byte is set to 1 to indicate a write operation. The remaining 7 bits specify the target register address.
3. Sends the data byte. These 8 bits are the value to be written to the specified register.
4. After data transfer completes, the MCU returns NCS to High to end the transaction.

*[Diagram: Write operation sequence (see datasheet Figure 9)]*

#### Read Operation

The operation to read values from sensor registers also consists of a 2-byte transaction, but the data direction differs.

1. The MCU sets NCS to Low to begin communication.
2. Sends the address byte. The MSB of this byte is set to 0 to indicate a read operation. The remaining 7 bits specify the register address to read.
3. After sending the address, the sensor returns the data byte through the MISO line during the next 8 clock cycles. The MCU receives this data.
4. After receiving data, the MCU returns NCS to High to end the transaction.

*[Diagram: Read operation sequence (see datasheet Figure 11)]*

#### Timing Specifications Between Commands

To maintain stable communication, the datasheet specifies minimum wait times between commands. These delays are essential for the sensor's internal processor to complete operations such as writing and data preparation and become ready to accept the next command.

- **t_SWW (45 µs)**: Minimum time from write to next write
- **t_SWR (45 µs)**: Minimum time from write to next read
- **t_SRR/t_SRW (20 µs)**: Minimum time from read to next command (read/write)
- **t_SRAD (35 µs)**: Minimum wait time from sending address in read command until data is ready

If these timing specifications are not observed, commands may not execute correctly, so inserting sufficient delays during firmware design is recommended.

After understanding these basic SPI operations, the next chapter explains the specific initialization procedures for properly starting up the sensor.

---

## 3.0 Sensor Initialization Procedures

### 3.1 Power-On Sequence and Reset

For the sensor to generate accurate motion data, executing a strict initialization sequence after power-on is essential. By performing this procedure correctly, the sensor's internal registers are set to appropriate states and a foundation for stable operation is established.

The recommended power-on sequence is as follows:

1. **Power Supply**: Supply power to VDDIO (I/O voltage), then supply power to VDD (core voltage) with a delay not exceeding 100ms. Wait for both power supplies to stabilize.
2. **Wait**: After power stabilization, wait at least 40ms.
3. **SPI Port Reset**: Drive the NCS pin High once, then Low to reset the SPI port.
4. **Power-Up Reset**: Write 0x5A to the Power_Up_Reset register (address 0x3A). This resets the entire chip.
5. **Wait**: After writing the reset command, wait at least 1ms.
6. **Initial Readout**: Read registers 0x02 through 0x06 (motion-related registers) once, regardless of motion pin state.

#### Supplement: Hardware Reset

By using the NRESET pin (active low), a full chip reset similar to software reset (writing to the Power_Up_Reset register) can also be executed. Reset occurs by holding the NRESET pin Low for 100ns or more. This pin cannot be left floating (unconnected).

### 3.2 Performance Optimization Register Configuration

After completing the power-on sequence, specific register values must be written to a series of registers to optimize sensor performance. These settings are PixArt's proprietary adjustment items, and detailed functions of each register are not disclosed in the datasheet, but they are essential procedures for obtaining optimal performance.

The following shows the register sequence to write. As it includes conditional branches, refer to comments for accurate implementation.

```c
// Address, Value
{0x7F, 0x00},
{0x55, 0x01},
{0x50, 0x07},
{0x7F, 0x0E},

// --- Conditional write sequence start ---
// Note: If 0x08 cannot be read from 0x47, write 0x10 to 0x43 and read 0x47 again.
// Try this up to 3 times. If unsuccessful, re-power the sensor and start over.
{0x43, 0x10}, // Write as needed
// Verify Read(0x47) becomes 0x08

// Read Bit7 of 0x67 and determine value to write to 0x48 based on it
// value = Read(0x67)
// if (value & 0x80) { Write(0x48, 0x04) } else { Write(0x48, 0x02) }
// --- Conditional write sequence end ---

{0x7F, 0x00},
{0x51, 0x7B},
{0x50, 0x00},
{0x55, 0x00},
{0x7F, 0x0E},

// --- Conditional configuration block ---
// value = Read(0x73)
// if (value != 0x00) {
//   // If Read(0x73) is not 0x00, execute the following calculation and writes
//   C1 = Read(0x70);
//   if (C1 <= 0x1C) { C1 += 0x0E; } else { C1 += 0x0B; }
//   if (C1 > 0x3F) { C1 = 0x3F; }
//   C2 = Read(0x71);
//   C2 = (C2 * 45) / 100;
//
//   {0x7F, 0x00},
//   {0x61, 0xAD},
//   {0x51, 0x70},
//   {0x7F, 0x0E},
//   {0x70, C1},
//   {0x71, C2},
// }
// --- Conditional configuration block end ---

{0x7F, 0x00},
{0x61, 0xAD},
{0x7F, 0x03},
{0x40, 0x00},
{0x7F, 0x05},
{0x41, 0xB3},
{0x43, 0xF1},
{0x45, 0x14},
{0x5B, 0x32},
{0x5F, 0x34},
{0x7B, 0x08},
{0x7F, 0x06},
{0x44, 0x1B},
{0x40, 0xBF},
{0x4E, 0x3F},
{0x7F, 0x08},
{0x65, 0x20},
{0x6A, 0x18},
{0x7F, 0x09},
{0x4F, 0xAF},
{0x5F, 0x40},
{0x48, 0x80},
{0x49, 0x80},
{0x57, 0x77},
{0x60, 0x78},
{0x61, 0x78},
{0x62, 0x08},

// --- Insert 10ms delay ---
delay(10);

{0x32, 0x44},
{0x7F, 0x07},
{0x40, 0x41},
{0x63, 0x50},
{0x7F, 0x0A},
{0x45, 0x60},
{0x7F, 0x00},
{0x4D, 0x11},
{0x55, 0x80},
{0x74, 0x1F},
{0x75, 0x1F},
{0x4A, 0x78},
{0x4B, 0x78},
{0x44, 0x08},
{0x45, 0x50},
{0x64, 0xFF},
{0x65, 0x1F},
{0x7F, 0x14},
{0x65, 0x67},
{0x66, 0x08},
{0x63, 0x70},
{0x7F, 0x15},
{0x48, 0x48},
{0x7F, 0x07},
{0x41, 0x0D},
{0x43, 0x14},
{0x4B, 0x0E},
{0x45, 0x0F},
{0x44, 0x42},
{0x4C, 0x80},
{0x7F, 0x10},
{0x5B, 0x02},
{0x7F, 0x07},
{0x70, 0x00},

// 10ms delay not needed (Delay in datasheet Table 10 is only once)
// Wait 15ms after the write to 0x70, then read register 0x15.
// The read back value should be 0xBF. This is part of observation check.
// The sequence continues here.
{0x40, 0x40},
{0x7F, 0x06},
{0x62, 0xF0},
{0x63, 0x00},
{0x7F, 0x0D},
{0x48, 0xC0},
{0x6F, 0xD5},
{0x7F, 0x00},
{0x5B, 0xA0},
{0x4E, 0xA8},
{0x5A, 0x50},
{0x40, 0x80}
```

Once this initialization and optimization is complete, the sensor is ready to generate motion data. The next section explains procedures for actually reading out this data (X-Y displacement).

---

## 4.0 Motion Data (Displacement) Readout Procedures

### 4.1 Standard Motion Data Readout

The sensor's primary purpose is to detect displacement on the X-Y plane. After initialization completes, tracking device position and movement by periodically reading out this motion data. To ensure data reliability, performing quality checks along with readout procedures is important.

The standard motion data readout procedure is as follows:

1. **Check for Motion Occurrence**: Read the Motion register (address 0x02). This read operation freezes (fixes) the displacement data register values (Delta_X_L/H, Delta_Y_L/H).
2. **Determine Data Presence**: Verify whether bit 7 (MOT7) of the Motion register is 1. If 1, it means new motion has been detected since the last readout.
3. **Acquire Displacement Data**: If MOT7 is 1, read the following registers in order:
   - Delta_X_L (address 0x03)
   - Delta_X_H (address 0x04)
   - Delta_Y_L (address 0x05)
   - Delta_Y_H (address 0x06)
4. Combine _L (low byte) and _H (high byte) to synthesize X-direction and Y-direction displacement data as 16-bit signed integers (two's complement representation).
5. **Data Quality Verification**: To evaluate acquired data reliability, read the SQUAL register (address 0x07) and Shutter_Upper register (address 0x0C) values. According to the datasheet, if the SQUAL value is less than 0x19 AND the Shutter_Upper value is 0x1F, that data may have low reliability and discarding it is recommended. This condition suggests poor visibility such as featureless surfaces or insufficient lighting where the sensor cannot reliably track movement.

By repeating this procedure, continuous motion data can be acquired.

### 4.2 High-Speed Data Readout: Utilizing Burst Mode

For faster and more efficient bulk acquisition of motion-related data, burst mode is effective. Using burst mode enables continuous readout of up to 12 bytes of data with a single address specification, reducing SPI transaction overhead.

The burst mode usage procedure is as follows:

1. Set NCS Low to begin communication.
2. Send the Motion_Burst register address (0x16).
3. Wait for t_SRAD (35µs) time.
4. Continuously drive SCLK and read up to 12 bytes of data from MISO.
5. After completing readout, return NCS to High to end burst mode.

The breakdown of 12 bytes of data read in burst mode is as follows:

- BYTE[00]: Motion
- BYTE[01]: Observation
- BYTE[02]: Delta_X_L
- BYTE[03]: Delta_X_H
- BYTE[04]: Delta_Y_L
- BYTE[05]: Delta_Y_H
- BYTE[06]: SQUAL
- BYTE[07]: RawData_Sum
- BYTE[08]: Maximum_RawData
- BYTE[09]: Minimum_RawData
- BYTE[10]: Shutter_Upper
- BYTE[11]: Shutter_Lower

This mode is particularly useful for comprehensively monitoring sensor status including not just displacement but also surface quality (SQUAL) and shutter values.

Motion data acquisition is the sensor's basic function, but for debugging or specialized image analysis, the raw image data that the sensor captures may be needed. The next chapter explains frame capture procedures for this purpose.

---

## 5.0 Frame Capture Procedures

### 5.1 Transition to Frame Capture Mode and Data Acquisition

Frame capture is a special function for directly downloading raw image data of 35x35 pixels (total 1225 pixels) captured by the sensor's imaging element. This function is mainly used for debugging, surface texture analysis, or advanced image processing algorithm development.

**Important**: During frame capture mode, normal navigation (motion detection) functionality is disabled. Also, strict procedures are required for mode transitions and data acquisition.

The frame capture execution process is divided into three major steps.

#### 1. Transition to Frame Capture Mode

Execute the following register write sequence in order to set the sensor to frame capture mode.

#### 2. Data Acquisition Preparation and Execution

- **a. Initialization**: Write 0x00 to register 0x70, then write 0xFF to the RawData_Grab register (address 0x58) to begin the data acquisition process.
- **b. Readiness Polling**: Repeatedly read the RawData_Grab_Status register (address 0x59) and wait until both bit 6 (data starts from position 0,0) and bit 7 (data acquisition enabled) are 1.
- **c. Data Readout and Reconstruction**: Repeatedly read the RawData_Grab register (0x58). Each pixel's data (8 bits) is returned split across two reads.
  - **First read**: RDG[7:6] becomes 01, and RDG[5:0] contains the upper 6 bits of pixel data.
  - **Second read**: RDG[7:6] becomes 10, and RDG[3:2] contains the lower 2 bits of pixel data.

  Combine these to restore 8-bit data for one pixel. Repeat this process for 1225 pixels.

#### 3. Exit from Frame Capture Mode

Execute the following register write sequence to prepare to return the sensor to normal mode.

**Caution**: After executing this sequence, manual reset is required to restore navigation functionality. Write 0x5A to the Power_Up_Reset register (0x3A) or toggle the NRESET pin.

Following this advanced function, we finally explain power-down procedures, which are particularly important for battery-powered devices - sensor power-saving management.

---

## 6.0 Power-Down (Shutdown) Procedures

### 6.1 Transition to and Return from Shutdown Mode

Minimizing system power consumption is extremely important, especially in battery-powered applications. The PMW3901MB features a shutdown mode that reduces current consumption to approximately 12µA when inactive. Properly utilizing this mode can significantly extend device operating time.

#### Transition to Shutdown Mode

The procedure to transition the sensor to shutdown mode is very simple:

- Write 0xB6 to the Shutdown register (address 0x3B).

When this command is executed, the sensor enters a low-power state. During shutdown, do not access the SPI port (except for return commands).

#### Return from Shutdown Mode

The procedure to return from shutdown mode to normal run mode is almost the same as the power-on sequence. This is because shutdown resets internal state, requiring re-initialization.

1. **SPI Port Reset**: Drive the NCS pin High once, then Low.
2. **Power-Up Reset**: Write 0x5A to the Power_Up_Reset register (address 0x3A).
3. **Wait and Initial Readout**: After waiting 1ms or more, read registers 0x02 through 0x06 once.
4. **Re-initialization**: Re-execute the series of register settings explained in "3.2 Performance Optimization Register Configuration".

#### Notes on Pin States During Shutdown

In system design during shutdown mode, pin states must be considered.

| Pin Name | State During Shutdown | Notes |
|----------|----------------------|-------|
| NRESET | High | Must be held at High level. |
| NCS | High | If sharing SPI bus with other devices, must be held High. |
| MISO | Hi-Z | External pull-up or pull-down recommended to avoid undefined state. |
| SCLK | Ignore (if NCS=High) | SCLK signal is ignored if NCS is High. |
| MOSI | Ignore (if NCS=High) | MOSI signal is ignored if NCS is High. |
| MOTION | Output High | Motion interrupt pin outputs at High level. |

---

## Summary

This guide has explained all basic procedures necessary for implementing the PMW3901MB optical motion sensor. By understanding proper initialization, accurate SPI communication, and data readout methods appropriate to the purpose, this sensor can be effectively integrated into drones and navigation systems.

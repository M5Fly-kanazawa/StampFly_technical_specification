# BMI270 IMU SPI Connection Guide: Startup, Initialization, and Data Reading with Microcontrollers

## 1.0 Introduction: BMI270 Overview and Guide Purpose

### 1.1 Introduction to BMI270's Key Features

The BMI270 from Bosch Sensortec is not just a collection of accelerometers and gyroscopes. It is a high-performance 6-axis Inertial Measurement Unit (IMU) equipped with intelligent on-chip features. It can achieve high-precision motion detection while reducing the load on the host microcontroller, making it invaluable especially in resource-constrained embedded systems such as wearable devices and battery-powered IoT equipment. This section explains the main technical features that the BMI270 offers.

- **6-Axis Sensor**: Integrates a 3-axis accelerometer and 3-axis gyroscope in one compact package. This enables simultaneous and synchronized measurement of linear motion (acceleration) and rotational motion (angular velocity) of the device. This data forms the foundation for orientation estimation in all motion-based applications.
- **Low Power Consumption**: The standard current consumption when operating both the accelerometer and gyroscope in normal mode is only 685 µA. This extremely low power consumption provides a significant advantage in wearable and hearable devices where battery life directly affects product competitiveness.
- **On-Chip FIFO**: Built-in 2 KB large-capacity FIFO (First-In, First-Out) buffer. This eliminates the need for the microcontroller to continuously poll sensor data in real-time. By accumulating a certain amount of data in the FIFO before reading it in bulk, the microcontroller can maximize sleep time and dramatically reduce overall system power consumption. It also prevents data loss during high-speed data sampling.
- **Intelligent Features**: The BMI270 does not merely measure data but also processes it internally with numerous intelligent functions. Examples include wrist-worn step counter, various motion detection (Significant Motion/Any Motion), and activity recognition (still/walking/running). By utilizing these features, complex algorithm processing traditionally handled by the host processor can be offloaded to the sensor side, simplifying system design.

### 1.2 Target Audience and Goals of This Guide

This guide targets all embedded system developers aiming to connect the BMI270 to a microcontroller via **SPI (Serial Peripheral Interface)** and perform basic sensor data reading.

The goal of this guide is to master the complete flow from power-on to stable data acquisition, with specific procedures and important notes. Ultimately, we aim to enable readers to confidently integrate the BMI270 into their own projects.

The next section explains the details of the SPI protocol, which forms the foundation for all communication with the BMI270. Accurately understanding this protocol is the first step toward achieving stable sensor operation.

---

## 2.0 SPI Communication Protocol Basics

### 2.1 Introduction: Establishing MCU-BMI270 Communication

All operations from configuration changes to data reading on the BMI270 are performed via a digital interface. SPI in particular enables high-speed communication and is supported by many microcontrollers. This section explains the protocol for establishing SPI communication with the BMI270, physical connections, and specific operation sequences. Accurately understanding and implementing these rules is essential for reliable data acquisition.

The BMI270 starts in I²C mode by default after power-on, but can be easily switched to SPI mode.

1. **CSB Pin Operation After Power-On**: The most common and reliable method. The BMI270 interface defaults to I²C mode immediately after power-on. Executing an initial dummy read provides the necessary rising edge (Low→High) on the CSB pin and is the standard technique for reliably switching the interface to SPI mode. If this operation (e.g., reading CHIP_ID register 0x00) is omitted, the sensor will not respond to SPI commands.
2. **Permanent SPI Mode Configuration**: By setting the spi_en bit of the NV_CONF register (address 0x70) to 1 and writing to non-volatile memory (NVM), the sensor can be set to start permanently in SPI mode without requiring CSB pin manipulation at each power-on.

### 2.2 SPI Modes and Physical Connection

The main specifications for connecting the BMI270 to a microcontroller via SPI are as follows:

- **Supported Modes**: Multiple SPI modes (combinations of clock polarity CPOL and clock phase CPHA) exist, but the BMI270 supports the following two modes:
  - Mode '00' (CPOL=0, CPHA=0)
  - Mode '11' (CPOL=1, CPHA=1)

  The microcontroller's SPI settings must be configured to match one of these.

- **4-Wire and 3-Wire**:
  - **4-Wire (Default)**: Uses four signal lines: CSB (chip select), SCx (clock), SDx (input from master), and SDO (output from sensor).
  - **3-Wire**: Can be switched to 3-wire mode by setting the spi3 bit of the IF_CONF register (address 0x6B) to 1. In this mode, the data line is consolidated to the SDx pin and used for bidirectional communication.

| Interface | BMI270 Pin | Description |
|-----------|------------|-------------|
| **4-Wire SPI** | CSB | Chip Select |
| | SCx | Serial Clock |
| | SDx | Serial Data Input (SDI) |
| | SDO | Serial Data Output (SDO) |
| **3-Wire SPI** | CSB | Chip Select |
| | SCx | Serial Clock |
| | SDx | Serial Data I/O (SDA) |

### 2.3 Read Operation

The SPI communication sequence for reading one byte of data from a register consists of the following steps. Pay particular attention to the existence of a dummy byte as specified by the BMI270.

1. The master (microcontroller) sets CSB to Low to begin communication.
2. The master sends an 8-bit command from the SDx (SDI) pin. This command consists of a read bit (R/W=1) and a 7-bit register address.
3. The master continues sending clock signals and receives the first dummy byte from the SDO pin. This dummy byte is invalid data and must be discarded.
4. The master sends 8 more clock cycles and receives the target register data (2nd byte) from the SDO pin.
5. The master returns CSB to High to end communication.

**Important**: The rule **"BMI270's SPI read always generates one dummy byte"** is one of the most critical points in driver implementation. If this is not considered, data will be shifted by one byte and correct values cannot be read. This is a common failure point that is often overlooked during implementation.

### 2.4 Write Operation

The SPI communication sequence for writing one byte of data to a register is simpler than the read operation.

1. The master (microcontroller) sets CSB to Low to begin communication.
2. The master sends an 8-bit command from the SDx (SDI) pin. This command consists of a write bit (R/W=0) and a 7-bit register address.
3. The master continues by sending the 8-bit data to be written from the SDx (SDI) pin.
4. The master returns CSB to High to end communication.

### 2.5 Important Communication Timing Requirements

To achieve reliable communication, the following timing requirements must be observed:

- **Maximum Clock Frequency**: When I/O supply voltage (VDDIO) is 1.62V or higher, the maximum SPI clock frequency is **10 MHz**.
- **Idle Time After Write**: The wait time (idle time) required between a register write operation and the start of the next access varies significantly depending on the sensor's operating mode. Observing this delay is very important. Ignoring this wait time may prevent writes from being properly reflected.
  - **Normal mode**: At least **2 µs** (t_IDLE_wr_act) wait time is required.
  - **Suspend/Low-power mode**: At least **450 µs** (t_IDLE_wacc_sum) wait time is required. This time is very long compared to normal mode, so special care is needed when configuring registers in low-power mode.

Accurately implementing these protocols and timing conventions is an absolute prerequisite for successfully executing the sensor initialization sequence in the next step.

---

## 3.0 Startup Sequence and Required Initialization

### 3.1 Introduction: Key to Making the Sensor Operational

The BMI270 does not function as a sensor simply by applying power. A strict initialization sequence is required to boot the processor governing the sensor's internal intelligent functions and load Bosch Sensortec's proprietary configuration file. Accurately executing this sequence is key to maximizing the BMI270's performance. If even one step is incorrect, the sensor will not operate properly and may cause project delays, so careful implementation is required.

### 3.2 Detailed Initialization Procedure

The specific initialization procedure that the microcontroller should execute is as follows. Follow the register names, addresses, write values, and wait times accurately.

1. **Wait After Power-On**: After the power supplies (VDD and VDDIO) stabilize, wait at least **450 µs** until registers become accessible. This wait is very important. If neglected, commands will be sent before the sensor is internally ready, which is a common cause of initialization failure.

2. **Establish SPI Interface**: As explained in Section 2.1, execute a dummy register read (e.g., CHIP_ID register 0x00) to reliably set the interface to SPI mode.

3. **Disable Advanced Power Save**: Write 0x00 to the PWR_CONF register (address 0x7C) to disable Advanced Power Save mode (APS). This setting is essential because the sensor's internal processor must be active during initialization.
   ```c
   write_reg(0x7C, 0x00);
   ```

4. **Wait**: After writing to the register, wait at least **450 µs** for the internal state to stabilize. This wait time is also critical and cannot be omitted.

5. **Prepare for Config File Load**: Write 0x00 to the INIT_CTRL register (address 0x59) to notify the sensor that preparation for configuration file upload is complete.
   ```c
   write_reg(0x59, 0x00);
   ```

6. **Upload Config File**:
   - Prepare the official configuration file (bmi270_config_file, approximately 8KB) provided by Bosch Sensortec. This file can be obtained from the official GitHub repository.
     - Reference: https://github.com/BoschSensortec/BMI270-Sensor-API/blob/master/bmi270.c
   - Send this configuration data in bulk to the INIT_DATA register (address 0x5E) using SPI's burst write function.
   - **Note (for memory-constrained microcontrollers)**: If the host microcontroller's RAM buffer is smaller than 8KB, the configuration file may not be sendable at once. In that case, it is possible to write the data in smaller chunks. Between writing each chunk, the start address for the next data to be written must be written to the INIT_ADDR_0 (0x5B) and INIT_ADDR_1 (0x5C) registers. Specifically, increment the address by (bytes written in chunk ÷ 2).

7. **Complete Config File Load**: After upload is complete, write 0x01 to the INIT_CTRL register (address 0x59) to inform the sensor that the initialization process is complete.
   ```c
   write_reg(0x59, 0x01);
   ```

### 3.3 Verifying Initialization Completion

Confirming that the initialization sequence completed correctly is very important for ensuring subsequent operation reliability. Follow these steps for verification:

- After writing 0x01 to the INIT_CTRL register, periodically read (poll) the INTERNAL_STATUS register (address 0x21).
- Wait for the message field consisting of the lower 4 bits of this register to become 0b0001 (0x01). This value means "ASIC initialized".
- This confirmation process may take up to **20 ms**.
- If a value other than 0x01 is returned (e.g., 0x02 "Initialization error"), it indicates a problem with the initialization sequence. In that case, recheck for errors in wait times or write procedures.

When this initialization process succeeds, the BMI270 transitions to "Configuration Mode". Only in this state is the sensor ready to proceed to the next steps such as sensor enabling and data rate configuration.

---

## 4.0 Reading Sensor Data

### 4.1 Introduction: Preparing for Data Acquisition After Initialization

The strict initialization sequence is complete, and the BMI270 has entered "Configuration Mode" ready to accept settings. However, at this stage the sensor is not yet generating data. As the next step, to acquire the desired data, the accelerometer and gyroscope must be individually enabled and their respective operating modes and output data rates (ODR) configured. This configuration is an important process that determines the trade-off between the data update frequency required by the application and system power consumption.

### 4.2 Sensor Enabling and Basic Configuration

The procedure for enabling the accelerometer and gyroscope and performing basic configuration is as follows:

#### Sensor Enabling

Use the PWR_CTRL register (address 0x7D) to individually control the power of each sensor.

- To enable the accelerometer, set the acc_en bit (bit 2) to 1.
- To enable the gyroscope, set the gyr_en bit (bit 1) to 1.
- To enable the temperature sensor, set the temp_en bit (bit 3) to 1.

#### Operating Mode and Data Rate Configuration

Sensor performance (filter characteristics) and output data rate (ODR) are configured in dedicated configuration registers.

- **Accelerometer**: Use the ACC_CONF register (address 0x40).
  - Set the ODR with the acc_odr field (lower 4 bits) (e.g., 0x08 for 100Hz).
  - Set the filter performance with the acc_filter_perf bit (bit 7) (1=Performance, 0=Power optimized).

- **Gyroscope**: Use the GYR_CONF register (address 0x42).
  - Set the ODR with the gyr_odr field (lower 4 bits) (e.g., 0x09 for 200Hz).
  - Set the filter performance with the gyr_filter_perf bit (bit 7).

#### Configuration Example

To operate the accelerometer at 100Hz and gyroscope at 200Hz in "Normal Mode", write the following values:

```c
// 1. Enable accelerometer, gyro, and temperature sensors
write_reg(0x7D, 0x0E);  // PWR_CTRL: 0b00001110

// 2. Configure accelerometer (100Hz, Performance mode)
write_reg(0x40, 0xA8);  // ACC_CONF

// 3. Configure gyroscope (200Hz, Performance mode)
write_reg(0x42, 0xA9);  // GYR_CONF
```

### 4.3 Direct Register Reading

Once sensors are enabled and configuration is complete, sensor values can be read directly from data registers.

#### Data Register Addresses

Data for each axis is stored in 16 bits (2 bytes).

- **Accelerometer Data (X, Y, Z)**: DATA_8 (0x0C) to DATA_13 (0x11)
- **Gyroscope Data (X, Y, Z)**: DATA_14 (0x12) to DATA_19 (0x17)

#### Data Format

Data is in 16-bit two's complement representation. After reading from two 8-bit registers (LSB: low byte, MSB: high byte), combine them into a 16-bit signed integer for use.

#### Importance of Shadowing Procedure

To ensure data integrity, the BMI270 has an important mechanism called "shadowing". This prevents data corruption where upper and lower bytes contain data from different sample times because values are updated while multi-byte data is being read.

**This rule is very simple: Always read each axis's data starting from the LSB (low byte) register.** When the LSB register is read, the corresponding MSB (high byte) register value is internally locked (shadowed) at that moment. This ensures that when the MSB register is read next, it contains a value from the same sample time as the LSB.

Direct register reading is the simplest and most intuitive data acquisition method, but when reading data at high frequency, microcontroller processing load increases. For more efficient data collection, using the FIFO explained in the next chapter is recommended.

---

## 5.0 High-Speed Data Reading Using FIFO

### 5.1 Introduction: FIFO Value and Usage Scenarios

The FIFO (First-In, First-Out) buffer is one of the BMI270's most powerful features. Using it enables dramatically reduced microcontroller load while achieving high-speed, lossless data sampling. The FIFO is on-chip memory that temporarily stores data acquired by the sensor. Instead of constantly monitoring the sensor, the microcontroller can wait for data to accumulate in the FIFO and read it in bulk when needed.

This approach provides tremendous value especially in the following scenarios:

- **High ODR (Output Data Rate) Data Collection**: When performing high-speed sampling at hundreds of Hz or more, polling tends to lose data, but using FIFO reliably captures data.
- **Low Power Applications**: The microcontroller can sleep most of the time and wake up briefly only when triggered by FIFO interrupts (watermark, etc.). This can dramatically reduce average system power consumption.

### 5.2 FIFO Enabling and Configuration

To use FIFO, you must configure which sensor data to store in FIFO and how to behave when FIFO becomes full.

#### Enabling Sensor Data in FIFO

Use the FIFO_CONFIG_1 register (address 0x49) to select data sources to store in FIFO.

- To store accelerometer data: Set the fifo_acc_en bit (bit 6) to 1.
- To store gyroscope data: Set the fifo_gyr_en bit (bit 7) to 1.

#### FIFO Mode Configuration

**Header Mode / Headerless Mode**: Toggle with the fifo_header_en bit (bit 4) of the FIFO_CONFIG_1 register. When header mode is enabled (1), a header containing information such as which sensor data is included is added to the beginning of each data frame. This enables flexible data analysis even when multiple sensor data are sampled at different rates.

**Behavior When FIFO Full**: Select the behavior when FIFO becomes full with the fifo_stop_on_full bit (bit 0) of the FIFO_CONFIG_0 register (address 0x48).

- `0` (fifo_stop_on_full = 0) corresponds to streaming mode. The oldest data is overwritten and the latest data is always retained.
- `1` (fifo_stop_on_full = 1) corresponds to FIFO mode. When FIFO becomes full, new data writing stops and data is discarded.

### 5.3 FIFO Data Reading Procedure

The procedure for the microcontroller to read data accumulated in FIFO is as follows:

1. **Check FIFO Data Amount**: First, read the two registers FIFO_LENGTH_0 (address 0x24) and FIFO_LENGTH_1 (address 0x25) to confirm the total number of bytes currently accumulated in FIFO.

2. **Execute Burst Read**: Execute SPI burst read from the FIFO_DATA register (address 0x26) for the number of bytes confirmed in step 1.

3. **Parse Data**: Parse the read byte sequence according to the configured frame structure. In header mode (fifo_header_en=1), the byte stream consists of "frames", with a 1-byte header added before each sensor data payload. When accelerometer and gyroscope ODRs differ, parsing this header to identify the frame contents (which sensor data is included) is essential for accurate data extraction. Extract accelerometer and gyroscope data from each frame and convert to 16-bit sensor values.

### 5.4 Advanced Usage: FIFO for Low Power Mode

FIFO's true value is demonstrated in low power applications. By using the fifo_self_wakeup function, system power consumption can be minimized. The behavior of this function varies significantly based on the fifo_self_wakeup bit (bit 1) setting in the PWR_CONF register (address 0x7C), so accurate understanding is important.

- **PWR_CONF.fifo_self_wakeup = 0b0**: With this setting, even when FIFO interrupt (watermark or full) occurs, the sensor maintains low power mode (with Advanced Power Save enabled). To read FIFO data, the host microcontroller must first explicitly disable Advanced Power Save mode (PWR_CONF.adv_power_save = 0b0), wait 450 µs, then access FIFO.

- **PWR_CONF.fifo_self_wakeup = 0b1 (Maximum Power Efficiency)**: This is the setting for true low-power FIFO operation. When FIFO interrupt occurs, the sensor temporarily becomes ready to accept FIFO reads from the host microcontroller. This allows the microcontroller to wake from sleep and retrieve FIFO data in bulk with a single burst read without disabling Advanced Power Save. After data acquisition, the microcontroller can immediately return to sleep, dramatically reducing average system power consumption. Understanding this behavioral difference is key to successful low-power design.

---

## Summary

By mastering the procedures explained in this guide—establishing reliable SPI communication, executing the strict initialization sequence, and selecting the appropriate data reading method (direct read or FIFO) according to purpose—you will be able to maximize the advanced performance of the BMI270.

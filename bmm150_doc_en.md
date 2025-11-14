# BMM150 Magnetic Sensor Implementation Guide for Microcontrollers

## Introduction

This guide provides technical documentation on the implementation procedures for integrating the Bosch Sensortec BMM150, a high-performance 3-axis magnetic sensor, with microcontrollers. The BMM150 enables a wide range of applications including electronic compasses, navigation systems, and augmented reality (AR). This guide offers step-by-step explanations from sensor initialization to data acquisition, as well as operational methods for efficiently utilizing system resources, with the goal of resolving technical challenges that developers may encounter.

The main topics covered in this guide include:

- **Understanding Basic Specifications**: Sensor performance, communication interfaces, and coordinate system definitions
- **Initialization Sequence**: Proper startup procedures and mode transitions after power-on
- **Data Reading**: Measurement data structure and reading methods that ensure consistency
- **Efficient Data Acquisition Techniques**: Utilizing interrupts and on-demand measurements as alternatives to polling

Let's begin by examining the essential basic functions and specifications of the BMM150 before starting the implementation in the next section.

---

## 1. Understanding BMM150 Basic Functions and Specifications

To effectively use the sensor, understanding its basic capabilities and specifications is the first critical step. This section provides prerequisite knowledge for later programming, including the sensor's performance limits, communication methods with microcontrollers, and the coordinate system that defines how measurement data corresponds to physical orientation. By properly grasping these fundamentals, you can prevent rework during implementation and maximize the sensor's performance.

### Key Features of BMM150

The main features of the BMM150 as described in the datasheet are as follows:

- **3-Axis Magnetic Sensor**: Measures magnetic fields along three mutually orthogonal axes (X, Y, Z).
- **Ultra-Compact Package**: Adopts a Wafer Level Chip Scale Package (WLCSP) of only 1.56 x 1.56 mm², offering high placement flexibility on PCBs.
- **Digital Interfaces**: Supports both SPI (3-wire/4-wire) and I²C communication protocols.
- **Low Voltage Operation**: Operates across a wide voltage range (VDD: 1.62V to 3.6V, VDDIO: 1.2V to 3.6V), suitable for low-power systems.
- **Magnetic Field Detection Range**: Can detect a wide range of magnetic fields: ±1300µT for x and y axes, ±2500µT for z axis.
- **Built-in Interrupt Controller**: Incorporates interrupt functionality to notify the microcontroller of events such as Data Ready and threshold detection.

### Digital Interface Selection

The BMM150's digital interface is determined by the state of the PS (Protocol Select) pin. You must set the pin state correctly according to your connection with the microcontroller.

- **I²C Mode**: Set the PS pin to "1" (VDDIO level).
- **SPI Mode**: Set the PS pin to "0" (GND level).

### Sensor Coordinate System

To correctly interpret data obtained from the sensor, it is extremely important to understand how the sensor's physical orientation corresponds to the X, Y, and Z axes. The diagram below shows the direction of each axis relative to the sensor package. For example, the Z-axis extends in the positive direction from the top surface of the sensor mounted on the PCB, while the X and Y axes are defined along the sides of the package.

> **Explanation based on Figure 21: Orientation of sensing axes**
> (Note: The above diagram is a schematic representation of Figure 21 from the datasheet)

When the X-axis points to magnetic north, the X-axis output shows a positive value. This coordinate system definition forms the basis for calculations when implementing electronic compasses and similar applications.

Now that the sensor's basic characteristics are clear, let's explain the specific initialization sequence for starting up the sensor with a microcontroller.

---

## 2. Initialization Sequence: From Suspend to Active Mode

Correctly managing the state transitions (power modes) after powering on the BMM150 is key to ensuring stable sensor operation. Immediately after power-on, the BMM150 starts in "Suspend Mode" with all functions stopped. This section details the specific procedures for transitioning from Suspend Mode through "Sleep Mode," where register configuration becomes possible, and finally to "Active Mode," where magnetic field measurements are performed.

BMM150 initialization follows these steps:

### 1. Power-On and Suspend Mode

When power is supplied to VDD and VDDIO, a Power-On Reset (POR) is executed, and the device automatically starts in Suspend Mode. In this mode, power consumption is minimized, but most functions are stopped.

- **Important Note**: In Suspend Mode, only the Power Control Register (address `0x4B`) is accessible. Attempting to read other registers, such as the Chip ID register (`0x40`), in this state will not return the correct value and will instead read `0x00`.

### 2. Transition to Sleep Mode

To access and configure the sensor's internal registers, you must first transition to Sleep Mode.

- **Operation**: Write `1` to the "Power Control bit" at bit 0 of the Power Control Register (`0x4B`).
- **Result**: This operation transitions the BMM150 to Sleep Mode, enabling read/write access to all registers including the Chip ID.

### 3. Chip ID Verification (Communication Check)

After transitioning to Sleep Mode, verify that communication with the sensor has been properly established. This is commonly referred to as a "Health Check" process.

- **Operation**: Read the Chip ID register (`0x40`).
- **Verification**: If the read value is `0x32`, it indicates that the sensor has started normally and communication with the microcontroller is functioning correctly.

### 4. Transition to Active Mode

Once communication verification is complete, transition to Active Mode to actually start magnetic field measurements.

- **Operation**: Write `00b` to "Opmode <1:0>", consisting of bits 2 and 1 of the Operation Mode Control Register (`0x4C`).
- **Result**: This transitions the sensor to "Normal Mode" and begins periodic measurements at the configured data rate.

### Utilizing Recommended Presets

While the BMM150 allows fine-tuning of measurement repetition counts and other parameters, finding optimal settings for your application requirements can be complex. Therefore, the datasheet provides four recommended presets corresponding to common use cases. These presets allow you to easily balance power consumption and measurement accuracy by appropriately setting the values of registers `0x51` (REPXY) and `0x52` (REPZ). The table below shows the performance and power consumption trade-offs for each preset, helping with design decisions.

#### Table 3: Recommended Presets

| Preset Name | Rep. X/Y (nXY) | Rep. Z (nZ) | RMS Noise x/y/z (µT) | Average Current Consumption (mA @ ODR) |
|---|---|---|---|---|
| Low power | 3 | 3 | 1.0/1.0/1.4 | 0.17 @ 10Hz |
| Regular | 9 | 15 | 0.6/0.6/0.6 | 0.5 @ 10Hz |
| Enhanced regular | 15 | 27 | 0.5/0.5/0.5 | 0.8 @ 10Hz |
| High accuracy | 47 | 83 | 0.3/0.3/0.3 | 4.9 @ 20Hz |

The sensor initialization and measurement preparation are now complete. In the next section, we'll examine in detail how to read and interpret the measured magnetic data with the microcontroller.

---

## 3. Reading and Interpreting Magnetic Data

When the sensor starts measurements in Active Mode, measurement results are stored in data registers. The process by which the microcontroller accurately acquires this data and converts it to meaningful physical quantities is a critical step that affects application reliability. This section explains the structure of data registers, safe reading methods to maintain data consistency, and an overview of converting raw data to actual magnetic field data (µT).

### 3.1. Checking Data Ready Status

To know when new measurement data is available, check the Data Ready status. The "Data Ready Status" flag at bit 0 of the RHALL register (`0x48`) serves this role.

- **Polling**: The microcontroller periodically reads this bit and waits for it to become `1`. When it becomes `1`, it means new data is ready.
- **Interrupt**: As a more efficient method, you can also use the DRDY interrupt described later.

This flag is automatically cleared to `0` immediately after any data register read operation completes and communication ends (STOP condition issued for I²C, or CSB transitions to High for SPI).

### 3.2. Data Register Structure

Magnetic data for the X, Y, and Z axes, along with RHALL (Hall element resistance value) data used for temperature compensation, are stored in registers `0x42` through `0x49`. Each data type has different bit widths and formats and must be correctly combined and interpreted.

| Data | Register (LSB, MSB) | Bit Width | Data Format |
|---|---|---|---|
| X-axis | 0x42 (LSB[4:0]), 0x43 (MSB[12:5]) | 13-bit | 2's complement |
| Y-axis | 0x44 (LSB[4:0]), 0x45 (MSB[12:5]) | 13-bit | 2's complement |
| Z-axis | 0x46 (LSB[6:0]), 0x47 (MSB[14:7]) | 15-bit | 2's complement |
| RHALL | 0x48 (LSB[5:0]), 0x49 (MSB[13:6]) | 14-bit | Unsigned |

### 3.3. Safe Data Reading (Burst Read)

If values are updated inside the sensor during data reading, a problem called "axis mix-up" can occur, where old data from the X-axis gets mixed with new data from the Y-axis. To prevent this, the BMM150 incorporates a shadow register function.

To ensure data consistency, the datasheet strongly recommends reading registers `0x42` through `0x49` together in a single communication sequence called **"Burst Read"**. During a burst read, data register updates are blocked, and new data is temporarily stored in shadow registers. When communication completes, the shadow register contents are reflected in the data registers. This ensures you always obtain a consistent dataset with synchronized 3-axis + RHALL data.

### 3.4. Interpreting Raw Data

Values read from registers are **"raw data"** that includes sensor-specific variations and temperature characteristics. These cannot be used directly as physical magnetic field units of µT (microtesla).

> **Figure 3: Calculation flow of magnetic field data**
> (Note: The above diagram is a schematic representation of Figure 3 from the datasheet)

To obtain final magnetic field data, temperature compensation calculations using the RHALL register value are necessary. This complex compensation calculation is implemented in the official API/driver provided by Bosch Sensortec. By using this API, developers can easily obtain accurate magnetic field data in µT units from raw data without implementing the compensation algorithm themselves.

Now that we've mastered basic data reading methods, more advanced applications require more efficient data acquisition methods than polling to conserve CPU resources. The next section introduces techniques for this purpose.

---

## 4. Efficient Data Acquisition Techniques

In many embedded systems, conserving CPU resources and ensuring real-time performance are extremely important. While simple polling is easy to implement, it is inefficient because the CPU must constantly monitor the sensor's state. This section introduces more advanced and efficient data acquisition methods as alternatives to polling: event-driven approaches using interrupts, and methods to actively trigger measurements at specific timings.

### 4.1. Event-Driven Data Acquisition Using DRDY Interrupts

The DRDY (Data Ready) interrupt is a function that sends a notification to the microcontroller when the sensor completes measurement and storage of new data.

- **Advantages**: Using this function eliminates the need for the microcontroller to constantly monitor (poll) the Data Ready flag. The CPU can enter sleep mode or execute other tasks until data is ready, improving overall system processing efficiency and significantly reducing power consumption.
- **Enabling Procedure**: To use the DRDY interrupt, set the "Data Ready Pin En" at bit 7 of register `0x4E` to `1`. This enables the DRDY pin.

> **Sequence based on Figure 5: Data acquisition and DRDY operation**

1. The sensor starts measurement.
2. When measurement and writing to data registers complete, the DRDY pin is asserted (High active by default).
3. The microcontroller detects the interrupt and begins data read processing.
4. When data reading completes, the DRDY pin is automatically cleared (deasserted) and enters a state waiting for the next measurement completion.

### 4.2. Synchronous Data Acquisition Using Forced Mode

Forced Mode is an on-demand operation mode that executes a single measurement upon command from the host (microcontroller), rather than periodic automatic measurements like Normal Mode.

- **Advantages**: This mode is very effective when you need to strictly synchronize measurement timing with other sensors like accelerometers, or when you want to acquire data at arbitrary intervals other than those configurable in Normal Mode.
- **Usage Procedure**:
  1. Write `01b` to "Opmode <1:0>" in the Operation Mode Control Register (`0x4C`) to trigger Forced Mode.
  2. The sensor executes a single measurement sequence (X, Y, Z, RHALL).
  3. When measurement data is written to data registers, the sensor automatically returns to Sleep Mode (Opmode returns to `11b`). The microcontroller triggers Forced Mode again when the next measurement is needed.

The maximum data rate achievable in Forced Mode depends on the measurement repetition counts (nXY, nZ) and can be approximated by the following formula. This allows you to estimate how quickly you can measure with your configured accuracy (repetition count).

```
f_max,ODR ≈ 1 / (145µs * nXY + 500µs * nZ + 980µs)
```

Note that nXY and nZ are not the raw values from registers `0x51` and `0x52`, but the actual measurement repetition counts calculated from them.

By selectively using these two techniques (DRDY interrupt and Forced Mode) according to application requirements, you can build a flexible and efficient data acquisition architecture.

---

## 5. Summary

This guide has explained the important steps for using the BMM150 magnetic sensor in conjunction with a microcontroller. The key points for implementation are summarized below:

- **Initialization**: Accurate power mode transition from Suspend → Sleep → Active after power-on is essential for stable sensor operation. Note especially that register configuration is not possible without transitioning to Sleep Mode.
- **Data Reading**: To prevent "axis mix-up" where X, Y, and Z axis data become out of sync, it is strongly recommended to always read data registers (`0x42`~`0x49`) in a batch using burst read.
- **Efficient Acquisition**: To use CPU resources efficiently, consider introducing an event-driven architecture utilizing DRDY interrupts instead of polling. Additionally, Forced Mode is an effective option when synchronization with other sensors or arbitrary data rates are required.

By properly implementing these procedures and techniques, you can maximize the potential of the BMM150 and build highly accurate and reliable applications.

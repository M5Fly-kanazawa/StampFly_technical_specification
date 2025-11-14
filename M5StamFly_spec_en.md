# M5Stamp Fly Technical Specification

## 1.0 Product Overview

This section provides an overview of the M5Stamp Fly quadcopter kit, its key features, and expected application areas. The M5Stamp Fly is an open-source programmable quadcopter kit that adopts the M5StampS3 as its main controller. Its design aims to serve as a strategic platform for education, research, and diverse drone development projects.

The core concept of the M5Stamp Fly is to achieve precise autonomous flight through advanced sensor fusion. The aircraft integrates a 6-axis gyroscope (BMI270) and a 3-axis magnetometer (BMM150) for attitude and heading detection, a barometric pressure sensor (BMP280) and two distance sensors (VL53L3) for accurate altitude maintenance and obstacle avoidance (Note: connected to a single I2C bus). Additionally, an optical flow sensor (PMW3901MB-TXQT) captures ground displacement, enabling stable hovering and position holding even in GPS-denied environments. These sensor systems work in coordination to enable stable flight control in complex environments.

The main features of the M5Stamp Fly are as follows:

- **M5StampS3 as Main Controller**: Equipped with a high-performance Xtensa LX7 processor and Wi-Fi connectivity for advanced computational processing and wireless communication.
- **BMP280 Barometric Pressure Detection**: Precisely measures atmospheric pressure changes, forming the foundation for stable altitude hold functionality.
- **VL53L3 Distance Sensor**: Directly measures physical distance to the ground, enabling accurate altitude maintenance at low altitudes and obstacle avoidance.
- **6-axis Attitude Sensor**: Detects aircraft tilt and angular velocity in real-time, serving as the basis for attitude control.
- **3-axis Magnetometer for Heading Detection**: Determines the aircraft's absolute orientation using Earth's magnetic field, supporting accurate navigation.
- **Optical Flow Sensor for Displacement Detection**: Detects horizontal movement with high precision, contributing to position hold in indoor environments where GPS is unavailable.
- **Buzzer**: Notifies users of flight status and warnings through sound, enabling interactive operation.
- **300mAh High-Voltage Power Battery**: Ensures sufficient flight time despite its compact size.
- **Current/Voltage Detection Function**: Monitors battery status in real-time, supporting safe operation and flight time management.
- **Grove Interface Expandability**: Allows easy addition of external sensors and modules, expanding project possibilities.

With its high functionality and expandability, this product is expected to be utilized in a wide range of fields including:

- Education
- Research
- Drone Development
- DIY Projects

The following chapters will analyze in detail the specific hardware components that realize these diverse functions.

## 2.0 Main Component Analysis

This section provides detailed explanations of the roles and performance of the main electronic components that constitute the M5Stamp Fly. Stable autonomous flight of a quadcopter is only achieved through the close coordination of these control and sensor systems. Understanding how each component collects and processes information and contributes to the overall aircraft behavior is essential for maximizing the product's capabilities.

### Main Controller (M5StampS3)

The M5StampS3, based on the ESP32-S3, serves as the central brain of this aircraft. Equipped with a high-performance Xtensa LX7 processor and 8MB of flash memory, it executes complex flight control algorithms and real-time sensor data processing with ease. The built-in Wi-Fi functionality enables remote control, telemetry data transmission, and wireless firmware updates. It also supports OTG/CDC functions, facilitating debugging and communication via USB.

### 6-axis Attitude Sensor (BMI270)

The BMI270 is an Inertial Measurement Unit (IMU) that integrates a 3-axis accelerometer and a 3-axis gyroscope. It detects the aircraft's tilt (roll, pitch) and rotation (yaw) in real-time with high precision. Data obtained from this sensor serves as the primary input to the PID control loop for stabilizing the aircraft and forms the foundation of attitude maintenance in all flight operations.

### 3-axis Magnetometer (BMM150)

The BMM150 detects Earth's magnetic field and plays the role of determining the aircraft's absolute orientation. By providing this global reference frame, it enables intuitive control in "headless mode" and more accurate autonomous navigation. However, caution is needed regarding the surrounding magnetic environment, as the sensor readings may be affected when used near products containing magnets or ferromagnetic materials.

### Barometric Pressure Sensor (BMP280)

The BMP280 precisely measures atmospheric pressure. It calculates the aircraft's relative altitude using the principle that air pressure decreases with increasing altitude, and is essential for realizing the "altitude hold function" that maintains stable hovering at a specified altitude. By capturing minute environmental pressure changes, it enables smooth altitude control.

### Distance Sensor (VL53L3)

The VL53L3 is a laser ranging sensor using Time-of-Flight (ToF) technology. It directly and accurately measures the physical distance to objects such as the ground. Particularly during low-altitude flight or landing sequences, it provides more accurate altitude information than the barometric pressure sensor, improving aircraft stability. It can also be applied to obstacle avoidance functions.

### Optical Flow Sensor (PMW3901MB-TXQT)

The PMW3901MB-TXQT is a type of "visual odometry" sensor that detects the aircraft's horizontal movement (displacement) by tracking ground patterns. In environments such as indoors where GPS signals are unavailable, it detects aircraft drift due to disturbances and autonomously corrects position. This realizes a high-precision "position hold" function.

### Current/Voltage Sensor (INA3221AIRGVR)

The INA3221AIRGVR is an important component that monitors battery voltage and current consumption in real-time. Information from this sensor enables accurate battery remaining capacity assessment to predict available flight time, and detection of abnormalities such as overcurrent or voltage drops, supporting safe aircraft operation.

The following section will review the specific numerical specifications of these components in a list format to further explore their hardware performance.

## 3.0 Detailed Technical Specifications

This section presents a comprehensive list of detailed technical specifications that define the performance and functionality of the M5Stamp Fly. Understanding these precise values is essential for maximizing hardware capabilities and verifying compatibility with external components.

| Specification Item | Parameter |
|-------------------|-----------|
| M5StampS3 | ESP32-S3@Xtensa LX7, 8MB Flash, WIFI, OTG/CDC function |
| Motor | 716-17600kv |
| Distance Sensor | VL53L3C 0x29(7-bits) @up to 3 meters |
| Optical Flow Sensor | PMW3901MB-TXQT |
| Barometric Pressure Sensor | BMP280(0x76)@300-1100hPa |
| 3-axis Magnetometer | BMM150(0x10) |
| 6-axis Attitude Sensor | BMI270 |
| Grove | I2C+UART |
| Battery | 300mAh High-voltage Lithium Battery |
| Battery Output Voltage | 4.35V |
| Flight Time | Approx. 4 minutes |
| Charging Time (Input:5V@1A) | 55 minutes |
| Current/Voltage Detection | INA3221AIRGVR(0x40) |
| Buzzer | Passive Onboard Buzzer@5020 |
| Operating Temperature | 0-40°C |
| Product Dimensions | 81.5 x 81.5 x 31mm |
| Package Dimensions | 162 x 99 x 36mm |
| Product Weight | 36.8g |
| Package Weight | 70.7g |

The next chapter will examine the pin assignments of each interface in detail to understand how these hardware components are physically connected.

## 4.0 Interface and Pin Mapping

This section provides detailed explanations of each interface and pin mapping between the M5Stamp Fly and the M5StampS3 controller. Accurately understanding the role and connection destination of each pin is extremely important for adding external sensors or developing custom firmware.

### 4.1 I2C Interface

I2C is a serial bus that communicates with multiple devices using two signal lines (SDA, SCL). In the M5Stamp Fly, the current/voltage sensor, magnetometer, barometric pressure sensor, and distance sensor are connected to this shared bus, realizing efficient data communication.

| StampFly (Stamp-S3) | Connected Devices |
|---------------------|-------------------|
| G3 (SDA) | INA3221, BMM150, BMP280, VL53L3 |
| G4 (SCL) | INA3221, BMM150, BMP280, VL53L3 |

### 4.2 SPI Interface

SPI is a serial communication protocol that enables faster data transfer than I2C. Each device is controlled by an individual chip select (CS) pin. In the M5Stamp Fly, the 6-axis attitude sensor and optical flow sensor, which require high-speed data readout, are connected to the SPI bus.

| StampFly (Stamp-S3) | BMI270 | PMW3901MB-TXQT |
|---------------------|--------|----------------|
| G14 | MOSI | MOSI |
| G44 | SCK | SCK |
| G43 | MISO | MISO |
| G46 | CS | - |
| G12 | - | CS2 |

### 4.3 Grove Interface

To ensure expandability, two Grove interface systems are provided. This allows easy connection of various commercially available third-party sensors and modules without soldering, enabling simple expansion of project functionality.

- **Grove (RED - I2C)**:
  - G13: SDA
  - G15: SCL
- **Grove (BLACK - UART)**:
  - G1: GROVE I (RX)
  - G2: GROVE O (TX)

### 4.4 VL53L3CX ToF Sensor Control Pins

The two VL53L3CX distance sensors (forward and bottom) share the same I2C address, so the XSHUT pins must be used to individually initialize and change their addresses. Each sensor also has an interrupt pin (GPIO1/INT) that can be used for measurement completion notification.

- **Forward ToF Sensor XSHUT**: G9
- **Forward ToF Sensor INT**: G8
- **Bottom ToF Sensor XSHUT**: G7
- **Bottom ToF Sensor INT**: G6

### 4.5 Other

A buzzer and full-color RGB LED are mounted for user feedback and status display. The control pins for these are as follows:

- **Buzzer**: G40
- **WS2812 RGB LED**: G39

The next chapter will explain in detail the power system that supports the operation of these hardware components.

## 5.0 Power Supply and Battery

This section details the power system and battery management, which are extremely important for ensuring the performance and safety of the M5Stamp Fly. Stable drone flight depends on reliable power supply, and proper understanding and handling of the battery are essential.

The M5Stamp Fly is equipped with a 300mAh high-voltage lithium battery. This battery has an output voltage of 4.35V, higher than standard lithium polymer batteries, and achieves high energy density while maintaining a compact size, enabling approximately 4 minutes of flight time.

To ensure safe operation and maximize battery life, please observe the following precautions:

- **Charging Precautions**:
  - Charge the included battery by connecting it to the charging slot of the dedicated remote controller Atom JoyStick, and connecting a data cable to the Atom JoyStick.
- **Battery Maintenance**:
  - During flight, ensure that the voltage per battery cell does not fall below 3V. Over-discharge causes serious damage to the battery.
  - Do not store a fully charged battery for more than 3 days. For long-term storage, it is recommended to maintain the voltage between 3.8V and 3.9V.

The INA3221AIRGVR sensor mounted on the aircraft monitors battery current and voltage in real-time. This functionality enables the flight control system to accurately assess battery remaining capacity and issue low voltage warnings. It also contributes to improving aircraft safety by analyzing power consumption patterns to detect abnormal power consumption.

The next chapter will explain software-related information for controlling the hardware.

## 6.0 Software and Development Resources

This section introduces the software, development tools, and technical documentation provided to maximize the potential of the M5Stamp Fly. Effective utilization of these resources is key to successful custom application development and research projects.

The main available resources are summarized below.

### Firmware

- **Arduino**: StampFly Firmware - Firmware source code for customization and development using the Arduino IDE.
- **Easyloader**: StampFly Firmware Easyloader - A tool that allows easy installation of factory default firmware by simply connecting to a PC.

### Schematics

- **StampS3_Fly_Hat Schematics PDF** - Schematics for the expansion board where sensors and motor drivers are implemented. Essential for custom hardware debugging.
- **Stamp_Fly Schematics PDF** - Integrated schematics for the entire aircraft including M5StampS3 and expansion board.
- **PMW3901MB Schematics PDF** - Schematics for the optical flow sensor module.

### Main Component Datasheets

- VL53L3(ToF)
- PMW3901MB-TXQT(Optical Flow Sensor Chip)
- BMP280(Pressure Sensor Chip)
- BMM150(3D Magnetometer Compass)
- BMI270(6-axis Inertial Measurement Unit (IMU) Sensor)

### 3D Models

- **Battery Protection Component STL File** - 3D printer data for battery protection components.

---

We hope this technical specification document serves as a solid starting point for your educational, research, and creative development projects using the M5Stamp Fly.

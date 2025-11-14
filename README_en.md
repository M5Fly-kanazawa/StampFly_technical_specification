# StampFly Technical Specification

English | [日本語](README.md)

This repository provides technical documentation to support the development of the M5Stamp Fly (StampFly).

## Overview

The M5Stamp Fly is an open-source programmable quadcopter kit that adopts the M5StampS3 as its main controller. It achieves precise autonomous flight through advanced sensor fusion, making it an ideal platform for education, research, and drone development.

## Documentation

### M5Stamp Fly Technical Specification

Comprehensive technical documentation detailing the overall specifications, major components, interfaces, pin assignments, and more of the M5Stamp Fly.

- [日本語版](M5StamFly_spec_ja.md)
- [English](M5StamFly_spec_en.md)

**Contents:**
- Product overview and key features
- Major component analysis (M5StampS3, BMI270, BMM150, BMP280, VL53L3, PMW3901MB, etc.)
- Detailed technical specifications (power, motors, sensor performance, etc.)
- Interface and pin mapping (I2C, SPI, Grove)
- Power supply and battery management
- Software and development resources

---

### BMM150 Magnetic Sensor Implementation Guide

Technical documentation explaining the implementation procedures for integrating the Bosch Sensortec BMM150 3-axis magnetic sensor with microcontrollers.

- [日本語版](bmm150_doc_ja.md)
- [English](bmm150_doc_en.md)

**Contents:**
- BMM150 basic functions and specifications
- Initialization sequence (startup procedures and mode transitions after power-on)
- Data reading methods (measurement data structure and consistency guarantee)
- Efficient data acquisition techniques (interrupts, on-demand measurement)
- Coordinate system definition and magnetic data handling

---

### PMW3901MB Optical Motion Sensor Implementation Guide

Technical documentation explaining the implementation procedures for integrating the PixArt Imaging PMW3901MB-TXQT optical flow sensor with microcontrollers.

- [日本語版](pmw3901_doc_ja.md)
- [English](pmw3901_doc_en.md)

**Contents:**
- PMW3901MB sensor overview and functionality
- SPI interface establishment (4-wire SPI communication protocol)
- Initialization and configuration sequence (from power-on to register settings)
- Motion data readout (standard mode and burst mode)
- Data filtering and best practices

---

## Key Sensor List

| Sensor | Model | Function | Communication |
|--------|-------|----------|---------------|
| Main Controller | M5StampS3 (ESP32-S3) | Flight control & wireless communication | - |
| 6-axis IMU | BMI270 | Accelerometer & Gyroscope | SPI |
| 3-axis Magnetometer | BMM150 | Geomagnetic detection & heading sensor | I2C |
| Barometric Sensor | BMP280 | Altitude measurement | I2C |
| Distance Sensor | VL53L3 | ToF distance measurement | I2C |
| Optical Flow | PMW3901MB-TXQT | Horizontal displacement detection | SPI |
| Current/Voltage Sensor | INA3221AIRGVR | Battery monitoring | I2C |

## Development Resources

- **Firmware**: Arduino IDE compatible source code and Easyloader
- **Schematics**: Various circuit diagrams (PDF)
- **Datasheets**: Detailed specifications of major components
- **3D Models**: STL files for battery protection components, etc.

For details, refer to the "6.0 Software and Development Resources" section of the [M5Stamp Fly Technical Specification](M5StamFly_spec_en.md).

---

## License

This documentation is provided to support the development and community growth of the M5Stamp Fly.

## Contributions

Improvements to technical specifications and additional information are welcome.

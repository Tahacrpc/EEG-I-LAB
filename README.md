# EEG-I-LAB

## 8-Channel EEG Acquisition System Based on ADS1299

EEG-I-LAB is a hardware and software development project focused on building a functional, eight-channel electroencephalography (EEG) acquisition system.

The project uses the open-source [HackEEG Shield](https://github.com/starcat-io/hackeeg-shield) as its initial hardware reference. The primary objective is to develop and validate a reliable EEG acquisition platform that can support future research, signal processing, and brain-computer interface (BCI) applications.

The initial development phase focuses on reproducing and validating the reference hardware before introducing additional functionality or design modifications.

## 1. System Overview

The system is based on the Texas Instruments ADS1299, an eight-channel, 24-bit analog front-end integrated circuit designed for biopotential measurements.

The initial acquisition architecture consists of:

**Electrode Interface → Analog Input Circuitry → ADS1299 → SPI → Arduino Due → USB → Computer**

The ADS1299 handles analog signal acquisition and analog-to-digital conversion. The microcontroller manages device configuration, data acquisition, and communication with the host computer.

The host computer is responsible for recording, visualizing, and processing the acquired signals.

### Hardware Specifications

| Parameter | Specification |
|---|---|
| Analog Front End | Texas Instruments ADS1299 |
| Number of Channels | 8 |
| ADC Resolution | 24 bits |
| Sampling Rate | Up to 16 kSPS |
| Programmable Gain | 1, 2, 4, 6, 8, 12, 24 |
| Reference Microcontroller | Arduino Due |
| ADC Communication | SPI |
| Host Communication | USB |
| Digital Supply | 3.3 V |
| Analog Supply | ±2.5 V (default configuration) |
| PCB | 4 layers |
| PCB Design Software | Altium Designer |

These specifications describe the reference architecture. The actual performance of the manufactured prototype has not yet been measured.

## 2. Hardware Design

The initial hardware design is based on the original HackEEG Shield.

The main hardware sections include:

- **Analog Front End:** ADS1299 for simultaneous, eight-channel signal acquisition.
- **Input Circuitry:** Electrode connections and analog input filtering.
- **Power Management:** Analog and digital power regulation.
- **Reference and Bias Circuitry:** Reference voltage and bias signal management.
- **Digital Interface:** SPI communication, logic-level conversion, and control signals.
- **Configuration:** Hardware jumpers and identification EEPROM.

The reference design uses a four-layer PCB with dedicated ground and digital power layers.

The original EAGLE design has been imported into Altium Designer to establish an editable hardware development environment.

The initial implementation will preserve the reference circuit architecture and PCB layout as closely as possible.

## 3. Current Development Status

**Project phase: Hardware design verification and manufacturing preparation.**

### Completed

- Reference hardware and system architecture selected.
- Original HackEEG hardware resources obtained.
- EAGLE schematic imported into Altium Designer.
- Original PCB layout imported into Altium Designer.
- Main analog, digital, and power sections visually inspected.
- Initial electrical rules check performed.
- Preliminary schematic-to-PCB consistency analysis initiated.

### In Progress

- Verification of electrical connectivity.
- Investigation of schematic-to-PCB inconsistencies.
- Component and footprint verification.
- Review of the existing PCB layout.
- Preparation for Bill of Materials (BOM) extraction.

### Pending

- Final schematic and PCB verification.
- Complete BOM and component availability assessment.
- Manufacturing file verification.
- Component procurement.
- PCB manufacturing and assembly.
- Firmware preparation and integration.
- Hardware bring-up.
- Functional acquisition and signal quality testing.

### Current Verification Results

The imported Altium project currently reports:

| Check | Result |
|---|---|
| ERC Errors | 0 |
| ERC Warnings | 83 |
| Information Messages | 1 |
| Schematic-to-PCB Differences | 260 |

The remaining differences are under investigation. They may include connectivity, component identity, and CAD import-related inconsistencies.

No destructive Engineering Change Order (ECO) modifications have been applied.

The absence of ERC errors does not establish that the design is electrically correct or ready for manufacturing.

**The physical prototype has not yet been manufactured or functionally tested.**

## 4. Software and Data Acquisition

The initial software implementation will use the existing HackEEG Arduino driver and Python client.

The planned software functionality includes:

1. ADS1299 initialization and register configuration.
2. SPI communication and acquisition control.
3. Continuous, eight-channel data acquisition.
4. Data transfer to the host computer.
5. Recording and visualization of acquired signals.
6. Frequency-domain analysis and basic digital signal processing.

Initial testing will use the ADS1299 internal test-signal generator, with 500 SPS as the first target sampling rate.

The system will subsequently be evaluated using appropriate laboratory signals before progressing toward more advanced EEG experiments.

The initial firmware and acquisition workflow have not yet been validated on the project's physical hardware.

## 5. Development Roadmap

### Phase 1: Hardware Verification

Verify the schematic, PCB, component assignments, electrical connectivity, and power architecture.

**Target:** Establish a verified hardware design suitable for manufacturing preparation.

### Phase 2: PCB Manufacturing and Assembly

Finalize the BOM, procure the required components, prepare manufacturing files, and assemble the prototype.

**Target:** Produce a complete physical prototype.

### Phase 3: Hardware Bring-Up

Perform visual inspection, unpowered electrical checks, power-rail verification, and ADS1299 communication tests.

**Target:** Establish reliable operation of the acquisition hardware.

### Phase 4: Data Acquisition

Integrate the firmware and host software. Validate the internal test signal, acquisition channels, and data transfer.

**Target:** Demonstrate stable, eight-channel data acquisition.

### Phase 5: Signal Processing

Implement data recording, visualization, filtering, and frequency-domain analysis.

**Target:** Establish a usable platform for EEG signal processing and experimental BCI applications.

## 6. Potential Future Development

Following the successful validation of the reference hardware, several extensions may be considered.

**ESP32-S3 Integration**

Replace the Arduino Due with an ESP32-S3 to support a more compact system and additional communication capabilities.

**Wireless Communication**

Investigate Bluetooth LE or Wi-Fi for wireless data transmission while evaluating throughput, data integrity, and interference.

**Wearable EEG Hardware**

Explore a smaller PCB, battery-powered operation, and an electrode interface suitable for a wearable system.

**BCI Applications**

Investigate blink detection, EEG frequency-band analysis, and experimental attention or relaxation indicators.

These are possible future development directions rather than features of the current prototype.

## 7. Electrical Safety

The original HackEEG Shield does not include patient isolation circuitry.

Human-connected measurements must not be performed until the complete system has undergone an appropriate electrical safety assessment and the necessary protective measures have been implemented and verified.

Initial validation will be limited to electrical checks, internal ADC test signals, and suitable laboratory signal sources.

The system is being developed for research and educational purposes. No medical-device certification or clinical suitability is claimed.

## 8. Reference Resources

### Hardware

[HackEEG Shield](https://github.com/starcat-io/hackeeg-shield)

### Firmware

[HackEEG Arduino Driver](https://github.com/starcat-io/hackeeg-driver-arduino)

### Software

[HackEEG Python Client](https://github.com/starcat-io/hackeeg-client-python)

### Technical Documentation

[Texas Instruments ADS1299](https://www.ti.com/product/ADS1299)

[HackEEG Hardware Configuration](https://github.com/starcat-io/hackeeg-shield/blob/master/docs/configuration.md)

[HackEEG Typical Configuration](https://github.com/starcat-io/hackeeg-shield/blob/master/docs/typical-configuration.md)

The original HackEEG hardware is released under the CERN Open Hardware Licence v1.2. Applicable licensing and attribution requirements must be preserved when distributing derived hardware.

---

**Last Updated:** October 3, 2026  
**Current Milestone:** Hardware design verification and manufacturing preparation.

# InkTime - Smartwatch Project Technical Documentation

## 1. Overview
InkTime is an ultra-low-power open-hardware smartwatch designed around the nRF52840 SoC and a 1.54" E-paper display. The project focuses on extreme battery longevity, utilizing a high-efficiency dual-stage power supply and hardware-level power gating to achieve a standby target of 30 days or more.

---

## 2. System Architecture

### 2.1 Block Diagram
The system utilizes a dual-stage power topology to maximize battery discharge efficiency.

    [ USB Type-C ]     [ LiPo Battery (250mAh) ]
           | 5V VBUS          |
           v                  | BAT
    +---------------+         v
    |    BQ25180    |<--+ [MAX17048]
    |  (Charger IC) |   | (Fuel Gauge)
    +---------------+   |     |
           | VSYS       +-----+ I2C
           v                  v
    +---------------+      +-------------+
    |    RT6160     |      |             |
    | (Buck-Boost)  |      |  nRF52840   |
    +---------------+      |     MCU     |
           | 3.3V (VDD_3V3)|  (Control   |
           | System Rail   |    Hub)     |
           |               |             |
    +------+---------------+------+------+
    |      |               |      |
    |      v               |      |
    |  +-------------+     |      | SPI
    |  | PFET Switch |<--GPIO-----+      v
    |  +------+------+     |  +-------------+
    |      | VEPD          |  | 1.54" E-paper|
    |      v               |  |   Display   |
    |  +-------------+     |  +-------------+
    |  | [Display]   |     |
    |  +-------------+     | I2C Bus
    |                      |
    +---I2C==> [ BMA423 Accel ]
    |                      |
    +---I2C==> [ DRV2605L Driver ] --> [ Vibration Motor ]
    |
    |          [ PCB Antenna ] <--> [ BLE RF ]
    |
    +---GPIO-> [ User Buttons ]

---

## 3. Bill Of Materials (BOM)

| Designator | Component | Package | Manufacturer | Description |
| :--- | :--- | :--- | :--- | :--- |
| U1 | nRF52840 | aQFN-73 | Nordic Semi | SoC Bluetooth 5.0 |
| IC1 | BQ25180Y | DSBGA-8 | Texas Inst. | Li-Ion Charger w/ Power Path |
| IC9 | RT6160AWSC | WLCSP-15 | Richtek | 3.3V Buck-Boost Regulator |
| IC3 | BMA423 | LGA-12 | Bosch | Triaxial Accelerometer |
| U3 | MAX17048 | TDFN-8 | Maxim/ADI | 1-Cell Fuel Gauge |
| ANT1 | 2450AT18B100E| 1.4mm | Johanson | 2.45GHz Chip Antenna |
| J1 | 503480-2400 | FPC-24 | Molex | 0.5mm Pitch FPC Connector |
| J4 | KH-TYPE-C-16P| SMT | Kinghelm | USB Type-C 16-pin |
| Q3 | SI1308EDL | SOT-323 | Vishay | P-Channel PFET (Power Gate) |
| D3 | USBLC6-2 | SOT-23-6 | STMicro | ESD Protection |

---

## 4. Hardware Implementation Details

### 4.1 Power Management Strategy
The BQ25180 handles LiPo charging and provides the VSYS rail. This rail feeds the RT6160 Buck-Boost converter, which maintains a stable 3.3V output. This configuration is critical for longevity, as the Buck-Boost can maintain regulation even when the battery voltage drops below 3.3V, effectively utilizing the entire discharge curve of the LiPo cell.

### 4.2 Display Power Gating
To minimize parasitic leakage during deep sleep, the E-paper power rail (VEPD) is controlled via a SI1308EDL PFET high-side switch. The MCU toggles this switch via P1.01, physically disconnecting the display logic and charge pump from the power rail when no refresh cycle is active.

### 4.3 Haptic and Motion Sensing
* DRV2605L: Provides I2C-controlled haptic feedback for ERM/LRA motors with integrated waveform libraries.
* BMA423: Low-power accelerometer used for step counting and gesture recognition. It is configured to wake the MCU via a hardware interrupt on P0.08.

---

## 5. Pinout Mapping (nRF52840)

### 5.1 Communication Interfaces
* I2C (SDA/SCL): P0.06 / P0.07 (Shared bus for Charger, Fuel Gauge, Accel, and Haptics).
* SPI (Display): P0.02 (SCK), P0.03 (MOSI), P0.05 (CS), P0.15 (DC), P0.16 (RST).

### 5.2 GPIO Control Lines
* Display Busy: P0.17 (Input - monitors refresh state).
* Power Control: P1.01 (Output - PFET Enable).
* Haptic Enable: P0.12 (Output - DRV2605L EN).
* BMA Interrupt: P0.08 (Input - Motion wake-up).
* Battery Alert: P0.10 (Input - Fuel Gauge alert).
* Charger Interrupt: P0.11 (Input - Charging state change).

---

## 6. Layout Guidelines

### 6.1 RF Section
* The path from U1 Pin 55 (ANT) to ANT1 must be as short as possible.
* Maintain a 50-ohm impedance-controlled trace.
* Implement a ground keep-out zone on all layers directly beneath the ANT1 chip antenna.

### 6.2 Power Integrity
* Place the 0.47uH inductor (L7) and decoupling capacitors (22uF/10uF) as close as possible to the RT6160 (IC9) pins.
* Use wide traces for VBUS, VSYS, and VDD_3V3 to minimize voltage drops.

### 6.3 Grounding
* Utilize a solid ground plane on Layer 2.
* Stitch ground planes with multiple vias, especially near decoupling capacitors and the MCU thermal pad.

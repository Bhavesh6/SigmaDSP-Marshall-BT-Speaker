# 🎸 DIY Marshall Middleton-Style Smart DSP Speaker

> **"Professional-grade portable Bluetooth speaker with SigmaDSP processing, LDAC audio, and full live tuning — built from scratch."**

[![Status](https://img.shields.io/badge/Status-Concept%20%2F%20Design%20Phase-yellow?style=for-the-badge)]()
[![Hardware](https://img.shields.io/badge/Hardware-Not%20Started%20Yet-red?style=for-the-badge)]()
[![DSP](https://img.shields.io/badge/DSP-ADAU1701-blue?style=for-the-badge)]()
[![BT Codec](https://img.shields.io/badge/Bluetooth-LDAC%20%7C%20aptX%20%7C%20AAC%20%7C%20SBC-green?style=for-the-badge)]()
[![MCU](https://img.shields.io/badge/MCU-ESP32--WROVER-orange?style=for-the-badge)]()

---

## ⚠️ Project Status

```
🔵 Phase 1 — Concept & Research         ██████████ 100% COMPLETE
🔵 Phase 2 — Architecture Design        ██████████ 100% COMPLETE  
🔵 Phase 3 — Component Selection        ██████████ 100% COMPLETE
🔵 Phase 4 — Firmware Planning          ████░░░░░░  40% IN PROGRESS
🔴 Phase 5 — PCB / Schematic Design     ░░░░░░░░░░   0% NOT STARTED
🔴 Phase 6 — Hardware Assembly          ░░░░░░░░░░   0% NOT STARTED
🔴 Phase 7 — Firmware Implementation    ░░░░░░░░░░   0% NOT STARTED
🔴 Phase 8 — Testing & Tuning           ░░░░░░░░░░   0% NOT STARTED
```

> **This repository currently documents all theoretical design decisions, component choices, architecture plans, and programming strategies worked out before hardware build begins. All hardware work, code, and schematics will be committed here as the project progresses.**

---

## 🎯 Project Goal

Build a **DIY portable Bluetooth speaker** that closely replicates the acoustic performance, feature set, and aesthetic of the **Marshall Middleton** — using a custom PCB, dedicated SigmaDSP hardware, LDAC Bluetooth audio, and full live parameter tuning via a mobile app.

### Why Marshall Middleton as Target?

| Marshall Middleton Spec | Value |
|---|---|
| Total Amplifier Power | 80W (2×30W woofer + 2×10W tweeter) |
| Drivers | 2× 3" woofers + 2× 0.6" tweeters |
| Bass Extension | 2× passive radiators |
| Frequency Response | 50Hz – 20kHz |
| Max SPL | 87 dB @ 1m |
| Sound Stage | 360° True Stereophonic |
| Bluetooth | BT 5.3, LDAC, SBC |
| Battery | Replaceable Li-ion, 30hr playtime |
| Controls | Bass knob, treble knob, volume, BT |
| IP Rating | IP67 |
| Dimensions | 230 × 98 × 110mm |
| Retail Price | ₹21,000+ |

### Our DIY Target

| DIY Clone Spec | Target Value |
|---|---|
| Total Power | ~80W (matched) |
| Drivers | 2× 3" woofers + 2× 1" dome tweeters |
| Bass Extension | 2× passive radiators (matched) |
| Frequency Response | ~55Hz – 20kHz |
| Sound Stage | 360° stereo (front + rear firing) |
| Bluetooth | BT 4.2, LDAC + aptX + AAC + SBC |
| DSP | ADAU1701 — hardware SigmaDSP (better than Marshall's) |
| Battery | 3× 18650 (3S), ~20–25hr |
| Controls | Physical knobs + mobile app |
| Dimensions | ~240 × 105 × 115mm |
| **Estimated Build Cost** | **₹5,000 – ₹6,000** |

---

## 🏗️ System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        SIGNAL FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📱 Phone (LDAC/aptX/AAC/SBC)                                  │
│       │                                                         │
│       │ Bluetooth A2DP Classic                                  │
│       ▼                                                         │
│  [ESP32-WROVER] ←──── BLE GATT ←──── 📱 Mobile App            │
│   WillyBilly06        (live tune)      (EQ/presets/vol)        │
│   LDAC decoder                                                  │
│       │                                                         │
│       │ I2S Digital Audio (24-bit / 48kHz or 96kHz)            │
│       ▼                                                         │
│  [ADAU1701 DSP] ←──── I²C ←──── ESP32 (live parameter writes) │
│   SigmaStudio program:                                          │
│   • 5-band Parametric EQ                                        │
│   • Linkwitz-Riley 4th-order crossover @ 3kHz                  │
│   • Bass shelf / Treble shelf                                   │
│   • Soft limiter / Compressor                                   │
│   • Output routing (4-channel)                                  │
│       │                                                         │
│   ┌───┴────────────────────┐                                    │
│   ▼                        ▼                                    │
│ [DAC L/R — Woofer]   [DAC L/R — Tweeter]  ← Built-in DACs     │
│       │                        │                                │
│  [TPA3116D2]            [TPA3110D2]                             │
│   2×30W Class-D          2×10W Class-D                         │
│       │                        │                                │
│  [Woofer L] [Woofer R]  [Tweeter L] [Tweeter R]                │
│  3" 4Ω      3" 4Ω       1" dome 8Ω  1" dome 8Ω                │
│                                                                 │
│  [Passive Radiator 1 — Back]                                    │
│  [Passive Radiator 2 — End]   ← Bass extension (no power)      │
└─────────────────────────────────────────────────────────────────┘
```

### Programming / Setup Flow (One-Time)

```
[PC + SigmaStudio]
       │
       │ USB
       ▼
[CY7C68013A FX2LP]  ← flashed with freeUSBi firmware
       │
       │ I²C
       ▼
[ADAU1701 DSP]  ← program designed + compiled in SigmaStudio
       │
       ▼
[24LC256 EEPROM]  ← DSP program stored here for self-boot
```

### Runtime Control Flow

```
[ESP32-WROVER]
  Core 0:  Bluetooth A2DP sink (LDAC/aptX/AAC decode) + I2S output
  Core 1:  BLE GATT server + I²C ADAU1701 live writes + LED + buttons
       │
       ├── I2S → ADAU1701 (audio data)
       ├── I²C → ADAU1701 Parameter RAM (EQ, crossover, gain)
       ├── I²C → 24LC256 EEPROM (preset save/load)
       └── GPIO → Rotary encoders, push buttons, WS2812B LEDs
```

---

## 🔩 Hardware Components

### Core Electronics

| Component | Part Number | Purpose | Status |
|---|---|---|---|
| **DSP Chip** | ADAU1701JSTZ-RL (LQFP-48) | All audio processing | ✅ Selected |
| **Microcontroller** | ESP32-WROVER-E | BT audio + smart control | ✅ Selected |
| **USBi Programmer** | CY7C68013A FX2LP dev board | SigmaStudio programming | ✅ Selected |
| **DSP EEPROM** | 24LC256 (32KB I²C) | DSP self-boot storage | ✅ Selected |
| **Woofer Amp** | TPA3116D2 module | 2×30W Class-D | ✅ Selected |
| **Tweeter Amp** | TPA3110D2 module | 2×10W Class-D | ✅ Selected |
| **3.3V LDO** | AMS1117-3.3 | ESP32 + ADAU1701 DVDD | ✅ Selected |
| **1.8V LDO** | AMS1117-1.8 | ADAU1701 AVDD (dedicated) | ✅ Selected |

### Speaker Drivers

| Driver | Spec | Qty | Role |
|---|---|---|---|
| **Woofer** | 3" full-range, 4Ω, 15W | 2 | Bass + mids, front-firing |
| **Tweeter** | 1" dome, 8Ω, 5W | 2 | Highs, front-firing |
| **Passive Radiator** | 3–4", tuned 60–80Hz | 2 | Bass extension (back + end) |

### Power System

| Component | Part | Purpose |
|---|---|---|
| Battery | 3× 18650 Li-ion (3S1P, ~9600mAh) | Main power |
| BMS | 3S BMS 20A | Protection + balancing |
| Charger | USB-C PD input + TP5100 module | Charging |
| Power Rail 1 | 12V direct from battery | Both amp boards |
| Power Rail 2 | AMS1117-3.3 | ESP32 + DSP digital |
| Power Rail 3 | AMS1117-1.8 (separate LDO) | DSP analog (AVDD only) |

### User Interface

| Component | Part | Function |
|---|---|---|
| Volume/Power | EC11 Rotary encoder (push) | Power on/off + volume |
| Bass control | B10K rotary pot | Bass shelf adjust → ADAU1701 |
| Treble control | B10K rotary pot | Treble shelf adjust → ADAU1701 |
| Bluetooth button | Push button | BT pairing |
| LED indicator | WS2812B strip (10 LEDs) | Battery + status + VU meter |
| AUX input | 3.5mm panel mount jack | Wired audio input |
| USB-C port | Panel mount USB-C | Charging + power bank |

---

## 🧠 The USBi Problem — How We Solved It

### The Problem

The **EVAL-ADUSB2EBZ** (official Analog Devices USBi programmer) is required to program the ADAU1701 via SigmaStudio. It costs ₹5,000–₹8,000 and is not easily available in India.

### How We Analyzed It

The official USBi is simply:
- A **CY7C68013A** USB microcontroller chip
- With Analog Devices proprietary firmware
- Acting as a **USB → I²C bridge** to the DSP

Since the CY7C68013A is just a general-purpose USB chip, and open-source firmware exists that clones the USBi protocol, **we can replicate it exactly**.

### Our Solution — Option A (DIY USBi)

```
Official USBi (EVAL-ADUSB2EBZ)         DIY Equivalent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Chip: CY7C68013A              =        Chip: CY7C68013A (identical)
Firmware: ADI proprietary     →        Firmware: freeUSBi (open-source)
Interface: USB → I²C          =        Interface: USB → I²C (identical)
SigmaStudio: Native support   =        SigmaStudio: Works identically
Cost: ₹5,000–₹8,000           →        Cost: ₹300–₹400
```

### Programming Flow

1. Buy EZ-USB FX2LP dev board (CY7C68013A, available on Robu.in for ~₹350)
2. Flash [freeUSBi firmware](https://github.com/DatanoiseTV/freeUSBi) to board's onboard EEPROM
3. Install WinUSB driver via [Zadig](https://zadig.akeo.ie/)
4. Open SigmaStudio → Hardware Configuration → Add USBi → detects automatically
5. Design DSP program → compile → write to ADAU1701 or EEPROM
6. Done — FX2LP board is now permanently a USBi clone

> **Key insight:** The ADAU1701 uses I²C for all programming. The FX2LP's I²C master maps directly to SigmaStudio's communication protocol. No hardware difference exists between the DIY and official programmer for I²C-based DSPs.

### Why Not Option B, C, or D?

We evaluated four approaches:

| Option | Approach | Decision | Reason |
|---|---|---|---|
| **A** | FX2LP + freeUSBi → SigmaStudio | ✅ **CHOSEN** | Professional, exact workflow, one-time tool |
| B | ESP32 writes .BIN directly to EEPROM | ✅ Also used (runtime) | For OTA preset updates |
| C | ESP32 as software DSP | ❌ Rejected | Insufficient CPU precision for multi-band real-time audio |
| D | STA350/STA328 alternative DSP | ❌ Rejected | Far less flexible, no graphical design tool |

**Final decision: A + B combined.** FX2LP for initial programming, ESP32 for live runtime control.

---

## 🎵 Bluetooth Audio — LDAC on ESP32 BT 4.2

### The Question We Had

*"Can ESP32's Bluetooth 4.2 support LDAC? Can we upgrade to BT 5.0?"*

### The Answer

**Bluetooth version ≠ codec quality.** This is a common misconception.

```
BT Version controls:
  4.2 → Classic BT + BLE, A2DP profile support
  5.0 → Longer range, higher BLE throughput, LE Audio (LC3)
  5.3 → Improved connection stability

A2DP (audio) codec support is determined by:
  → The Bluetooth HOST STACK software
  → NOT the hardware Bluetooth version

LDAC streams over Classic Bluetooth A2DP — available since BT 2.0
BT 4.2 hardware + patched host stack = full LDAC support ✅
```

### Solution — WillyBilly06 Repo

Repository: [`WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC`](https://github.com/WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC)

Also see updated version: [`WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED`](https://github.com/WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED)

This project patches the **Bluedroid host stack** inside ESP-IDF to add LDAC, aptX, aptX-HD, and AAC decoders — bypassing the hardware version entirely.

### Codec Support Matrix

| Codec | Bitrate | Sample Rate | PSRAM Required | Phone Support |
|---|---|---|---|---|
| SBC | 328 kbps | 44.1kHz | ❌ No | Universal |
| AAC | 256 kbps | 44.1kHz | ✅ Yes (WROVER) | iPhone + Android |
| aptX | 352 kbps | 44.1kHz | ❌ No | Android (Qualcomm) |
| aptX HD | 576 kbps | 48kHz | ✅ Yes | Android (Qualcomm) |
| **LDAC** | **990 kbps** | **96kHz** | **✅ Yes (WROVER)** | **Android (Sony)** |

**ESP32-WROVER with 4MB PSRAM → all codecs enabled including LDAC at 96kHz.**

### Why LDAC Matters Here

The ADAU1701 DSP processes audio at full 24-bit / 48kHz precision. Feeding it compressed SBC (lossy, 328kbps) wastes that precision. LDAC at 990kbps is nearly lossless — the DSP gets clean source material to work with, resulting in noticeably better output especially at high sample rates.

---

## 🎛️ DSP Design — ADAU1701 SigmaStudio Program

### Signal Chain (Planned)

```
I2S Input (stereo, 24-bit, 48kHz from ESP32)
    │
    ├─ [Input Volume / Gain Stage]
    │
    ├─ [5-Band Parametric EQ]
    │       Band 1: 80Hz   — Bass shelf        (+/-12dB)
    │       Band 2: 300Hz  — Low-mid           (+/-8dB)
    │       Band 3: 1kHz   — Presence          (+/-8dB)
    │       Band 4: 3kHz   — Upper-mid boost   (+/-8dB) ← Marshall character
    │       Band 5: 10kHz  — Treble shelf      (+/-12dB)
    │
    ├─ [Dynamic Processor — Soft Limiter]
    │       Threshold: -3dBFS
    │       Ratio: 4:1
    │       Attack: 5ms, Release: 100ms
    │
    ├─ [Linkwitz-Riley 4th-Order Crossover @ 3kHz]
    │       Woofer output  → Low-pass  (< 3kHz)
    │       Tweeter output → High-pass (> 3kHz)
    │
    ├── DAC 0 (L) → Woofer Left  → TPA3116D2 Ch1
    ├── DAC 1 (R) → Woofer Right → TPA3116D2 Ch2
    ├── DAC 2 (L) → Tweeter Left  → TPA3110D2 Ch1
    └── DAC 3 (R) → Tweeter Right → TPA3110D2 Ch2
```

### Why Linkwitz-Riley Crossover?

LR4 crossovers sum flat at the crossover point — both drivers output equal amplitude at 3kHz, and they sum to 0dB with correct phase. This eliminates the "notch" or "bump" at the crossover frequency that cheaper Butterworth crossovers produce. Critical for clean integration between woofer and tweeter.

### Live Parameter Control (ESP32 → ADAU1701 I²C)

The ADAU1701 has two memory regions accessible over I²C at runtime:

```
Program RAM  → DSP algorithm (written once, from EEPROM on boot)
Parameter RAM → Filter coefficients, gain, crossover freq (writable ANYTIME)
```

This means EQ, crossover frequency, volume, and gain can all be updated in real time without any audio interruption — exactly like dragging sliders in SigmaStudio.

### ADAU1701 Self-Boot Configuration

```
SELFBOOT pin (Pin 10) → 3.3V via 10kΩ resistor
= DSP reads program from 24LC256 EEPROM at power-on automatically
= No ESP32 intervention needed for boot
= ESP32 only needed for live parameter tuning
```

---

## 🔌 Pin Assignments & Wiring

### I²C Bus — Shared (ESP32 + ADAU1701 + EEPROM)

```
ESP32 GPIO21 (SDA) ──┬── 4.7kΩ pull-up to 3.3V
                     ├── ADAU1701 SDIO (Pin 9)
                     └── 24LC256 SDA

ESP32 GPIO22 (SCL) ──┬── 4.7kΩ pull-up to 3.3V
                     ├── ADAU1701 SCLK (Pin 8)
                     └── 24LC256 SCL

I²C Addresses:
  ADAU1701 → 0x34 (ADDR0=GND, ADDR1=GND)
  24LC256  → 0x50 (A0=A1=A2=GND)
```

### I2S Bus — ESP32 → ADAU1701 (Audio)

```
ESP32 GPIO26 → ADAU1701 BCLK  (Pin 13)
ESP32 GPIO25 → ADAU1701 LRCLK (Pin 14)
ESP32 GPIO22 → ADAU1701 SDATA_IN0 (Pin 16)

Mode: ADAU1701 as I2S SLAVE
Format: I2S standard, 24-bit, 48kHz stereo
```

### FX2LP → ADAU1701 (Programming only — bench use)

```
FX2LP PA0 (SCL) → ADAU1701 SCLK (Pin 8) + 4.7kΩ to 3.3V
FX2LP PA1 (SDA) → ADAU1701 SDIO (Pin 9) + 4.7kΩ to 3.3V
FX2LP GND       → ADAU1701 GND
FX2LP 3.3V      → ADAU1701 IOVDD (via 10Ω series resistor)
```

### ADAU1701 Power Rails (Critical)

```
3.3V → DVDD (Pin 5, 6)        + 100nF + 10µF per pin
3.3V → IOVDD (Pin 28)         + 100nF
1.8V → AVDD (Pin 1, 2, 3)     + 100nF + 10µF per pin  ← SEPARATE LDO
VREF (Pin 27) → 1µF to GND   (reference only, no load)

⚠️ NEVER power AVDD from same LDO as DVDD
⚠️ AGND and DGND star-grounded at single point near PSU input
```

---

## 📐 Enclosure Design (Planned)

```
Target dimensions: 240 × 105 × 115mm
(Marshall Middleton: 230 × 98 × 110mm — matched closely)

Front face:
  [Tweeter L] [Woofer L]          [Woofer R] [Tweeter R]

Back face:
  [Passive Radiator 1] — bass porting

Left end:
  [Passive Radiator 2] — additional bass extension

Top panel (Marshall-style):
  [Power/Volume knob] [BASS] [TREBLE] [BT LED] [Battery indicator]

Rear panel:
  [USB-C charge port] [AUX 3.5mm input]

Material options under consideration:
  Option 1: 3D printed PLA/PETG + fabric grille (Marshall texture)
  Option 2: Laser-cut MDF + vinyl wrap
  Option 3: Bamboo shell (premium look)
```

---

## 📦 Complete Bill of Materials

| # | Component | Part | Qty | Est. Cost (₹) | Source |
|---|---|---|---|---|---|
| 1 | DSP | ADAU1701JSTZ-RL LQFP-48 | 1 | ₹350 | Robu.in ✅ |
| 2 | Programmer | CY7C68013A FX2LP dev board | 1 | ₹350 | Robu.in ✅ |
| 3 | Microcontroller | ESP32-WROVER-E | 1 | ₹500 | Amazon/Robu |
| 4 | DSP EEPROM | 24LC256 | 1 | ₹40 | Robu/Evelta |
| 5 | Woofer | 3" full-range 4Ω 15W | 2 | ₹600 | Amazon/AliExpress |
| 6 | Tweeter | 1" dome 8Ω 5W | 2 | ₹300 | Amazon |
| 7 | Passive Radiator | 3–4" | 2 | ₹800 | AliExpress |
| 8 | Woofer Amp | TPA3116D2 module | 1 | ₹200 | Robu |
| 9 | Tweeter Amp | TPA3110D2 module | 1 | ₹150 | Robu |
| 10 | LDO 3.3V | AMS1117-3.3 | 2 | ₹40 | Robu |
| 11 | LDO 1.8V | AMS1117-1.8 | 1 | ₹20 | Robu |
| 12 | Battery | 18650 3000mAh (Samsung/LG) × 3 | 3 | ₹900 | Amazon |
| 13 | BMS | 3S 20A BMS | 1 | ₹150 | Robu |
| 14 | Charger IC | TP5100 / USB-C PD module | 1 | ₹100 | Robu |
| 15 | Volume knob | EC11 rotary encoder | 1 | ₹30 | Robu |
| 16 | Bass/Treble | B10K rotary pot × 2 | 2 | ₹40 | Local |
| 17 | LED indicator | WS2812B strip (10 LEDs) | 1 | ₹80 | Robu |
| 18 | USB-C port | Panel mount USB-C | 1 | ₹50 | Amazon |
| 19 | AUX jack | 3.5mm panel mount | 1 | ₹30 | Local |
| 20 | Enclosure | 3D print / MDF | 1 | ₹500–1000 | Custom |
| 21 | Passive components | Caps, resistors, pull-ups | — | ₹200 | Local |
| | | | | **≈ ₹5,400–6,000** | |

---

## 🗺️ Development Roadmap

### ✅ Phase 1 — Concept (COMPLETE)
- [x] Defined project goal: Marshall Middleton-style portable speaker
- [x] Chose ADAU1701 SigmaDSP as audio processing core
- [x] Researched Bluetooth codec options (LDAC, aptX, AAC, SBC)
- [x] Decided on ESP32-WROVER as controller + BT receiver
- [x] Confirmed WillyBilly06 repo for LDAC on ESP32 BT 4.2

### ✅ Phase 2 — Architecture (COMPLETE)
- [x] Designed full signal chain (BT → ESP32 → I2S → ADAU1701 → Amp → Speakers)
- [x] Designed USBi clone solution (FX2LP + freeUSBi)
- [x] Planned DSP program structure (EQ + crossover + limiter)
- [x] Defined power architecture (3S battery + separate LDO rails)
- [x] Confirmed dual-amp architecture (TPA3116 + TPA3110)
- [x] Planned enclosure layout (360° stereo, Marshall-style top panel)

### ✅ Phase 3 — Component Selection (COMPLETE)
- [x] All ICs selected and sourced/verified
- [x] Speaker drivers specified (3" woofer + 1" dome tweeter + passive radiators)
- [x] BOM finalized with Indian sourcing

### 🔄 Phase 4 — Firmware Planning (IN PROGRESS)
- [x] WillyBilly06 LDAC repo identified and studied
- [x] MCUdude SigmaDSP library identified for ESP32 I²C control
- [x] Wei1234c TCPi identified for optional SigmaStudio TCP bridge
- [ ] FreeRTOS task architecture planned for ESP32
- [ ] BLE GATT service design for mobile app control
- [ ] OTA preset update flow design

### 🔴 Phase 5 — Schematic & PCB (NOT STARTED)
- [ ] ADAU1701 breakout schematic (EasyEDA)
- [ ] ESP32 + ADAU1701 integration schematic
- [ ] Power supply schematic (3S BMS + LDOs)
- [ ] Amp board integration
- [ ] PCB layout and gerber export

### 🔴 Phase 6 — Hardware Assembly (NOT STARTED)
- [ ] Solder ADAU1701 LQFP-48 (0.5mm pitch — hot air required)
- [ ] Build power supply board
- [ ] Flash FX2LP with freeUSBi
- [ ] Connect FX2LP → ADAU1701, verify SigmaStudio detection
- [ ] Mount amp modules, wire to speakers
- [ ] Build enclosure

### 🔴 Phase 7 — Firmware (NOT STARTED)
- [ ] Flash WillyBilly06 LDAC firmware to ESP32-WROVER
- [ ] Implement MCUdude SigmaDSP I²C control layer
- [ ] Design SigmaStudio DSP program (EQ + crossover + dynamics)
- [ ] Export DSP program to 24LC256 EEPROM
- [ ] Implement BLE GATT server for mobile app
- [ ] Implement physical controls (rotary encoders, buttons)
- [ ] Implement WS2812B LED animations (VU meter, battery, spectrum)

### 🔴 Phase 8 — Tuning & Testing (NOT STARTED)
- [ ] Verify ADAU1701 self-boot from EEPROM
- [ ] Test LDAC/aptX/AAC codec switching
- [ ] Measure frequency response (woofer + tweeter + passive radiator)
- [ ] Tune Linkwitz-Riley crossover frequency (target: 3kHz)
- [ ] Tune EQ for Marshall sound signature (upper-mid presence boost)
- [ ] Battery life testing
- [ ] SPL measurement

---

## 💻 Software Stack

### Firmware — ESP32

| Layer | Library/Tool | Purpose |
|---|---|---|
| BT Audio | [WillyBilly06 LDAC repo](https://github.com/WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC) | LDAC/aptX/AAC/SBC decode |
| DSP Control | [MCUdude/SigmaDSP](https://github.com/MCUdude/SigmaDSP) | I²C parameter writes to ADAU1701 |
| BLE App Control | ESP-IDF BLE GATT | Mobile app interface |
| LED | FastLED / NeoPixel | WS2812B VU meter + animations |
| Storage | LittleFS | Preset storage on ESP32 flash |
| OTA | ESP-IDF OTA | Firmware + DSP preset updates |

### DSP Design

| Tool | Purpose |
|---|---|
| [SigmaStudio](https://www.analog.com/en/design-center/evaluation-hardware-and-software/software/ss_sigst_02.html) | Graphical DSP program design |
| [Linkwitz-Riley Calculator](https://www.linkwitzlab.com/) | Crossover coefficient calculation |
| ADI AN-1006, AN-1168 | Application notes: crossovers + EQ |

### Programming Tools

| Tool | Purpose |
|---|---|
| [freeUSBi](https://github.com/DatanoiseTV/freeUSBi) | FX2LP firmware for USBi clone |
| [Zadig](https://zadig.akeo.ie/) | WinUSB driver install for FX2LP |
| Cypress EZ-USB Suite | FX2LP EEPROM flashing |
| [esptool](https://github.com/espressif/esptool) | ESP32 firmware flashing |
| EasyEDA | PCB schematic design |

---

## 🔑 Key Technical Decisions & Rationale

### 1. Why ADAU1701 over software DSP on ESP32?

The ADAU1701 runs a 50MHz dedicated audio DSP core with 56-bit double-precision arithmetic, specifically designed for biquad filter computation. It processes 4-channel audio with sub-1ms latency completely independently of any MCU. An ESP32 running software biquads in floating-point at the same sample rate would consume ~60–80% CPU, leave little headroom for Bluetooth decode, and introduce measurable latency and rounding noise. The ADAU1701 was the only correct choice for a speaker claiming professional audio quality.

### 2. Why ESP32-WROVER specifically?

PSRAM is mandatory for LDAC decode. The LDAC decoder is a complex floating-point algorithm originally written for desktop hardware. On ESP32, it requires extended buffers that exceed the 520KB internal SRAM. The WROVER's 4MB PSRAM provides this headroom. Without PSRAM (e.g., bare ESP32-DevKitC), LDAC fails to initialize and the system falls back to SBC.

### 3. Why two separate amplifier ICs (TPA3116 + TPA3110)?

Each driver type has a different impedance (4Ω woofer vs 8Ω tweeter) and power requirement (30W vs 10W). A single stereo amp IC would require both channels to be configured for the same load impedance, compromising either the woofer or tweeter drive level. Separate amp ICs allow optimal biasing per driver type and independent gain control — matching how the real Marshall Middleton separates its 4-channel amplifier.

### 4. Why passive radiators instead of a bass port?

Passive radiators provide bass extension equivalent to a tuned reflex port, but without port chuffing noise at high excursion levels. In a compact sealed enclosure at moderate to high volumes, a port can produce audible turbulence in the bass range. The Marshall Middleton uses passive radiators for this exact reason. In a DIY build, passive radiators are also mechanically simpler to mount correctly — no tuning pipe length calculation needed.

### 5. Why I2S from ESP32 to ADAU1701 instead of using ADAU1701's built-in ADC?

The ADAU1701's built-in ADCs are excellent for analog inputs, but when receiving Bluetooth audio, the signal is already in digital PCM format after decoding. Passing it through an unnecessary DAC → ADC conversion (ESP32 DAC → ADAU1701 ADC) would introduce noise and degrade the audio path. I2S slave input on the ADAU1701 accepts the PCM data directly in digital domain — zero additional conversion, maximum quality.

---

## 📚 References & Key Resources

| Resource | Link | Purpose |
|---|---|---|
| ADAU1701 Datasheet | [Analog Devices](https://www.analog.com/en/products/adau1701.html) | DSP chip reference |
| SigmaStudio | [ADI Download](https://www.analog.com/en/design-center/evaluation-hardware-and-software/software/ss_sigst_02.html) | DSP programming IDE |
| WillyBilly06 LDAC Repo | [GitHub](https://github.com/WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC) | ESP32 LDAC firmware |
| WillyBilly06 Updated | [GitHub](https://github.com/WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED) | ESP32 IDF 5.5.2 version |
| MCUdude SigmaDSP | [GitHub](https://github.com/MCUdude/SigmaDSP) | Arduino I²C DSP control |
| freeUSBi Firmware | [GitHub](https://github.com/DatanoiseTV/freeUSBi) | DIY USBi clone |
| pschatzmann ESP32-A2DP | [GitHub](https://github.com/pschatzmann/ESP32-A2DP) | Alternative A2DP lib |
| EZ-USB TRM | [Infineon](https://www.infineon.com) | CY7C68013A reference |
| Wei1234c TCPi | [GitHub](https://github.com/Wei1234c/TCPi) | SigmaStudio TCP bridge |
| Linkwitz-Riley Calc | [linkwitzlab.com](https://www.linkwitzlab.com/) | Crossover design |
| ADI AN-1006 | Analog Devices | DSP crossover design note |
| ADI AN-1168 | Analog Devices | EQ filter design note |
| ADI EngineerZone | [ez.analog.com](https://ez.analog.com/) | ADAU1701 community forum |

---

## 🤝 Contributing

This is a personal DIY project currently in design phase. Once hardware build begins, all schematics, firmware, SigmaStudio project files, and gerbers will be uploaded here.

If you're working on a similar ADAU1701 or ESP32 audio project:
- Open an **Issue** for questions or suggestions
- Open a **Discussion** for general exchange

---

## 📜 License

This project is open-source under the **MIT License**.

Third-party components used:
- WillyBilly06 ESP32 LDAC repo — see their respective license
- MCUdude SigmaDSP — MIT License
- freeUSBi — GPL v3

---

## 👤 Author

**Ghost**


> *"Built from scratch, tuned by ear, designed to last."*

---

*Last updated: May 2026 — Design phase complete, hardware build upcoming.*

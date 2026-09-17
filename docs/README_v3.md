# 🎸 SigmaDSP-Marshall-BT-Speaker

> **A fully engineered DIY portable Bluetooth speaker built from raw chips — targeting Marshall Middleton acoustic performance using ADAU1701 SigmaDSP, ESP32-WROVER with LDAC, 4-channel Class-D amplification, and a precision-tuned acoustic enclosure.**

[![Status](https://img.shields.io/badge/Status-V3%20Deep%20Research%20Complete-yellow?style=for-the-badge)]()
[![Hardware](https://img.shields.io/badge/Hardware-Not%20Started-red?style=for-the-badge)]()
[![DSP](https://img.shields.io/badge/DSP-ADAU1701%20SigmaDSP-blue?style=for-the-badge)]()
[![BT](https://img.shields.io/badge/Bluetooth-LDAC%20%7C%20aptX%20%7C%20AAC%20%7C%20SBC-green?style=for-the-badge)]()
[![MCU](https://img.shields.io/badge/MCU-ESP32--WROVER-orange?style=for-the-badge)]()
[![License](https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge)]()

---

## Project Status

```
V1 — Initial Concept                        COMPLETE
V2 — Architecture + Gap Analysis R1         COMPLETE
V3 — Deep Research + All New Challenges     COMPLETE
V4 — Controller Architecture Decision      IN PROGRESS (zero PCB testing needed)
V5 — Firmware Planning (Both MCUs)         IN PROGRESS
--- HARDWARE NOT STARTED ---
Testing Phase 1 — Zero PCB Prototyping     NOT STARTED
Testing Phase 2 — Individual Module Test   NOT STARTED
Final Phase   — All-in-One Custom PCB      NOT STARTED
Acoustic Phase — Enclosure + Tuning        NOT STARTED
App Development                            NOT STARTED
```

> This repository is the complete living design document of every decision made, every problem found, and every solution planned — before a single component is soldered. Hardware will be committed here as it is built and tested.

---

## How This Project Evolved — V1 to V3

### V1 — The Naive First Approach

What we thought we were building:
```
Phone -> Bluetooth -> ESP32 -> I2S -> ADAU1701 -> DAC -> Amp -> Speaker
```

What we got wrong:
- I2S clock direction backwards (ESP32 master → ADAU1701 slave) — would cause complete silence
- Missing 12.288MHz crystal oscillator — DSP literally cannot start without it
- No Thiele-Small analysis — "3 inch 4Ω 15W" is meaningless without T/S parameters
- Thought True Stereophonic was DSP widening — it is physical driver placement
- Generic EQ curve — does not sound like Marshall at all
- No PCB grounding plan — Class-D amp + 24-bit DSP on shared trace = noise
- Assumed one ESP32 handles everything without analyzing memory usage
- No battery cell-level monitoring plan
- I2S DAC module in signal path — ADAU1701 has 4 built-in DACs, DAC module is redundant

V1 was a list of components, not an engineering design.

---

### V2 — Architecture Defined, First Round of Mistakes Corrected

Fixed issues:
- I2S corrected: ADAU1701 is master (generates BCLK + LRCLK from crystal)
  MP10/MP11 loopback traces to MP5/MP4 on PCB
  ESP32 configured as I2S slave
- 12.288MHz crystal oscillator added to BOM
- 48kHz sample rate lock — WillyBilly06 firmware must resample all BT audio to 48kHz
- True Stereophonic = physical diagonal tweeter placement (confirmed by Middleton teardown)
- PCB 2-layer ground plane strategy defined (AGND / DGND zones, star point)
- Zobel networks added to amp outputs (8.2Ω + 100nF per channel)
- Input filter added DAC to amp (100Ω + 10nF)
- Pop prevention RC circuit added (10MΩ + 47µF on amp MUTE pin)
- Marshall EQ curve corrected (50Hz sub-bass, 2.5kHz presence, mud cut at 250Hz)
- Baffle step correction identified (+4dB low-shelf around 1kHz)
- MAX17043 fuel gauge added to BOM

V2 was a correct architecture. But still missing the deep layers.

---

### V3 — Everything We Now Know

New discoveries in V3:

1. ESP32 memory with LDAC + BLE simultaneously is critically tight
2. Single MCU may not be enough — decision requires physical testing
3. Individual cell voltage monitoring needs dedicated resistor divider circuit
4. Battery runtime is much lower than expected at real listening volumes
5. Testing strategy must be zero PCB first — not straight to custom PCB
6. App design is far more complex — 25 BLE GATT characteristics needed
7. Acoustic tuning is three layers (physical, correction, signature) not one
8. DSP signal chain needs 11 blocks in specific order — not arbitrary
9. Baffle step correction formula is enclosure-width dependent
10. Passive radiator mass calculation needs actual T/S data — cannot guess

This README documents V3.

---

## Project Goal

Build a DIY portable Bluetooth speaker that matches the Marshall Middleton in actual acoustic performance — not just appearance.

### Why Marshall Level Is Hard for DIY

The Middleton is expensive because:
- Matched driver pairs (measured and selected)
- Enclosure tuned per driver T/S parameters
- Professional DSP tuning with hundreds of measurements + EQ corrections
- 4-channel amp (woofer and tweeter driven independently)
- Placement compensation that adjusts EQ with environment
- SNR greater than 90dB (requires proper grounding and low-noise PSU)

We do all of this from scratch, with raw chips, no reference design.

### Our Advantage Over Real Middleton

Because we use ADAU1701 (fully programmable hardware DSP) instead of Marshall's fixed DSP:

| Feature | Marshall Middleton | Our DIY Build |
|---|---|---|
| DSP access | Locked — no user access | Every coefficient editable live |
| EQ | App: bass + treble only | 10-band parametric, adjustable Q |
| Crossover | Fixed factory setting | Adjustable LR4, any frequency |
| Codec | LDAC + AAC + SBC (BT 5.3) | LDAC + aptX HD + aptX + AAC + SBC |
| Placement compensation | Fixed algorithm | Programmable EQ preset switch |
| Dynamic Loudness | Fixed | Fully programmable |
| OTA | Marshall controls | Full user control |
| Cost | Rs 21,000+ | Rs 5,900 – 6,700 |

---

## The Big Challenge — Raw Chips, Real Engineering

We are NOT using:
- Pre-made ADAU1701 breakout module
- Pre-made audio amplifier shield
- Generic BT audio module
- Pre-designed speaker crossover

We ARE using:
- Bare ADAU1701JSTZ-RL LQFP-48 (0.5mm pitch) on custom PCB
- ESP32-WROVER-E bare module
- Custom 2-layer PCB designed in EasyEDA
- Acoustic enclosure designed from Thiele-Small parameters
- Raw 18650 cells in 3S pack with custom cell monitoring

Every layer — power supply, grounding, signal path, acoustic enclosure, firmware, DSP program — is our responsibility. Things that will go wrong on first try: LQFP-48 solder bridges, I2S slave mode instability, noise floor from grounding, passive radiator needing mass iteration, EQ needing multiple measurement rounds. This is why we test on zero PCBs first.

---

## ESP32 Memory Reality Check

### What LDAC Actually Consumes

Classic Bluetooth alone uses roughly 80-100KB of internal DRAM. LDAC decoder buffers require PSRAM. Running Classic BT (A2DP audio) and BLE simultaneously causes significant additional heap loss of 40-50KB compared to using either alone.

```
ESP32-WROVER Internal DRAM: 520KB total

Allocation:
  Bluetooth controller (Classic BT):    ~80KB
  Bluedroid host stack (A2DP):          ~60KB
  LDAC decoder buffers:         → PSRAM (200KB+, explicit SPIRAM malloc)
  BLE GATT stack (simultaneous):        ~30KB
  FreeRTOS + task stacks:               ~40KB
  I2S DMA buffers (MUST be internal):    ~8KB
  Application code + variables:         ~20KB
  LittleFS + NVS:                       ~10KB
  Remaining internal DRAM:              ~70KB (tight — must be managed)
  PSRAM (4MB):           LDAC buffers + audio queue (safe here)
```

ESP32-WROVER with 4MB PSRAM is mandatory — not optional.

Key firmware rules:
- LDAC decode buffers: `heap_caps_malloc(MALLOC_CAP_SPIRAM)` — put in PSRAM
- I2S DMA buffers: `heap_caps_malloc(MALLOC_CAP_INTERNAL)` — must stay in internal DRAM
- Monitor: `heap_caps_get_free_size(MALLOC_CAP_INTERNAL)` — alert if below 20KB

---

## Single vs Dual Controller Decision

### Option A — Single ESP32-WROVER

```
Core 0: Classic BT A2DP + LDAC decode + resample + I2S TX
Core 1: BLE GATT + I2C DSP writes + encoders + LED + battery monitoring

Pros: Simpler hardware, single firmware, lower cost
Cons: Memory tight, BLE + audio may interfere, any crash kills all functions
```

### Option B — Dual MCU (ESP32-WROVER + STM32/ESP32-C3)

```
ESP32-WROVER: Audio ONLY (BT A2DP + LDAC + I2S TX)
Secondary MCU: BLE GATT + I2C ADAU1701 + GPIO + LED + battery monitoring
Inter-MCU: UART command protocol

Pros: ESP32 has all memory for audio, independent failure domains, cleaner
Cons: More complex hardware, two firmwares, inter-MCU protocol to design
```

### Decision Process

We START with Option A on zero PCB testing.

Zero PCB Stage 4 (BLE GATT + audio simultaneously) is the decision point:
- If free internal DRAM stays above 20KB under full load → Option A confirmed, proceed to custom PCB
- If free internal DRAM drops below 20KB → switch to Option B for final design

The zero PCB stage will stress-test:
- LDAC decoding while BLE GATT actively receives parameter updates
- LED animation running with audio
- I2C writes to ADAU1701 during streaming
- Memory headroom monitoring under full simultaneous load

---

## Battery Management — Cell-Level Monitoring

### Why Standard BMS Is Not Enough

The 3S BMS board we use provides overcharge protection, over-discharge protection, overcurrent protection, and passive cell balancing. What it does NOT provide: individual cell voltages over any digital bus, SOC percentage, temperature monitoring (on basic modules).

### What the System Must Know

```
For accurate app battery display:
  1. Total pack voltage (resistor divider on ESP32 ADC)
  2. Individual cell voltages (dedicated divider circuit per cell tap)
  3. State of Charge % (MAX17043 fuel gauge IC)
  4. Charging status (BMS CHRG pin to GPIO)
  5. Cell balance status (derived from cell voltage differences)
  6. Pack temperature (NTC thermistor on ADC)
```

### 3S Individual Cell Voltage Circuit

A fully charged 3S pack sits at 12.6V (4.2V per cell). The BMS monitors individual cells — if any single cell exceeds 4.25V, charging cuts off immediately.

```
3S pack cell tap points (from B- ground):
  B-  = 0V
  B1  = 0 to 4.2V  (Cell 1)
  B2  = 0 to 8.4V  (Cell 1+2)
  B+  = 0 to 12.6V (full pack)

Voltage dividers (all resistors to B- reference, output to ESP32 ADC):
  Cell 1: B- to B1: 100kΩ / 100kΩ divider → max 2.1V → GPIO34 (ADC1_CH6)
  Cell 2: B- to B2: 200kΩ / 100kΩ divider → max 2.8V → GPIO35 (ADC1_CH7)
  Pack:   B- to B+: 330kΩ / 100kΩ divider → max 2.93V → GPIO32 (ADC1_CH4)
  NTC:    10kΩ NTC + 10kΩ divider → GPIO33 (ADC1_CH5)

Software:
  Cell 1 voltage = ADC34_reading × 2
  Cell 2 alone   = ADC35_reading × 3 − Cell1_voltage
  Cell 3 alone   = ADC32_reading × 4.3 − Cell1 − Cell2

Important:
  Use ADC averaging (32 samples) to reduce noise
  Add 100nF cap across lower resistor of each divider
  Use ADC1 only (ADC2 conflicts with BT/WiFi on ESP32)
```

### Cell Health Alerts

```
Healthy:  All cells within 50mV of each other → Green LED
Drifting: Any cell differs by more than 100mV → Yellow LED + app warning
Critical: Any cell below 3.0V or above 4.2V → Red LED + app alert + volume reduce
Hot:      Pack temperature above 45°C → app warning + volume reduction
Very hot: Above 55°C → hard mute + charge disable
```

---

## App Design — What We Can and Cannot Do

### Full App Feature List

```
Audio Controls:
  Master Volume slider (0–100%)
  EQ Visualizer (10-band graphic, read from DSP)
  Bass Shelf slider (±12dB)
  Treble Shelf slider (±12dB)
  Presence slider at 2.5kHz (±6dB)
  EQ Presets: Marshall Rock / Flat / Bass Boost / Vocal / Open Space / Near Wall

Status Display:
  Battery bar (5-segment, SOC %)
  Per-cell voltage readout (Cell 1 / Cell 2 / Cell 3)
  Pack temperature (°C with color coding)
  Charging status (Charging / Full / Discharging)
  Active codec indicator (LDAC 990kbps / aptX HD / aptX / AAC / SBC + sample rate)
  DSP connection status

Advanced Controls:
  Crossover frequency slider (1kHz – 5kHz, moves LR4 XO point)
  Tweeter delay (0–1ms, time alignment fine-tune)
  Dynamic Loudness toggle (ON / OFF)
  Placement Compensation (Open Space / Near Wall)
  Limiter threshold (-6 to 0 dBFS)
  Save/Load preset (name + save to LittleFS)
  Auto-off timer (0 / 10 / 30 / 60 minutes)

System:
  Firmware OTA (push new ESP32 firmware)
  DSP Program OTA (push new .bin to 24LC256 EEPROM)
  Factory reset
  Device name customization
```

### BLE GATT Design (25 Characteristics)

```
Audio Service:
  Volume (Write)
  EQ Band 1–10 coefficients (Write × 10)
  Active preset index (Read/Write)
  Crossover frequency (Write)
  Tweeter delay (Write)
  Limiter threshold (Write)
  Placement mode (Write)
  Dynamic loudness (Write)

Status Service (Notify):
  Battery SOC % (Read + Notify)
  Cell 1/2/3 voltages (Read + Notify)
  Pack temperature (Read + Notify)
  Charging status (Read + Notify)
  Active codec string (Read + Notify)
  DSP connection (Read + Notify)

System Service:
  OTA firmware trigger (Write)
  OTA DSP program (Write, long)
  Factory reset (Write)
  Device name (Read/Write)
  Auto-off timer (Read/Write)
```

### Real-Time EQ Update Latency

```
User drags EQ slider → BLE write (10–50ms) → ESP32 receives
→ I2C write to ADAU1701 Parameter RAM (~1ms)
→ ADAU1701 applies coefficient on next audio frame (<1ms)
Total: ~11–51ms — acceptable for live EQ control
```

---

## Acoustic Engineering — The Hard Part

### Why This Is the Most Underestimated Section

DSP and firmware follow documented procedures. Acoustics is where guessing costs the most time. A speaker with wrong enclosure volume or untuned passive radiator cannot be fixed by DSP alone. EQ cannot add bass extension below the enclosure's tuning frequency, fix passive radiator over-excursion distortion, or correct tweeter-woofer phase misalignment at crossover. All acoustic problems must be solved in hardware first. DSP is the final 10%.

### Three Layers of Tuning

```
Layer 1: Physical (hardware — cannot change after build)
  Driver selection (Fs, Qts, Xmax from T/S parameters)
  Enclosure volume calculated from T/S
  Passive radiator mass tuned to target frequency
  Driver placement and angle (True Stereophonic = physical)

Layer 2: Acoustic Correction (DSP — fixes hardware imperfections)
  Baffle step correction (+4dB low-shelf at ~1kHz)
  Room resonance notch filters (from REW measurement)
  Driver frequency response correction (from measurement)
  Tweeter level matching to woofer sensitivity
  Tweeter time alignment delay (from REW step response)

Layer 3: Sound Signature (DSP — adds Marshall character)
  Marshall EQ curve (sub-bass, presence, treble — see below)
  Dynamic loudness (Fletcher-Munson at low volumes)
  Placement compensation (open vs wall preset)
  Limiter / compressor

DO LAYER 2 BEFORE LAYER 3.
Get flat response first, then add character on top.
```

### Driver Selection Rules — Mandatory

You cannot choose a driver without these T/S parameters published:

| Parameter | Symbol | Target for Our Build |
|---|---|---|
| Resonant frequency | Fs | Below 120Hz for woofer |
| Total Q | Qts | 0.4 – 0.6 (passive radiator sweet spot) |
| Compliance volume | Vas | 1–3L (determines box volume) |
| Sensitivity | dB/W/m | Above 84dB |
| Max excursion | Xmax | Above 3mm |

If the seller does not publish T/S parameters, skip that driver.

### Enclosure Volume Calculation

```
Using Dayton CE90-4 as target driver (Fs=115Hz, Qts=0.56, Vas=1.2L):

QB3 alignment box volume per driver:
  Vb = Vas × (Qts / 0.38)^2.87 = 1.2 × (0.56/0.38)^2.87 = approx 3.1L total
  Per driver: 1.55L

Enclosure gross dimensions target: 250 × 110 × 120mm
Internal after 12mm MDF walls: 226 × 86 × 96mm = 1.87L per side
After internal displacement (battery, PCB, drivers, damping): ~1.55L net per woofer

This matches the calculation. Enclosure size is confirmed.
```

### Passive Radiator Tuning Formula

```
Tuning frequency formula:
  fp = Fs_woofer × sqrt(Mms / Mmd)

Where:
  Fs = woofer free-air resonance (from T/S data)
  Mms = woofer moving mass (from T/S data, typically 8–12g for 3" driver)
  Mmd = passive radiator moving mass (add weights to tune)

Target fp: 55–65Hz

Example (Fs=115Hz, Mms=9g, target fp=58Hz):
  Mmd = Mms × (Fs/fp)^2 = 9 × (115/58)^2 = 35.4g

Physical implementation:
  Stick metal washers or fishing sinkers to PR cone center
  Start light, measure with REW, add mass until fp reaches 60Hz
  Each gram shifts fp down slightly
  This requires iteration — cannot be calculated to exact final value without measurement
```

### Internal Damping Rules

```
Use: 25mm open-cell polyurethane acoustic foam on all walls EXCEPT
  - Passive radiator mounting area (must breathe freely)
  - Woofer front baffle area (behind cone)

Use: Loosely stuffed polyester fill in remaining volume (~40% fill)
  Effect: Effectively increases Vas by 15–25%, lowers tuning slightly, removes box resonances

Do NOT:
  Overstuff (raises tuning frequency, reduces PR efficiency)
  Use closed-cell foam (reflects instead of absorbs)
  Block passive radiator cavities with any material
```

### Baffle Step Correction

```
Physical effect: High frequencies radiate into 180° half-space,
low frequencies into 360° full space → 6dB bass drop above transition frequency.

Transition frequency = 115 / baffle_width_metres
For 110mm baffle: fc = 115 / 0.11 = 1045Hz

DSP correction in ADAU1701:
  Type: Low-shelf boost
  Frequency: ~1kHz (adjust from actual measurement)
  Gain: +4dB
  This is in Layer 2 (acoustic correction), NOT in Marshall EQ (Layer 3)
```

### Acoustic Measurement Plan

Required equipment:
- UMIK-1 calibrated USB microphone (~Rs 5,000) or calibrated phone mic
- REW (Room EQ Wizard, free software)
- Outdoor or large room for measurement (minimize reflections)

Measurements to take and what to do with them:

```
1. Individual woofer response (L and R, separately, near-field at 15cm)
   → Identifies Fs peak, rolloff, cone breakup modes
   → Used to design notch filters in Layer 2 EQ

2. Individual tweeter response
   → Finds rolloff point, sensitivity vs woofer
   → Used to set tweeter level trim in ADAU1701

3. Combined response (both woofer + tweeter, in-box)
   → Check crossover region 2–4kHz for peaks, dips, phase errors
   → If dip at XO: adjust XO frequency or tweak filter Q
   → If peak at XO: slight adjustment to filter type or level

4. Bass extension (woofer + passive radiator in sealed box)
   → Confirm tuning peak near 60Hz
   → If too high: add mass to passive radiators
   → If too low: remove mass from passive radiators
   → Target: -10dB point at or below 60Hz

5. Step response (time domain in REW)
   → Shows whether tweeter and woofer arrive at mic simultaneously
   → If tweeter leads: add delay to tweeter path in ADAU1701 (Block 10)
   → Typical correction: 0.1–0.3ms

6. THD measurement at 75dB SPL at 1m
   → Target below 1% at all frequencies except near Fs
   → High THD at high volume = over-excursion → reduce volume or adjust limiter

7. Max SPL sweep
   → Find where distortion rises sharply → set limiter threshold there
```

---

## DSP Tuning — Complete ADAU1701 Signal Chain

### All 11 Processing Blocks in Order

```
I2S INPUT (L+R, 48kHz, 24-bit, from ESP32 I2S slave)
    |
[BLOCK 1] Input Volume Control
  Parameter: VOLUME_ADDR
  Range: -80dB to +6dB, live via ESP32 I2C

[BLOCK 2] Dynamic Loudness (Fletcher-Munson compensation)
  SigmaStudio: Dynamics → Loudness block
  Low volumes (<40% output): +5dB @ 50Hz, +3dB @ 12kHz
  High volumes: curves flatten automatically
  Prevents thin sound at low volume

[BLOCK 3] Acoustic Correction EQ (from REW measurement — fill in after build)
  These values cannot be set now.
  After measurement: load REW inverse filter here.
  Removes driver peaks, dips, resonances — makes response flat.

[BLOCK 4] Baffle Step Correction
  Type: Low-shelf boost
  Frequency: ~1kHz (from actual baffle width calculation)
  Gain: +4dB, Q: 0.7

[BLOCK 5] Marshall Sound Signature EQ (Layer 3 — character)
  Applied on top of flat baseline from Blocks 3+4:
  50Hz   Peak  +4.0dB  Q=0.7  Sub-bass punch (leather bass hit)
  120Hz  Peak  +1.5dB  Q=1.0  Bass body and weight
  250Hz  Peak  -1.0dB  Q=1.5  Mud cut (keeps mids clean)
  500Hz  Peak  -0.5dB  Q=1.0  Low-mid clarity
  800Hz  Peak  +0.5dB  Q=2.0  Guitar fundamental
  1.5kHz Peak  +1.0dB  Q=1.5  Vocal and instrument presence
  2.5kHz Peak  +2.5dB  Q=1.2  Marshall presence peak (key signature)
  5kHz   Peak  +1.5dB  Q=1.5  Air and transient detail
  10kHz  Shelf +2.0dB         Treble sparkle
  14kHz  Peak  +1.0dB  Q=0.8  High-end extension

[BLOCK 6] Placement Compensation (switchable via GPIO or app)
  Open Space: corrections as above
  Near Wall:  Add low-shelf cut: -3dB @ 80Hz (room bass buildup)
  Switch via: ADAU1701 MP13 GPIO input triggered from app or button

[BLOCK 7] Stereo Width (M-S based, subtle, optional)
  Mid-Side matrix → slight side boost → convert back to L/R
  Complements physical diagonal tweeter placement
  Depth: 0% (off) to 20% (subtle) — adjustable from app

[BLOCK 8] Soft Limiter + Compressor
  Compressor: Threshold -12dBFS, Ratio 2:1, Attack 10ms, Release 200ms
  Limiter: Threshold -3dBFS, Ratio inf:1, Attack 0.5ms, Release 100ms
  Soft knee on both — prevents harsh clipping character

[BLOCK 9] Linkwitz-Riley 4th Order Crossover at 2.8kHz
  Woofer path:  Low-pass LR4 at 2.8kHz
  Tweeter path: High-pass LR4 at 2.8kHz
  LR4 = sums flat, correct polarity, -24dB/octave

[BLOCK 10] Tweeter Time Alignment Delay
  Delay on tweeter path: 0–0.3ms
  Set from REW step response measurement after assembly

[BLOCK 11] Output Mute (pop prevention)
  MP12 GPIO: LOW during boot → all outputs muted
  After self-boot complete (~200ms): MP12 → HIGH → amps enable

DAC 0 (L) → [100Ω + 10nF filter] → TPA3116 Ch1 → Woofer L (front)
DAC 1 (R) → [100Ω + 10nF filter] → TPA3116 Ch2 → Woofer R (front)
DAC 2 (L) → [100Ω + 10nF filter] → TPA3110 Ch1 → Tweeter L (FRONT)
DAC 3 (R) → [100Ω + 10nF filter] → TPA3110 Ch2 → Tweeter R (REAR diagonal)
```

---

## PCB Strategy — Zero PCB First, Custom PCB Second

### Zero PCB Testing Stages

Stage 1: ADAU1701 standalone
- Build on perfboard: ADAU1701 + crystal + 24LC256 + 1.8V + 3.3V power
- Flash via FX2LP + SigmaStudio — write simple tone generator program
- Verify: DAC output on scope, SELFBOOT works on power cycle
- Pass: move to Stage 2

Stage 2: ESP32 I2S slave to ADAU1701
- Add ESP32-WROVER to Stage 1 board
- Flash minimal I2S slave test firmware (not WillyBilly06 yet)
- ESP32 sends 1kHz test tone over I2S slave to ADAU1701
- Verify: Clean audio output, stable clocks on scope
- Tune: DMA buffer size, APLL settings until zero dropouts
- Pass: move to Stage 3

Stage 3: WillyBilly06 LDAC streaming
- Flash full WillyBilly06 firmware
- Connect phone via Bluetooth, stream music (LDAC if available)
- Verify: Audio quality, no dropouts
- Monitor: `heap_caps_get_free_size(MALLOC_CAP_INTERNAL)` under load
- Decision: If free RAM below 20KB → dual MCU for final design
- Pass: move to Stage 4

Stage 4: BLE GATT + audio simultaneously
- Add BLE GATT control alongside audio in firmware
- Stream audio while sending BLE parameter writes continuously
- Verify: No audio glitches during BLE activity
- Test: Live EQ slider from app while music plays — dropout acceptable?
- Pass: Single MCU confirmed → proceed to custom PCB

Stage 5: Amplifier integration
- Add TPA3116 + TPA3110 on separate perfboard sections
- Wire ADAU1701 DAC outputs through input filters to amp inputs
- Add Zobel networks on outputs, pop prevention RC
- Test with any available speakers — check noise floor (should be silent with no audio)
- Verify SNR: no hiss or hum at any volume setting

Stage 6: Battery and monitoring
- Build 3S 18650 pack with BMS
- Wire cell voltage dividers to ESP32 ADC pins
- Wire MAX17043 to I2C bus
- Verify per-cell voltage readings, SOC % via BLE notification to phone
- Test cell alert thresholds

Only after all stages pass: design custom PCB.

---

## Complete Pin Assignments

### I2S (ADAU1701 Master, ESP32 Slave — Corrected)

```
ADAU1701 MP10 (OUTPUT_BCLK)  ────────────→ ESP32 GPIO26 (BCLK in, slave)
ADAU1701 MP11 (OUTPUT_LRCLK) ────────────→ ESP32 GPIO25 (LRCLK in, slave)
ADAU1701 MP0  (SDATA_IN0)    ←──────────── ESP32 GPIO22 (I2S DOUT)

PCB loopback traces (not wires — PCB copper traces):
ADAU1701 MP10 ──────────────────────────→ MP5 (INPUT_BCLK)
ADAU1701 MP11 ──────────────────────────→ MP4 (INPUT_LRCLK)
```

### I2C Bus

```
ESP32 GPIO21 SDA ──┬── 4.7kΩ → 3.3V
                   ├── ADAU1701 Pin 9 SDIO   addr 0x34
                   ├── 24LC256 SDA           addr 0x50
                   └── MAX17043 SDA          addr 0x36

ESP32 GPIO22 SCL ──┬── 4.7kΩ → 3.3V
                   ├── ADAU1701 Pin 8 SCLK
                   ├── 24LC256 SCL
                   └── MAX17043 SCL
```

### Battery ADC (All ADC1 — no conflict with BT/WiFi)

```
Cell 1 divider output (100k/100k)   → GPIO34 (ADC1_CH6)
Cell 2 divider output (200k/100k)   → GPIO35 (ADC1_CH7)
Pack total divider (330k/100k)      → GPIO32 (ADC1_CH4)
NTC thermistor divider (10k/10k)    → GPIO33 (ADC1_CH5)
BMS charge status GPIO              → GPIO27
```

### User Interface GPIO

```
Volume encoder CLK  → GPIO18 (interrupt)
Volume encoder DT   → GPIO19
Volume encoder SW   → GPIO5 (push = power toggle)
Bass potentiometer  → GPIO36 (ADC1_CH0)
Treble potentiometer→ GPIO39 (ADC1_CH3)
Bluetooth button    → GPIO4 (interrupt, pairing)
Preset button       → GPIO2 (placement comp toggle)
WS2812B data        → GPIO13
AUX detect switch   → GPIO14
```

### ADAU1701 Control Pins

```
Pin 10 SELFBOOT: 10kΩ → 3.3V → HIGH = self-boot from EEPROM on power-on
Pin 24 XTALI: → 12.288MHz crystal + 22pF to GND
Pin 25 XTALO: → 12.288MHz crystal + 22pF to GND
MP12 (GPIO out): → RC delay → amp MUTE pin (pop prevention)
MP13 (GPIO in): → Placement button (10kΩ pull-up to 3.3V)
```

---

## Power Architecture

```
3× Samsung/LG 18650 3000mAh (3S1P pack)
  Nominal 11.1V | Full 12.6V | Cutoff 9.0V

→ [3S BMS 20A]
    Overcharge protection, over-discharge, overcurrent, cell balancing

→ [USB-C PD module]
    Charges to 12.6V CC/CV (correct 3S charging voltage)

B+ distribution:
  ├── TPA3116D2 woofer amp (direct 12V)  — 470µF + 100nF per VCC pin
  ├── TPA3110D2 tweeter amp (direct 12V) — 470µF + 100nF per VCC pin
  └── AMS1117-5V LDO
           ├── AMS1117-3.3V → ESP32-WROVER + ADAU1701 DVDD + IOVDD
           └── AMS1117-1.8V → ADAU1701 AVDD ONLY (dedicated, shared with NOTHING)

ADAU1701 decoupling per power pin:
  AVDD (Pins 1,2,3): 100nF + 10µF per pin
  DVDD (Pins 5,6):   100nF + 10µF per pin
  IOVDD (Pin 28):    100nF
  VREF (Pin 27):     1µF to GND — reference only, zero load

Power rules:
  AVDD LDO: dedicated — never shared
  AGND and DGND: star-join at ONE point near PSU entry only
  Class-D switching traces: NEVER under analog signal traces
  Bulk cap (470µF) at PCB 12V entry point
  No copper pour under ESP32 antenna area
```

### Realistic Battery Runtime

```
Power at 50% volume:
  Woofer amps (2× 8W average):  16W
  Tweeter amps (2× 3W average):  6W
  ESP32 + DSP + LEDs:            2W
  Total: ~24W at 50% volume

Battery usable capacity (80% depth of discharge):
  3× 3000mAh at 11.1V = 99.9Wh × 80% = ~80Wh

Runtime:
  20% volume: ~20–25 hours
  50% volume: ~3–4 hours  (lower than Marshall's 30hr — see note)
  80% volume: ~1.5–2 hours

Note: Marshall Middleton's 30hr claim likely uses a much larger pack
(possibly 10,000mAh+ equivalent). Our 3S 3000mAh pack is small.

If longer battery life needed:
  Option A: Use 6× 18650 in 3S2P (doubles capacity, same voltage)
  → Runtime: 40–50hr at 20%, 6–8hr at 50%
  → Adds weight and enclosure volume
```

---

## Complete BOM v3.0

### Core ICs

| # | Part | Qty | Cost (Rs) | Status |
|---|---|---|---|---|
| 1 | ADAU1701JSTZ-RL LQFP-48 | 1 | Have | 0.5mm pitch, hot air required |
| 2 | CY7C68013A FX2LP board | 1 | Have | freeUSBi USBi clone |
| 3 | ESP32-WROVER-E | 1 | 500 | PSRAM mandatory |
| 4 | 24LC256 EEPROM | 1 | 40 | DSP self-boot storage |
| 5 | 12.288 MHz crystal oscillator | 1 | 100 | WAS MISSING — CRITICAL |
| 6 | MAX17043 fuel gauge | 1 | 90 | Pack SOC percentage |
| 7 | AMS1117-3.3 LDO | 2 | 40 | ESP32 + DSP DVDD |
| 8 | AMS1117-1.8 LDO | 1 | 20 | DSP AVDD dedicated |
| 9 | AMS1117-5.0 LDO | 1 | 20 | Pre-regulator |

### Amplifiers

| # | Part | Qty | Cost (Rs) | Notes |
|---|---|---|---|---|
| 10 | TPA3116D2 module | 1 | 200 | 2×30W, drives woofers |
| 11 | TPA3110D2 module | 1 | 150 | 2×10W, drives tweeters |
| 12 | 100Ω + 10nF input filter | 4 sets | 20 | DAC to amp per channel |
| 13 | 8.2Ω + 100nF Zobel | 4 sets | 30 | Amp output per channel |
| 14 | 10MΩ + 47µF pop delay RC | 1 | 10 | Amp mute delay |

### Acoustic Drivers (T/S parameters mandatory before ordering)

| # | Driver | Spec Required | Qty | Cost (Rs) |
|---|---|---|---|---|
| 15 | Woofer | 3", 4Ω, Fs below 120Hz, Qts 0.4–0.6, T/S published | 2 | 600 |
| 16 | Tweeter | 3/4"–1" dome, 8Ω, Fs below 2kHz | 2 | 300 |
| 17 | Passive Radiator | 3–4", adjustable Mmd (accepts added weights) | 2 | 800 |

### Power System

| # | Part | Qty | Cost (Rs) |
|---|---|---|---|
| 18 | 18650 Li-ion 3000mAh Samsung/LG | 3 | 900 |
| 19 | 3S BMS 20A | 1 | 150 |
| 20 | USB-C PD charging module | 1 | 100 |
| 21 | 470µF 25V electrolytic caps | 2 | 40 |

### Battery Monitoring

| # | Part | Qty | Cost (Rs) |
|---|---|---|---|
| 22 | 100kΩ resistors | 6 | 10 |
| 23 | 200kΩ and 330kΩ resistors | 2 each | 10 |
| 24 | 100nF caps (ADC noise filter) | 4 | 10 |
| 25 | 10kΩ NTC thermistor | 1 | 20 |

### User Interface

| # | Part | Qty | Cost (Rs) |
|---|---|---|---|
| 26 | EC11 rotary encoder (push switch) | 1 | 30 |
| 27 | B10K rotary potentiometer | 2 | 40 |
| 28 | Tactile push buttons | 3 | 15 |
| 29 | WS2812B LED strip 10 LEDs | 1 | 80 |
| 30 | 3.5mm panel mount AUX jack | 1 | 30 |
| 31 | Panel mount USB-C connector | 1 | 50 |

### PCB and Enclosure

| # | Item | Cost (Rs) |
|---|---|---|
| 32 | Perfboard / veroboard for zero PCB testing | 150 |
| 33 | Custom 2-layer PCB JLCPCB 5 pieces | 600–900 |
| 34 | 12mm MDF for enclosure | 200 |
| 35 | Speaker grille fabric | 150 |
| 36 | Knob caps Marshall style | 100 |
| 37 | Passive components misc | 300 |

Total: approximately Rs 5,900 – 6,700

---

## Enclosure Design

```
External dimensions: 250 × 110 × 120mm (close to Middleton 230×98×110mm)
Material: 12mm MDF all walls
Internal acoustic volume: ~1.55L net per woofer (calculated from T/S)

FRONT FACE:
  [Tweeter L]  [Woofer L]              [Woofer R]
  1" dome      3" woofer               3" woofer

REAR FACE:
  [Passive Radiator 1]               [Tweeter R]  ← REAR FACING
  bass extension                      diagonal!

LEFT END:
  [Passive Radiator 2]

TOP PANEL:
  [Power+Volume] [Bass] [Treble] [BT] [Preset] [LED bar]

REAR BOTTOM:
  [USB-C charge] [AUX 3.5mm]

KEY POINT: Tweeter L fires FORWARD. Tweeter R fires REARWARD.
This diagonal physical arrangement creates True Stereophonic.
No DSP widening algorithm needed — purely physical.
```

---

## Development Roadmap

### Completed

- V1: Initial concept — component list
- V2: Architecture, first gap analysis, I2S clock fix, crystal added
- V3: Deep research — ESP32 memory, dual MCU decision, cell monitoring, acoustic layers, app design, zero PCB strategy

### In Progress

- V4: Zero PCB testing stages 1–4 (single vs dual MCU decision)
- V5: Firmware architecture based on testing results

### Not Started

- Source drivers with T/S parameters
- Speaker Box Lite enclosure simulation
- Custom PCB design in EasyEDA
- PCB order and assembly
- SigmaStudio DSP program (11 blocks)
- EEPROM write and self-boot verification
- Amplifier integration
- Acoustic measurement with REW
- Acoustic correction EQ from measurement
- Marshall signature EQ tuning
- Passive radiator mass iteration
- App development (BLE GATT + UI)
- A/B comparison vs real Marshall Middleton

---

## References

| Resource | Link | Purpose |
|---|---|---|
| ADAU1701 Datasheet | analog.com | DSP chip reference |
| SigmaStudio | analog.com | DSP graphical programming IDE |
| WillyBilly06 LDAC | github.com/WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC | LDAC on ESP32 |
| WillyBilly06 v2 | github.com/WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED | ESP-IDF 5.5 version |
| MCUdude SigmaDSP | github.com/MCUdude/SigmaDSP | Arduino I2C DSP control |
| freeUSBi | github.com/DatanoiseTV/freeUSBi | DIY USBi firmware |
| pschatzmann A2DP | github.com/pschatzmann/ESP32-A2DP | Alternative A2DP library |
| Speaker Box Lite | speakerboxlite.com | Enclosure and PR simulation |
| WinISD | winisd.com | Advanced enclosure simulation |
| REW | roomeqwizard.com | Acoustic measurement software |
| ADI AN-1006 | Analog Devices | Crossover design in SigmaDSP |
| ADI AN-1168 | Analog Devices | EQ filter design in SigmaDSP |
| ADI EngineerZone | ez.analog.com | ADAU1701 community forum |
| ESP-IDF RAM Guide | Espressif docs | ESP32 memory optimization |
| Infineon AN-1135 | infineon.com | Class-D amplifier PCB layout |

---

## Author

Ghost 

"Marshall-level sound from raw chips. Not because it is easy — because it is the right way to build it."

Last updated: May 2026 — V3 design and deep research complete. Zero PCB testing phase upcoming.

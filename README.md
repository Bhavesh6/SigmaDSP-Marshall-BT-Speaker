# SigmaDSP-Marshall-BT-Speaker

> **A fully engineered DIY portable Bluetooth speaker built from raw chips — ADAU1701 SigmaDSP, ESP32-WROVER with LDAC and lossless WiFi/microSD sources, 4-channel Class-D amplification, active crossover, and live tuning from a phone.**

[![Status](https://img.shields.io/badge/Status-V4%20Architecture%20Locked-yellow?style=for-the-badge)]()
[![Hardware](https://img.shields.io/badge/Hardware-Not%20Started-red?style=for-the-badge)]()
[![DSP](https://img.shields.io/badge/DSP-ADAU1701%20SigmaDSP-blue?style=for-the-badge)]()
[![BT](https://img.shields.io/badge/Bluetooth-LDAC%20%7C%20aptX%20%7C%20AAC%20%7C%20SBC-green?style=for-the-badge)]()
[![MCU](https://img.shields.io/badge/MCU-ESP32--WROVER--IE-orange?style=for-the-badge)]()
[![License](https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge)]()

---

## Project Status

```
V1 — Initial Concept                        COMPLETE
V2 — Architecture + Gap Analysis R1         COMPLETE
V3 — Deep Research + All New Challenges     COMPLETE   (archived: docs/README_v3.md)
V4 — Architecture Locked Against Rev. C     COMPLETE   <-- this document
V5 — Firmware Planning                      NEXT
--- HARDWARE NOT STARTED ---
Stage 0  SigmaStudio program, no hardware   IN PROGRESS
Stage 1  Power rails verified               NOT STARTED
Stage 2  ADAU1701 alive standalone          NOT STARTED
Stage 3  SigmaStudio over WiFi (TCPi)       NOT STARTED
Stage 4  Tone out of VOUT0                  NOT STARTED
Stage 5  EEPROM self-boot                   NOT STARTED
Stage 6  ESP32 I2S clock + real audio       NOT STARTED
Final    All-in-one custom PCB              NOT STARTED
Acoustic Enclosure + tuning                 NOT STARTED
App      BLE GATT control                   NOT STARTED
```

> This repository is the complete living design document of every decision made, every problem found, and every solution planned — before a single component is soldered. Hardware will be committed here as it is built and tested.

**Supporting documents**

| File | What it holds |
|---|---|
| `docs/pre-build-research-report.md` | The final pre-build research report. Primary source for V4 — ADI quotes, the Elektor board confirmation, instruction budgets, amp measurements, verified pitfalls, and the confidence caveats on every claim |
| `docs/README_v3.md` | V1–V3 history, BLE GATT app design, cell-monitoring circuit, full REW measurement protocol |

---

## What V4 Changed

V4 is not a refinement of V3. Working from the **Rev. C datasheet** instead of the 2006 preliminary, and from documented ESP32 I2S slave-mode failures, six things in V3 turned out to be wrong — two of them badly enough to destroy the chip or produce permanent silence.

| # | V3 said | V4 says | Severity if built as V3 |
|---|---|---|---|
| 1 | AVDD = 1.8 V, DVDD = 3.3 V | **AVDD = 3.3 V, DVDD = 1.8 V** | Chip destroyed instantly (DVDD abs max 2.2 V) |
| 2 | ADAU1701 is I2S master, ESP32 slave | **ESP32 is I2S master, BCLK feeds MCLKI** | No audio — ESP32 slave mode FIFO-underruns |
| 3 | Split AGND / DGND, star point | **One single ground plane** | Worse noise, not better |
| 4 | Crystal mandatory | **Optional, on a jumper, bench use only** | Not fatal — but debugging becomes blind |
| 5 | Pin numbers from the preliminary datasheet | **Every pin number corrected** | Miswired board |
| 6 | Passive radiators, no high-pass | **Mandatory excursion-protecting HPF + limiter** | Cone bottoming below tuning |

Plus five things V3 never mentioned at all: **PVDD/PGND**, the **PLL loop filter**, **DAC Setup register 0x0827**, the **2 mA / 0.6 V GPIO limit**, and **MP pins floating HIGH at power-up**.

---

## Corrected System Architecture

```
  Phone / PC / NAS
        |
        +-- Classic BT A2DP (LDAC / aptX HD / aptX / aptX-LL / AAC / SBC)
        +-- WiFi: AirPlay (ALAC) / Squeezelite (FLAC) / DLNA      <- LOSSLESS
        +-- BLE GATT  (app control, runs alongside any ONE audio source)
        |
   [ESP32-WROVER-IE-N16R8]  external IPEX antenna
        |  I2S MASTER, FIXED 48 kHz, 24-bit  -- never 44.1 kHz
        |  BCLK 3.072 MHz ---+--> ADAU1701 MP5   (INPUT_BCLK, pin 9)
        |                    +--> ADAU1701 MCLKI (pin 32)   <- same wire, 64 x fs
        |  LRCLK 48 kHz  -------> ADAU1701 MP4   (INPUT_LRCLK, pin 8)
        |  DATA          -------> ADAU1701 MP0   (SDATA_IN0, pin 11)
        |
        +-- I2C --> ADAU1701 0x34, 24LC256 0x50, MAX17043 0x36, optional OLED
        +-- SPI --> microSD (local FLAC/WAV)
        +-- ADC1 -> cell voltages, pack total, NTC
        |
   [ADAU1701 SigmaDSP]   AVDD 3.3 V / DVDD 1.8 V (internal reg) / PVDD 3.3 V
        |  DC-block, volume, HPF, BSC, PEQ, LR4 @ 4 kHz, trims, limiters
        |
        +-- VOUT0 (46) --> filter --> TPA3116D2 ch1 @ 20 dB --> Woofer L
        +-- VOUT1 (45) --> filter --> TPA3116D2 ch2 @ 20 dB --> Woofer R
        +-- VOUT2 (44) --> filter --> TPA3110D2 ch1 @ 20 dB --> Tweeter L
        +-- VOUT3 (43) --> filter --> TPA3110D2 ch2 @ 20 dB --> Tweeter R
        |
        +-- ADC0 (2) / ADC1 (4) <-- 3.5 mm AUX jack (analog, lossless path)

   Programming: ESP32 running the TCPi bridge --> SigmaStudio over WiFi
                (FX2LP + freeUSBi kept as backup only)
```

---

## 1. Clock Architecture — The Decision That Unblocks Everything

### Why V3's fix was a dead end

V3 corrected V1 by making the ADAU1701 the I2S master and the ESP32 the slave. That removes the DSP's mute condition but creates a worse one:

- The ESP32's I2S block clocks its state machine from an **internal** clock. When that internal clock runs faster than an externally supplied MCLK, the FIFO underruns. Espressif's own position is that the codec must be the slave and the ESP32 the master.
- Builders attempting exactly this topology report fragmented noise on the data line with APLL both on and off, and in several cases no data signal at all.
- Clemens Valens, author of the Elektor Audio DSP FX Processor (ESP32-PICO + ADAU1701, Elektor 358): *"there is only one solution because the ESP32 software libraries do not (yet) support I²S slave mode. Therefore, the ESP32 must be the master."*

So both of these are true at once:

1. The ADAU1701 mutes every output if the serial-port clocks are not synchronous with its master clock.
2. The ESP32 cannot be trusted as an I2S slave.

### The V4 solution — make BCLK *be* MCLK

Setting **PLL_MODE0 = 0 and PLL_MODE1 = 0** puts the ADAU1701 in **64 × fs** mode, which is exactly the BCLK frequency for stereo I2S. One wire serves as both. An ADI applications engineer on EngineerZone: *"Connecting the 3.072 MHz clock to both BCLK and MCLK would be the best solution. In this configuration, please set the PLL to 64 × fs by setting PLL_MODE0 and PLL_MODE1 both to GND, and also take into account the increased amount of time required for PLL lock."*

```
ESP32 (I2S MASTER)
  BCLK  3.072 MHz ---+---> ADAU1701 MP5   (INPUT_BCLK, pin 9)
                     +---> ADAU1701 MCLKI (pin 32)
  LRCLK 48 kHz    -------> ADAU1701 MP4   (INPUT_LRCLK, pin 8)
  DATA            -------> ADAU1701 MP0   (SDATA_IN0, pin 11)

PLL_MODE0 (pin 38) -> GND
PLL_MODE1 (pin 39) -> GND

MCLK and BCLK are the SAME signal, therefore synchronous by construction.
The DSP never mutes, and the ESP32 never leaves master mode.
```

The MP10 → MP5 and MP11 → MP4 loopback traces specified in V3 are **deleted**.

The same ADI engineer states the underlying rule: *"the MCLK and BCLK/LRCLK must be synchronous, but not necessarily phase aligned. If there are crossing edges between MCLK and BCLK, there will be some unpredictable audio artifact."*

### The four rules that come with 64 × fs mode

**1. Force 48 kHz everywhere. Never 44.1 kHz.**
3.072 MHz is the *lowest* MCLKI in the datasheet's PLL table. A 44.1 kHz source makes BCLK 2.8224 MHz — about 8% below the nearest listed lock point. It is inside the PLL's nominal ±20% window and ADI expressed confidence it still works, but the safe design is to resample every source (BT, WiFi, SD) to a fixed 48 kHz so the DSP only ever sees 3.072 MHz.

**2. The clock must never stop — including in AUX mode.**
ADI: *"the I2S must always be present and it cannot stop even if you are only using the analog ADC input."* When the speaker is playing analog audio through the ADAU1701's own ADCs, the ESP32 must keep its I2S master clock running or the DSP mutes.

**3. Budget 260 ms before audio is valid.**
Rev. C: *"The PLL start-up time lasts for 2^18 cycles of the clock on the MCLKI pin. This time ranges from 10.7 ms for a 24.576 MHz (512 × fS) input clock to 85.3 ms for a 3.072 MHz (64 × fS) input clock."* 64 × fs is the slowest mode by a wide margin.

```
Hold ADAU1701 in RESET until the ESP32 I2S clock is confirmed running
Release reset -> 85.3 ms PLL lock -> boot cycle -> ~260 ms total
Keep the amps muted for at least 300 ms after reset to kill the boot pop
Self-boot from EEPROM also waits for MCLK before it initialises
```

**4. Do not use GPIO0's APLL MCLK.**
A widely-cited build log measured an ESP32's GPIO0 APLL-derived MCLK output jumping between roughly 7 MHz and 13 MHz. That is the failure mode this design sidesteps: BCLK is a cleanly divided clock, not an APLL output. This is a concrete advantage of BCLK-as-MCLK over the "separate MCLK on GPIO0" alternative.

### Crystal: fit it, but understand what it is for

The crystal is not required for normal operation, and it is **not a fallback for the live digital path**:

```
Crystal on MCLKI = 256 x fs, while the ESP32 is still the I2S master
  -> MCLK and BCLK are now ASYNC
  -> the DSP mutes the digital path

So the crystal position is for:
  - standalone bench bring-up (analog AUX in -> DSP -> DAC out)
  - isolating "is the chip alive?" from "is the I2S config right?"
It is NOT a runtime fallback for Bluetooth or I2S audio.
```

That isolation is worth ₹100 on a first build:

```
WITH crystal (jumper in XTAL position):
  Power the DSP alone -> load a tone generator over TCPi -> scope VOUT0
  Tone present? Chip alive, solder good, rails right, PLL locking.
  THEN flip the jumper to the ESP32 clock and debug I2S as a separate problem.

WITHOUT crystal:
  Power up -> silence -> soldering? rails? I2S clock? PLL mode? program?
  Five unknowns at once.
```

The Elektor board ships exactly this jumper — **JP1 pins 1&2 = crystal X1, pins 2&3 = ESP32 clock** — and its documentation says to short pins 2 & 3 whenever I2S is in use. Copy that.

**Crystal spec matters:** 12.288 MHz, **AT-cut, parallel resonance, fundamental mode** (Rev. C: *"the oscillator circuit should be an AT-cut, parallel resonator operating at its fundamental frequency"*). A 24 MHz part is third-overtone and will not oscillate. Reference part: Abracon ABLS-12.288MHZ-B4-T. Circuit per Rev. C Figure 16: crystal between MCLKI (32) and OSCO (31), a **100 Ω** series damping resistor, **22 pF** to ground each side, traces as short as physically possible.

### Rejected: 512 × fs / 24.576 MHz

Faster lock (10.7 ms) and it would enable 96 kHz — but it requires the ESP32 to generate a stable 24.576 MHz, which reintroduces the GPIO0 APLL jitter problem, and it halves the instruction budget to 512. **Stay at 48 kHz / 64 × fs.**

---

## 2. Power Rails — The Error That Would Have Killed the Chip

V3 specified AVDD = 1.8 V and DVDD = 3.3 V. **Rev. C Table 1 says the opposite**, and the Absolute Maximum Ratings list **DVDD to GND: 2.2 V max**. Building V3's power section would have put 3.3 V onto a 1.8 V pin and destroyed the part on first power-up.

```
CORRECT:
  AVDD  (pins 36, 48)  = 3.3 V
  IOVDD (pin 18)       = 3.3 V   <- verify this trace exists; a documented
                                    build failed with bizarre GPIO behaviour
                                    and no audio because it was missing
  PVDD  (pin 34)       = 3.3 V   <- PLL supply, missing from V3 entirely
  DVDD  (pins 13, 24)  = 1.8 V   <- from the chip's INTERNAL regulator
  PGND  (pin 33)       = GND     <- missing from V3 entirely
```

### There is no 1.8 V LDO in the BOM any more

The ADAU1701 has a built-in 1.8 V regulator. You only supply the pass transistor:

```
3.3 V --+-- PNP emitter (2N3906 or FZT953, hFE >= 100)
        |
      [1 kOhm]
        |
  VDRIVE (pin 17) --- PNP base

  PNP collector --+--> DVDD (pins 13, 24)
                  +--> 10 uF bulk (one, shared)
                  +--> 100 nF at EACH DVDD pin

Dissipation: (3.3 - 1.8) x 60 mA = 90 mW. A SOT-23 handles it.
If the internal regulator is not used, VDRIVE must be tied to ground.
```

**Removed from BOM:** AMS1117-1.8. **Added:** 2N3906 ×2 (₹10), 1 kΩ.

### PVDD is the cleanest rail on the board

The PLL's VCO is referenced to PVDD. Any AC on that rail modulates the VCO, the PLL struggles to hold lock, MCLK jitters — and because BCLK is derived from MCLK, the jitter propagates into the audio clock.

```
Main 3.3 V --[ferrite bead]--+-- 10 uF --+-- 100 nF --+--> PVDD (34)
                                                       |
                                                      PGND (33)
```

### PLL loop filter — without it the PLL never locks

Rev. C Figure 17, on PLL_LF (pin 35):

```
3.3 V (from AVDD) --[475 Ohm]--+--> PLL_LF (pin 35)
                               |
                            3.3 nF        56 nF --> GND

Tolerances: 10% resistor, 20% caps. Not critical, but the parts must be there.
```

### Grounding — V3 had it backwards

V3 specified split AGND / DGND copper zones joined at a star point. Rev. C says:

> "A single ground plane should be used in the application layout."
> "The AGND, DGND, and PGND pins can be tied directly together in a common ground plane."

**Use one ground plane.** Separate analog and digital by *placement*, not by cutting copper. ADI designed the chip; follow ADI.

### Reference decoupling

```
CM    (pin 40) -> 47 uF to GND    (reduces ADC/DAC crosstalk)
FILTD (pin 41) -> 10 uF to GND
FILTA (pin 47) -> 10 uF to GND
100 nF at every single power pin, as close as the layout allows
```

---

## 3. Complete Pin Map (ADAU1701, Rev. C verified)

Every pin number in V3 came from the 2006 preliminary datasheet and was wrong. This table is from Rev. C and is authoritative. **Cross-check against the Rev. C pin-function table only — not timing tables, not the preliminary.**

| Function | Pin | Notes |
|---|---|---|
| ADC0 | 2 | Swapped vs preliminary — wiring from the old sheet reverses L/R |
| ADC_RES | 3 | 18 kΩ to GND, 1% |
| ADC1 | 4 | |
| RESETB | 5 | Active low, to an ESP32 GPIO |
| SELFBOOT | 6 | 10 kΩ to 3.3 V = boot from EEPROM |
| ADDR0 | 7 | GND |
| MP4 / INPUT_LRCLK | 8 | from ESP32 LRCLK |
| MP5 / INPUT_BCLK | 9 | from ESP32 BCLK |
| MP0 / SDATA_IN0 | 11 | from ESP32 DATA |
| DGND | 12, 25 | |
| DVDD | 13, 24 | **1.8 V**, 100 nF each |
| MP10 / OUTPUT_LRCLK | 16 | unused in V4 |
| VDRIVE | 17 | 1 kΩ to 3.3 V + PNP base |
| IOVDD | 18 | 3.3 V — do not forget this trace |
| MP11 / OUTPUT_BCLK | 19 | unused in V4 |
| ADDR1 | 20 | GND |
| WP | 21 | 10 kΩ to 3.3 V. **Low ENABLES writes** (preliminary said the opposite) |
| SDA | 22 | 2.2 kΩ pull-up |
| SCL | 23 | 2.2 kΩ pull-up |
| RSVD | 30 | **Tie to GND.** Easy to miss, not optional |
| OSCO | 31 | crystal + 100 Ω, 22 pF |
| MCLKI | 32 | ESP32 BCLK (or crystal). MCLKI range 3–25 MHz |
| PGND | 33 | |
| PVDD | 34 | 3.3 V via ferrite |
| PLL_LF | 35 | 475 Ω / 3.3 nF / 56 nF |
| AVDD | 36, 48 | **3.3 V**, 100 nF + 10 µF each |
| AGND | 1, 37, 42 | |
| PLL_MODE0 | 38 | **GND** |
| PLL_MODE1 | 39 | **GND** |
| CM | 40 | 47 µF to GND |
| FILTD | 41 | 10 µF to GND |
| VOUT3 | 43 | Tweeter R |
| VOUT2 | 44 | Tweeter L |
| VOUT1 | 45 | Woofer R |
| VOUT0 | 46 | Woofer L |
| FILTA | 47 | 10 µF to GND |

Other Rev. C facts that change the design:

| Item | Value | V3 said |
|---|---|---|
| I2C pull-ups | **2.2 kΩ** | 4.7 kΩ |
| I2C address | 0x34 (7-bit) / 0x68 write | correct |
| EEPROM self-boot address | **0xA0 write / 0xA1 read** (7-bit 0x50) | 0x60 / 0x61 — self-boot would never work |
| Program limit | **1019 usable** of 1024 (5 reserved for safeload) | 1024 |
| Safeload registers | 0x0810–0x0819 (data + address), IST bit in 0x081C | not documented |
| Operating temperature | **0 °C to 70 °C only** | never mentioned |
| GPIO drive | **2 mA** | 5 mA |
| Max delay memory | ~43 ms across all blocks | correct |
| End-to-end dynamic range | 98.5 dB | ">100 dB" |

**Known typo in Rev. C itself:** the Digital Timing table calls OUTPUT_BCLK "Pin 11". The pin function table is correct — MP11 / OUTPUT_BCLK is **pin 19**; pin 11 is MP0 / SDATA_IN.

---

## 4. Mandatory Initialization Sequence

Rev. C adds a Control Registers Setup section and a **register that does not exist in the preliminary datasheet at all**. Skip this and the outputs stay muted — one of the most common first-boot failures on this chip.

```
1. Bring up 3.3 V. Confirm the ESP32 I2S clock is running.
2. Release RESETB
3. Wait for PLL lock  (85.3 ms in 64 x fs mode)
4. Load the SigmaDSP program and parameters
5. DSP Core Control, register 2076 (0x081C): set ADM, DAM, CR (bits 4:2) = 1
6. DAC Setup, register 2087 (0x0827): set DS[1:0] = 01      <- NEW, mandatory
7. Release the amp mute  (>= 300 ms after reset)
```

The SigmaDSP holds no non-volatile memory — the entire program must be loaded into RAM on every boot, from EEPROM self-boot or from the MCU.

Also: the ADAU1701 **cannot be taken out of SPI mode without a full reset**. Once it latches SPI, only RESETB brings it back to I2C.

---

## 5. Analog Interfaces — Corrected Component Values

### DAC output filter (Rev. C Figure 18) — ×4

V3 specified 100 Ω + 10 nF. That is not ADI's filter.

```
VOUTn --[47 uF]--[560 Ohm]--+--> amp input
                            |
                          5.6 nF
                            |
                           GND        ~50 kHz corner

Full scale: 0.9 Vrms (2.5 Vpp). Keep the output loaded >= 2 kOhm.
The DACs are INVERTING -- invert in SigmaStudio or account for it.
```

### DAC to amplifier input

Do not feed raw 0.9 Vrms into a module's input. A series R+C into the module forms a high-pass with its input impedance — target **20–30 Hz**. ADI's CN-0162 reference design (ADAU1701 → SSM2306) uses 0.10 µF + 13.0 kΩ for a 28 Hz corner; scale to whatever the chosen module's input impedance actually is. At 20 dB gain a TPA3116's input impedance is ~60 kΩ, so ~1.5 µF coupling caps are sufficient.

### AUX input into the ADAU1701's own ADCs

The chip has two 24-bit Σ-Δ ADCs that V3 never used. This is the wired lossless path and it costs about ₹20 in passives.

```
Jack L --[47 uF]--[7 kOhm 1%]--> ADC0 (pin 2)
Jack R --[47 uF]--[7 kOhm 1%]--> ADC1 (pin 4)
                  [18 kOhm 1%]--> ADC_RES (pin 3) --> GND
```

The ADC inputs are **current** inputs (100 µA rms full scale), which is why the series resistor sets the range. Rev. C Table 13:

| Full-scale input | ADC_RES | ADC0/ADC1 series R |
|---|---|---|
| **0.9 Vrms** | 18 kΩ | **7 kΩ** — matches DAC full scale exactly |
| 1.0 Vrms | 18 kΩ | 8 kΩ |
| 2.0 Vrms | 18 kΩ | 18 kΩ |

Use the 0.9 V row, 1% tolerance on all three. The ADCs have **no built-in high-pass**, so they show a DC offset — put a DC-blocking high-pass at the top of the DSP chain or you get pops.

---

## 6. Logic-Level Constraints

Three separate limits, all easy to trip over:

```
1. GPIO drive is 2 mA (Rev. C, halved from the preliminary's 5 mA).
   Rev. C explicitly warns against driving many LEDs directly from MPx pins.
   Even one ordinary LED needs a transistor or logic-level MOSFET.

2. The MP pins only pull down to about 0.6 V.
   Downstream logic (MERUS/MA12070 amp enable among others) does not
   reliably read 0.6 V as a valid low.
   -> The amp MUTE/ENABLE line must go through a MOSFET (2N7002) or a
      74LVC1G17 buffer. Never drive it directly.
   -> Our TPA3116/TPA3110 audio inputs are analog, so the audio path
      itself is unaffected.

3. MP pins power up as INPUTS and float HIGH until the program loads
   and reconfigures them.
   -> 47 kOhm pull-down (or pull-up, for the wanted idle state) on any
      MP pin that drives external logic.
   -> Prefer active-high control wherever the external part allows it.
```

---

## 7. Controller Decision — Single Original ESP32

This was the open question V4 was supposed to answer. It is answered, and not by testing — by elimination.

### The hard rule

**Only the original ESP32 has Classic Bluetooth (BR/EDR).** S2, S3, C3, C6, H2 and P4 are BLE-only. A2DP — and therefore LDAC, aptX and AAC — requires BR/EDR. This single fact eliminated most of the field.

### Everything evaluated and why it lost

| Candidate | Verdict |
|---|---|
| ESP32-S3-WROOM-1 N16R8 | Better in every spec except the one that matters — no Classic BT, confirmed by Espressif. Excellent *control-plane* MCU if we ever go dual |
| ESP32-C3 / C6 / H2 / S2 | BLE only |
| Ai-Thinker PB-03 (PHY6252) | BLE 5.2 only. Good, cheap secondary MCU (₹180) — nothing more |
| nRF52820 | BLE only, 32 KB RAM, no I2S, comparator instead of ADC, bare QFN, needs a J-Link |
| Bouffalo BL618 / AiPi-SCP-2.4 | BLE only in practice; Classic BT is in the marketing, not in the SDK |
| CSR8675 (BTM875 / PA214) | Native certified LDAC + I2S out, genuinely good — but locked firmware, no I2C control, no BLE, no WiFi, no SD. Needs an ESP32 next to it anyway |
| QCC3034 / QCC5125 boards | No LDAC (Qualcomm never licenses it). "Lossless decoding" in the listings is false |
| CSR8645 | BT 4.0, no LDAC, I2S output effectively unobtainable with stock firmware |
| ZK-TB21 / MT21 / ST21 | Duplicates the whole project — built-in BT + tone controls + 2.1 topology. Useful only as a borrowed bench amp |
| Raspberry Pi Zero 2 W | Everything works natively via PipeWire, including LDAC and AirPlay. Killed by ~20 s boot and ~8 h battery — both fatal for a portable. Worth having on the bench as a fallback tool |
| TAS5805M / TAS5825M | Better *architecture* — but no safeload equivalent, see below |
| ESP32-S31 | The dream chip. Announced, not shipping: 0 distributor stock, 21-week lead. RISC-V, so the LDAC firmware would need porting. Its Classic BT (+EDR) is claimed in the announcement but **not confirmed in the distributor parametric data** — verify against the datasheet before counting on it |

### Why the TAS5825M was rejected despite being the better chip

It removes about 30 components, needs no MCLK, keeps the signal digital all the way to the speaker terminals, and protects driver excursion in hardware. But its DSP is programmed by pasting a PPC3-generated **whole-register blob** into firmware. TI's own engineer recommends against hand-writing individual settings. There is no clean live-parameter path.

```
ADAU1701 safeload: drag a slider in the app
  -> BLE -> ESP32 -> 5 coefficients over I2C -> new sound in <50 ms, no pop

TAS5825M: drag a slider
  -> open PPC3 on a Windows PC -> regenerate blob -> recompile -> reflash
```

This project is not "a speaker". It is "a speaker I can retune from my phone." Safeload is the hardware feature that makes that possible. **The TAS5825M is the right chip for a V2 custom PCB; the ADAU1701 is the right chip for this one.**

### Final module choice

```
FINAL:      ESP32-WROVER-IE-N16R8
            16 MB flash, 8 MB PSRAM, IPEX/U.FL external antenna, ECO V3 silicon

PROTOTYPE:  7Semi ESP32-DevKitC WROVER (16 MB / 8 MB / IPEX)
            USB, CP2102N, BOOT/EN buttons, 2.54 mm headers, breadboardable

RIGHT NOW:  any ESP32 already on the bench is fine for Stages 1-6.
            PSRAM is only needed once LDAC decoding starts.

NEVER:      any WROOM (no PSRAM), any R2 variant (EOL, 2 MB),
            any S3/C3/C6 (no Classic Bluetooth)
```

**Why the external antenna is not optional polish:** the module sits inside a sealed MDF box with four neodymium driver magnets, metal PR frames, cells, and Class-D amps switching at 400 kHz. A PCB antenna in that environment will drop WiFi audio. Route a U.FL pigtail to a rear-panel SMA antenna (₹140 total), away from magnets and switching nodes.

**PSRAM reality check:** the ESP32 can only map 4 MB of external RAM into its address space; the upper 4 MB of an R8 module needs the `himem` API. Internal DRAM is the scarce resource, not PSRAM — Classic BT alone takes ~100 KB. Rough live-stack budget: **BLE ~50 KB, Classic BT ~100 KB, WiFi ~70 KB — do not hold all three.** Use the PSRAM branch of the A2DP firmware so audio buffers live in PSRAM.

**Buying warning for India:** insist on the full part number printed on the shield — `ESP32-WROVER-IE-N16R8`. Boards sold as "4 MB" have been found containing 2 MB flash, which produces cryptic partition errors.

### Radio coexistence — the V3 "must test" item, resolved

| Combination | Works | Notes |
|---|---|---|
| A2DP + BLE GATT | Yes | Set `ESP_BT_MODE_BTDM`. This is the app-control case |
| WiFi + BLE GATT | Yes | Time-sliced, well tested |
| WiFi audio + BLE GATT | Yes | No Classic BT involved |
| **A2DP + WiFi** | **No — do not design for it** | Espressif: *"BR/EDR and WIFI coexistence performance is in optimizing on ESP32."* Users report massive WiFi packet loss and disassociation during A2DP |
| A2DP + WiFi audio | Irrelevant | Two audio sources at once — never happens |

Every scenario this speaker actually enters is supported, because **source selection is mutually exclusive by design**: Bluetooth **or** WiFi **or** microSD **or** AUX, with BLE control alongside whichever is active.

Coexistence tuning that helps where it is needed: pin the BT and WiFi controllers to different cores (`CONFIG_BTDM_CTRL_PINNED_TO_CORE_CHOICE`, `CONFIG_ESP_WIFI_TASK_CORE_ID`), enable software coexistence, set Core Debug Level to None, and enlarge the I2S DMA buffer to ride out RF gaps.

If Stage 9 testing does show A2DP + BLE stuttering, the fix is a PB-03 on UART (₹180) as a dedicated BLE radio. **Do not design for it up front** — leave four spare pads for a UART header.

---

## 8. Source Architecture — Lossless Added

No Bluetooth codec is lossless, LDAC included (990 kbps vs 1411 kbps for CD). If lossless matters, it has to arrive by another path — and the WROVER already has the hardware for three of them.

| Rank | Path | Rate | Lossless | Extra hardware |
|---|---|---|---|---|
| 1 | microSD FLAC/WAV | 1411 kbps | Yes | microSD module ₹80 + card |
| 2 | WiFi AirPlay (ALAC) / Squeezelite (FLAC) / DLNA | 1411 kbps | Yes | none |
| 3 | AUX 3.5 mm into the ADAU1701 ADCs | analog | Yes* | ~₹20 passives |
| 4 | Bluetooth LDAC | 990 kbps | No | none |
| 5 | Bluetooth SBC | 328 kbps | No | none |

\* no digital compression; quality depends on the source device's DAC.

**microSD playback is both the best quality and the longest runtime** — the radio is off entirely.

### The firmware fork in the road

The two candidate stacks **cannot be merged into one binary**:

- **WillyBilly06 `ESP32-A2DP-SINK-WITH-CODECS-UPDATED`** — ESP-IDF 5.5.2, patched Bluetooth stack, LDAC / aptX HD / aptX / aptX-LL / Opus / AAC / SBC, LDAC to 96 kHz/24-bit. Has a PSRAM branch (WROVER) and an internal-SRAM branch (WROOM). Bundles BLE GATT, DSP, level meters and WS2812B effects — closest to this project's feature set. Needs modifying to force fixed 48 kHz I2S master output. *(The older `esp32-a2dp-sink-with-LDAC-APTX-AAC` repo is superseded — its own README points here.)*
- **sle118/squeezelite-esp32** — AirPlay, Squeezelite/LMS, Spotify Connect, BT sink, multi-room, display and encoder support. But it *is* the firmware, with one active mode at a time, and its own foreword warns that everything other than LMS playback is "stitched on".

**V5 must pick one as the primary stack.** Current lean: WillyBilly06 for Bluetooth-first with SD playback added via `schreibfaul1/ESP32-audioI2S`, which conveniently **always outputs 48 kHz regardless of source** — exactly what a fixed-rate ADAU1701 needs. AirPlay would then be the feature deferred, or handled by a later firmware swap. (`rbouteiller/airplay-esp32` is a newer AirPlay-2 option but is oriented at the TAS5825M.)

Practical file-format note: 16/44.1 FLAC decodes comfortably on the ESP32; 24/96 is at the edge and stutters. Target 16–24 bit, 44.1–48 kHz files.

---

## 9. Amplifiers

### Power reality at 12 V

The "50 W" on a TPA3116D2 module assumes 21–24 V. At 12 V into 4 Ω the real figure is **~17 W/ch at usable THD** (~22 Vpp swing); TI's own 25 W number is at 14.4 V and 10% THD. Into 8 Ω at 12 V it is nearer 10 W, and TI state you must push past 1% THD to get there. That is still fine — the drivers are Xmax-limited long before the amps clip.

### The gain setting matters more than the chip

```
ADAU1701 DAC full scale: 0.9 Vrms -- a strong signal.
Stock TPA3116 modules ship at 32-36 dB gain (loudest sells best).

At 36 dB (x63):  0.9 Vrms x 63 = 56 Vrms demanded
                 a 12 V rail delivers about 4 Vrms
                 -> you use the bottom 7% of the volume range
                 -> and you amplify the noise floor by 63x

At 20 dB (x10):  0.9 Vrms x 10 = 9 Vrms
                 -> matched to the rail
                 -> 16 dB less amplified hiss
                 -> TI's own low-noise reference point:
                    65 uV (-80 dBV) A-weighted, SNR 102 dB
```

**Set every amp board to 20 dB gain.** On common boards that is two SMD resistors next to the chip: many are 36 dB via 75 kΩ ("753") + 47 kΩ, and removing the 75 kΩ parts gives 20 dB master mode. TI's clean 20 dB is a single 5.6 kΩ gain resistor with the second removed. **Identify the actual gain resistors on your specific board against the TI datasheet before removing anything** — clone designators differ (R1/R2 vs R2/R3).

### Module surgery checklist

1. Gain to 20 dB. All modules as master, identical resistors.
2. Bypass or remove the onboard volume pot — the DSP does volume digitally. If it must stay, fit ~4.7 kΩ across the pot output; builders report the hiss becoming "barely noticeable".
3. Replace the input coupling caps with matched film caps (~1.5 µF at 20 dB) — matched L/R values also reduce turn-on pop.
4. Feed the module through our own series R+C (~20–30 Hz corner), not raw DAC output.
5. Add Zobel networks on the outputs (8.2 Ω + 100 nF per channel) — cheap modules omit them.
6. Check the output inductors: 10–22 µH shielded. Tiny unshielded coils — or missing inductors entirely — are the classic shortcut.
7. One thick ground wire from module GND to the board's star point. Multiple ground paths are where hum comes from.
8. 1000–2200 µF low-ESR bulk at each amp's VCC — this is what supplies the 4 A bass transients, not the battery.

### Chips considered

| Amp | @12 V, 4 Ω | Idle | India stock | Verdict |
|---|---|---|---|---|
| TPA3116D2 | ~17 W | ~40 mA | Modules easy, chips via LCSC | **Woofers** |
| TPA3110D2 | ~10 W (8 Ω) | ~30 mA | Easy | **Tweeters** |
| TPA3156D2 | ~15 W | <23 mA | Hard | Best on paper — adaptive modulation, programmable power limit (kills battery sag), master/slave sync, fault reporting. Unavailable locally |
| MA12070 (MERUS) | ~14 W | 52 mW | Hard | Best noise floor, used by decaVox. QFN 6×6, hard to source and solder, and the 0.6 V logic-low issue originates here |
| TPA3255 | ~15 W | high | Easy | Needs 24 V+, pointless here |
| PAM8610 / PAM8403 | ~10 W | high | Easy | Too noisy, no gain control |

TPA3110D2 / TPA3118D2 / TPA3116D2 / TPA3156D2 **share the HTSSOP-32 footprint** — design the PCB once and populate whatever is available.

### The amp is not the bottleneck

```
TPA3116D2 THD:            0.03 - 0.1%
PC83-4 driver THD @ 85dB:    1 - 3%
```

The driver distorts 10–100× more than the amplifier. The correct response is not a better amp — it is to **measure acoustically, with a microphone in front of the speaker**, and build the correction EQ from that measurement. One measurement covers the DSP, the amp's output LC filter interacting with the driver's non-flat impedance, the driver, the box and the baffle. Frequency-domain errors from *any* source get corrected together. Only noise floor, distortion and clipping cannot be fixed this way.

---

## 10. Drivers — Locked

```
WOOFERS:   2x Dayton Audio PC83-4     3", 4 ohm
TWEETERS:  2x Dayton Audio ND16FA-4   5/8" soft dome, 4 ohm
CROSSOVER: LR4 @ 4.0 kHz
TWEETER TRIM: -6 dB
Both from diyaudiocart.com -- one order, matched impedance
```

> The research report predates the tweeter decision and specifies the HiVi T20-8 at
> −2.2 dB trim. The ND16FA-4 replaced it afterwards; its 93 dB sensitivity is why the
> trim is now −6 dB. Everything else in the report's crossover section still applies.

### Dayton PC83-4 (woofer)

| Parameter | Value |
|---|---|
| Fs | 80.1 Hz |
| Qts | 0.54 |
| Vas | 1.98 L |
| Xmax | **2.0 mm** — the binding constraint |
| Sd | 30.2 cm² |
| Vd | **6.0 cm³** |
| Mms | 2.7 g |
| Sensitivity | 86.8 dB @ 2.83 V/1 m |
| Power | 30 W RMS |
| Response | 80 Hz – 20 kHz |
| Cone | Poly damped woven glass fibre, copper cap |
| Parts Express boxes | sealed 1.4 L → F3 126 Hz / vented 2.83 L → F3 62 Hz |

### Dayton ND16FA-4 (tweeter) — switched from HiVi T20-8

| | ND16FA-4 | HiVi T20-8 |
|---|---|---|
| Impedance | **4 Ω — matches the woofer** | 8 Ω |
| Sensitivity | **93 dB** | 89 dB |
| Power RMS | 30 W | 15 W |
| Fs | 2246 Hz | 2000 Hz |
| Faceplate | 45 mm | 50 mm |
| Depth | 11.4 mm | 12.7 mm |
| Ferrofluid | Yes | Yes |

The 4 Ω match means both amp channels see identical loads, and the amp delivers ~17 W into 4 Ω versus ~10 W into 8 Ω at 12 V. The tweeter being 6.2 dB hotter than the woofer is headroom in reserve — the right direction to be wrong in.

### Drivers rejected, and why

| Driver | Reason |
|---|---|
| Dayton ND64-4 | Sealed F3 **283 Hz**. Not a bass driver. Sd 15.6 cm², and ₹2,615 |
| Dayton ND65-4 | 83 dB sensitivity (−3.8 dB), 15 W, ~2× the price. Matching ND65-PR is its only real advantage |
| Dayton ND90-4 | Needs 4.5 L per driver — the budget is ~2.8 L — and rolls off at 15 kHz |
| Dayton PC105-8 | Needs 4.25 L sealed, 126 mm frame on a 110 mm baffle, 8 Ω halves amp power, ₹4,800 the pair |
| Dayton PC83-8 | Same cone, 8 Ω: about 3 dB total loss and a 1.8× bigger box |
| Generic Amazon "3 inch hi-fi tweeter" | 35 mm voice coil (a midrange, not a tweeter), contradictory impedance, no T/S data, 75 mm body |
| Inkocean 48 mm square | Car-audio square faceplate, zero published data, ₹1,749 |
| Unnamed 3.6 Ω silk dome | Fs 1.5 kHz and 30 kHz extension are excellent, but 3.6 Ω trips the TPA3110's thermal protection, and 92 dB needs a −5 dB trim |

**Rule applied throughout: if the seller does not publish T/S parameters, the driver does not enter the design.**

---

## 11. Enclosure — Open Decision, Deliberately

This is the one place where the research report and the current plan disagree, so both are recorded.

### The report's position: sealed

> The 2 mm Xmax is decisive. A passive radiator unloads the driver near tuning, letting
> excursion spike below tuning. A sealed box rolls off at 12 dB/oct and inherently limits
> low-frequency excursion. Use ~1.5–2.5 L per driver (Qtc ~0.7–0.8) and extend the bottom
> electronically. And since a ~70–80 Hz woofer HPF is mandatory anyway, tuning a PR below
> that corner yields limited benefit.

### The counter-argument: an octave is a lot to give up

```
PC83-4 sealed 1.4 L  -> F3 = 126 Hz   <- loses the kick drum fundamental
                                         and a bass guitar's low E (41 Hz)
PC83-4 vented 2.83 L -> F3 =  62 Hz
```

And the excursion risk is asymmetric: **at** the tuning frequency the radiator does the air-moving and cone excursion actually *drops*. The danger is only **below** tuning — which is exactly what the high-pass filter removes, for about 20 instructions. For reference, the Marshall Middleton reaches a claimed 50 Hz from roughly 1.8 L using small PRs plus DSP correction.

### The decision: build so both can be tested

```
Box:        ~2.8 L per driver  (~5.5 L total internal)
PR tuning:  ~60 Hz, tuned by adding washers until the impedance
            minimum sits at 60 Hz
PR spec:    must displace >= 12 cm3 (2x the woofer's 6.0 cm3),
            3-4", generous Xmax, adjustable mass
            (Dayton makes no PR matched to the PC83 -- DMA series,
             ND90-PR, or generic adjustable-mass)
DSP:        high-pass 70-80 Hz, 2nd-4th order, on the woofer path
            -- NON-NEGOTIABLE in either alignment
Build:      mount the passive radiators on a REMOVABLE panel
```

- **Test A** — PR installed, HPF at ~70 Hz. Measure and listen.
- **Test B** — PR aperture closed with a blank plate: now a 2.8 L sealed box. Measure and listen.

A 2.8 L sealed box is oversized for the PC83-4 so F3 rises slightly, but it works — the fallback is genuinely usable. Cost of keeping the option open: one blank panel. **Model any PR in WinISD with the actual PR's parameters before buying** — the numbers above are starting estimates.

### Other enclosure rules

- Line the walls with 10–20 mm acoustic foam or polyester; light fill for a sealed box (it adds apparent volume); never over-stuff near a PR.
- Flush-mount both drivers.
- Chamfer or round the baffle edges — even a 3–5 mm radius measurably cuts diffraction on a narrow baffle.
- Keep the tweeter close to the woofer to limit lobing around the 4 kHz crossover.
- Brace, and seal every joint. A sealed alignment demands a genuinely airtight box.

### Honest expectation

Two Xmax-limited 86.8 dB / 30 W three-inch drivers give lively near- and mid-field levels, not outdoor-party SPL. The DSP loudness and limiter are what make it sound *good* rather than merely loud.

---

## 12. Power System — Corrected Maths

V3 stated that a 3S 3000 mAh pack holds "99.9 Wh". That is wrong, and the error mattered: **series cells add voltage, not capacity.**

```
3S 3000 mAh:   3.0 Ah x 11.1 V = 33.3 Wh   (not 99.9 Wh)
4S 2600 mAh:   2.6 Ah x 14.8 V = 38.5 Wh
4S2P 5200 mAh: 5.2 Ah x 14.8 V = 77 Wh
```

Realistic runtime, not the 20 hours V3 claimed:

| Listening level | Draw | 3S 3000 mAh | 4S2P 5200 mAh |
|---|---|---|---|
| Quiet background | ~5 W | ~5.3 h | ~12 h |
| Normal | ~8 W | ~3.3 h | ~7.7 h |
| Loud | ~20 W | ~1.3 h | ~3 h |
| Mixed real use | — | ~3 h | **~5–6 h** |

### Go 4S

```
Amp power at 4 ohm:
  12.0 V (3S)        ~17 W per channel, sagging badly toward cutoff
  14.8 V (4S nom)    ~22 W per channel
  16.8 V (4S full)   ~28 W per channel

+2.5 dB output, and it stays there instead of getting quieter
as the pack drains. TPA3116 handles up to 26 V.

TARGET PACK: 4S2P, 5200 mAh or better, 14.8 V, BMS included,
             16.8 V charge, ~12 V cutoff.
AVOID:       4S1P anything -- one evening of listening, no more.
```

Watch for mislabelled listings: a "4S1P 26000 mAh" pack is physically impossible (4S1P capacity equals one cell, and the largest 18650 is ~3500 mAh). It is a typo for 2600 mAh.

### Current budget

| Load | Power | Current @ 14.8 V |
|---|---|---|
| Idle (everything on, no music) | ~4 W | 0.27 A |
| Quiet (~70 dB) | ~8 W | 0.54 A |
| Normal (~80 dB) | ~15 W | 1.0 A |
| Loud (~90 dB) | ~30 W | 2.0 A |
| Bass transients | ~60 W | 4.0 A |

Baseline electronics (ESP32 + DSP + LEDs + amp idle) total about 0.22 A — music dominates completely.

```
BMS:      20 A
Fuse:     5 A slow-blow inline from battery +
Charger:  16.8 V @ 2 A  -> ~3 h for a 5200 mAh pack
Wiring:   18 AWG battery main, 20-22 AWG signal
Bulk:     1000-2200 uF at each amp VCC -- these supply the 4 A peaks
```

**Correction to V3's monitoring plan:** the MAX17043 is a **single-cell** fuel gauge. Use it on one representative cell or on a scaled measurement, and get balance/health from the per-cell resistor dividers into ADC1. The V3 divider ratios were sized for 3S and must be re-scaled for a 4S pack. Enforce a hard low-voltage cutoff in firmware regardless of what the BMS does.

---

## 13. Expected Output

```
Watch the units. Both drivers are quoted at 2.83 V, which into 4 ohms
is 2 W -- not 1 W. Subtract 3 dB before doing any power maths:

  PC83-4    86.8 dB @ 2.83 V/1 m  ->  83.8 dB/W
  ND16FA-4  93.0 dB @ 2.83 V/1 m  ->  90.0 dB/W

Woofers:   83.8 dB/W, ~12 W each after the limiter
           -> 94.6 dB each, ~98 dB for the pair
Tweeters:  90.0 dB/W, ~5 W each
           -> 97.0 dB each, ~100 dB for the pair (headroom to spare)

Bass is excursion-limited, not power-limited:
  Vd = 30.2 cm2 x 2 mm = 6.0 cm3 per driver, 12.1 cm3 for the pair

  200 Hz  ~105 dB
  150 Hz  ~100 dB
  100 Hz   ~94 dB
   80 Hz   ~90 dB
   65 Hz   ~86 dB
```

| | Max SPL @ 1 m | Low end |
|---|---|---|
| **This build** | ~95–100 dB | ~65–70 Hz |
| Marshall Middleton | 87 dB | 50 Hz |
| JBL Flip 6 | ~85 dB | 63 Hz |
| Genelec 8010A | 96 dB | 74 Hz |

Louder than the Middleton; the Middleton goes deeper on bigger radiators and more cone area. On paper the amps total ~50 W; real usable output is ~34 W, and the DSP limiter engages before the amps ever run out.

### Can it be made louder?

| Change | Gain | Cost | Worth it |
|---|---|---|---|
| 4S battery (14.8 V) | +2.5 dB | ₹300 | Yes, but it only helps mids/highs — bass is still Xmax-bound |
| 4 woofers instead of 2 | +6 dB and double the displacement | ~₹1,500 + bigger box + third amp | The only real fix for bass |
| Bigger driver | +3–4 dB | ₹2,000 + a much bigger box | Not portable any more |
| Bigger amp (TPA3255) | ~0 dB | ₹1,000 | Pointless — the driver is the ceiling |

100 dB at 1 m is uncomfortable to sit next to. Build it at ~50 W, listen, then decide.

---

## 14. Goal — Settled

A studio-monitor target was evaluated and **rejected**, because it is the opposite of a Marshall-flavoured target:

```
MARSHALL                           STUDIO / AUDIOPHILE
coloured, characterful       vs     flat, neutral
+4 dB bass, +2.5 dB presence vs     +/-2.5 dB max deviation
fun, forgiving               vs     revealing, unforgiving
tuned by ear                 vs     tuned by measurement
```

Chasing flat would have meant a calibrated UMIK-1 (~₹8,000), a full REW protocol, 3D-printed tweeter waveguides, 18 mm braced walls, an external PCM5102A/ES9038 DAC, and probably an ADAU1452 (80-bit ALU, ASRC) instead of the ADAU1701 (56-bit ALU, 1019 instructions, ~90 dB DAC THD).

**The goal is a speaker with good bass, clear mids and smooth highs.** What is kept is the short list that actually determines the result:

1. Correct enclosure volume from the T/S data
2. A tested alignment (PR at ~60 Hz, or sealed)
3. LR4 crossover at 4.0 kHz
4. Tweeter trimmed −6 dB, polarity verified
5. Limiter set conservatively for the 2 mm Xmax
6. Airtight box
7. Foam damping inside

A free phone RTA app (Spectroid / AudioTool) is enough to catch gross peaks. The full REW protocol in `docs/README_v3.md` stays available for anyone who wants to go further later.

---

## 15. DSP Program — V4 Signal Chain

```
I2S IN (48 kHz, 24-bit, ESP32 master)   /   or AUX via ADC0/ADC1
  |
  [1]  DC-block high-pass                    the ADCs have none -- prevents pops
  [2]  Input volume                          live via I2C safeload
  [3]  Dynamic loudness                      low-shelf + gentle high-shelf,
                                             scaled inversely with volume,
                                             cornered ABOVE the woofer HPF
  [4]  Woofer high-pass 70-80 Hz, 2nd-4th    MANDATORY -- protects 2 mm Xmax
  [5]  Baffle step correction  +3 to +4 dB low-shelf at ~1.0-1.2 kHz
                                             (115 / 0.11 m baffle; +6 dB is the
                                             theoretical max -- keep it a preset)
  [6]  PEQ voicing, ~4 bands per channel:
         80 Hz   shelf  +3.0 dB    body and punch
         250 Hz  peak   -2.0 dB    removes boxiness
         2.5 kHz peak   +1.5 dB    vocal clarity
         10 kHz  shelf  +2.0 dB    air
         (acoustic correction filters added here after measurement)
  [7]  Crossover LR4 @ 4.0 kHz               = two cascaded 2nd-order
                                             Butterworth biquads (Q 0.707)
                                             per band, or SigmaStudio's
                                             2-way crossover block
  [8]  Tweeter trim -6.0 dB + polarity check invert the woofer if the
                                             summation shows a null
  [9]  Tweeter time-alignment delay          from step response, 0-0.3 ms
  [10] Woofer limiter:  threshold -6 to -3 dBFS, ratio >=10:1,
                        attack 1-5 ms, release 100-300 ms
       Tweeter limiter: lower threshold, protects against clipping bursts
  [11] Output mute control                   released >= 300 ms after reset
  |
  VOUT0/1 -> woofers (TPA3116D2)    VOUT2/3 -> tweeters (TPA3110D2)
```

### Instruction budget

| Block | Instructions |
|---|---|
| Input DC-block + volume | ~10–20 |
| LR4 crossover (16 biquads) | ~80–120 |
| Woofer HPF (4 biquads) | ~20–30 |
| PEQ voicing (8 bands) | ~40–60 |
| Baffle-step shelf ×2 | ~15 |
| Tweeter trim + polarity | ~5 |
| Woofer + tweeter limiters | ~60–120 |
| Dynamic loudness | ~30–60 |
| **Total** | **~300–500 of 1024** |

Rule of thumb: a double-precision biquad is 5–7 instructions, single-precision 3–4. **Use single precision only on benign filters** — never on very low-frequency, high-Q filters, where the reduced resolution adds audible noise.

**Check `[project]/IC1_[name]/net_list_out2/compiler_output.txt` after every compile.** It prints e.g. `Number of instructions used (out of a possible 1024) = 458`. Going over the limit produces no clear error — the DSP just goes silent after Link/Compile/Download. Trust the compiler output over any estimate in this document. 96 kHz would halve the budget to 512 and is not an option: **48 kHz only.**

### Live control

Use **safeload for every runtime change** — volume, EQ, crossover, limiter thresholds. Values are 5.23 fixed-point; write the safeload data/address registers (0x0810–0x0819), then set the IST bit in 0x081C. MCUdude's library wraps all of it.

---

## 16. Programming Path — WiFi, Not USB

**rarranzb/ADAU1701-TCPi-ESP32** turns an ESP32 into a TCPi bridge on port 8086, so SigmaStudio connects to the DSP over WiFi (SigmaStudio: USBi → TCP/IP → 8086) with **hardware safeload** — no pops during live parameter changes — and can write the program to EEPROM from its web UI.

```
Repo default wiring           Our wiring
  SCL      -> GPIO 17           must be remapped  <- GPIO16/17 are PSRAM on
  SDA      -> GPIO 16           must be remapped     WROVER and CANNOT be used
  RESET    -> GPIO 21           see the pin map in section 18
  SELFBOOT -> GPIO 19
  Reconfigure at http://<esp-ip>/config
```

Caveats: older TCPi variants were **write-only with no readback**, and the tooling can need a recent or beta SigmaStudio build. Verify readback early if you intend to rely on it, keep SigmaStudio current, and back up project files — SigmaStudio projects are version-sensitive.

This makes the **FX2LP + freeUSBi programmer optional** — keep it as a backup for initial bring-up only.

Do **both** boot paths: EEPROM self-boot as primary (works even if the ESP32 crashes), with the ESP32 verifying the DSP is running and re-pushing the program if not.

After the first SigmaStudio compile, run **MCUdude's `DSP_parameter_generator`** — SigmaStudio scatters parameter macros across several header files; the script merges them into one usable file.

---

## 17. Reference Repositories

| Repo | What it gives us |
|---|---|
| `rarranzb/ADAU1701-TCPi-ESP32` | SigmaStudio over WiFi with hardware safeload — replaces the FX2LP |
| `Thenicolaibulow/decaVox` | Full KiCad 6 PCB: ADAU1701 + ESP32-WROVER + 4× MA12070P. Closest existing project — study its power sequencing, reset and clocking before drawing ours |
| `ClemensAtElektor/Elektor_AudioDSP` | ESP32-PICO + ADAU1701, the JP1 clock-source jumper, the exact clocking topology we use. Arduino IDE 1.8.19 / ESP32 core 2.0.17 |
| `freedsp.github.io` | Open ADAU1701 hardware, I2C and I2S getting-started guides, hand-soldering video for the QFP package |
| `MCUdude/SigmaDSP` | Arduino I2C library, safeload wrapper, DSP_parameter_generator |
| `Wei1234c/SigmaDSP` | Python control from PC or ESP32 — bench tool, not the shipping path |
| `WillyBilly06/ESP32-A2DP-SINK-WITH-CODECS-UPDATED` | **Current** LDAC / aptX HD / aptX / aptX-LL / Opus / AAC / SBC sink, ESP-IDF 5.5.2, PSRAM branch + BLE GATT + LED effects |
| `WillyBilly06/esp32-a2dp-sink-with-LDAC-APTX-AAC` | Superseded — its README points to the repo above |
| `sle118/squeezelite-esp32` | AirPlay, Squeezelite/LMS, Spotify Connect, multi-room, display, encoder |
| `schreibfaul1/ESP32-audioI2S` | SD/web FLAC, WAV, MP3 — always outputs 48 kHz |
| `pschatzmann/ESP32-A2DP` | Simplest reliable A2DP sink (SBC/AAC/aptX), good fallback |
| `rbouteiller/airplay-esp32` | AirPlay 2 on ESP32 — oriented at TAS5825M, worth watching |

---

## 18. ESP32 Pin Map

Baseline from the research report. **Reconcile against the chosen firmware's hard-coded defaults before wiring** — squeezelite-esp32 and WillyBilly06's sink both have opinions, and the TCPi bridge defaults to the unusable GPIO16/17.

```
UNUSABLE:  GPIO 6-11  (SPI flash)
           GPIO 16,17 (PSRAM on WROVER modules)
           GPIO 1,3   (USB serial)
STRAPPING: GPIO 0, 2, 5, 12, 15 -- handle with care at boot
ADC:       ADC2 does not work while WiFi is active. Use ADC1 only.
           GPIO 34-39 are input-only and have NO internal pull-ups.

I2S to ADAU1701 (ESP32 = master):
  GPIO 26 -> BCLK    (also wired to ADAU1701 MCLKI pin 32)
  GPIO 25 -> LRCLK / WS
  GPIO 22 -> DOUT

I2C to ADAU1701 + EEPROM + fuel gauge + optional OLED:
  GPIO 21 -> SDA      2.2 kOhm pull-up
  GPIO 4  -> SCL      2.2 kOhm pull-up

ADAU1701 RESET:
  GPIO 23

microSD (SPI):
  GPIO 13 MOSI | GPIO 27 MISO | GPIO 14 SCK | GPIO 5 CS (strapping: HIGH at boot)

Battery / NTC sense (ADC1):
  GPIO 32 | GPIO 33 | GPIO 34 | GPIO 35     (34/35 input-only, ideal for sense)

Rotary encoder:
  GPIO 36 (VP) A | GPIO 39 (VN) B | GPIO 18 switch
  36/39 need EXTERNAL pull-ups

Buttons:
  GPIO 19, GPIO 2 (strapping -- must not be held at boot), one spare

WS2812B data:
  GPIO 12 (strapping, must be LOW at boot -- WS2812 idles low, but verify)

Amp mute / enable:
  a spare GPIO through a buffer transistor (never direct)
```

---

## 19. Zero-PCB Build Plan

```
Stage 0  SigmaStudio only, no hardware
         Build the full chain, compile, check the instruction count,
         Export System Files -> param_data.h

Stage 1  Power rails ONLY on perfboard. DSP not connected.
         Multimeter: 3.30 V +/-0.1 and 1.80 V +/-0.05
         AVDD = 3.3 V, DVDD = 1.8 V. A wrong rail kills the chip silently.

Stage 2  ADAU1701 on the TQFP->DIP adapter.
         Crystal, decoupling, PLL loop filter, EEPROM, RSVD to GND, IOVDD.
         Continuity-test all 48 pins to their DIP holes.
         Bring the DSP up STANDALONE first: crystal clock, analog AUX in,
         DAC out. Confirm 0x0827 = 01 and the core register un-muted.
         This isolates DSP problems from ESP32 problems.

Stage 3  ESP32 running the TCPi bridge, I2C pins remapped off GPIO16/17.
         >>> MILESTONE: SigmaStudio connects to the chip over WiFi <<<
         Reach this and the riskiest part of the project is behind you.

Stage 4  Sine generator -> VOUT0. Scope it, or listen through any small amp.

Stage 5  Write the program to EEPROM. Power-cycle with the ESP32 detached.
         Still generating a tone? Self-boot works.

Stage 6  Flip the jumper to the ESP32 clock. Scope for a steady 3.072 MHz.
         Hold reset until the clock is up; mute the amps ~300 ms.
         Confirm the DSP does not mute. This proves PLL_MODE 0/0.

Stage 7  Amps: set 20 dB gain, rework inputs, then drivers.
         Listen for hiss with no signal.
Stage 8  Sources one at a time: AUX, microSD, Bluetooth, WiFi.
Stage 9  BLE GATT alongside audio -- the single/dual MCU decision point.
```

Stages 1–5 need no amplifier and no speakers at all.

### Benchmarks that change the plan

| Symptom | Response |
|---|---|
| PLL won't lock reliably in 64 × fs on perfboard (intermittent mute, unstable 3.072 MHz on the scope) | Fall back to the crystal for bench work, shorten and clean the clock wiring, move the clock section to a small PCB |
| `compiler_output.txt` nears ~900 instructions | Switch benign filters to single precision, or cut PEQ bands |
| BT audio stutters with BLE active | Enlarge the PSRAM audio buffer, reduce BLE advertising/connection frequency, confirm core pinning. Only then consider the PB-03 |
| Bass distorts / cone bottoms | Tighten the woofer HPF and lower the limiter threshold. The 2 mm Xmax is the hard limit |

---

## 20. Phase 1 BOM (DSP only)

Everything needed to reach Stage 6. Speakers, amps, battery and enclosure are deliberately excluded.

### AliExpress — order first, 15–20 day lead

| Item | Qty | ₹ |
|---|---|---|
| TQFP48 → DIP adapter, **0.5 mm pitch, 7×7 mm body** | 3 | 200 |
| No-clean flux paste (syringe) | 1 | 150 |

The dual-sided "TQFP32/44/64/80/100 → DIP" adapters sold by Robu/Sharvi work — one side is 0.5 mm. Sellers contradict each other on which side is labelled A or B; ignore the labels and use the side with the finer pads (12 pads per edge = 48).

### Evelta / Robu

| Item | Qty | ₹ |
|---|---|---|
| ADAU1701JSTZ-RL | 1–2 | 350–700 |
| 24LC256 EEPROM | 2 | 80 |
| 12.288 MHz crystal — AT-cut, fundamental, parallel | 2 | 100 |
| 2N3906 PNP (or FZT953) | 2 | 10 |
| AMS1117-3.3 module | 2 | 60 |

### Passives

| Value | Purpose | Qty |
|---|---|---|
| 100 Ω | crystal damping | 5 |
| 475 Ω | PLL loop filter | 5 |
| 560 Ω | DAC output filter ×4 | 5 |
| 1 kΩ | internal regulator base | 5 |
| 2.2 kΩ | I2C pull-ups | 5 |
| 5.6 kΩ | TPA3116 gain set to 20 dB (later) | 5 |
| 7 kΩ 1% | ADC input | 5 |
| 10 kΩ | SELFBOOT, WP pull-ups | 10 |
| 18 kΩ 1% | ADC_RES | 5 |
| 47 kΩ | MP pin pull-downs | 10 |
| 100 nF | every power pin (8 needed) | 20 |
| 22 pF | crystal load | 4 |
| 3.3 nF / 56 nF | PLL loop filter | 5 each |
| 5.6 nF | DAC output filter | 5 |
| 10 µF | FILTA, FILTD, bulk | 10 |
| 47 µF | CM pin, DAC coupling, ADC coupling | 10 |

≈ ₹300

### Build supplies

| Item | ₹ |
|---|---|
| Perfboard ×3, headers, jumper wires | 250 |
| 0.5 mm solder + 1.5 mm desoldering braid | 210 |
| 99% isopropyl alcohol | 60 |
| 12 V 2 A bench adapter | 250 |
| Multimeter (mandatory, if not already owned) | 500 |

### Total

```
Without multimeter:  ~ Rs 1,760 - 2,360
With multimeter:     ~ Rs 2,260 - 2,860

A second ADAU1701 is Rs 350. It is worth it -- the fear of
destroying the only chip is what makes people rush 0.5 mm soldering.
```

### Not yet

Tweeters, woofers, amp modules, microSD module, ESP32-WROVER-IE, battery, BMS, charger, passive radiators, enclosure materials, encoders, buttons, WS2812B, jacks. Roughly ₹5,500 more, all of it after Stage 6 proves the DSP chain works.

An ESP32 already on the bench is fine for Stages 1–6 — check the shield: WROVER has PSRAM, WROOM does not, and PSRAM is only needed once LDAC decoding starts.

---

## 21. Soldering the LQFP-48

The single highest-risk manual operation in the project. Budget a full hour, good light, no rush.

```
 1. Clean the adapter pads with IPA, let dry
 2. Tin ONE corner pad, lightly
 3. Place the chip. Check the pin-1 dot against the adapter marking. Twice.
 4. Reflow that corner while nudging into alignment. Every pin centred
    on its own pad -- the 0.5 mm footprint is universal (32-100 pins),
    so a 48-pin chip sits on the inner portion of the pad ring
 5. Tack the opposite corner. The chip can no longer move
 6. Flood one side with flux, generously
 7. Drag-solder that side in one slow continuous motion
 8. Bridges will happen: more flux, braid over the bridge, press, lift
 9. Repeat all four sides
10. Clean with IPA and a soft brush
11. Inspect every pin under magnification (phone camera zoom works)
12. Continuity-test all 48 pins to their DIP holes, and check for
    shorts to neighbours -- especially the power and PLL pins

Iron: 300-320 C, fine conical or <=1 mm chisel tip.
Practice on a scrap QFP first -- that is what the third adapter is for.
```

---

## 22. Deferred to V2

| Idea | Status |
|---|---|
| TAS5825M / TAS5805M smart amp | Right architecture for a custom PCB — JLCPCB reflows the QFN, ~30 fewer parts, no MCLK, hardware excursion protection. Blocked today by VQFN (unsolderable by hand) and the lack of a safeload equivalent |
| ESP32-S31 | Classic BT + BLE 5.4 + WiFi 6 + dual I2S with hardware BT audio sync on one chip. 0 stock, 21-week lead, RISC-V port needed for the LDAC firmware, and its +EDR support needs confirming in the datasheet. Set a distributor stock alert |
| Voice assistant / smart-home hub | Wake word needs an ESP32-S3 (microWakeWord is S3-only) **and** acoustic echo cancellation — Home Assistant's own Voice PE pairs an S3 with a dedicated XMOS XU316 DSP just for AEC. Two MCUs and a hard DSP problem on top of this build is how projects die |
| Home Assistant integration | **Free today, firmware only:** media_player entity, MQTT control from the rotary encoder, ESP-NOW to other projects, Matter over WiFi, TTS announcements. No microphone, no wake word, no AEC |
| External DAC (PCM5102A / ES9038Q2M) | Takes DAC THD from ~−90 dB to ~−110 dB. Route the ADAU1701 I2S output to a header on the PCB even if unpopulated. Needs a 74LVC1G17 buffer because of the 0.6 V logic low |
| ADAU1452 | 80-bit ALU, ~295 MIPS, 32-bit coefficients, built-in ASRC — which would eliminate every clock problem in §1. For a genuine studio-monitor build |
| 4 woofers | The only real fix for the bass ceiling: +6 dB and double the displacement |

---

## 23. Still Open for V5

1. **Primary firmware stack** — WillyBilly06's codec sink (Bluetooth-first, LDAC) or squeezelite-esp32 (WiFi-first, AirPlay). They cannot be merged. Current lean: WillyBilly06 + ESP32-audioI2S for SD, AirPlay deferred.
2. **Source-switching state machine** — WiFi and Classic BT share one radio, so the firmware must cleanly tear one down before starting the other, while keeping the I2S clock running the whole time (including AUX mode).
3. **Single vs dual MCU** — decided provisionally as single, confirmed at Stage 9 by streaming A2DP while pushing continuous BLE GATT writes. Fallback: PB-03 on UART, four spare pads reserved.
4. **Enclosure alignment** — PR at ~60 Hz vs sealed. Build for both, decide by measurement.
5. **Passive radiator mass** — cannot be calculated to a final value. Model in WinISD with the real PR's parameters, buy after the drivers are measured, then iterate with washers.
6. **BLE GATT map** — the 25-characteristic design in `docs/README_v3.md` stands, but needs the new parameters folded in (HPF corner, alignment mode, source select, limiter thresholds).
7. **4S divider ratios** — the cell-monitoring dividers in V3 were sized for 3S and must be re-scaled; the MAX17043 is single-cell only.

---

## 24. Confidence and Caveats

Recorded so future-me knows which claims are verified and which are inferred. Full list in `docs/pre-build-research-report.md` §9.

- The ADI "best solution" clock endorsement is a **design recommendation**, not a published measured build with jitter data. The strongest empirical proof is the shipping Elektor board using the same ESP32-master topology — but its article does not spell out whether JP1 routes BCLK or a separate MCLK, nor the exact PLL mode. **Verify on the bench that the DSP does not mute.**
- The 44.1 kHz → 2.8224 MHz case is ~8% below the nearest listed PLL point. Inside the ±20% window, ADI is confident, but force 48 kHz so it never arises.
- Instruction-count estimates are approximate. **Trust `compiler_output.txt`.**
- TPA3116 noise and power behaviour varies board to board, and gain-resistor designators differ between clones. Identify the real resistors against the TI datasheet before removing anything.
- Passive-radiator numbers are starting estimates and need WinISD modelling with the actual PR's parameters.
- Pin-map suggestions must be reconciled with each firmware's hard-coded defaults and the WROVER-IE datasheet.
- GPIO drive and the "0.6 V low" behaviour differ between datasheet revisions. **Rev. C governs this build.**

---

## 25. References

| Resource | Purpose |
|---|---|
| ADAU1701 Datasheet **Rev. C** (analog.com) | The only datasheet to work from. Figure 12 System Block Diagram, Figure 16 crystal, Figure 17 PLL filter, Figure 18 DAC filter, Table 12 PLL modes, Table 13 ADC resistors |
| ADI EngineerZone | PLL_MODE / MCLK guidance, BCLK jitter, PVDD, MP pin float-high |
| ADI AN-1006 / AN-1168 / CN-0162 | Crossover design, EQ design, DSP-to-amp interface |
| Elektor issue 358 (Nov/Dec 2024) | Audio DSP FX Processor — the shipping ESP32 + ADAU1701 board with the JP1 clock jumper |
| SigmaStudio (analog.com) | Free, Windows. Works with no hardware attached |
| Espressif ESP-IDF docs | I2S, PSRAM/himem, coexistence, strapping pins |
| WinISD / speakerboxlite.com | Enclosure and passive radiator simulation |
| REW (roomeqwizard.com) | Acoustic measurement, if pursued later |
| Infineon AN-1135 | Class-D amplifier PCB layout |
| `docs/pre-build-research-report.md` | Primary source for V4, with quotes and caveats |
| `docs/README_v3.md` | V1–V3 history, app/BLE design, cell monitoring, REW protocol |

---

## Author

Ghost

*Marshall-level sound from raw chips. Not because it is easy — because it is the right way to build it.*

**V4 — architecture locked against Rev. C and the pre-build research report. Phase 1 parts on order. Hardware next.**

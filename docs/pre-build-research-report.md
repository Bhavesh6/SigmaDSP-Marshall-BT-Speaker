# Final Pre-Build Research Report

**Portable Bluetooth DSP Speaker (ADAU1701 + ESP32-WROVER)**

> Source document for V4. Archived verbatim. Where V4 departs from this report, the
> README says so explicitly and gives the reason — the main case is the tweeter, which
> changed from HiVi T20-8 to Dayton ND16FA-4 after this report was written.

---

## TL;DR

- **The clock architecture is CORRECT and ADI-endorsed** — feeding the ESP32's 3.072 MHz BCLK to both MCLKI and INPUT_BCLK with PLL_MODE0=0 / PLL_MODE1=0 (64×fs) is exactly what an ADI applications engineer called "the best solution," and it is the same scheme used in the shipping Elektor Audio DSP FX Processor board (Elektor issue 358, Nov/Dec 2024). But 64×fs is the slowest-locking mode (85.3 ms PLL start-up) and demands a stable, always-running clock — keep a 12.288 MHz crystal on a jumper as a fallback, exactly as Elektor's JP1 does.
- **The firmware plan is viable but forces trade-offs**: use WillyBilly06's actively-maintained A2DP multi-codec sink for Bluetooth, keep the ESP32 as I2S master at a fixed 48 kHz, and drive the ADAU1701 live over I2C with MCUdude/SigmaDSP + safeload. You cannot reliably run Classic-BT A2DP, WiFi AirPlay, and BLE GATT all at once on the original ESP32 — plan mutually-exclusive source modes, not simultaneous radios.
- **The acoustic/amp design needs two corrections**: the PC83-4's 2 mm Xmax (Vd only 6.0 cm³) makes a sealed box safer than a passive radiator unless you add a strict excursion-protecting high-pass + limiter; and you must reduce the TPA3116 modules to 20 dB gain (TI's own low-noise reference point) and rework their inputs to avoid hiss and clipping from the ADAU1701's 0.9 Vrms output.

## Key Findings

1. **Clock architecture: verified.** ADI staff explicitly endorse BCLK-as-MCLK in 64×fs. 3.072 MHz sits at the low edge of the PLL range but inside spec. PLL start-up in 64×fs mode is 85.3 ms (vs ~21 ms at 256×fs) — you must hold everything muted through init.
2. **BCLK must never stop.** In 64×fs mode the ADAU1701 mutes if the clock dies or drifts async. The ESP32 I2S master clock must be running before the DSP finishes booting and must stay running even during silence (including in analog-AUX mode).
3. **Firmware repos are current:** WillyBilly06's codec sink is actively developed (2026, ESP-IDF 5.5.2); rarranzb/ADAU1701-TCPi-ESP32 gives WiFi SigmaStudio programming with hardware safeload; MCUdude/SigmaDSP is the standard I2C control library.
4. **Instruction budget is comfortable:** a full 2-way active stereo (LR4 crossover + PEQ + baffle step + limiters + loudness) lands around 300–500 of the 1024 instructions. The DSP goes silently mute if you exceed the budget, so always check `compiler_output.txt`.
5. **Sealed box is the safer alignment** for this low-Xmax driver; a passive radiator is possible but risky without excursion control.
6. **Amp gain must drop to 20 dB** and inputs reworked, or you get hiss and easy clipping.
7. **Many small ADAU1701 gotchas are real** (DAC setup register, GPIO 2 mA drive, float-high MP pins, crystal type, self-boot IOVDD) — all fixable if known in advance.

---

## 1. Clock Architecture (highest-risk item) — CONFIRMED CORRECT

The plan — ESP32 as I2S master, its 3.072 MHz BCLK wired to both the ADAU1701 MCLKI and INPUT_BCLK (MP5), with PLL_MODE0=0 and PLL_MODE1=0 for 64×fs — is a legitimate, ADI-endorsed topology.

- On the ADI EngineerZone thread "MCLKI with I2S in ADAU1701," an ADI applications engineer (ColemanR) wrote: *"Connecting the 3.072 MHz clock to both BCLK and MCLK would be the best solution. In this configuration, please set the PLL to 64 × fs by setting PLL_MODE0 and PLL_MODE1 both to GND, and also take into account the increased amount of time required for PLL lock."*
- The same engineer stressed the fundamental rule: *"the MCLK and BCLK/LRCLK must be synchronous, but not necessarily phase aligned. If there are crossing edges between MCLK and BCLK, there will be some unpredictable audio artifact."* Wiring BCLK to both pins makes them inherently synchronous — the whole point.
- Multiple other ADI threads confirm the same, e.g. *"you can use a 3.072MHz bit clock as a master clock as long as you set the PLL Mode to 64x fs... the I2S must always be present and it cannot stop."*

**Real-world confirmation — the Elektor Audio DSP FX Processor** (ESP32-PICO + ADAU1701, Elektor issue 358, Nov/Dec 2024). A shipping, working board using exactly this ESP32-master topology. IC1 is the ADAU1701JSTZ; X1 is a 12.288 MHz / 20 pF crystal; JP1 selects the DSP clock source. Author Clemens Valens wrote: *"there is only one solution because the ESP32 software libraries do not (yet) support I²S slave mode. Therefore, the ESP32 must be the master."* And: *"A subtle (undocumented?) detail of the ADAU1701 is that it requires the I²S clock signals to be synchronous to its master clock. If not, it mutes the audio outputs... The only way to ensure synchronicity when the DSP is the I²S slave, is by clocking the DSP from the ESP32. This is where JP1 comes in. It allows choosing the DSP clock source: crystal X1 or the ESP32's MCLK signal... when I²S is to be used, make sure to short pins 2 & 3 of JP1."* The crystal position is pins 1 & 2. This directly validates the jumper-selectable crystal fallback. (The board's DAC outputs are also rated 0.9 V RMS / 600 Ω.)

**Does 3.072 MHz meet the minimum?** Yes, but only just. It is the lowest MCLKI in the datasheet's PLL table, and the PLL operating range is nominal ±20%. A 48 kHz-only fixed source keeps you safely on the 3.072 MHz point. The danger is 44.1 kHz sources: BCLK becomes 2.8224 MHz, ~8% low and not a listed lock point. ADI expressed confidence it still works within the ±20% window, but the safe fix is to **force the ESP32 I2S output to a fixed 48 kHz always** (resample BT/WiFi/SD to 48 kHz) so the DSP only ever sees 3.072 MHz.

**PLL lock timing (verified from datasheet).** *"The PLL start-up time lasts for 2^18 cycles of the clock on the MCLKI pin. This time ranges from 10.7 ms for a 24.576 MHz (512 × fS) input clock to 85.3 ms for a 3.072 MHz (64 × fS) input clock."* So 64×fs is by far the slowest: 85.3 ms PLL start-up plus boot cycle — budget up to ~260 ms before audio is valid, vs ~21 ms for the 12.288 MHz (256×fs) mode. Practical consequences:

- Hold the ADAU1701 in RESET until the ESP32 I2S clock is confirmed running.
- Keep amps muted (or unpowered) for at least ~300 ms after DSP reset to avoid the boot pop.
- If self-booting from EEPROM, the part waits for MCLK before it initializes — fine, but budget the extra time.

**Jitter concern (real, but sidestepped).** The ESP32's I2S output jitter is generally acceptable for the ADAU1701's PLL, which is designed to lock to imperfect clocks. Note the widely-cited itohi.com build log where an ESP32's MCLK output on GPIO0 was measured jumping between ~7 MHz and 13 MHz — but that was the APLL-derived GPIO0 MCLK output behaving badly, not the BCLK. This design deliberately avoids that trap by using BCLK (a cleanly-divided clock) as the master clock, not the flaky GPIO0 APLL MCLK. A point in favour of this architecture over the "separate MCLK on GPIO0 via APLL" alternative.

**Verdict on alternatives:**

- Keep the 64×fs BCLK-as-MCLK plan as primary. Proven; needs no extra oscillator.
- Do keep the 12.288 MHz crystal on a jumper (exactly like Elektor's JP1). If the PLL ever fails to lock reliably on perfboard (long noisy wires), fall back to the crystal for analog-only bench testing. **Be aware the crystal is genuinely only a fallback:** with the crystal on MCLKI (256×fs) but the ESP32 as I2S master, MCLK and BCLK are async and the DSP will mute the digital path — so the crystal path is for standalone/analog operation or debugging, not the live digital operating mode.
- 512×fs / 24.576 MHz mode gives faster lock (10.7 ms) and enables 96 kHz, but requires the ESP32 to produce a stable 24.576 MHz, reintroducing the GPIO0 APLL jitter problem, and halves the instruction budget to 512. Not worth it. Stay at 48 kHz / 64×fs.

---

## 2. Firmware / Software Evaluation

**rarranzb/ADAU1701-TCPi-ESP32** — Works and is the modern replacement for a USBi/FX2LP. A WiFi bridge so SigmaStudio connects over TCP/IP (in SigmaStudio: USBi → TCP/IP → port 8086). Its README explicitly claims *"No pops or clicks thanks to hardware safeload"* and it has a "Save to EEPROM (selfboot)" web function. **Important:** its default wiring is SCL→GPIO17, SDA→GPIO16 — but GPIO16/17 are the PSRAM pins on the WROVER-IE, so the I2C pins must be reconfigured from its web UI (`http://ESP32-IP/config`). It descends from the older EngineerZone TCPi project (Rocha) and hasaranga/SigmaESP32. **Caveat:** the older TCPi variants were write-only (no readback) and may need a recent/beta SigmaStudio; verify readback if you rely on it. This is the recommended programming path — no FX2LP needed.

**FX2LP / freeUSBi (CY7C68013A)** — Still the "official" USBi-clone path; works with SigmaStudio's USBi driver but is fiddlier (firmware loading, driver signing). Use the ESP32 TCPi bridge as primary and keep the FX2LP as backup.

**WillyBilly06 A2DP sinks** — Two repos. The older `esp32-a2dp-sink-with-LDAC-APTX-AAC` (ESP-IDF v5.1.4, WROVER + PSRAM required) is superseded; its README states *"Active development has moved to ESP32-A2DP-SINK-WITH-CODECS-UPDATED. New projects should start there."* The current `ESP32-A2DP-SINK-WITH-CODECS-UPDATED` targets ESP-IDF v5.5.2 with a patched Bluetooth stack and supports *"LDAC | aptX HD | aptX | aptX-LL | Opus | AAC | SBC."* It has a PSRAM branch (WROVER) and an internal-SRAM branch (WROOM); LDAC runs up to 96 kHz/24-bit. It bundles BLE GATT, DSP, level meters, and WS2812B/LED effects — very close to the target feature set — and uses a native ESP-IDF A2DP/AVRCP wrapper (not Arduino audio libs). For the WROVER-IE, use the PSRAM branch. It will need modifying to force fixed 48 kHz I2S out (master) into the ADAU1701.

**sle118/squeezelite-esp32** — Mature; does AirPlay (v1), Spotify Connect, BT sink, and streaming on a WROVER (needs ≥4 MB flash + 4 MB PSRAM). **Critical constraint:** it is one firmware with one active mode at a time. The project's own foreword warns: *"More and more people seems to use this without a LMS server, just for BT, AirPlay or Spotify. It's fine but understand that squeezeliteESP32 is primarily a Logitech Media Server player... All the others are add-ons stitched to it, so other modes have their shortcomings."* It does not coexist with a separate BT A2DP firmware — it **is** the firmware, and you pick a mode at runtime. **You cannot merge squeezelite-esp32 and WillyBilly06's codec sink into one binary. Choose one primary streaming stack.** (If AirPlay is a must-have, the standalone `rbouteiller/airplay-esp32` AirPlay-2 project is a newer alternative but is oriented at TAS5825M, not ADAU1701.)

**schreibfaul1/ESP32-audioI2S** — Good for microSD FLAC/WAV/MP3 with I2S output; can be configured for fixed I2S output. 16-bit/44.1 kHz and typical FLAC play fine; 24-bit/96 kHz FLAC is near the ESP32's real-time decode limit and may stutter. Since everything is resampled to 48 kHz for the DSP anyway, target 16–24 bit / 44.1–48 kHz files.

**MCUdude/SigmaDSP (Arduino)** — The recommended live-control library. It exposes `safeload_writeRegister()` and biquad/gain/EQ/dynamic-bass/clip helpers, plus EEPROM self-boot support and a `DSP_parameter_generator` workflow. Its header confirms the safeload data/address registers (0x0810–0x0819) and core register (0x081C). This is the MCU→DSP live-EQ path. Safeload is the correct mechanism for pop-free updates: write parameters to the safeload registers (value in 5.23 format), then set the IST bit.

**Wei1234c/SigmaDSP (Python)** — Alternative control path (PC or ESP32 as TCP server). Useful for scripting/bench work; keep as a tool, not the shipping embedded control path.

**decaVox (Thenicolaibulow/decaVox)** — Open-source KiCad hardware: *"an open source smart power audio amplifier... Powered by four Merus MA12070P Amplifiers, an ADAU1701 DSP Core & an ESP32 Wrover µC"* (requires KiCAD 6.0). The single most relevant open-hardware reference — same DSP + same MCU. Study its schematic for power sequencing, I2C, reset, and clocking. It uses Infineon MERUS amps, which is where the "0.6 V logic-low" GPIO concern originates (see §6).

**Elektor_AudioDSP (ClemensAtElektor)** — The other must-study schematic (ESP32-PICO + ADAU1701, JP1 clock jumper). Developed on Arduino IDE 1.8.19 / ESP32 core 2.0.17. Its clocking approach is this exact design.

**Can ONE firmware do BT + AirPlay + SD + AUX with runtime switching?** Realistically no — not all radios at once.

- Classic BT A2DP + WiFi simultaneously on the original ESP32 is officially problematic. Espressif staff: *"BR/EDR and WIFI coexistence performance is in optimizing on ESP32."* Users report *"massive packet loss on the WiFi side"* and WiFi disassociation when A2DP is active — they share one radio via time-division, so both degrade, and A2DP is jitter-sensitive.
- BLE GATT + WiFi, or BLE GATT + Classic BT, can work with tuning (pin BT and WiFi controllers to different cores via `CONFIG_BTDM_CTRL_PINNED_TO_CORE_CHOICE` and `CONFIG_ESP_WIFI_TASK_CORE_ID`; enable "Software controls WiFi/Bluetooth coexistence"; set Core Debug Level to None), but audio quality suffers under contention.
- **Recommended firmware architecture — one firmware, mutually-exclusive source modes selected at runtime/BLE:**
  - **Bluetooth mode:** Classic BT A2DP (WillyBilly06 stack) + BLE GATT control. WiFi off.
  - **WiFi/AirPlay mode:** WiFi streaming + BLE GATT control. Classic BT off.
  - **SD mode:** local FLAC/WAV playback + BLE GATT. Radios idle.
  - **AUX mode:** analog into the ADAU1701 ADC directly — but keep the I2S clocks running so the DSP does not mute (ADI: *"the I2S must always be present and it cannot stop even if you are only using the analog ADC input"*).
- BLE GATT for control is fine alongside any single audio path. Rough RAM: BLE ~50 KB, Classic BT ~100 KB, WiFi ~70 KB — do not hold all three live.

**Memory:** The WROVER-IE has 8 MB PSRAM but the ESP32 can only directly map 4 MB (the rest needs the himem API). LDAC A2DP + BLE GATT + I2S + I2C is feasible with PSRAM audio buffering. Internal DRAM is the scarce resource (Classic BT alone ~100 KB) — budget conservatively and prefer the PSRAM branch of WillyBilly06's sink so audio buffers live in PSRAM.

---

## 3. SigmaStudio DSP Program Design

**Crossover frequency.** The T20-8 has Fs = 2 kHz and HiVi rates it usable from 4 kHz. Crossing at 3.5 kHz with LR4 puts the −6 dB point 1.75 octaves above Fs; at 2 kHz the tweeter is ~24+ dB down — borderline at high SPL. **Recommendation: cross at 4.0 kHz with LR4.** 3.5 kHz LR4 is tolerable only with a conservative tweeter limiter. Implement LR4 as two cascaded 2nd-order Butterworth biquads (Q = 0.707) per band, or use SigmaStudio's built-in 2-way crossover block (defaults to LR4 24 dB/oct). ADI: *"A 4th-order Linkwitz-Riley filter can be implemented by connecting two 2nd-order biquad filters in series."*

**Tweeter level trim.** Sensitivity difference is 89 − 86.8 = 2.2 dB; pad the tweeter down ~2.2 dB with a gain block. Check polarity at crossover — invert the woofer if the summation shows a null (the hackaday DSP-01 2-way example did exactly this: *"to make them move in the same direction, the woofer's polarity is inverted"* with a −1.9 dB tweeter trim).

**Baffle step correction.** For a ~110 mm-wide baffle the step centre ≈ 115/W(m) ≈ 115/0.11 ≈ ~1.0–1.2 kHz. Apply a low-shelf BSC; the theoretical maximum is +6 dB, but in a small portable used near surfaces +3 to +4 dB usually suffices and preserves headroom/battery. Make BSC amount a tunable preset.

**Excursion protection (critical for 2 mm Xmax).** With only 2 mm Xmax and Fs 80 Hz, bass below ~80 Hz produces huge excursion for little output. You must high-pass the woofer:

- High-pass at ~70–80 Hz, 2nd–4th order on the woofer path.
- RMS/peak limiter on the woofer: start threshold conservative (around −6 to −3 dBFS), ratio ≥10:1 (brick-wall), attack 1–5 ms, release 100–300 ms; tune by watching cone excursion at low frequencies.
- A separate tweeter limiter with a lower threshold to protect the 15 W tweeter from clipping bursts.

**Dynamic loudness / Fletcher-Munson.** Use a low-shelf (and gentle high-shelf) boost that scales inversely with volume — bass/treble lift at low levels, flattening as volume rises. SigmaStudio and MCUdude's library expose dynamic bass-boost cells. Keep the low-shelf boost above the HPF corner so it does not fight the excursion limiter.

**Instruction budget.** The ADAU1701 executes 1024 instructions/sample at 48 kHz (512 at 96 kHz). Exceeding it makes the DSP silently mute on Link-Compile-Download — a very common trap. Check usage in `[project]/IC1_[name]/net_list_out2/compiler_output.txt`, which prints e.g. *"Number of instructions used (out of a possible 1024) = 458."* Rough per-block cost: a double-precision biquad ≈ 5–7 instructions; single-precision ≈ 3–4; dynamics processors and lookups more; delays cheap in instructions but use data RAM. Realistic budget for a 2-way stereo:

| Block | Instructions |
|---|---|
| Input DC-block + volume | ~10–20 |
| LR4 crossover (4 biquads/channel × 2 = 16 biquads) | ~80–120 |
| Woofer HPF (2 biquads × 2) | ~20–30 |
| PEQ voicing (~4/channel × 2 = 8) | ~40–60 |
| Baffle-step shelf (×2) | ~15 |
| Tweeter trim + polarity | ~5 |
| Woofer + tweeter limiters | ~60–120 |
| Dynamic loudness | ~30–60 |
| **Total** | **~300–500 of 1024** |

Comfortable, with room for more EQ. Use single-precision biquads only on benign filters (avoid them on very low-frequency, high-Q filters where they add noise). Always read `compiler_output.txt` before trusting a download.

**Safeload for live updates.** Use safeload for every runtime change (volume, EQ, crossover, limiter thresholds). MCUdude's library handles the sequence (write safeload data/address regs 0x0810–0x0819, set IST bit in 0x081C). Add a DC-blocking high-pass at the input to avoid the ADC DC-offset pops ADI warns about (*"The ADC converters do not have a built in high pass filter so you will see a DC offset"*).

**Example projects to start from:** ADI's "ADAU1701 2-Way Crossover" video/project (standard 2-way crossover block + pushbutton volume), the audiodevelopers.com ADAU1701 tutorial series, and the emumannen.blogspot.com tutorial (uses the Automatic Speaker EQ / Two-Way Speaker PEQ block that does crossover + phase + EQ to a target).

---

## 4. Enclosure & Acoustic Design

**Driver reality check.** PC83-4: Fs 80.1 Hz, Qts 0.54, Vas 1.98 L, Mms 2.7 g, Xmax 2.0 mm, Sd 30.2 cm², **Vd 6.0 cm³** (Parts Express official spec), 86.8 dB, 30 W RMS. Parts Express's staff-recommended enclosures: sealed 0.05 ft³ (≈1.4 L) → F3 126 Hz; vented 0.1 ft³ (≈2.83 L) → F3 62 Hz. Fs 80 Hz with Qts 0.54 is a driver that wants a small box and does not go truly deep.

**Sealed vs passive radiator — recommendation: SEALED** (or PR only with strict DSP protection). The 2 mm Xmax is decisive. A passive radiator unloads the driver near tuning, letting excursion spike below tuning — dangerous for a 2 mm driver without an aggressive high-pass. A sealed box rolls off gently at 12 dB/oct and inherently limits low-frequency excursion; the DSP then extends the low end electronically (within excursion/power limits).

- **Sealed volume:** ~1.5–2.5 L per driver gives a sealed Qtc ~0.7–0.8 and F3 in the ~100–126 Hz region; use a DSP low-shelf/Linkwitz-transform to lift the bottom.
- **If you insist on a passive radiator** (for the Marshall-style bass bump and higher efficiency near tuning): use ~2.8–4 L per driver and pick a PR displacing at least double the driver's Vd. With PC83-4 Vd = 6.0 cm³, the PR must displace **≥ ~12 cm³** — a 3–4" PR with generous Xmax and adjustable mass (Dayton DMA-series, ND90-PR, or generic adjustable-mass PRs; Dayton makes no PR matched to the PC83). Tune by adding washers until the impedance minimum sits at ~60 Hz. Because a woofer HPF at ~70–80 Hz is mandatory anyway, **tuning a PR below that corner yields limited benefit** — another argument for sealed. Model any PR in WinISD with the actual PR parameters before buying.

**Realistic expectations** in ~1.4–2.5 L per driver: sealed F3 ~100–126 Hz before DSP (a PR can reach ~60–65 Hz but with excursion risk). Max clean SPL is modest — two Xmax-limited 86.8 dB / 30 W 3" drivers give lively near/mid-field levels, not outdoor-party SPL. The DSP loudness/limiter is what makes it sound "genuinely good" rather than merely loud.

**Damping/fill:** line internal walls with ~10–20 mm acoustic foam/polyester; light fill for sealed (adds apparent volume); don't over-stuff near a PR.
**Baffle & mounting:** flush-mount both drivers; chamfer/round baffle edges (even 3–5 mm radius) to cut diffraction on the narrow baffle; keep the tweeter close to the woofer to limit lobing at the ~4 kHz crossover; brace and seal thoroughly — sealed alignment demands a genuinely airtight box.

---

## 5. Amplifier Optimization

**Power reality at 12 V.** The TPA3116D2 label ("50 W") assumes ~21–24 V. At 12 V into 4 Ω you get **~17 W/ch at usable THD** (*"At 12V, maximum undistorted sine-wave output into 4Ω is ~17W per channel (≈22V_pp swing)"*); TI's datasheet quotes 25 W only at 14.4 V / 10% THD. Into 8 Ω at 12 V you're closer to ~10 W and hit 1% THD getting there (TI E2E: at 12 V into 8 Ω *"you need to push past 1% THD to get to 10W"*). This is fine — the drivers are 30 W (woofer) / 15 W (tweeter) and Xmax-limited, so the DSP limiter engages well before amp clipping. Note the tweeter amp into 8 Ω will be especially power-limited as a 3S pack sags toward 9 V.

**Gain setting — reduce to 20 dB.** The ADAU1701 DAC outputs 0.9 Vrms full scale; TPA3116 modules ship at 32–36 dB, which clips almost immediately from 0.9 Vrms and amplifies hiss. Set both amps to 20 dB — also TI's low-noise reference point (datasheet: *"Output integrated noise, 20 Hz–22 kHz, A-weighted, Gain = 20 dB → 65 µV (–80 dBV)"*, SNR 102 dB). On common single-chip Chinese boards, gain is two SMD resistors (datasheet R1/R2, silkscreened near the chip):

- Many boards are 36 dB via 75 kΩ ("753") + 47 kΩ. Removing the 75 kΩ resistor(s) gives 20 dB master mode (*"Changing the gain to 20dB (as master) just requires the two 75K resistors (marked with '753') to be removed"*).
- TI's clean 20 dB is a single 5.6 kΩ gain resistor (*"replaced with 5.6k for first resistor and completely remove the second one"*). At 20 dB the input impedance is ~60 kΩ, so ~1.5 µF input caps suffice.
- On a 4-channel build from multiple modules, set all as master with identical gain resistors (a builder ran *"all three boards as master with a 5.6k gain setting"* at 20 dB with near-silent hiss).

**Noise reduction (cheap TPA3116 modules)** — from the long diyAudio TPA3116 thread:

- Do volume in the DSP; bypass/remove the onboard pot. If keeping it, fit ~4.7 kΩ across the pot output (*"set the 4K7 resistance at the output of the potentiometer... the hiss became barely noticeable"*).
- Set gain to 20 dB (biggest single hiss reduction).
- Match L/R input coupling caps (film preferred) to reduce turn-on pop; match values between channels.
- Verify/add Zobel networks and the LC output filter — some tiny boards omit output inductors entirely.
- Add bulk capacitance on the 12 V rail (1000–2200 µF low-ESR) close to each module.
- Star-ground analog inputs to DSP ground; join DSP and amp grounds at one point.

**DAC-to-amp interface.** The datasheet's passive DAC filter (Fig. 18: 47 µF series cap + 560 Ω + 5.6 nF, ~50 kHz corner) is a fine reconstruction filter. Feeding a TPA3116 module (which has its own input cap and input impedance) needs no heavy extra buffering — a series R+C into the module input is enough. ADI's CN-0162 reference (ADAU1701→SSM2306) uses *"0.10 μF capacitors and 13.0 kΩ resistors in series"* forming a 28 Hz high-pass; adapt the R+C to the module's input impedance for a ~20–30 Hz high-pass. **The ADAU1701 DACs are inverting** — invert in the DSP or account for it. Keep the DAC output loaded ≥2 kΩ; never feed raw 0.9 Vrms into a 36 dB input.

**Class-D LC filter vs speaker impedance.** The output LC filter interacts with the rising speaker impedance, giving a small load-dependent HF bump/rolloff. At this level, and with DSP correction available, it is minor — measure the final acoustic response and EQ it flat in the DSP rather than pre-compensating in hardware.

---

## 6. Known Pitfalls (verified) with fixes

- **DAC Setup register 2087 (0x0827) DS[1:0] = 01 is mandatory.** Datasheet: *"To properly initialize the DACs, Bits DS[1:0] in this register should be set to 01."* This register does not exist in the old preliminary datasheet — confirmed real, and a very common first-boot failure. Also set core control register (2076/0x081C) bits ADM/DAM/CR to un-mute. Skip these and outputs stay muted.
- **GPIO output drive is 2 mA in Rev C** (was 5 mA in the preliminary — a real revision change). Do not drive standard LEDs, relays, or the MERUS amp enable directly; use a transistor/MOSFET buffer. electro-dan.co.uk: *"since these pins can only drive 2mA - even an LED will need an external transistor or logic level MOSFET unless it's high brightness."*
- **GPIO "0.6 V logic low" / floats high at power-up** — confirmed real and widespread. MP pins power up as inputs and float high until the DSP program loads and reconfigures them; the reported ~0.6–1.2 V "lows" come from external pull circuitry, not a true logic low. ADI: *"all pins are configured as inputs upon powering up and as such it will float high."* Fix: add a ~47 kΩ pull-down (or pull-up for the desired idle) on any MP pin driving external logic, design for active-high control where possible, or use an inverting pre-biased transistor. This is exactly the trap that bites people driving amp mute/enable pins.
- **Rev C datasheet typos:** OUTPUT_BCLK is called "Pin 11" in the timing table vs Pin 19 (MP11) in the pin table; ADC0/ADC1 pin numbers were swapped between the 2006 preliminary and Rev C. Cross-check every pin against the Rev C pin-function table only.
- **Crystal must be fundamental-mode AT-cut, parallel-resonant, ~12.288 MHz.** Datasheet: *"the oscillator circuit should be an AT-cut, parallel resonator operating at its fundamental frequency."* A 24 MHz third-overtone crystal will NOT work. Use e.g. Abracon ABLS-12.288MHZ-B4-T with 22 pF load caps and a 100 Ω series (damping) resistor.
- **64×fs is the slowest startup** — 85.3 ms PLL lock + boot, up to ~260 ms total. Hold reset/mute accordingly.
- **EEPROM self-boot problems:** verify IOVDD (3.3 V) is actually connected — a documented build failed because the 3.3 V-to-IOVDD trace was missing, causing bizarre GPIO behaviour and no audio. SELFBOOT must be high at power-up; the part waits for MCLK first. Use the 24LC256 at the correct I2C address (0xA0 write / 0xA1 read).
- **Program exceeds 1024 instructions → silent mute.** Check `compiler_output.txt`.
- **Power-up sequencing / pops:** mute amps until the DSP is booted and un-muted; DC-block the DSP input; bring up 3.3 V before releasing RESET; keep the I2S clock running before de-asserting reset.
- **SigmaStudio version compatibility:** TCPi/USBi tools sometimes need a recent (or specific beta) SigmaStudio. Install the latest stable; if TCPi readback fails, try a newer build; back up project files (SigmaStudio projects are version-sensitive).
- **LQFP-48 0.5 mm-pitch hand-soldering:** plenty of no-clean flux; tack two opposite corners first to align; drag-solder with a fine chisel tip (or hot-air + paste); clear bridges with wick; verify every pin for shorts/continuity before powering. Work under magnification — unforgiving, especially on the power/PLL pins.

---

## 7. ESP32 Specifics

**Pin constraints (WROVER-IE).** GPIO16/17 are consumed by PSRAM — do NOT use them (reconfigure rarranzb's default I2C off 16/17). GPIO6–11 are flash — unusable. GPIO34–39 are input-only (good for ADC/sense). Strapping pins 0, 2, 5, 12, 15 need care at boot. **ADC2 does not work while WiFi is active** — use ADC1 for all analog sense.

**Recommended clean pin map** (verify against the WROVER-IE datasheet and the chosen firmware's defaults):

- **I2S to ADAU1701:** BCLK→GPIO26, LRCLK/WS→GPIO25, DOUT→GPIO22 (BCLK also physically wired to ADAU1701 MCLKI).
- **I2C to ADAU1701 (control):** SDA→GPIO21, SCL→GPIO4, with 2.2 kΩ pull-ups.
- **ADAU1701 RESET:** GPIO23.
- **microSD (SPI):** MOSI→GPIO13, MISO→GPIO27, SCK→GPIO14, CS→GPIO5 (or GPIO15 as output CS if not held low at boot).
- **4 battery/NTC ADC inputs (ADC1):** GPIO32, GPIO33, GPIO34, GPIO35 (34/35 input-only — ideal for sense).
- **Rotary encoder:** A→GPIO36 (VP), B→GPIO39 (VN), SW→GPIO18. 36/39 are input-only with no internal pull-ups — add external pull-ups (squeezelite-esp32 specifically notes 36/39 for its volume encoder).
- **2–3 buttons:** GPIO19, GPIO2 (strapping — ensure not held during boot), plus one spare.
- **WS2812B data:** GPIO12 (strapping — must be low at boot; the WS2812 idle line is low, so usually OK, but verify) or GPIO4 if not used for I2C.
- **Amp mute/enable:** a spare GPIO through a buffer transistor.

**BT/BLE/WiFi coexistence** (see §2): do not run Classic BT A2DP + WiFi together; pin BT and WiFi controllers to different cores; use mutually-exclusive audio-source modes with BLE GATT for control in all modes.

**BLE GATT control app.** WillyBilly06's repo already includes BLE GATT + DSP + level meters + WS2812B — the closest existing open-source starting point. Model the GATT service with characteristics for volume, per-band EQ gains, preset select, crossover frequency, and a read/notify characteristic for battery %/voltage. There's no dominant "ESP32 speaker app" standard — build a simple custom service or fork WillyBilly06's.

**Battery/power.** MAX17043 on I2C (**a single-cell gauge** — use it on one representative cell or a scaled measurement; per-cell resistor dividers into ADC1 give balance/health). Enforce the pack cutoff to protect the 18650s. Expect amp output to drop as the pack sags.

---

## 8. Recommendations (staged)

**Stage 0 — before soldering the DSP:**

1. Keep a 12.288 MHz AT-cut fundamental crystal + 22 pF caps + 100 Ω on a JP1-style jumper as clock fallback. Wire PLL_MODE0=PLL_MODE1=GND for 64×fs as primary.
2. Verify every breakout connection against the Rev C pin-function table only — MCLKI, MP5/INPUT_BCLK, IOVDD 3.3 V, PLL_MODE pins.
3. Study the decaVox and Elektor_AudioDSP schematics; copy their proven power-sequencing/reset/clock topology.

**Stage 1 — bench bring-up (12 V bench supply):**

4. Solder the LQFP-48 with flux + drag-solder; short-check all pins.
5. Bring up the ADAU1701 stand-alone first (crystal clock, analog AUX → DAC out); confirm pass-through with DAC setup reg 0x0827=01 and core reg un-muted. This isolates DSP from ESP32 issues.
6. Switch to ESP32-as-I2S-master, BCLK→MCLKI+INPUT_BCLK, 64×fs. Scope for a steady 3.072 MHz; confirm the DSP does not mute. Hold reset until the clock is up; mute amps ~300 ms.
7. Program via the rarranzb ESP32 TCPi WiFi bridge (reconfigure its I2C off GPIO16/17). Load a minimal crossover.

**Stage 2 — DSP program:**

8. Build: input DC-block → volume → LR4 @ 4.0 kHz → woofer HPF ~75 Hz → PEQ voicing → BSC (~1.1 kHz, +3–4 dB) → tweeter trim + polarity check → woofer & tweeter limiters → dynamic loudness. Check `compiler_output.txt` < 1024 every download.
9. Wire live control via MCUdude/SigmaDSP with safeload for all runtime changes.

**Stage 3 — amps & acoustics:**

10. Set both TPA3116 modules to 20 dB (remove 75 kΩ / fit 5.6 kΩ), bypass pots, match input caps, add bulk caps, verify LC output filters exist.
11. Interface DACs to amps via series R+C (~20–30 Hz high-pass), not raw.
12. Build a sealed box ~1.5–2 L per driver first; measure with REW; voice in the DSP. Attempt a PR only after you understand the driver's excursion behaviour — and if so, add washers to a PR displacing ≥12 cm³ to hit ~60 Hz, and keep the woofer HPF.

**Stage 4 — firmware integration:**

13. Use WillyBilly06 `ESP32-A2DP-SINK-WITH-CODECS-UPDATED` (PSRAM branch, ESP-IDF 5.5.2) for BT; force fixed 48 kHz I2S out.
14. Implement mutually-exclusive source modes (BT / WiFi-AirPlay / SD / AUX) + BLE GATT control in all modes. Never run Classic BT + WiFi at once; pin BT and WiFi to different cores.
15. Add battery monitoring (MAX17043 + dividers on ADC1) with a hard low-voltage cutoff.

**Benchmarks that change the plan:**

- PLL won't lock reliably in 64×fs on perfboard (intermittent mute; unstable 3.072 MHz on scope) → fall back to crystal for bench work, shorten/clean the clock traces, move the clock section to a small PCB.
- `compiler_output.txt` nears ~900 instructions → switch benign filters to single precision or cut PEQ count.
- BT audio stutters with BLE active → enlarge the PSRAM audio buffer, reduce BLE advertising/connection frequency, confirm core pinning.
- Bass distorts / cone bottoms → tighten the woofer HPF and lower the limiter threshold; the 2 mm Xmax is the hard limit.

---

## 9. Caveats

- The ADI "best solution" clock endorsement is a design recommendation, not a published measured build with jitter data; the strongest empirical proof is the shipping Elektor board using the same ESP32-master topology. The Elektor article's prose says "the ESP32's MCLK signal" via JP1 and does not spell out whether it routes BCLK or a separate MCLK, nor the exact PLL mode — verify the exact routing on its schematic PDF and confirm on the bench that the DSP does not mute.
- The 44.1 kHz → 2.8224 MHz case is ~8% below the nearest listed PLL point; it's within the ±20% window and ADI expressed confidence, but force 48 kHz everywhere so it never arises.
- Instruction-count estimates per block are approximate — trust `compiler_output.txt` over estimates.
- TPA3116 power/hiss behaviour varies board-to-board, and the gain-resistor designators (R1/R2 vs R2/R3) differ between clones — identify the actual gain resistors on the specific board against the TI datasheet before removing anything.
- Passive-radiator tuning numbers require modelling in WinISD with the actual PR's parameters; the Vd/mass figures here are starting estimates (PC83-4 Vd = 6.0 cm³; PR must displace ≥ ~12 cm³).
- Pin-map suggestions must be reconciled with the specific firmware's hard-coded defaults (especially squeezelite-esp32 and WillyBilly06's) and the WROVER-IE datasheet; treat strapping pins GPIO0/2/5/12/15 with care.
- GPIO drive/LED and "0.6 V low" details differ between the preliminary and Rev C datasheets; the Rev C 2 mA figure governs this build.

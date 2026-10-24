# LR2021 crystal temperature compensation: bench test plan

This plan is for testing `LR2021::setTempCompensation()` on a Meshnology W12, with an RTL-SDR Blog V3 as the main measuring instrument and a Heltec V4 as a helper. It is written so that a fresh session on the machine with the hardware can carry it out without any other context.

## 1. What is under test

The driver change is on branch `h3lix1/keen-mccarthy-mqhe9k`, commit `9e0332d`. This branch is cut from it.

```cpp
int16_t setTempCompensation(uint8_t mode, bool useNtc = false, float rRatio = 1.0f,
                            uint16_t betaK = 4250, uint8_t delay = 5);
```

- **What it sends:**
  - `SetTempCompCfg` (0x0132), plus `SetNtcParams` (0x0133) when `useNtc` is true.
  - `mode` is one of `RADIOLIB_LR2021_TEMP_COMP_DISABLED`, `_RELATIVE` or `_ABSOLUTE`.
- **Refusals:**
  - With a TCXO configured (`tcxoVoltage > 0`) it returns `RADIOLIB_ERR_INVALID_TCXO_VOLTAGE` (-703).
  - An invalid mode returns `RADIOLIB_ERR_UNSUPPORTED` (-25).
- **After every `begin()`:** call it again, because `begin()` hardware-resets the chip.
- **Already checked without hardware:**
  - The SPI bytes, using a mock HAL. The W12 values `(1.2f, 4250, 5)` send `01 33 02 66 08 4D 05` and then `01 32 05`.
  - Compiles: the native build, AVR Mega and ESP32-S3.
  - cppcheck.

What the chip does, according to Semtech's datasheet section 6.12 and app note AN1200.106:
- **Cfine:** an on-die sensor next to the crystal oscillator's tuning capacitors corrects the chip's own capacitance shift. It needs no external parts and is active whenever a mode is set.
- **NTC:** an external thermistor at the crystal feeds a built-in crystal model. On the W12 this is R21, 100 kΩ, B = 4250 K, biased by R22 = 120 kΩ, so the ratio is 1.2.
- **Relative mode:** corrects the temperature change between the start and the end of a packet. Semtech prefers it.
- **Absolute mode:** corrects any measured difference, but Semtech warns it is more sensitive to errors in the temperature reading.
- **Transmit only:** the datasheet describes the block as acting during transmission.

Semtech published only one measurement: drift over a packet reduced by 39% with Cfine alone and 43% with Cfine and NTC, at SF12/62.5 kHz on the LR2021's own PA at up to +22 dBm. The W12 drives an external GC1109 PA to about 1 W. Its drift has never been measured.

### Questions this plan answers

1. How fast does the W12's carrier drift during a long transmission at full power, in Hz/s and in Hz over 1, 2, 4, 8 and 12 s?
2. How much does each mode reduce that drift: Relative or Absolute, with or without NTC?
3. Which NTC settings work best (`delay`, ratio, beta)? Does the ratio field mean bias/NTC (encoded 614) or NTC/bias (encoded 427)?
4. Does the setting survive `sleep(true)`, or does Meshtastic need to send it again on every wake? We already expect it to be lost on `begin()`.
5. Does the compensation work during CW test mode (`transmitDirect()`), or only during packets?
6. Does it change packet loss on a link that is limited by drift?

## 2. Equipment

| Item | Role |
|---|---|
| Meshnology W12: ESP32-S3, LR2021 with a plain 32 MHz crystal, GC1109 sub-GHz PA, NTC at the crystal | The board under test (transmitter) |
| RTL-SDR Blog V3: R820T2 tuner, 1 ppm TCXO | Main instrument: records the carrier frequency over time |
| Heltec V4: ESP32-S3, SX1262 with a 1.8 V TCXO, GC1109 (V4.2) or KCT8103L (V4.3) front end | (a) a TCXO-stable reference tone to measure the SDR's own drift; (b) the receiver for the packet-loss test |
| Attenuation: about 10 dB is on the W12 at the moment | See the safety section |

## 3. Safety: read this before transmitting

- **The RTL-SDR input.**
  - At full power the W12 puts out roughly +28 to +30 dBm. Through only a 10 dB pad, about +19 dBm reaches the next device.
  - The RTL-SDR's front end is damaged at around +10 dBm and clips far below that. Never connect the W12 to the SDR through only 10 dB.
  - With only the 10 dB pad available, run the W12 into an antenna or a dummy load and pick the signal up over the air. Set the SDR's gain to 0 dB and add distance, or remove the SDR's antenna and let it pick up leakage.
  - For a cabled connection, use about 60 to 80 dB in total, with the first stage rated for at least 2 W.
  - Aim for −40 to −60 dBm at the SDR with no clipping.
- **Power ramp.** Start at the lowest W12 power setting (−9) and raise it in steps. At each step check the SDR's peak level; the host script should refuse to continue if the samples clip.
- **Never transmit from the W12 without a load.** The GC1109 PA is always in the RF path.
- **Keep the pad off the W12's connector.** The pad absorbs about 90% of the W12's output, roughly 0.9 W, and gets hot.
  - It must be rated for at least 2 W.
  - Connect it through a short length of coax, not screwed directly onto the W12's SMA connector. Otherwise it heats the W12 board through the connector, which is the very thing being measured.
- **Leave the RTL-SDR's bias tee off** (this is the default), so it doesn't put DC on the RF line.
- **Keep the Heltec V4's input below about −20 dBm.** Its front end includes an LNA.
- **Keep runs short or into a dummy load.** Long continuous carriers at about 1 W are a test condition, not normal operation.

## 4. Board details

These are taken from Meshtastic's `variants/esp32s3/meshnology-w12/variant.h` and `variants/esp32s3/heltec_v4/variant.h`.

| | W12 (LR2021) | Heltec V4 (SX1262) |
|---|---|---|
| SPI pins | SCK 9, MISO 11, MOSI 10, CS 8 | SCK 9, MISO 11, MOSI 10, CS 8 |
| IRQ, reset, busy | IRQ on GPIO14, which is radio DIO8, so set `radio.irqDioNum = 8` before `begin()` (RadioLib's default of 5 is not wired); NRESET 12; BUSY 13 | DIO1 14, RST 12, BUSY 13 |
| Oscillator | Crystal: `begin(..., tcxoVoltage = 0)` | TCXO: `begin(..., 1.8)`, and `setDio2AsRfSwitch(true)` |
| Front end | The GC1109 is always in the path. The radio drives its RF switching, and the PA enable (IO3) is pulled high, so no pins need driving. Meshtastic doesn't drive GPIO4 either; its variant file names IO4 as the 2.4 GHz PA enable, also pulled high. RadioLib accepts LR2021 power settings from −9 to +22, which gives roughly +28 to +30 dBm out at the top (not measured). | Set GPIO7 HIGH to power the front end, then read GPIO2 as an input. **HIGH means V4.3 (KCT8103L):** set GPIO2 HIGH; set GPIO5 LOW for the LNA, or HIGH to bypass the LNA for strong signals. **LOW means V4.2 (GC1109):** set GPIO2 HIGH and GPIO46 LOW. |
| Other | OLED supply GPIO45 is active LOW; leave it HIGH (off) to cut heat. The USB port is the ESP32-S3's native USB CDC. | Vext GPIO36 is active LOW. |

Use LoRa sync word 0x12 (private) so that Meshtastic traffic can't mix in. `f0` below means the test frequency for your region, for example 915.0 or 869.525 MHz.

## 5. Measurement method

### SDR capture
- Tune the SDR 250 kHz below `f0`, so the SDR's DC spike falls outside the signal.
- Sample at 1.024 MS/s, with manual gain and AGC off. Shift digitally to `f0` and decimate to 250 kS/s for bandwidths up to 125 kHz. For 250 kHz LoRa, keep 1.024 MS/s.
- Record continuously during each trial, plus 1 s before and after. Find the transmission by its power.
- Save the frequency trace for every trial. Save raw IQ only for the first trial of each configuration, as a spot check.

### Measuring frequency over time
- **CW (`transmitDirect()`):** the frequency over a window is the change in unwrapped phase divided by 2π times the window length. Use 10 ms windows.
- **LoRa:** a chirp's instantaneous frequency averages to zero over any whole number of symbols. So the same formula, applied to windows exactly N symbols long, gives the carrier offset with the chirp removed. This assumes the modulator's phase is continuous; Phase 0, step 5 checks it on real hardware. For example, 50 symbols at SF7/125 kHz is a 51.2 ms window.
  - This works whatever the window's timing relative to the preamble, because the preamble is periodic.
  - For the payload, align the windows to the symbol boundaries found from the preamble. Each data symbol is still one complete cyclic sweep.
  - The phase only unwraps correctly if the sample rate is above 2 × (BW/2 + the largest offset). The W12's static error is ±10 ppm, about ±9 kHz.
- **Main stimulus: a LoRa packet with a long preamble.**
  - The preamble lasts P seconds, up to 12 s. 12 s is 11,719 symbols at SF7/125 kHz, within the 65,535-symbol limit, and about the airtime of a full-size Meshtastic packet at SF12/125 kHz with coding rate 4/8.
  - Because it is a single real packet, Relative mode is exercised exactly as Semtech defines it.
  - One 12 s trial gives the drift at every point from 0 to 12 s, so no separate sweep over packet length is needed.
  - The W12 transmit sketch must use `startTransmit()` and wait for the IRQ, with a timeout longer than the packet.
- **CW** is used only to learn whether compensation acts in test mode (question 5).

### The SDR's own drift
- The RTL-SDR's TCXO is rated at 1 ppm. After a 20-minute warm-up, its drift over a 12 s capture should be negligible. Phase 0.3 measures it.
- If it wanders by more than a few Hz over 10 minutes, use the V4 as a pilot tone in the long runs:
  - The V4 transmits CW at `f0 + 150 kHz` at low power, outside the LoRa signal's bandwidth.
  - Subtract the pilot's measured movement from the W12 trace.

### Why the SDR is the main instrument rather than the V4's frequency-error reading
- The SX1262 returns one frequency-error value per packet, taken at the start of the packet. It can't show drift within a packet.
- RadioLib also notes that this register is undocumented on the SX126x.

## 6. Configurations

| Name | Call (after every `begin()`) |
|---|---|
| Off | none |
| Rel | `setTempCompensation(RADIOLIB_LR2021_TEMP_COMP_RELATIVE)` |
| Rel+NTC | `setTempCompensation(RADIOLIB_LR2021_TEMP_COMP_RELATIVE, true, 1.2f, 4250, 5)` |
| Abs | `setTempCompensation(RADIOLIB_LR2021_TEMP_COMP_ABSOLUTE)` |
| Abs+NTC | `setTempCompensation(RADIOLIB_LR2021_TEMP_COMP_ABSOLUTE, true, 1.2f, 4250, 5)` |

Rules for every trial:
- **Order:** rotate through the configurations and shuffle the order within each round, so slow changes in the room don't favour one mode.
- **Cool-down:** between trials, wait until the W12's on-chip crystal-area temperature, `getTemperature(RADIOLIB_LR2021_TEMP_SOURCE_XOSC)`, is within 0.3 °C of the session's starting value. Wait at least 20 s and at most 180 s.
- **When to read temperature:** only while the radio is in standby, before and after each transmission.
- **Keep conditions constant:** same position, no fans switching on and off, same USB power, and the W12's Wi-Fi, Bluetooth and OLED off.

## 7. Phases

### Phase 0: set up and validate (about 1.5 h, including warm-up)

1. **Check the method on simulated data, before using any hardware.**
   - Generate LoRa preamble IQ with an 8 ppm static error and about 0.1 ppm/s of drift. At 915 MHz that is roughly 90 Hz/s.
   - Run the symbol clock from the same simulated crystal.
   - Add noise for 10 dB SNR and round the samples to 8 bits.
   - Do the same with random data symbols.
   - The 50-symbol estimator should recover the drift to within about 1 Hz rms per window. If it doesn't, fix the estimator first.
2. **Check signal levels.**
   - Start with the W12 at power −9 and the SDR gain fixed.
   - Raise the power in steps up to 22, checking that the SDR's peak sample level stays below about 70% of full scale.
   - Record the final attenuation and SDR gain, and keep them the same for all later phases.
3. **Measure the SDR's stability.**
   - Warm the SDR up for 20 minutes.
   - Record the V4 transmitting a low-power CW tone for 10 minutes. On a V4.3, bypass the PA if possible to minimise heating.
   - Report the drift in Hz/s and the total wander. This decides whether a pilot tone is needed (section 5).
4. **Check the scale and sign.**
   - At low power, transmit W12 CW at `f0`, `f0 ± 2 kHz` and `f0 ± 5 kHz`. The measured steps should match.
   - Use steps of 2 kHz or more: `setFrequency(float MHz)` only resolves about 61 Hz near 900 MHz.
5. **Check that CW and LoRa agree.**
   - At full power with compensation Off, run 3 × 12 s of CW and 3 × 12 s of long-preamble LoRa.
   - The drift curves should overlay each other. This validates the LoRa estimator on real hardware.
6. **Basic checks.**
   - All five `setTempCompensation()` calls return 0.
   - Log the crystal-area temperature at rest, and the W12's static offset in ppm (expected within ±10 ppm).

### Phase 1: main comparison (about 1 h)

- **Setup:** full power (LR2021 setting 22), long-preamble LoRa at SF7/125 kHz with P = 12 s, all five configurations, 10 repeats each, rotated.
- **Results per configuration:**
  - the mean drift curve with a 95% confidence band
  - the drift at 0.5, 1, 2, 4, 8 and 12 s, in Hz and ppm
  - the drift rate over the first second
  - the drift expressed in SF12 frequency bins (BW/4096: 30.5 Hz at 125 kHz, 15.3 Hz at 62.5 kHz)
  - the % reduction compared with Off
- **CW check:** repeat with CW for the same five configurations, 3 repeats each, to answer question 5.
- **What to look for:**
  - Is a mode's reduction outside the noise, with its confidence interval not overlapping Off's?
  - NTC over-correcting, which shows as drift reversing sign later in the packet. Semtech's own result showed this.

### Phase 2: power sweep (about 45 min)

- LR2021 power settings 0, 10, 16 and 22. If the W12's PA gain is similar to the V4's, that is roughly 11, 21, 26 and 29 dBm out; measure it if a power meter is available.
- Off against the best configuration from Phase 1, 5 repeats, P = 12 s.
- This shows whether a Meshtastic default should depend on transmit power.

### Phase 3: NTC tuning (about 1 h)

- Use the better NTC mode from Phase 1, at full power with P = 12 s and 5 repeats.
- Settings to try:
  - `delay` = 1, 3, 5, 8 and 12
  - `rRatio` = 1.2 (encoded 614) and 0.833 (encoded 427). This answers question 3: the wrong way round should visibly over- or under-correct.
  - `betaK` = 4250 (B25/50) and 4300 (B25/85)
- Keep the flattest result, and report the chosen values for the Meshtastic variant file.

### Phase 4: stress

- **4a: sustained transmission.**
  - Send 12 s packets back to back for 10 minutes at full power, which keeps the transmitter on about 95% of the time.
  - Run this for Off, Rel, Abs and the best NTC configuration, with 15 minutes of cool-down between runs.
  - Use the V4 pilot tone if Phase 0.3 says it is needed.
  - Look for:
    - the start frequency creeping as the whole board warms, which can amount to ppm over minutes (Relative is not expected to fix this; Absolute might)
    - whether drift within each packet stays corrected on a hot board
    - any transmit errors or timeouts.
- **4b: Meshtastic-like duty cycle.**
  - Packets similar to LONG_FAST (SF11/250 kHz, about 1 s each) at 25% duty for 10 minutes, sampled at 1.024 MS/s.
  - Measure the drift within each packet using windows aligned to the payload's symbols.
- **4c: hot and cold.**
  - Put the W12 in a closed box at 50 to 60 °C, or test straight after 4a. Use a fridge for the cold case if practical.
  - Repeat Phase 1 for Off and the best configuration.
  - Watch Absolute mode in particular, since Semtech says it is sensitive to temperature-reading errors.
- **4d: what survives sleep and reset (question 4).**
  - Set compensation once, then before each trial call `sleep(true)` and wake the radio.
  - Compare against calling `setTempCompensation()` again after every wake.
  - Also confirm that `begin()` without a new call behaves like Off.
- **Throughout:** log every non-zero RadioLib return code, every transmit timeout and every unexpected reset.

### Phase 5: does it change packet loss? (uses the V4, about 1.5 h)

- **Wiring:** W12 → attenuation → 2-way splitter → SDR on one output and V4 on the other, with extra attenuation on the V4 side so it sees −60 to −90 dBm. Without a splitter, the V4 can receive over the air.
- **Push bandwidth down until drift breaks the link.**
  - Keep the signal strong and step the bandwidth down, so that drift rather than noise is what fails: SF12 at 62.5 kHz → 31.25 kHz → 15.6 kHz.
  - Stop at the first bandwidth where Off loses packets.
  - Tune the V4's receive frequency to the W12's static offset as measured by the SDR. That way offset, which at 15.6 kHz is only ±3.9 kHz of tolerance, is ruled out as the cause.
- **Volume:** 100 packets per configuration, in alternating blocks of 20.
- **Logging:** the V4 records CRC result, RSSI and SNR for each packet; the SDR records each packet's drift. Then correlate the failures with the measured drift.

## 8. Analysis and reporting

- **Per trial:** a CSV of time against frequency offset, plus metadata (configuration, power, P, temperatures before and after, return codes, SDR gain).
- **Summary:**
  - a table of the Phase 1 results with confidence intervals
  - plots of drift against time for each configuration
  - the power sweep, NTC tuning and stress plots
  - start frequency against temperature for the 4a runs
- **Interpreting the drift rate** (915 MHz, using the datasheet's limits for under 2 dB loss at 125 kHz: 200 Hz/s at SF10, 100 Hz/s at SF12 with low-data-rate optimisation):
  - under 100 Hz/s: within spec for every preset, so compensation is a refinement
  - 100 to 300 Hz/s: SF12 presets are degraded, so compensation is needed
  - over 300 Hz/s: compensation alone is not enough; reduce power or improve the thermal path.
- **For pull-request text:** quote the measured reduction, with Semtech's 43% only as a reference point. Cite app notes by section number only; don't copy their figures, as they are marked confidential.

## 9. Tools to build (in this directory)

| Path | What it does |
|---|---|
| `W12_TxRig/W12_TxRig.ino` | The W12 transmitter, controlled over serial (protocol below) |
| `V4_Helper/V4_Helper.ino` | Pilot tone and packet-loss receiver for the V4 (protocol below) |
| `host/sim_check.py` | The simulated-data check from Phase 0, step 1 |
| `host/run.py` | Runs the test matrix: drives the W12 and V4 over serial, controls the SDR (pyrtlsdr), shuffles the order, waits on temperature for cool-down, writes the trial files |
| `host/analyze.py` | Builds the tables and plots from the trial files |

Build the sketches against this checkout. For example:

```
arduino-cli compile --fqbn esp32:esp32:esp32s3:CDCOnBoot=cdc --library <repo root> W12_TxRig
```

Check how each board shows up over USB and choose board options to match. The W12 uses the ESP32-S3's native USB CDC, 16 MB flash and OPI PSRAM.

Host dependencies: librtlsdr / rtl-sdr tools, and Python packages `numpy`, `scipy`, `matplotlib`, `pyserial` and `pyrtlsdr`.

### W12_TxRig serial protocol

Commands are lines of text; every reply line starts with a tag so it is easy to parse.

```
radio <f0_MHz> <bw_kHz> <sf> <cr> <power>
    -> OK radio <code>
    Calls begin() with tcxoVoltage=0, irqDioNum=8, sync word 0x12,
    then re-applies the last 'comp' setting.

comp off|rel|abs [ntc <ratio> <betaK> <delay>]
    -> OK comp <code>
    Calls setTempCompensation(); the setting is kept for later 'radio' commands.

temp
    -> TEMP <xosc_C> <vbe_C>

lora <preamble_ms> <payload_len>
    -> TXSTART <ms>
    -> TXDONE <ms> <code> <xosc_before> <xosc_after>

cw <duration_ms>
    -> TXSTART <ms>
    -> TXDONE ...
    Uses transmitDirect(); returns to standby afterwards.

burst <count> <payload_len> <gap_ms>
    Packets for the duty-cycle tests: one TXSTART/TXDONE pair per packet.

sleep <ms>
    -> OK sleep <code>
    Calls sleep(true) then standby(), for test 4d.

freq <MHz>
    -> OK freq <code>
```

### V4_Helper serial protocol

```
On boot: detect the front end (section 4) and print FEM V4.2|V4.3.

pilot <f_MHz> <power>   -> OK pilot <code>
    CW from the TCXO-locked SX1262.

pilot off               -> OK pilot 0

rx <f_MHz> <bw_kHz> <sf> <cr>
    -> PKT <ms> <len> <crc_ok> <rssi> <snr> <fei_Hz>
    Continuous receive, one PKT line per packet.
```

## 10. Known unknowns

- **CW test mode:** whether compensation acts during `SetTxTestMode` CW. Phase 1 answers this.
- **Sleep retention:** whether the 0x0132/0x0133 settings survive `sleep(true)`. Phase 4d answers this.
- **NTC filter:** the units of `delay` and the filter's time constant are not documented. Semtech's value of 5 (about 100 ms) is for its own reference board.
- **NTC temperature readout:** RadioLib's `getTemperature()` converts readings with the formula for the on-chip sensor, so an NTC reading through it (source 0x20) needs its own conversion. Phase 3's ratio A/B answers the question of which way round the ratio field works without needing that readout.
- **The W12's real output power** at each setting is not measured. Note it if a power meter is available.

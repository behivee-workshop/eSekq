Hardware + firmware design doc · Rev A

# eSekq

An ESP32-based MIDI thru-box with a loop recorder: it sits between your keyboard and a chained line of synths, keeping time and letting you capture a pass over the top.

Keyboard→ **eSekq**→ Synth 1→ Synth 2…N

01

## Overview

eSekq is a single-board ESP32 project that lives in the MIDI cable between a controller keyboard and the first synth in a chain. In **thru** mode it's transparent — everything played arrives at the synths with no added latency. In **rec** mode it keeps forwarding live, but also captures the pass into a one-shot loop buffer, which it can then play back over the same output at a BPM you set on the encoder.

There's no mixing, no multi-track, no DAW transport — one buffer, one tempo, two buttons. That scope is deliberate; see [§02](#assumptions) for what's been left out and why.

02

## Signal path & design assumptions

Keyboard feeds eSekq's MIDI IN only. eSekq's single MIDI OUT drives the first synth; downstream synths are daisy-chained through their own MIDI THRU ports — eSekq does not fan out to more than one device itself.

### Reading the ambiguous bits this way

- **Rec / thru toggle** — one capture axis: **thru** passes input straight through untouched; **rec** passes it through *and* writes it into the loop buffer, overwriting whatever was captured before.
- **Play / pause button** — a separate transport axis: starts or stops the internal MIDI clock and the loop playback. It's independent of the capture toggle, so you can record while paused (building a part before the loop ever plays) or let the loop run while switching back to thru for live playing on top.
- **One take, no overdub** — a new recording replaces the old one rather than layering. Simplest thing that's actually useful solo; add multi-take/overdub if the live-looper use case shows up.
- **Downstream fan-out** — relies on each synth having a hardware MIDI THRU jack. A synth without one needs an external thru box; eSekq doesn't grow a multi-out splitter for this.
- **No persistence** — the loop buffer is RAM only and clears on power-down. Add SPIFFS/SD storage if loops need to survive a reboot.

03

## Bill of materials

| Ref | Part | Qty | Notes |
| --- | --- | --- | --- |
| U1 | ESP32 DevKitC (WROOM‑32) | 1 | Plain WROOM, not WROVER — keeps GPIO16/17 free (see §04 footnote). |
| U2 | SSD1306 OLED, 128×64, I²C | 1 | 0.96″ module, 4-pin (VCC/GND/SDA/SCL). |
| U3 | 6N138 optocoupler | 1 | MIDI-spec recommended part for the IN isolation. |
| J1 / J2 | 5-pin DIN jack, panel mount | 2 | One IN, one OUT. |
| ENC1 | Rotary encoder with push (EC11-type) | 1 | BPM entry + confirm. |
| SW1 | Tactile button — Play/Pause | 1 |  |
| SW2 | Tactile button — Rec/Thru | 1 | Long-press clears the loop buffer — reuses the one switch instead of adding a third. |
| D1 | 1N4148 diode | 1 | Reverse-protects the opto LED on MIDI IN. |
| R1, R3, R4 | 220Ω resistor | 3 | Current-limit: MIDI IN loop, MIDI OUT loop ×2. |
| R2 | 4.7kΩ resistor | 1 | Opto-output pull-up, referenced to 3V3 (not 5V — see §05). |
| C1 | 100nF ceramic capacitor | 1 | Decouples opto VCC. |
| — | 5V supply, ≥500mA | 1 | USB or barrel jack. Powers the ESP32's own 3V3 regulator, the opto VCC, and the MIDI OUT current loop. |

04

## ESP32 pinout

| GPIO | Function | Connects to |
| --- | --- | --- |
| 17 | TX2 (UART2) | MIDI OUT, via R4 |
| 16 | RX2 (UART2) | MIDI IN, from opto pin 7 / R2 node |
| 21 | SDA | OLED |
| 22 | SCL | OLED |
| 32 | Encoder A | ENC1 |
| 33 | Encoder B | ENC1 |
| 25 | Encoder push | ENC1 |
| 26 | Button | SW1 — Play/Pause |
| 27 | Button | SW2 — Rec/Thru |
| 2 | Activity LED | Onboard LED, reused — blinks on MIDI traffic |

UART2's default pins (16/17) sit on the external PSRAM bus on WROVER modules. On a WROVER board, remap UART2 to any free pair — e.g. GPIO4/GPIO5 — via `Serial2.begin(31250, SERIAL_8N1, RX, TX)`; the ESP32's UART is routed through the GPIO matrix, so this costs nothing but a constant.

05

## MIDI interface circuits

Both circuits follow the MIDI 1.0 electrical spec's standard reference design. The one adaptation for a 3.3V MCU: the opto's output pull-up (R2) returns to the ESP32's **3V3** rail rather than 5V, so RX2 never sees more than 3.3V — no separate level shifter needed. The opto's input side and VCC still want the full 5V for the LED's rated current.

MIDI IN — opto-isolated per spec. DIN pin 4 sources current through R1 into the opto LED (pin 2); DIN pin 5 returns it (pin 3). D1 clamps reverse spikes across the LED. The output transistor (pin 7) is pulled to 3V3 through R2 and feeds RX2 directly.

MIDI OUT — the classic two-resistor current loop. R3 sources from 5V into DIN pin 4; TX2 sinks it through R4 into DIN pin 5. DIN pin 2 is the shield, grounded here. Driving this loop straight off a 3.3V push-pull pin is standard practice at MIDI's slow 31.25kbaud; swap in a 74HCT14 buffer if you're feeding a long cable run or several thru boxes.

06

## Controls & display

Idle screen: tempo, capture mode, and transport state at a glance, plus an activity dot for MIDI traffic.

Rotate encoder

±1 BPM per detent, 20–300 range.

Press encoder

Reserved for a settings menu — no-op for now.

Play / Pause

Starts or stops the internal clock + loop playback.

Rec / Thru — tap

Toggles capture mode.

Rec / Thru — hold 800ms

Clears the loop buffer.

07

## Firmware architecture

Transport: paused

Transport: playing

Capture: thru

Silent passthrough. Nothing recorded, nothing looping.

Loop plays into the chain; live playing passes through on top, unrecorded.

Capture: rec

Playing is captured and forwarded live — building a part before the loop ever plays.

New notes overwrite the loop in place while the old loop keeps playing underneath.

Two independent axes, not one combined mode — capture and transport can be in any of the four combinations.

### Main loop, in order

1\) Drain MIDI IN. In **thru**, forward each message immediately. In **rec**, forward it *and* append it (status, data1, data2, elapsed-ms) to the loop buffer.

2\) If **playing**: emit a clock byte (`0xF8`) every tick interval (60,000,000 ÷ BPM ÷ 24 µs, 24 ppqn), and emit any buffered events whose timestamp has come due, wrapping at the recorded loop length.

3\) Poll the encoder delta and the three debounced buttons; update BPM / capture / transport state.

4\) Redraw the OLED only when something on screen actually changed — skips most I²C traffic on an otherwise idle loop.

### Libraries

| Job | Library | Notes |
| --- | --- | --- |
| MIDI framing | FortySevenEffects/MIDI | On `Serial2` (HardwareSerial), 31250 baud. |
| OLED | Adafruit_SSD1306 + Adafruit_GFX | I²C at the default 0x3C address. |
| Encoder, buttons, clock, loop buffer | Hand-rolled | Interrupt-driven quadrature decode, millis()-based debounce, a flat struct array in RAM. Each is a few dozen lines — not worth a dependency. |

08

## Not yet specified

Enclosure and panel layout, MIDI channel filtering (currently assumed omni thru), and whether the loop buffer's few-thousand-event capacity needs to be user-visible. None of these change the circuit or the pinout above — worth pinning down before fabrication, not before a breadboard.

eSekq · design doc rev A ESP32 · MIDI thru + 1-take looper
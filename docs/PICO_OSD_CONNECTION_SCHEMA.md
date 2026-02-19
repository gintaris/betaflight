# PICO OSD Connection Schema for Madflight FC3

## Overview

Betaflight PICO OSD implementation (FB_OSD - Framebuffer OSD) uses RP2350's PIO (Programmable I/O) to overlay OSD graphics onto analog video signal. This requires 3 GPIO pins and external video mixer circuitry.

## PIO Pixel Encoding

The FB_OSD uses 2-bit per pixel encoding with these states:

| W_PIN | EN_PIN | Result | Description |
|-------|--------|--------|-------------|
| X | 0 | Transparent | Video passes through unchanged |
| 0 | 1 | Black | OSD draws black pixel |
| 1 | 1 | White | OSD draws white pixel |

## CRITICAL: Sync Signal Requirements

The PIO program expects **ACTIVE HIGH** sync detection:
- **HIGH (3.3V)** during sync pulses (when video sync is LOW/0V)
- **LOW (0V)** during active video

This means the sync signal must be **INVERTED** from the video source!

Video composite sync is ~0V during sync, ~0.3-1V during video.
RP2350 GPIO needs >2V for HIGH detection.

**Therefore: Simple RC coupling WILL NOT WORK! Active circuitry required.**

## Required GPIO Pins (Must be consecutive!)

| Signal | Function | Direction | Description |
|--------|----------|-----------|-------------|
| OSD_W_PIN | White/Black | Output | Controls OSD pixel brightness (white=1, black=0) |
| OSD_EN_PIN | Enable | Output | Enables OSD overlay (1=show OSD, 0=pass video) |
| OSD_SYNC_PIN | Sync | Input | Receives **INVERTED** composite sync |

**Important:** These 3 pins MUST be consecutive GPIO numbers (e.g., PA22, PA23, PA24).

## Suggested Pin Assignment for Madflight FC3

Using available PINIO pins on the third row of external connectors:

```
OSD_W_PIN    = PA22  // PINIO1
OSD_EN_PIN   = PA23  // PINIO2  
OSD_SYNC_PIN = PA24  // PINIO3
```

These pins are already broken out and easily accessible.

## Hardware Connection Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MADFLIGHT FC3 (RP2350B)                               │
│                                                                                 │
│   PA22 (OSD_W)  ───────────────────────────────► To Video Mixer                 │
│   PA23 (OSD_EN) ───────────────────────────────► To Video Mixer                 │
│   PA24 (OSD_SYNC) ◄─────────────────────────────  From Sync Inverter (3.3V!)    │
│   GND ──────────────────────────────────────────► Common Ground                 │
│   3.3V ─────────────────────────────────────────► Sync Inverter Power           │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                           OSD EXTERNAL CIRCUITRY                                │
│                                                                                 │
│   ┌──────────┐     ┌─────────────────┐     ┌──────────────────┐                 │
│   │  CAMERA  │     │ SYNC SEPARATOR  │     │   VIDEO MIXER    │     ┌───────┐  │
│   │          │     │    + INVERTER   │     │  (Resistor DAC)  │     │  VTX  │  │
│   │ Video Out├──┬──►    LM1881 +     │     │                  │     │       │  │
│   │          │  │  │  NPN Inverter   │     │ W_PIN ──[470Ω]───┤     │       │  │
│   └──────────┘  │  │                 │     │                  │     │       │  │
│                 │  │  Csync ─► Inv ──┼─────► EN_PIN ──[1kΩ]───┼─────► Video │  │
│                 │  │                 │     │                  │     │   In  │  │
│                 │  └─────────────────┘     │ Vid In ──[75Ω]───┤     │       │  │
│                 │                          │                  │     └───────┘  │
│                 └──────────────────────────►                  │                 │
│                                            └──────────────────┘                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Option 1: LM1881 + NPN Inverter (Recommended)

This is the most reliable solution with proper sync detection.

### Complete Bill of Materials

| Ref | Component | Value | LCSC Part # | Price | Description |
|-----|-----------|-------|-------------|-------|-------------|
| U1 | LM1881M/TR | - | C518922 | €0.57 | Sync Separator (SOIC-8) |
| U1 alt | LM1881N | - | C518923 | €0.59 | Sync Separator (DIP-8) |
| Q1 | 2N3904 | NPN | C18536 | €0.02 | Sync inverter transistor |
| Q1 alt | BC547B | NPN | C427315 | €0.02 | Alternative NPN |
| R1 | Resistor | 680kΩ | C103420 | €0.01 | LM1881 timing filter |
| R2 | Resistor | 4.7kΩ | C99782 | €0.01 | Q1 collector pull-up |
| R3 | Resistor | 10kΩ | C98220 | €0.01 | Q1 base resistor |
| R4 | Resistor | 470Ω | C114660 | €0.01 | W_PIN to video |
| R5 | Resistor | 1kΩ | C17513 | €0.01 | EN_PIN to video |
| R6 | Resistor | 75Ω | C174231 | €0.01 | Video termination |
| C1 | Capacitor | 100nF | C1591 | €0.01 | LM1881 input coupling |
| C2 | Capacitor | 100nF | C1591 | €0.01 | Power decoupling |

**Total cost: ~€1.30**

### Schematic

```
                              +5V (from camera/VTX power)
                               │
                       ┌───────┴───────┐
                       │    [100nF]    │
                       │      C2       │
                       │       │       │
    ┌──────────────────┼───────┼───────┼──────────────────┐
    │                  │       │       │                  │
    │            ┌─────┴───────┴───────┴─────┐            │
    │            │                           │            │
    │            │   ┌───────────────────┐   │            │
    │            │   │      LM1881       │   │            │
    │            │   │                   │   │            │
    │   Video ───┼───│1 VIN        VCC 8│───┼─── +5V      │
    │   In       │   │                   │   │            │
    │  ─[100nF]──┘   │2 GND       CSYNC 7│───┼──┐         │
    │     C1         │                   │   │  │         │
    │                │3 RSET    B.PORCH 6│   │  │         │
    │                │   │               │   │  │         │
    │              [680k]│4 VREF   O/E  5│   │  │         │
    │                │R1 │               │   │  │         │
    │                │   └───────────────┘   │  │         │
    │               GND                      │  │         │
    │                                        │  │         │
    └────────────────────────────────────────┼──┼─────────┘
                                             │  │
                                             │  │ LM1881 CSYNC
                                             │  │ (Active LOW during sync)
                                             │  │
              ┌──────────────────────────────┘  │
              │                                 │
              │    SYNC INVERTER (NPN)          │
              │    ════════════════════         │
              │                                 │
              │         +3.3V                   │
              │           │                     │
              │         [4.7k] R2               │
              │           │                     │
              │           ├─────────────────────┼────► PA24 (OSD_SYNC_PIN)
              │           │                     │      (Active HIGH during sync)
              │          |/  Q1                 │
              └──[10k]──|    2N3904             │
                   R3    |\                     │
                          │                     │
                         GND                    │
                                                │
                                                │
              VIDEO MIXER (Resistor DAC)        │
              ══════════════════════════        │
                                                │
    PA22 (W_PIN) ────[470Ω]───┐                 │
                      R4      │                 │
                              ├─────────────────┼────► VIDEO OUT ──► VTX
    PA23 (EN_PIN) ───[1kΩ]────┤                 │
                      R5      │                 │
                              │                 │
    VIDEO IN ────────[75Ω]────┤                 │
    (from camera)     R6      │                 │
                              │                 │
                             GND                │
```

### How It Works

#### Sync Detection:
1. **LM1881** extracts composite sync from video (output is active LOW)
2. **NPN transistor** inverts the signal:
   - When CSYNC = LOW (sync pulse): Q1 OFF → Output pulled HIGH by R2 → **3.3V**
   - When CSYNC = HIGH (active video): Q1 ON → Output pulled LOW → **0V**
3. **RP2350 PIO** sees proper 3.3V logic levels

#### Video Mixing:
1. When **EN_PIN = 0**: No current flows, video passes through R6 unchanged
2. When **EN_PIN = 1, W_PIN = 0**: Current sinks to GND → video level drops → **BLACK**
3. When **EN_PIN = 1, W_PIN = 1**: Current sources from 3.3V → video level rises → **WHITE**

### Timing Diagram

```
Video Signal:      0V ─┐    ┌─── 0.3V ────────────────────────── 1V (white) ───┐    ┌─ 0V
                       │    │   (black level)                                   │    │
                       └────┘                                                   └────┘
                       SYNC    BACK PORCH              ACTIVE VIDEO              SYNC
                       
LM1881 CSYNC:      HIGH ─────┐                                                ┌───── HIGH
                             │                                                │
                             └────────────────────────────────────────────────┘
                             LOW during sync pulse
                             
After Inverter:    LOW ──────┐                                                ┌───── LOW
(OSD_SYNC_PIN)               │                                                │
                             └────────────────────────────────────────────────┘
                             HIGH (3.3V) during sync - WHAT PIO EXPECTS!
```

---

## Option 2: Minimal Component (Comparator-based)

For even simpler build using just a comparator.

### Bill of Materials

| Ref | Component | Value | LCSC Part # | Price | Description |
|-----|-----------|-------|-------------|-------|-------------|
| U1 | LM393DR | - | C7955 | €0.08 | Dual comparator (SOIC-8) |
| U1 alt | LM393P | - | C387655 | €0.10 | Dual comparator (DIP-8) |
| R1-R6 | Resistors | various | - | €0.06 | See schematic |
| C1 | Capacitor | 100nF | C1591 | €0.01 | Decoupling |

**Total cost: ~€0.50**

### Schematic

```
                    +3.3V
                      │
                    [10k] R1
                      │
    Video In ──┬──────┼──────────► LM393 Pin 3 (+)
               │      │                    │
             [100nF]  │                    │
               │    [22k] R2               │        +3.3V
               │      │                    │          │
              GND    GND                   │        [4.7k] R3
                                          │          │
            Reference ────────────────────│──► (+)   ├─────► PA24 (OSD_SYNC)
            ~0.15V                        │          │
            (voltage    GND ──[47k]──┬────│──► (-)   │
            divider        +3.3V──[1M]┘   │        [10k] R4
            R5||R6)                        │          │
                                          └──────────┘
                                          LM393 open-collector
                                          output with pull-up
                              
    VIDEO MIXER (same as Option 1):
    
    PA22 (W_PIN) ────[470Ω]───┐
                              │
    PA23 (EN_PIN) ───[1kΩ]────┼────► VIDEO OUT ──► VTX
                              │
    VIDEO IN ────────[75Ω]────┤
                              │
                             GND
```

### Principle:
- Comparator compares video signal against ~0.15V reference
- During sync (0V): input < reference → output HIGH (pull-up)
- During video (0.3-1V): input > reference → output LOW
- This naturally inverts and level-shifts in one step!

---

## Option 3: MAX7456 Emulation (Advanced)

If you already have a MAX7456, the LM1881 sync output can drive it directly.
**Not recommended** - FB_OSD is designed to replace MAX7456.

---

## Software Configuration

### 1. Add OSD pin definitions to config.h

Add to `src/config/configs/MADFLIGHT_FC3/config.h`:

```c
// OSD (Framebuffer OSD for analog video overlay)
// Pins must be consecutive: W, EN, SYNC
#define USE_FB_OSD
#define OSD_W_PIN        PA22  // White/Black output (to video mixer)
#define OSD_EN_PIN       PA23  // Enable output (to video mixer)
#define OSD_SYNC_PIN     PA24  // Sync input (from inverter, ACTIVE HIGH!)
```

### 2. Enable OSD in CLI

After flashing, connect via CLI and configure:

```
# Enable OSD feature
feature OSD

# Set video standard (auto-detect, PAL, or NTSC)
set vcd_video_system = AUTO

# Configure OSD elements as desired
set osd_rssi_pos = 234
set osd_main_batt_voltage_pos = 12
# ... etc

# Save settings
save
```

## Video Signal Timing

The PICO OSD supports both PAL and NTSC:

| Parameter | PAL | NTSC |
|-----------|-----|------|
| Display Lines | 16 rows | 13 rows |
| Characters/Line | 30 | 30 |
| Char Resolution | 12x18 pixels | 12x18 pixels |
| Frame buffer | 92KB x2 (double-buffered) |
| Line period | 64µs | 63.5µs |
| Sync pulse | 4.7µs | 4.7µs |
| Back porch | 5.7µs | 4.7µs |

## Testing & Debugging

### Without Camera (oscilloscope test)

1. Generate 50Hz (PAL) or 60Hz (NTSC) square wave 0-3.3V
2. Connect to OSD_SYNC_PIN (PA24)
3. Observe OSD_W_PIN and OSD_EN_PIN on scope
4. You should see pixel data bursts synchronized to your signal

### With Camera (visual test)

1. Power camera and VTX
2. Verify video passes through (transparent mode)
3. Enable OSD: `feature OSD` then `save`
4. Reboot and check for OSD elements
5. If no OSD visible, check sync signal with scope

### Debug CLI Commands

```
# Check OSD status
status

# Check feature flags
feature

# View OSD element positions
get osd_
```

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| No OSD at all | Feature not enabled | `feature OSD` + `save` |
| No OSD at all | SYNC not detected | Check inverter circuit with scope |
| No OSD at all | SYNC polarity wrong | Verify signal is HIGH during sync pulse |
| OSD flickers/tears | Sync noise | Add 100nF capacitor near GPIO |
| OSD flickers/tears | Ground loop | Use single ground point |
| OSD wrong position | PAL/NTSC mismatch | `set vcd_video_system = PAL` or `NTSC` |
| OSD too dim | R4 too high | Decrease W_PIN resistor (try 330Ω) |
| OSD too bright/distorted | R4 too low | Increase W_PIN resistor (try 560Ω) |
| Black outline missing | R5 too high | Decrease EN_PIN resistor (try 680Ω) |
| Video distorted | Impedance mismatch | Verify 75Ω termination |
| Video distorted | Missing ground | Check GND connection |

## Resistor Value Tuning

The video mixer resistor values affect contrast and brightness:

```
Output voltage (simplified) ≈ (V_W × R6 × R5 + V_EN × R6 × R4 + V_IN × R4 × R5) / (R4×R5 + R4×R6 + R5×R6)

Where:
- V_W = W_PIN voltage (0 or 3.3V)
- V_EN = EN_PIN voltage (0 or 3.3V)  
- V_IN = Camera video (0-1V)
- R4 = 470Ω (W_PIN)
- R5 = 1kΩ (EN_PIN)
- R6 = 75Ω (Video termination)
```

### Typical adjustment:
| Adjustment | Change |
|------------|--------|
| Brighter white | R4: 470Ω → 330Ω |
| Dimmer white | R4: 470Ω → 680Ω |
| Darker black | R5: 1kΩ → 680Ω |
| Lighter black | R5: 1kΩ → 1.5kΩ |

## PCB Layout Recommendations

1. **Keep traces short** - especially SYNC input (high-frequency)
2. **Ground plane** - under all video signals
3. **Decoupling capacitors** - close to ICs (100nF)
4. **75Ω trace impedance** - for video signals if possible
5. **Star ground** - single point ground for analog and digital

## KiCad Footprints

| Component | Footprint |
|-----------|-----------|
| LM1881M | SOIC-8_3.9x4.9mm_P1.27mm |
| LM1881N | DIP-8_W7.62mm |
| 2N3904 | TO-92 |
| 0603 resistors | R_0603_1608Metric |
| 0603 capacitors | C_0603_1608Metric |

## References

- Betaflight PR #14882: https://github.com/betaflight/betaflight/pull/14882
- LM1881 Datasheet: https://www.ti.com/product/LM1881
- LM1881M/TR (LCSC): https://www.lcsc.com/product-detail/C518922.html
- LM1881N DIP-8 (LCSC): https://www.lcsc.com/product-detail/C518923.html
- LM393 Comparator: https://www.lcsc.com/product-detail/C7955.html
- 2N3904 Transistor: https://www.lcsc.com/product-detail/C18536.html
- RP2350 PIO Documentation: https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf

## Version History

- 2026-02-19: Complete rewrite with corrected sync inversion circuit
- 2026-02-19: Added LCSC part numbers and detailed schematics
- 2026-02-19: Initial documentation for PICO OSD integration

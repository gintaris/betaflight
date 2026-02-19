# PICO OSD Connection Schema for Madflight FC3

## Overview

Betaflight PICO OSD implementation (FB_OSD - Framebuffer OSD) uses RP2350's PIO (Programmable I/O) to overlay OSD graphics onto analog video signal. This requires 3 GPIO pins and external video mixer circuitry.

## Required GPIO Pins (Must be consecutive!)

| Signal | Function | Direction | Description |
|--------|----------|-----------|-------------|
| OSD_W_PIN | White/Black | Output | Controls OSD pixel brightness (white=1, black=0) |
| OSD_EN_PIN | Enable | Output | Enables OSD overlay (1=show OSD, 0=pass video) |
| OSD_SYNC_PIN | Sync | Input | Receives composite sync from video source |

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
                    ┌─────────────────────────────────────────────────────────────┐
                    │                    MADFLIGHT FC3 (RP2350B)                   │
                    │                                                             │
                    │   PA22 (OSD_W)  ──┬──►  To Video Mixer (White/Black control)│
                    │   PA23 (OSD_EN) ──┼──►  To Video Mixer (Enable/Key)         │
                    │   PA24 (OSD_SYNC)◄┴──   From Sync Separator                 │
                    │                                                             │
                    └─────────────────────────────────────────────────────────────┘
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────────┐
         │                                 │                                     │
         │    ┌────────────┐      ┌────────┴────────┐      ┌────────────┐        │
         │    │   CAMERA   │      │  SYNC SEPARATOR │      │ VIDEO MIXER│        │
         │    │            │      │   (LM1881 or    │      │  (Analog   │        │
         │    │  Video Out ├─────►│    AD8561)      │      │   Keyer)   │        │
         │    │            │      │                 │      │            │        │
         │    └────────────┘      │   Csync Out ────┼─────►│  OSD_W     │        │
         │                        │                 │      │  OSD_EN    │        │
         │                        └─────────────────┘      │            │        │
         │                                                 │   Video In ├◄───────┤ From Camera
         │                                                 │            │        │
         │                                                 │  Video Out ├────────┤►  To VTX
         │                                                 └────────────┘        │
         │                                                                       │
         └───────────────────────────────────────────────────────────────────────┘
                                    EXTERNAL OSD CIRCUITRY
```

## Required External Components

### Option 1: Simple OSD Board with LM1881 Sync Separator

#### Bill of Materials:
| Component | Value | Description |
|-----------|-------|-------------|
| U1 | LM1881 | Sync Separator IC |
| U2 | MAX454 or similar | Video Switch/Multiplexer |
| C1 | 100nF | Decoupling capacitor |
| C2 | 0.1µF | LM1881 input coupling |
| R1 | 680kΩ | LM1881 filter resistor |
| R2, R3 | 75Ω | Video termination |
| R4 | 1kΩ | Level shift resistor |

#### LM1881 Connections:
```
                 ┌─────────┐
    Video In ───►│1      8 │───► +5V (Vcc)
                 │  LM1881 │
        GND ─────│2      7 │───► Csync Out ──► OSD_SYNC_PIN (PA24)
                 │         │
   R1 (680k) ────│3      6 │     (Burst/Back Porch - not used)
      to GND     │         │
                 │4      5 │     (Odd/Even - not used)
                 └─────────┘
```

### Option 2: Simple Resistor DAC + Comparator

For minimal part count, using resistor ladder:

```
            OSD_W_PIN ────┬──[1k]──┐
                         │         │
                         │    ┌────┴────┐
            OSD_EN_PIN ──┴──[2k]──►│ VIDEO │───► To VTX
                                   │  MIX  │
            Video In ──────────────►│       │
                                   └───────┘

    Sync extraction (simple version):
    
    Video In ──[10k]──┬──────────────► OSD_SYNC_PIN
                      │
                    [4.7k]
                      │
                     GND
    
    Note: Add comparator (LM393) for cleaner sync extraction
```

## Software Configuration

### 1. Add OSD pin definitions to config.h

Add to `src/config/configs/MADFLIGHT_FC3/config.h`:

```c
// OSD (Framebuffer OSD for analog video overlay)
#define USE_FB_OSD
#define OSD_W_PIN        PA22
#define OSD_EN_PIN       PA23  
#define OSD_SYNC_PIN     PA24
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

## Testing Without Camera

For testing OSD output without a camera:

1. Connect oscilloscope to OSD_W_PIN and OSD_EN_PIN
2. Inject 50Hz/60Hz square wave (0-3.3V) to OSD_SYNC_PIN
3. Observe pixel output pattern on scope

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| No OSD display | Feature not enabled | `feature OSD` |
| OSD flickering | Sync signal noise | Add filtering/comparator |
| OSD misaligned | Wrong video std | Check PAL/NTSC setting |
| Characters garbled | Pin assignment wrong | Verify consecutive pins |
| No sync detection | Sync threshold wrong | Check sync circuit |

## References

- Betaflight PR #14882: https://github.com/betaflight/betaflight/pull/14882
- LM1881 Datasheet: https://www.ti.com/product/LM1881
- RP2350 PIO Documentation: https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf

## Version History

- 2026-02-19: Initial documentation for PICO OSD integration

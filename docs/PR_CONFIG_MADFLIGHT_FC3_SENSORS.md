# PR: MADFLIGHT_FC3 - Add internal I2C sensor configuration

**Repository:** betaflight/config  
**PR:** https://github.com/betaflight/config/pull/1030  
**Branch:** feature/madflight-fc3-sensors  
**Target:** master  
**Status:** Awaiting dependent firmware PRs

## Title
`MADFLIGHT_FC3: Add internal I2C sensor configuration`

## Description

### Summary
Update MADFLIGHT_FC3 board configuration to enable internal I2C sensors.

### Changes
- **I2C0 Configuration:** Added 100kHz clock speed and internal pull-ups for reliable sensor communication
- **BMP580/BMP581 Barometer:** Configured internal barometer on I2C0 (primary, with external fallback on I2C1)
- **MMC5603 Magnetometer:** Configured internal compass on I2C0 (primary, with external fallback on I2C1)
- **INA226 Power Monitor:** Added current/voltage sensor configuration on I2C0
  - `USE_CURRENT_METER_INA226` - Enable INA226 current meter
  - `DEFAULT_CURRENT_METER_SOURCE = CURRENT_METER_INA226`
  - `DEFAULT_VOLTAGE_METER_SOURCE = VOLTAGE_METER_INA226`
  - INA226 settings: 0x40 address, 2mΩ shunt, 50A max current
- **Motor Protocol:** Use default DSHOT600 (Oneshot125 commented out for analog ESC reference)
- **MSP Displayport:** Enabled OSD via PIOUART0 for digital VTX systems

### Dependent PRs (betaflight/betaflight)

This config PR requires the following firmware PRs to be merged first:

| PR | Description | Status |
|----|-------------|--------|
| [#14925](https://github.com/betaflight/betaflight/pull/14925) | BMP580/BMP581 barometer driver | In Review |
| [#14924](https://github.com/betaflight/betaflight/pull/14924) | MMC5603 magnetometer driver | In Review |
| [#14927](https://github.com/betaflight/betaflight/pull/14927) | INA226 power monitor driver | In Review |

### Hardware
- Board: [MADFLIGHT FC3](https://madflight.com/Board-FC3/)
- MCU: RP2350B (Raspberry Pi Pico 2)
- Internal sensors: BMP580/BMP581, MMC5603, INA226

### Testing
- Compiled locally with `make MADFLIGHT_FC3` (with driver PRs applied)
- Hardware testing completed with all three sensors working

### Reviewer Comment (haslinghuis)
> Awaiting updates to use `CURRENT_METER_INA226` and `VOLTAGE_METER_INA226`
> - there is no USE_VOLTAGE_METER_INA226 define.

**Response:** Added INA226 configuration with:
- `#define USE_CURRENT_METER_INA226`
- `#define DEFAULT_CURRENT_METER_SOURCE CURRENT_METER_INA226`
- `#define DEFAULT_VOLTAGE_METER_SOURCE VOLTAGE_METER_INA226`

Note: `USE_VOLTAGE_METER_INA226` is not separate - the INA226 driver provides both current and voltage from a single I2C read. The driver is enabled with `USE_CURRENT_METER_INA226` and voltage source is set via `DEFAULT_VOLTAGE_METER_SOURCE`.

---

## Merge Coordination

This PR should be merged after the three firmware PRs are merged to master:

1. ✅ Firmware PRs merged (betaflight/betaflight)
2. ✅ Config PR can then be merged (betaflight/config)

The config will compile but INA226, BMP580, and MMC5603 features will be ignored until the firmware support exists.

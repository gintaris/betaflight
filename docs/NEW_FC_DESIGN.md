# Naujo Skrydžio Valdiklio (FC) Projektavimas — RP2350B

**Data:** 2026-02-19  
**Platforma:** Betaflight PICO (RP2350B)  
**Branch:** `feature/new-fc-design`

---

## Apžvalga

Naujas skrydžio valdiklis su RP2350B mikrovaldikliu, skirtas tailsitter / VTOL
konfigūracijai su 4 motorais (DSHOT) ir 4 servo išvestimis. Plokštė turi
integruotus jutiklius (IMU, barometras, magnetometras), analoginį OSD, SD
kortelę blackbox įrašymui ir FRAM greitam parametrų saugojimui.

---

## Komponentų sąrašas

| Komponentas | Modelis | Sąsaja | Pastabos |
|-------------|---------|--------|----------|
| MCU | **RP2350B** | — | Dual-core ARM Cortex-M33, 150 MHz, 48 GPIO |
| IMU (giroskopas + akselerometras) | **LSM6DSV16X** | SPI1 | ±2000°/s, ±16g, DRDY INT |
| Barometras | **BMP580** | I2C0 | ±0.5 hPa absoliuti tikslumas, INT palaikymas |
| Magnetometras | **BMM150** | I2C0 | Bosch 3-ašis, ±1300/2500 µT, **draiveris bus rašomas** |
| Motorų ESC protokolas | **DSHOT600** | PIO0 | 4 motorai, bidirectional telemetrija |
| Servo | **PWM 50Hz** | Hardware PWM | 4 servo kanalai (slices 6-7) |
| OSD | **FB_OSD** (PIO) | PIO2 | Analoginis video overlay, LM393 sync |
| SD kortelė | **microSD** | SPI0 | Blackbox duomenų saugojimas |
| FRAM | **FM25V10 / MB85RS256** | SPI0 (bendra magistralė) | Greitas konfigūracijos saugojimas |
| 3.3V LDO #1 | **SK6019AD4-33** | — | Dedikuotas IMU maitinimui |
| 3.3V LDO #2 | **SK6019AD4-33** | — | MCU + kiti jutikliai |
| 5V Buck | **TPSM33625RDNR** | — | 4.5–36V įėjimas (iki 8S), 2.5A |
| OSD Sync | **LM393DR2G** | — | Komparatorius, ~€0.08 |
| Srovės/Įtampos monitorius | **INA226** | I2C0 | TI high-side galios monitorius, 36V max bus (8S), 1mΩ šuntas |

---

## Blokinė Schema

```
                                    ┌─────────────────────────────────┐
                                    │           RP2350B MCU           │
                                    │                                 │
  ┌─────────┐    UART0 (PA0-1)     │ GPIO 0-1   ── UART0 (RC Radio) │
  │ ELRS/   ├──────────────────────┤ GPIO 2-3   ── I2C1 (External)  │
  │ CRSF    │                      │ GPIO 4-5   ── UART1 (GPS)      │
  └─────────┘                      │ GPIO 6-9   ── DSHOT Motors(PIO0│)
                                    │ GPIO 10-11 ── PIOUART0 (SW)    │
  ┌─────────┐    UART1 (PA4-5)     │ GPIO 12-15 ── SERVO 1-4 (PWM)  │
  │ GPS     ├──────────────────────┤ GPIO 16-17 ── PIOUART1 (SW)    │
  │ Module  │                      │ GPIO 18-19 ── LED / Spare      │
  └─────────┘                      │ GPIO 20-22 ── OSD (PIO2)       │
                                    │ GPIO 23    ── LED Strip (PIO2) │
  ┌──────────┐   I2C1 (PA2-3)      │ GPIO 24-25 ── PINIO / Spare   │
  │ Ext.     ├─────────────────────┤                                 │
  │ Compass  │                      │ GPIO 26    ── GYRO CLKIN       │
  └──────────┘                      │ GPIO 27    ── GYRO EXTI (INT1) │
                                    │ GPIO 28-31 ── SPI1 (Gyro)      │
  ┌──────────────────┐              │ GPIO 32-33 ── I2C0 (Internal)  │
  │ LSM6DSV (SPI1)   ├────────────┤ GPIO 34-37 ── SPI0 (SD Card)   │
  │ INT1 → GPIO 27   │             │ GPIO 38    ── FRAM CS          │
  └──────────────────┘              │ GPIO 39    ── BARO INT (EOC)   │
                                    │ GPIO 40-44 ── ADC (RSSI,I,V)  │
  ┌──────────████──────┐            │ GPIO 45    ── Status LED       │
  │ BMP580 + BMM150    │            │ GPIO 46-47 ── Spare            │
  │ + INA226           │            │                                 │
  │ I2C0 (PA32-33)     ├──────────┤                                 │
  │ BMP580 INT → PA39  │           └─────────────────────────────────┘
  └────────────────────┘

  ┌──────────┐   SPI0 (PA34-37)
  │ microSD  ├───────────────── CS = PA37
  └──────────┘        │
  ┌──────────┐        │
  │ FRAM     ├────────┘ (bendra SPI0 magistralė, CS = PA38)
  └──────────┘
```

---

## PIO Blokų Paskirstymas

RP2350B turi 3 PIO blokus, kiekvienas su 4 state machines (SM) ir 32
instrukcijų atmintimi.

| PIO Blokas | Funkcija | SM Naudojama | Pastabos |
|------------|----------|-------------|----------|
| **PIO0** | DSHOT600 motorams | 4 SM (po 1 motorui) | Max 4 motorai. 13 arba 29 instr. |
| **PIO1** | Software UART | 2-4 SM (TX+RX × 2) | PIOUART0 + PIOUART1 |
| **PIO2** | OSD + LED Strip | 1-2 SM | OSD = 1 SM, LED = 1 SM |

### PIO GPIO Bazė

Kiekvienas PIO blokas gali pasiekti 32 GPIO langą nuo bazės 0 arba 16:

| PIO | Bazė | GPIO Diapazonas | Pinai šiame bloke |
|-----|------|-----------------|-------------------|
| PIO0 | 0 | GPIO 0-31 | Motorai (6-9) ✓ |
| PIO1 | 0 | GPIO 0-31 | PIOUART0 (10-11), PIOUART1 (16-17) ✓ |
| PIO2 | 0 arba 16 | GPIO 0-31 arba 16-47 | OSD (20-22), LED Strip (23) ✓ |

---

## SPI Magistralių Architektūra

### SPI1 — Giroskopas (LSM6DSV16X)

Dedikuota SPI magistralė giroskopu, nes tai yra labiausiai laikui jautrus
jutiklis (8 kHz ODR).

| Signalas | GPIO | Pastaba |
|----------|------|---------|
| SCK | PA30 | SPI1 SCK valid pin |
| MOSI (SDO) | PA31 | SPI1 TX valid pin |
| MISO (SDI) | PA28 | SPI1 RX valid pin |
| CS | PA29 | Gyro chip select |
| INT1 (EXTI) | PA27 | Data-ready interrupt |
| CLKIN | PA26 | Išorinis laikrodžio įėjimas (optional) |

**Max SPI dažnis:** 10 MHz (LSM6DSV16X limitas)

### SPI0 — SD Kortelė + FRAM (bendra magistralė)

SD kortelė ir FRAM dalijasi SPI0 magistrale su atskirais CS signalais.
Tik vienas įrenginys aktyvus vienu metu.

| Signalas | GPIO | Pastaba |
|----------|------|---------|
| SCK | PA34 | SPI0 SCK valid pin |
| MOSI | PA35 | SPI0 TX valid pin |
| MISO | PA36 | SPI0 RX valid pin |
| SD_CS | PA37 | SD kortelės CS |
| FRAM_CS | PA38 | FRAM chip select |

---

## I2C Magistralių Architektūra

### I2C0 — Vidiniai jutikliai (ant plokštės)

| Signalas | GPIO | Pastaba |
|----------|------|---------|
| SDA | PA32 | I2C0 SDA valid pin |
| SCL | PA33 | I2C0 SCL valid pin |

**Prijungti įrenginiai:**

| Įrenginys | I2C Adresas | Pastaba |
|-----------|-------------|---------|
| BMP580 | 0x47 (arba 0x46) | Barometras, INT → PA39 |
| BMM150 | 0x10 (default) | Magnetometras, CSB → VDD |
| INA226 | 0x40 (default) | Srovės/įtampos monitorius, A0=GND, A1=GND |

### I2C1 — Išoriniai jutikliai (jungtis)

| Signalas | GPIO | Pastaba |
|----------|------|---------|
| SDA | PA2 | I2C1 SDA valid pin |
| SCL | PA3 | I2C1 SCL valid pin |

Skirta GPS kompasui ar kitiems išoriniams jutikliams.

---

## Maitinimo Architektūra

```
  Baterija (2S-8S)
  4.2V – 33.6V
       │
       ├──[Įtampos daliklis]──→ ADC_VBAT (PA44) ← atsarginis
       │
       ├──[R_SHUNT 1mΩ]──┐
       │                  ▼
       │         ┌────────────────┐
       │         │    INA226      │
       │         │  I2C0 (0x40)   │──→ Srovė (mA) + Įtampa (mV) + Galia (mW)
       │         │  max 36V bus   │
       │         │  ±81.92mV šunt │
       │         └────────────────┘
       │
       ▼
  ┌─────────────────────┐
  │  TPSM33625RDNR      │
  │  5V Buck Converter   │
  │  Vin: 4.5–36V        │
  │  Vout: 5V / 2.5A     │
  └──────────┬──────────┘
             │ 5V
             ├──────────────→ ELRS/CRSF modulis
             ├──────────────→ GPS modulis
             ├──────────────→ VTX (per PINIO)
             │
       ┌─────┴─────┐
       │            │
       ▼            ▼
  ┌─────────┐  ┌─────────┐
  │SK6019   │  │SK6019   │
  │3.3V LDO │  │3.3V LDO │
  │(IMU)    │  │(MCU)    │
  │400mA    │  │400mA    │
  └────┬────┘  └────┬────┘
       │            │
       ▼            ├──→ RP2350B MCU
  LSM6DSV16X       ├──→ BMP580
  (izoliuotas       ├──→ BMM150
   maitinimas)      ├──→ SD kortelė
                    ├──→ FRAM
                    └──→ LM393 (OSD)
```

**Kodėl 2 atskiri LDO?**
- IMU (LSM6DSV) yra jautriausias triukšmui jutiklis
- Dedikuotas LDO izoliuoja nuo MCU skaitmeninio triukšmo
- Sumažina giroskopo driftą ir pagerina filtravimą
- 100nF + 10µF dekuplingas prie kiekvieno LDO išėjimo
- Ferito granulė tarp LDO ir gyro VDD (papildomas filtras)

---

## Programinės Įrangos Būsena

### Esami draiverai (veikia)

| Komponentas | Draiveris | Define | Pastaba |
|-------------|-----------|--------|---------|
| LSM6DSV16X | `accgyro_spi_lsm6dsv16x.c` | `USE_ACCGYRO_LSM6DSV16X` | Pilnai palaikomas |
| BMP580 | `barometer_bmp5xx.c` | `USE_BARO_BMP580` | INT palaikymas (pulsed, active HIGH) |
| DSHOT600 | `dshot_pico.c` | `USE_DSHOT` | PIO0, bidir telemetrija |
| FB_OSD | `osd_tx.pio` | `USE_FB_OSD` | PIO2, LM393 sync |
| SD Card | `bus_spi_pico.c` | `USE_SDCARD_SPI` | SPI režimu |
| Servo PWM | `pwm_servo_pico.c` | `USE_SERVOS` | Hardware PWM, 50Hz |
| PIO UART | `serial_uart_pico.c` | `USE_PIOUART0/1` | Software UART per PIO1 |
| INA226 | `ina226.c` | `USE_CURRENT_METER_INA226` | I2C srovės/įtampos monitorius, 36V max |

### Reikia rašyti

| Komponentas | Failas | Prioritetas | Pastaba |
|-------------|--------|-------------|---------|
| **BMM150** | `compass_bmm150.c` | **Aukštas** | ~250 eilučių, reikia TRIM kompensacijos |
| **FRAM** | Adaptuoti flash draiveri | Vidutinis | SPI FRAM panašus į NOR flash, bet be erase |

### BMM150 Draiverio Planas

Naudoti `compass_lis2mdl.c` kaip šabloną. Reikalingos funkcijos:

1. `bmm150Detect()` — WHO_AM_I (reg 0x40) = 0x32
2. `bmm150Init()` — nustatyti ODR (10-30 Hz), normal mode, XY/Z rep
3. `bmm150Read()` — nuskaityti 6 baitus (0x42-0x47), pritaikyti TRIM kompensaciją

**TRIM kompensacija** — BMM150 turi 16 trim registrų (0x5D-0x71), kurie
nuskaitomi init metu ir naudojami žaliems duomenims kompensuoti. Bosch
pateikia referencinį kodą (`bmm150.c` iš BMI270-Sensor-API pavyzdžių).

Integracijos taškai:
- Pridėti `MAG_BMM150` į `compass.h` enum
- Pridėti detect case į `compass.c`
- Pridėti `compass_bmm150.c` į `source.mk`
- Config: `#define USE_MAG_BMM150`

---

## config.h Pakeitimai (planuojami)

Nauji define'ai, kuriuos reikės pridėti:

```c
// Barometras (BMP580 ant I2C0)
#define USE_BARO_BMP580
#define BARO_I2C_INSTANCE    I2CDEV_0
#define BARO_EOC_PIN         PA39

// Magnetometras (BMM150 ant I2C0)
#define USE_MAG_BMM150
#define MAG_I2C_INSTANCE     I2CDEV_0

// Srovės/Įtampos monitorius (INA226 ant I2C0)
#define USE_CURRENT_METER_INA226
#define DEFAULT_VOLTAGE_METER_SOURCE   VOLTAGE_METER_INA226
#define DEFAULT_CURRENT_METER_SOURCE   CURRENT_METER_INA226
// CLI parametrai: ina226_i2c_device, ina226_address, ina226_shunt_resistance,
//                 ina226_max_expected_current, ina226_vbat_scale
// Default: I2CDEV_0, 0x40, 1mΩ (1000µΩ), 50A (50000mA), scale=100

// Servo (4 kanalai)
#define USE_SERVOS
#define SERVO1_PIN           PA12
#define SERVO2_PIN           PA13
#define SERVO3_PIN           PA14
#define SERVO4_PIN           PA15

// FRAM (ant SPI0)
#define USE_FLASH_FRAM
#define FRAM_CS_PIN          PA38
#define FRAM_SPI_INSTANCE    SPI0
```

---

## PCB Projektavimo Rekomendacijos

1. **IMU vieta:** Plokštės centre, kuo arčiau masės centro
2. **IMU maitinimas:** SK6019 LDO + ferito granulė + 100nF/10µF — kuo arčiau LSM6DSV
3. **I2C pull-up:** 4.7kΩ prie 3.3V ant I2C0 (BMP580+BMM150) ir I2C1 (išorinis)
4. **SPI terminacija:** 33Ω serijiniai rezistoriai SPI1 linijose (ypač SCK ir MOSI)
5. **Dekuplingas:** 100nF prie kiekvieno jutiklio VDD pino
6. **Ground plane:** Nepertraukiamas po IMU zona
7. **Motorų takeliai:** Kuo trumpesni, atskiri nuo jutiklių
8. **OSD signalai:** PA20-22 vedami kartu, toliau nuo motorų PWM
9. **SD kortelė:** 100nF prie SD VDD, ESD apsauga jei yra vietos
10. **FRAM:** Šalia SD kortelės (dalijasi SPI0 magistrale)
11. **INA226 + Šuntas:** Šunto rezistorius (1mΩ) kuo arčiau INA226 IN+/IN- pinų. Kelvin jungimas (4 laidai). INA226 maitinimas iš MCU 3.3V LDO (VS pin gali būti jungtas tiesiai prie baterijos, iki 36V). 100nF dekuplingas prie VS ir VDD.

---

## Sekantys Žingsniai

1. ☐ Finalizuoti GPIO pinout (žr. `NEW_FC_GPIO_PINOUT.md`)
2. ☐ KiCad schema su naujais komponentais
3. ☐ PCB layout (4 sluoksniai rekomenduojama)
4. ☐ Užsakyti PCB prototipus
5. ☐ Parašyti BMM150 draiveri
6. ☐ Adaptuoti FRAM draiveri
7. ☐ Sukurti naują config.h target
8. ☐ Įjungti INA226 (`USE_CURRENT_METER_INA226`) ir konfigūruoti šunto vertę
9. ☐ Testavimas ir kalibravimas

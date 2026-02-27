# Testavimo Planas — Pimoroni Pico Plus 2

**Data:** 2026-02-24  
**Dev plokštė:** Pimoroni Pico Plus 2 (RP2350B, 16MB Flash, USB-C)  
**Firmware:** `betaflight_2026.6.0-alpha_RP2350B_PIROMONI_PICO_DEV.uf2`  
**Konfigūracija:** `PIROMONI_PICO_DEV` (specialiai sukurta dev board'ui)  
**Tikslas:** Pratestuoti servo, motorų, OSD, IMU ir MAG veikimą prieš užsakant PCB

> **Pastaba:** Naudojame `PIROMONI_PICO_DEV` konfigūraciją, NE `DIVISION_RP_FPV_V2`.
> Dev board konfigūracija turi perkeltus IMU ir MAG pinus (GPIO 0–28 diapazone),
> be SD kortelės ir barometro. Žr. `src/config/configs/PIROMONI_PICO_DEV/config.h`.

---

## 1. Pimoroni Pico Plus 2 — GPIO Prieinamumas

Pico Plus 2 turi RP2350B (48 GPIO), bet header'iai išveda tik **GPIO 0–28**.
Vidiniai jutikliai (GPIO 30+) nebus testuojami — tai normalu.

### Ką GALIME testuoti

| Funkcija | GPIO | Prieinamas? | Prioritetas |
|----------|------|-------------|-------------|
| **MOTOR 1–4** (DSHOT600) | PA6–PA9 | ✅ | **Aukštas** |
| **SERVO 1–4** (50Hz PWM) | PA12–PA15 | ✅ | **Aukštas** |
| **OSD** (W, EN, SYNC) | PA20–PA22 | ✅ | **Aukštas** |
| **IMU** (LSM6DSV16X SPI1) | PA10, PA11, PA23, PA27, PA28 | ✅ | **Aukštas** |
| **MAG** (LIS2MDL I2C1) | PA2–PA3 | ✅ | **Aukštas** |
| UART0 (RC radio) | PA0–PA1 | ✅ | Vidutinis |
| UART1 (GPS) | PA4–PA5 | ✅ | Žemas |
| PIOUART1 | PA16–PA17 | ✅ | Žemas |
| LED0, LED1 | PA18–PA19 | ✅ | Informacinis |
| PINIO1, PINIO2 | PA24–PA25 | ✅ | Žemas |

### Ko NEGALIME testuoti (GPIO 30+)

| Funkcija | GPIO (PCB) | Priežastis |
|----------|------------|----------|
| Barometras BMP580 | I2C0 (PA32–PA33) | Nėra header'iuose |
| SD kortelė | SPI1 (PA28–PA31) | SPI1 perkeltas IMU; PA30-31 nėra header'iuose |
| ADC (VBAT, RSSI...) | PA40–PA44 | Nėra header'iuose |
| INA226 | I2C0 (PA32–PA33) | Nėra header'iuose |

> **Pastaba:** IMU ir MAG yra perkelti į GPIO 0–28 diapazoną PIROMONI_PICO_DEV
> konfigūracijoje, todėl juos GALIME testuoti ant dev board.

**Pastaba:** Pico Plus 2 LED yra ant GPIO 25 (= mūsų PINIO2_PIN). Betaflight
status LED0 (PA45) neveiks — statusą stebėsime per Betaflight Configurator.

---

## 2. Reikalingi Komponentai

### Iš karto (motorai + servo)

| # | Komponentas | Kiekis | Pastaba |
|---|-------------|--------|---------|
| 1 | Pimoroni Pico Plus 2 | 1 | RP2350B dev board |
| 2 | USB-C kabelis | 1 | Firmware flash + Configurator |
| 3 | Breadboard | 1 | 830 taškų |
| 4 | Jumper laidai (M-M) | ~20 | Jungimams |
| 5 | Servo motor (SG90 ar pan.) | 1–4 | 50Hz PWM testavimui |
| 6 | ESC + brushless motor | 1 | DSHOT600 testavimui (arba osciloskopas) |
| 7 | 5V maitinimo šaltinis | 1 | Servo maitinimui (NE iš Pico!) |
| 8 | Osciloskopas arba logic analyzer | 1 | Signalų verifikacijai (rekomenduojama) |
| 9 | LED + 330Ω rezistorius | 2–3 | Paprastas GPIO tikrinimas |

### Poryt (OSD komponentai)

| # | Komponentas | Kiekis | Pastaba |
|---|-------------|--------|---------|
| 10 | LM393DR2G (SOIC-8) | 1 | Sync komparatorius |
| 11 | Rezistorius 1MΩ | 1 | Įtampos daliklio viršus (R68 schemoje=100kΩ — naudok pagal schemą) |
| 12 | Rezistorius 4.7kΩ | 2 | Daliklio apačia + pull-up (R67, R3) |
| 13 | Rezistorius 470Ω | 1 | W_PIN → video (R77) |
| 14 | Rezistorius 1kΩ | 1 | EN_PIN → video (R76) |
| 15 | Rezistorius 75Ω | 1 | Video terminacija (R75) |
| 16 | Kondensatorius 100nF | 1 | LM393 dekuplingas (C121) |
| 17 | FPV kamera (analoginė) | 1 | Video šaltinis (PAL arba NTSC) |
| 18 | FPV monitorius arba VTX+goggles | 1 | OSD rezultato stebėjimas |

---

## 3. Firmware Flashinimas

### 3.1 Paruošti UF2

Firmware jau sukompiliuotas:
```
obj/betaflight_2026.6.0-alpha_RP2350B_PIROMONI_PICO_DEV.uf2
```

Jei reikia perkompiliuoti:
```bash
cd ~/Git/betaflight
make CONFIG=PIROMONI_PICO_DEV
```

### 3.2 Flashinti Pico Plus 2

1. **Laikyk BOOT mygtuką** ant Pico Plus 2
2. **Prijunk USB-C** kabelį prie kompiuterio (laikydamas BOOT)
3. **Atleisk BOOT** — pasirodys `RP2350` USB diskas
4. **Nukopijuok** UF2 failą į tą diską:
   ```bash
   cp obj/betaflight_2026.6.0-alpha_RP2350B_PIROMONI_PICO_DEV.uf2 /media/$USER/RP2350/
   ```
5. Pico automatiškai persikraus ir startuos Betaflight

### 3.3 Patikrinti ryšį

1. Atidaryti **Betaflight Configurator** (arba `screen /dev/ttyACM0 115200`)
2. Prisijungti — turėtų matytis `PIROMONI_PICO_DEV` kaip board name
3. CLI tab'e: `status` — patikrinti ar firmware veikia
4. CLI: `resource` — pamatyti GPIO priskirimus

---

## 4. Testas #1 — Servo Išvestys (PWM 50Hz)

**Tikslas:** Patikrinti ar hardware PWM veikia ant PA12–PA15.

### 4.1 Schema

```
    Pimoroni Pico Plus 2                    SG90 Servo
  ┌─────────────────────┐               ┌──────────────┐
  │                     │               │              │
  │  GPIO 12 (PA12) ────┼───────────────┤ Signal (geltonas)
  │                     │               │              │
  │  GND ───────────────┼───────┬───────┤ GND (rudas) │
  │                     │       │       │              │
  └─────────────────────┘       │       │ VCC (raudonas)│
                                │       └──────┬───────┘
                                │              │
                         ┌──────┴──────────────┴──┐
                         │   5V Maitinimo šaltinis │
                         │   (NE iš Pico VBUS!)    │
                         └─────────────────────────┘
```

**⚠️ SVARBU:** Servo maitink iš **atskiro 5V šaltinio**, ne iš Pico VBUS.
Servo gali traukti iki 500mA — per daug USB portui.
GND turi būti **bendras** (Pico GND + servo GND + 5V PSU GND).

### 4.2 Pinų Jungimas

| Pico GPIO | Header Pin # | Funkcija | Servo laidas |
|-----------|-------------|----------|-------------|
| GPIO 12 | Pin 16 | SERVO1 | Signal (geltonas) |
| GPIO 13 | Pin 17 | SERVO2 | Signal (geltonas) |
| GPIO 14 | Pin 19 | SERVO3 | Signal (geltonas) |
| GPIO 15 | Pin 20 | SERVO4 | Signal (geltonas) |
| GND | Pin 18 arba 23 | Bendra žemė | GND (rudas) |

### 4.3 Testavimo Procedūra

1. **Prijungti** servo prie GPIO 12 (SERVO1)
2. **Paleisti** Betaflight Configurator → Servo tab
3. **Patikrinti default'us:** Servo center = 1500µs, min = 1000µs, max = 2000µs

4. **Testas A — Center pozicija:**
   - Servo turi stovėti centrinėje pozicijoje
   - Osciloskope: 1500µs HIGH pulsas, 20ms periodas (50Hz)

5. **Testas B — Pilnas diapazonas:**
   - CLI: `set servo1_center = 1000`
   - Servo turi pasukti į vieną kraštą
   - CLI: `set servo1_center = 2000`
   - Servo turi pasukti į kitą kraštą

6. **Testas C — „Servo servo" slankiklis:**
   - Configurator Servo tab'e judinti slankiklį
   - Servo turi sekti
   
7. **Kartoti** GPIO 13, 14, 15 (SERVO2–4)

### 4.4 Tikėtini Rezultatai

| Testas | Laukiamas rezultatas | PASS / FAIL |
|--------|---------------------|-------------|
| Servo reaguoja į CLIkomandas | Servo juda | ☐ |
| PWM periodas = 20ms (50Hz) | Osciloskopas rodo 50Hz | ☐ |
| Pulsas min = 1000µs | Osciloskopas patvirtina | ☐ |
| Pulsas max = 2000µs | Osciloskopas patvirtina | ☐ |
| Visi 4 servo kanalai veikia | Kiekvienas juda nepriklausomai | ☐ |
| PWM slice 6: GPIO12+13 tos pačios freq | Abu 50Hz | ☐ |
| PWM slice 7: GPIO14+15 tos pačios freq | Abu 50Hz | ☐ |

---

## 5. Testas #2 — Motorų Išvestys (DSHOT600)

**Tikslas:** Patikrinti ar PIO0 DSHOT veikia ant PA6–PA9.

### 5.1 Variantas A — Su Osciloskopų / Logic Analyzer

Saugiausias būdas — nereikia tikro motoro.

```
    Pimoroni Pico Plus 2
  ┌─────────────────────┐
  │                     │
  │  GPIO 6 (PA6) ──────┼──────── Osciloskopo zondas (MOTOR1)
  │  GPIO 7 (PA7) ──────┼──────── Osciloskopo zondas (MOTOR2)
  │  GPIO 8 (PA8) ──────┼──────── Osciloskopo zondas (MOTOR3)
  │  GPIO 9 (PA9) ──────┼──────── Osciloskopo zondas (MOTOR4)
  │  GND ───────────────┼──────── Osciloskopo GND
  │                     │
  └─────────────────────┘
```

### 5.2 Variantas B — Su ESC + Motor

```
    Pimoroni Pico Plus 2               ESC                    Motor
  ┌─────────────────────┐          ┌─────────┐          ┌──────────┐
  │                     │          │         │          │          │
  │  GPIO 6 (PA6) ──────┼──────────┤ Signal  │          │          │
  │  GND ───────────────┼──────────┤ GND     │──────────┤ 3 fazės  │
  │                     │          │         │          │          │
  └─────────────────────┘          │ + ──────┼── LiPo   │          │
                                   │ - ──────┼── GND    │          │
                                   └─────────┘          └──────────┘
```

**⚠️ SVARBU:**
- **Nuimk propelerį!** Testavimui propeleris nereikalingas ir pavojingas.
- ESC maitink iš LiPo (2S–6S pagal ESC specifikaciją)
- **GND tarp Pico ir ESC turi būti sujungtas!**

### 5.3 Testavimo Procedūra

1. **CLI:** `set motor_protocol = DSHOT600`
2. **CLI:** `save` → Pico persikrauna

3. **Testas A — DSHOT signalo tikrinimas (osciloskopas):**
   - Betaflight Configurator → Motors tab
   - Pastumti MOTOR1 slankiklį virš minimumo
   - Osciloskope: turėtų matytis DSHOT600 paketai
   - DSHOT600 bitrate: 600 kbit/s → bit periodas ≈ 1.67µs
   - Paketo trukmė: 16 bitų × 1.67µs ≈ 26.7µs

4. **Testas B — Su ESC (jei naudojamas):**
   - **NEARMINK!** Testuojam tik per Motors tab slankiklį
   - Motors tab: pažymėti "I understand the risks"
   - Lėtai kelti MOTOR1 slankiklį — motoras turi suktis
   
5. **Testas C — Bidirectional DSHOT (jei ESC palaiko):**
   - CLI: `set dshot_bidir = ON`
   - CLI: `save`
   - Motors tab: turėtų rodyti RPM reikšmes

6. **Kartoti** kiekvienam motoro kanalui (GPIO 6–9)

### 5.4 Tikėtini Rezultatai

| Testas | Laukiamas rezultatas | PASS / FAIL |
|--------|---------------------|-------------|
| DSHOT600 paketas matomas osciloskope | 16-bit signalai, ~27µs | ☐ |
| 4 kanalai nepriklausomai valdomi | Kiekvienas GPIO generuoja signalą | ☐ |
| Motorai sukasi (jei ESC) | Reaguoja į throttle | ☐ |
| Bidir DSHOT RPM (jei ESC palaiko) | RPM rodomas Configurator | ☐ |

---

## 6. Testas #3 — OSD (Framebuffer OSD)

**Tikslas:** Patikrinti analoginio video overlay veikimą.  
**Komponentai atvyksta:** Poryt (2026-02-26)

### 6.1 LM393 Sync Grandinės Schema (Breadboard)

```
        ĮTAMPOS DALIKLIS — Referencinė ~0.15V
        ═══════════════════════════════════════

          Pico 3.3V (Pin 36)
               │
             [100kΩ] R68 (pagal schemą BO_RP2350_V2)
               │
               ├──── REF taškas → LM393 Pin 2 (IN1-)
               │
             [4.7kΩ] R63 (pagal schemą)
               │
              GND

        Vref = 3.3V × 4.7k / (100k + 4.7k) ≈ 0.148V


        LM393 KOMPARATORIUS (SOIC-8 ant adapterio arba DIP-8)
        ══════════════════════════════════════════════════════

            LM393 Pinout:
            ┌────────────┐
      OUT1 1│            │8  VCC ←── Pico 3.3V (Pin 36)
      IN1- 2│   LM393    │7  OUT2 ── NC
      IN1+ 3│            │6  IN2- ── 3.3V (neutralizuoti)
       GND 4│            │5  IN2+ ── NC
            └────────────┘


        Pajungimas:

        Pin 1 (OUT1) ──┬── [4.7kΩ pull-up] ──── Pico 3.3V
                       │
                       └──── GPIO 22 (PA22 — OSD_SYNC_PIN)

        Pin 2 (IN1-)  ──── REF taškas (0.148V)

        Pin 3 (IN1+)  ──── FPV kameros Video Out

        Pin 4 (GND)   ──── Pico GND

        Pin 6 (IN2-)  ──── Pico 3.3V (neutralizuoti nenaudojamą komparatorių)

        Pin 8 (VCC)   ──── Pico 3.3V + [100nF] → GND (dekuplingas)


        VIDEO MIKSERIS (Rezistorinis DAC)
        ═════════════════════════════════

        GPIO 20 (PA20 — W_PIN) ────[470Ω]────┐
                                              │
        GPIO 21 (PA21 — EN_PIN) ───[1kΩ]─────┤──── VIDEO OUT ──→ VTX arba monitorius
                                              │
        Camera Video Out ──────────[75Ω]──────┤
                                              │
                                             GND
```

### 6.2 Breadboard Jungimo Instrukcijos

#### Žingsnis 1: LM393 maitinimas
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| Pico 3.3V (Pin 36) | LM393 Pin 8 (VCC) | Laidas |
| Pico GND (Pin 38) | LM393 Pin 4 (GND) | Laidas |
| LM393 Pin 8 | LM393 Pin 4 | 100nF kondensatorius |
| Pico 3.3V | LM393 Pin 6 (IN2-) | Laidas (neutralizuoti) |

#### Žingsnis 2: Įtampos daliklis (REF)
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| Pico 3.3V | Daliklio viršus | Laidas |
| Daliklio viršus | REF taškas | 100kΩ rezistorius |
| REF taškas | GND | 4.7kΩ rezistorius |
| REF taškas | LM393 Pin 2 (IN1-) | Laidas |

#### Žingsnis 3: Video įėjimas
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| FPV kameros Video Out | LM393 Pin 3 (IN1+) | Laidas (tiesiogiai!) |

#### Žingsnis 4: SYNC → Pico
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| LM393 Pin 1 (OUT1) | GPIO 22 (PA22) | Laidas |
| LM393 Pin 1 (OUT1) | 3.3V | 4.7kΩ pull-up rezistorius |

#### Žingsnis 5: Video mikseris
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| GPIO 20 (PA20) | Video taškas | 470Ω rezistorius |
| GPIO 21 (PA21) | Video taškas | 1kΩ rezistorius |
| Kameros Video Out (tee) | Video taškas | 75Ω rezistorius |
| Video taškas | VTX / monitorius | Laidas |

#### Žingsnis 6: Masės
| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| Pico GND | Kameros GND | Laidas |
| Pico GND | VTX/monitoriaus GND | Laidas |

### 6.3 Pico Plus 2 Header Pinout (testuose naudojami pinai)

```
        Pimoroni Pico Plus 2 — Header pinout
        (žiūrint iš viršaus, USB-C kairėje)

                    USB-C
                 ┌─────────┐
    GPIO 0  ─── 1│         │40 ─── VBUS
    GPIO 1  ─── 2│         │39 ─── VSYS
         GND ── 3│         │38 ─── GND
    GPIO 2  ─── 4│         │37 ─── 3V3_EN
    GPIO 3  ─── 5│         │36 ─── 3V3 (OUT) ← LM393 VCC
    GPIO 4  ─── 6│         │35 ─── ADC_VREF
    GPIO 5  ─── 7│         │34 ─── GPIO 28
         GND ── 8│         │33 ─── GND
    GPIO 6  ─── 9│  MOTOR1 │32 ─── GPIO 27 (IMU_EXTI)
    GPIO 7  ──10│  MOTOR2 │31 ─── GPIO 26 (MAG_DRDY)
    GPIO 8  ──11│  MOTOR3 │30 ─── RUN
    GPIO 9  ──12│  MOTOR4 │29 ─── GPIO 22 ← OSD_SYNC
         GND ──13│         │28 ─── GND
    GPIO 10 └14│ SPI1_SCK│27 ─── GPIO 21 ← OSD_EN
    GPIO 11 └15│ SPI1_SDO│26 ─── GPIO 20 ← OSD_W
    GPIO 12 ──16│  SERVO1 │25 ─── GPIO 19 (LED2)
    GPIO 13 ──17│  SERVO2 │24 ─── GPIO 18 (LED1)
         GND ──18│         │23 ─── GND
    GPIO 14 ──19│  SERVO3 │22 ─── GPIO 17 (PIOUART1)
    GPIO 15 ──20│  SERVO4 │21 ─── GPIO 16 (PIOUART1)
                 └─────────┘
         (GPIO 23-25, 27, 28 ant Pico Plus 2 papildomų padų)
```

**Pastaba:** GPIO 23–25 ant Pico Plus 2 gali būti ant apatinės pusės padų
arba specialiuose header'iuose — patikrinti plokštės dokumentaciją.

### 6.4 Testavimo Procedūra (OSD)

**Prieš pradedant:**
- Kamera turi būti maitinama ir generuoti video signalą
- Monitorius/VTX turi būti prijungtas prie video mikserio išėjimo

1. **CLI tikrinimas:**
   ```
   get osd
   ```
   Turėtų rodyti OSD nustatymus. Patikrinti:
   - `osd_displayport_device = FB_OSD` arba panašiai

2. **CLI: Perjungti video standartą (jei reikia):**
   ```
   set vcd_video_system = AUTO
   save
   ```

3. **Testas A — Sync aptikimas:**
   - Prijungus kamerą, Betaflight turėtų aptikti PAL arba NTSC
   - CLI: `status` — ieškoti OSD statuso (detected NTSC/PAL)
   - Jei sync neaptiktas:
     - Patikrinti LM393 OUT1 su osciloskopų — ar matomi sync impulsai
     - Multimetru patikrinti REF tašką — turėtų būti ~0.15V
     - Patikrinti pull-up rezistorių prie OSD_SYNC_PIN

4. **Testas B — OSD vaizdas:**
   - Monitoriuje turėtų matytis kameros vaizdas su Betaflight OSD overlay
   - Turėtų matytis: baterijos įtampa, skrydžio laikas, craft name
   - Jei OSD neatsiranda:
     - Patikrinti ar W_PIN (GPIO20) generuoja signalą
     - Patikrinti ar EN_PIN (GPIO21) kyla HIGH kai OSD piešia

5. **Testas C — OSD elementų tikrinimas:**
   - Configurator → OSD tab
   - Įjungti/išjungti elementus
   - Per `save` — OSD turėtų atsinaujinti
   - Patikrinti: teksto pozicija, šrifto aiškumas

6. **Testas D — PAL vs NTSC:**
   - Jei kamera PAL: turėtų rodyti ~305 hsyncs, ~625 eilutės
   - Jei kamera NTSC: turėtų rodyti ~253 hsyncs, ~525 eilutės
   - CLI: `set vcd_video_system = PAL` ir `set vcd_video_system = NTSC`
   - Patikrinti ar veikia su kiekvienu

### 6.5 OSD Troubleshooting

| Problema | Galima priežastis | Sprendimas |
|----------|------------------|-----------|
| Matomas tik kameros vaizdas, be OSD | OSD nepaleistas arba sync neaptiktas | Tikrinti LM393 OUT prieš GPIO22 |
| Juodas ekranas | Video mikserio klaida | Tikrinti 75Ω rezistorių, GND sujungimus |
| OSD teksto mirksėjimas | Silpnas sync | Tikrinti pull-up, REF voltažą |
| "Sniego" arba triukšmo | EN_PIN nuolat HIGH | Tikrinti 1kΩ rezistorių EN linijoje |
| Configurator rodo "OSD not configured" | `USE_FB_OSD` neveikia | Patikrinti firmware kompiliaciją |

---

## 7. Testas #4 — IMU (LSM6DSV16X ant SPI1)

**Tikslas:** Patikrinti ar SPI1 IMU veikia su perkeltais pinais.  
**Reikia:** LSM6DSV16X breakout modulis (pvz. Adafruit LSM6DSV16X arba kitas SPI modulis).

### 7.1 IMU Prijungimas

| Pico GPIO | Header Pin # | Funkcija | IMU modulio pinas |
|-----------|-------------|----------|-------------------|
| GPIO 10 | Pin 14 | SPI1_SCK | SCK / CLK |
| GPIO 11 | Pin 15 | SPI1_SDO (MOSI) | SDI / MOSI |
| GPIO 28 | Pin 34 | SPI1_SDI (MISO) | SDO / MISO |
| GPIO 23 | Pad | GYRO_CS | CS / NCS |
| GPIO 27 | Pin 32 | GYRO_EXTI | INT1 |
| 3.3V | Pin 36 | Maitinimas | VDD / VIN |
| GND | Pin 38 | Žemė | GND |

**⚠️ SVARBU:**
- GPIO 23 ir 27 gali būti ant Pico Plus 2 **apatinių padų** — patikrinti plokštės dokumentaciją.
- IMU modulis turi veikti su **3.3V** logikos lygiu (ne 5V!).
- SPI mode: CPOL=1, CPHA=1 (SPI Mode 3).

### 7.2 Testavimo Procedūra

1. **Prijungti** IMU modulį pagal lentelę aukščiau
2. **Flash'inti** PIROMONI_PICO_DEV firmware (jei dar nepadarytas)
3. **Prisijungti** per Configurator

4. **Testas A — IMU aptikimas:**
   - CLI: `status` — ieškoti "Gyro detected" pranešimo
   - Jei aptiktas: rodys "LSM6DSV16X" arba panašiai
   - Jei neaptiktas: "NO GYRO" — tikrinti SPI laidus ir CS

5. **Testas B — Gyro duomenys:**
   - Configurator → Sensors tab
   - Turėtų matytis giroskopo kreivės (X, Y, Z)
   - Pasukti dev board — kreivės turi reaguoti

6. **Testas C — Akselerometras:**
   - Sensors tab: akselerometro duomenys
   - Pastatyti ant stalo: Z ≈ 1g, X ≈ 0, Y ≈ 0
   - Paversti — ašys turi keistis

### 7.3 Tikėtini Rezultatai

| Testas | Laukiamas rezultatas | PASS / FAIL |
|--------|---------------------|-------------|
| IMU aptiktas per SPI1 | CLI rodo "Gyro detected" | ☐ |
| Giroskopo duomenys kinta | Sensors tab rodo kreives | ☐ |
| Akselerometras rodo 1g | Z ašis ≈ 1.0g ramybėje | ☐ |
| INT1 (EXTI) veikia | Data-ready interrupt aktyvus | ☐ |

---

## 8. Testas #5 — Magnetometras (LIS2MDL ant I2C1)

**Tikslas:** Patikrinti ar I2C1 magnetometras veikia.  
**Reikia:** LIS2MDL breakout modulis (pvz. Adafruit LIS2MDL breakout, I2C adresas 0x1E).

### 8.1 MAG Prijungimas

| Pico GPIO | Header Pin # | Funkcija | MAG modulio pinas |
|-----------|-------------|----------|-------------------|
| GPIO 2 | Pin 4 | I2C1_SDA | SDA |
| GPIO 3 | Pin 5 | I2C1_SCL | SCL |
| 3.3V | Pin 36 | Maitinimas | VIN |
| GND | Pin 38 | Žemė | GND |

**Pastaba:** I2C pull-up rezistoriai (4.7kΩ) paprastai jau yra ant breakout modulio.
Jei nėra — pridėti 4.7kΩ nuo SDA→3.3V ir SCL→3.3V.

### 8.2 Testavimo Procedūra

1. **Prijungti** MAG modulį pagal lentelę
2. **CLI:** `set mag_hardware = AUTO` (arba `LIS2MDL`)
3. **CLI:** `save` → Pico persikrauna

4. **Testas A — MAG aptikimas:**
   - CLI: `status` — ieškoti "Mag detected"
   - Jei aptiktas: rodys "LIS2MDL" ir I2C adresą 0x1E

5. **Testas B — Kompaso duomenys:**
   - Configurator → Sensors tab
   - Turėtų matytis magnetometro kreivės (X, Y, Z)
   - Pasukti dev board aplink Z ašį — heading turi keistis

6. **Testas C — Kompaso kryptis:**
   - Configurator → Setup tab → 3D modelis
   - Sukant plokštę turėtų matytis heading pokytis

### 8.3 Tikėtini Rezultatai

| Testas | Laukiamas rezultatas | PASS / FAIL |
|--------|---------------------|-------------|
| LIS2MDL aptiktas per I2C1 | CLI rodo "Mag detected" | ☐ |
| Magnetometro duomenys kinta | Sensors tab rodo kreives | ☐ |
| Heading keičiasi sukant | 360° sukimas = pilnas ratas | ☐ |

---

## 9. Testas #6 — UART Komunikacija

**Tikslas:** Patikrinti ar UART0 veikia su ELRS/CRSF imtuvu.

### 9.1 ELRS Prijungimas

| Pico GPIO | Funkcija | ELRS pinas |
|-----------|----------|-----------|
| GPIO 0 | UART0_TX | RX |
| GPIO 1 | UART0_RX | TX |
| 5V (VBUS) | Maitinimas | VCC |
| GND | Žemė | GND |

### 9.2 Procedūra

1. CLI: `serial 0 64 115200 57600 0 115200` (nustatyti UART0 kaip SERIALRX)
2. CLI: `set serialrx_provider = CRSF`
3. CLI: `save`
4. Configurator → Receiver tab — turėtų rodyti kanalų vertes
5. Pajudinti siųstuvą — kanalai turi reaguoti

---

## 10. Tikrinimo Eiliškumas

### Diena 1 (Šiandien / Rytoj) — Be OSD komponentų

| # | Testas | Trukmė | Komentaras |
|---|--------|--------|----------|
| 1 | Flash firmware | 5 min | UF2 copy |
| 2 | USB + Configurator ryšys | 5 min | Patikrinti board detection |
| 3 | CLI `resource` | 5 min | Patikrinti GPIO priskirimus |
| 4 | LED0 (PA18) + LED1 (PA19) | 5 min | Prijungti LED per 330Ω, tikrinti flašinimą |
| 5 | **SERVO testas** | 30 min | 1–4 servo kanalai, osciloskopas |
| 6 | **MOTOR testas** (osciloskopas) | 20 min | 4 kanalai, DSHOT600 signalai |
| 7 | MOTOR testas (su ESC) | 20 min | Jei turi ESC, prisijungti 1 motorą |
| 8 | UART0 (ELRS) | 15 min | Jei turi ELRS imtuvą |
| 9 | **IMU testas** (SPI1) | 30 min | Jei turi LSM6DSV16X modulį |
| 10 | **MAG testas** (I2C1) | 20 min | Jei turi LIS2MDL modulį |

### Diena 2 (Poryt) — Su OSD komponentais

| # | Testas | Trukmė | Komentaras |
|---|--------|--------|-----------|
| 11 | LM393 grandinė ant breadboard | 30 min | Surinkti, patikrinti REF |
| 12 | Sync aptikimas | 15 min | Prijungti kamerą, tikrinti sync |
| 13 | **OSD vaizdas** | 30 min | Teksto overlay ant video |
| 14 | OSD nustatymai | 15 min | Elementų pozicijos, PAL/NTSC |
| 15 | Stabilumo testas | 30 min | Palikti veikti 30 min, stebėti |

---

## 11. Žinomi Apribojimai Testavimo Metu

1. **LED0 (PA18) — išorinė LED** — Pico Plus 2 onboard LED yra ant GPIO 25 (= PINIO2).
   Betaflight status LED0 perkeltas į PA18 — reikia prijungti išorinį LED per 330Ω.
   Arba stebėti per Configurator.

2. **IMU (LSM6DSV16X) ant SPI1** — Dev board konfigūracijoje IMU perkeltas
   į SPI1 (SCK=PA10, SDO=PA11, SDI=PA28, CS=PA23, EXTI=PA27).
   Reikia išorinio LSM6DSV16X modulio ant breadboard.
   **Jei modulio nėra — Configurator rodys "NO GYRO", bet motorai/servo/OSD vis tiek veikia.**

3. **MAG (LIS2MDL) ant I2C1** — Prijungtas per PA2/PA3.
   Reikia išorinio LIS2MDL modulio (pvz. Adafruit LIS2MDL breakout).
   **Jei modulio nėra — kompasas bus tiesiog išjungtas.**

4. **SD kortelė neveiks** — SPI1 perkeltas IMU naudojimui; PA30-31 neprieinami.
   Blackbox neįmanomas ant dev board.

5. **ADC neveiks** — PA40–PA44 neprieinami. VBAT/RSSI rodys 0.

6. **Barometras neveiks** — BMP580 ant I2C0 (PA32-33), neprieinami.

7. **PIOUART0 neprieinamas** — PA10/PA11 perkelti SPI1 (IMU).

8. **PINIO2 (PA25) = Pico Plus 2 LED** — Pico onboard LED gali mirksėti
   jei Betaflight valdo PINIO2.

---

## 12. Sėkmės Kriterijai

Prieš užsakant PCB, šie testai TURI praeiti:

| # | Kritinis testas | Statusas |
|---|----------------|---------|
| ✅ | Betaflight paleistas ant RP2350B | ☐ |
| ✅ | USB CDC ryšys su Configurator | ☐ |
| ✅ | SERVO1–4 generuoja 50Hz PWM (PA12–15) | ☐ |
| ✅ | MOTOR1–4 generuoja DSHOT600 (PA6–9) | ☐ |
| ✅ | OSD sync aptikimas (LM393 → PA22) | ☐ |
| ✅ | OSD teksto overlay ant video | ☐ |
| ⬜ | IMU LSM6DSV16X aptiktas (SPI1) — jei turi modulį | ☐ |
| ⬜ | MAG LIS2MDL aptiktas (I2C1) — jei turi modulį | ☐ |
| ⬜ | UART0 CRSF (PA0–1) — jei turi imtuvą | ☐ |

**Jei visi ✅ testai praeina → galima užsakyti PCB.**

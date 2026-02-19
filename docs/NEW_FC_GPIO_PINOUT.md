# RP2350B GPIO Pinų Paskirstymo Rekomendacijos

**Data:** 2026-02-19  
**MCU:** RP2350B (48 GPIO: PA0–PA47)  
**Konfigūracija:** 4 motorai (DSHOT) + 4 servo + OSD + SD + FRAM

---

## Turinys

1. [RP2350B GPIO Apžvalga](#rp2350b-gpio-apžvalga)
2. [Pilna GPIO Lentelė](#pilna-gpio-lentelė)
3. [Detalūs Paaiškinimai Pagal Grupes](#detalūs-paaiškinimai-pagal-grupes)
4. [PWM Slice Žemėlapis (Servo)](#pwm-slice-žemėlapis)
5. [PIO Blokų Paskirstymas](#pio-blokų-paskirstymas)
6. [SPI Pin Multipleksavimo Apribojimai](#spi-pin-multipleksavimo-apribojimai)
7. [I2C Pin Apribojimai](#i2c-pin-apribojimai)
8. [UART Pin Apribojimai](#uart-pin-apribojimai)
9. [ADC Kanalai](#adc-kanalai)
10. [Žinomi Konfliktai ir Sprendimai](#žinomi-konfliktai-ir-sprendimai)

---

## RP2350B GPIO Apžvalga

RP2350B turi **48 GPIO** (GPIO 0–47). Kiekvienas GPIO gali būti priskirtas
vienai iš šių funkcijų per multiplekserį:

| Funkcija | GPIO Func # | Pastaba |
|----------|-------------|---------|
| SPI | 1 | SPI0 arba SPI1, priklausomai nuo pin |
| UART | 2 | UART0 arba UART1 |
| I2C | 3 | I2C0 arba I2C1 |
| PWM | 4 | 12 PWM slices (0-11), 2 kanalai kiekvienas |
| PIO | 6, 7, 8 | PIO0, PIO1, PIO2 |
| GPIO | 5 | Paprastas skaitmeninis I/O |
| ADC | — | Tik GPIO 26-29 ir 40-47 |

**Svarbu:** Kiekvienas GPIO gali turėti tik VIENĄ aktyvią funkciją vienu metu.

---

## Pilna GPIO Lentelė

### Išorinių Jungčių Pinai (GPIO 0–25)

| GPIO | BF Pin | Funkcija | Magistralė | PWM Slice | Motyvas |
|------|--------|----------|------------|-----------|---------|
| 0 | PA0 | **UART0_TX** | UART0 | 0A | RC radijas (ELRS/CRSF). UART0 TX valid pin. Artimas PCB kraštui — patogu jungčiai. |
| 1 | PA1 | **UART0_RX** | UART0 | 0B | RC radijas RX. UART0 RX valid pin. Šalia TX — vienas konektorius. |
| 2 | PA2 | **I2C1_SDA** | I2C1 | 1A | Išorinė I2C magistralė (GPS kompasas, rangefinder). I2C1 SDA valid: GPIO 2,6,10,14... Pasirinktas 2 — šalia kitų jungčių. |
| 3 | PA3 | **I2C1_SCL** | I2C1 | 1B | Išorinis I2C clock. I2C1 SCL valid: GPIO 3,7,11,15... Pora su SDA=2. |
| 4 | PA4 | **UART1_TX** | UART1 | 2A | GPS UART TX. UART1 TX valid: GPIO 4,8,20,24,36,40. Pasirinktas 4 — arčiausiai prie I2C1 (GPS modulis naudoja ir UART ir I2C). |
| 5 | PA5 | **UART1_RX** | UART1 | 2B | GPS UART RX. UART1 RX valid: GPIO 5,9,21,25,37,41. Pora su TX=4. |
| 6 | PA6 | **MOTOR1** | PIO0/DSHOT | 3A | DSHOT Motor 1. Motorų grupė GPIO 6-9 — visos PIO0 bazės 0 lange (0-31). Gretimi pinai supaprastina PCB trasavimą. |
| 7 | PA7 | **MOTOR2** | PIO0/DSHOT | 3B | DSHOT Motor 2. Greta M1 — trumpi PWM takeliai prie ESC. |
| 8 | PA8 | **MOTOR3** | PIO0/DSHOT | 4A | DSHOT Motor 3. Tęsia gretimų pinų bloką. |
| 9 | PA9 | **MOTOR4** | PIO0/DSHOT | 4B | DSHOT Motor 4. Paskutinis motoro pinas. Visi 4 motorai GPIO 6-9 — PIO0 bazė=0 leidžia pasiekti visus. |
| 10 | PA10 | **PIOUART0_TX** | PIO1 | 5A | Software UART #3 TX (telemetrija, DJI goggles). PIO1 gali naudoti bet kurį GPIO, bet 10-11 tęsia sekvenciją po motorų. |
| 11 | PA11 | **PIOUART0_RX** | PIO1 | 5B | Software UART #3 RX. Šalia TX=10. |
| 12 | PA12 | **SERVO1** | PWM hw | 6A | Servo išvestis 1. PWM slice 6 — nekonliktuoja su motorais (slices 3-4) nei su kitomis funkcijomis. |
| 13 | PA13 | **SERVO2** | PWM hw | 6B | Servo išvestis 2. Tas pats slice 6, kitas kanalas — abu 50Hz, konflikto nėra. |
| 14 | PA14 | **SERVO3** | PWM hw | 7A | Servo išvestis 3. PWM slice 7 — naujas slice, nepriklausomas nuo slice 6. |
| 15 | PA15 | **SERVO4** | PWM hw | 7B | Servo išvestis 4. Slice 7 kanalas B. Visi 4 servo ant slices 6-7 — izoliuota nuo visko kito. |
| 16 | PA16 | **PIOUART1_TX** | PIO1 | 8A | Software UART #4 TX (ESC telemetrija, papildomas). PIO1 turi 4 SM — pakanka 2 UART (TX+RX × 2). |
| 17 | PA17 | **PIOUART1_RX** | PIO1 | 8B | Software UART #4 RX. Šalia TX=16. |
| 18 | PA18 | **LED0** | GPIO | 9A | Būsenos LED (mėlynas). Paprastas GPIO — nereikia specialios funkcijos. |
| 19 | PA19 | **LED1 / Spare** | GPIO | 9B | Papildomas LED arba laisvas pinas. Rezervas būsimiems poreikiams. |
| 20 | PA20 | **OSD_W_PIN** | PIO2 | 10A | OSD pikselių išvestis (baltas/juodas). **PRIVALOMA:** 3 gretimi pinai OSD. PIO2 bazė=0 arba 16 — GPIO 20-22 telpa abiejuose languose. |
| 21 | PA21 | **OSD_EN_PIN** | PIO2 | 10B | OSD perdengimo aktyvavimas. Turi būti W_PIN+1. |
| 22 | PA22 | **OSD_SYNC_PIN** | PIO2 | 11A | OSD sync įėjimas (iš LM393). Turi būti W_PIN+2. |
| 23 | PA23 | **LED_STRIP** | PIO2 | 11B | WS2812 RGB LED juosta. PIO2 (ta pati kaip OSD) — dalijasi PIO2 bloku, bet naudoja atskirą SM. GPIO 23 yra tame pačiame PIO2 bazės lange kaip OSD pinai (20-22). |
| 24 | PA24 | **PINIO1** | GPIO | 0A* | Buzzer / VTX maitinimo valdymas. Paprastas GPIO išėjimas. *PWM slice 0 perrašo su GPIO 0 — nesvarbu, nes PINIO naudoja GPIO, ne PWM. |
| 25 | PA25 | **PINIO2** | GPIO | 0B* | Laisvas GPIO / vartotojo funkcija. Galimas panaudojimas: kameros perjungimas, apšvietimas. |

### Plokštės Vidiniai Pinai (GPIO 26–47)

| GPIO | BF Pin | Funkcija | Magistralė | Motyvas |
|------|--------|----------|------------|---------|
| 26 | PA26 | **GYRO_CLKIN** | GPIO/Clock | LSM6DSV išorinio laikrodžio įėjimas. Opcionalus — leidžia tiksliau sinchronizuoti giroskopo ODR su MCU laikrodžiu. Šalia SPI1 grupės (27-31). |
| 27 | PA27 | **GYRO_EXTI** | GPIO/EXTI | LSM6DSV INT1 — data-ready pertraukimas. **KRITINIS** skrydžio valdymui: užtikrina, kad giroskopo duomenys nuskaitomi tiksliai laiku. Bet kuris GPIO tinka EXTI — pasirinktas 27 dėl artumo prie SPI1 pinų. |
| 28 | PA28 | **SPI1_SDI** | SPI1 RX | LSM6DSV MISO (duomenys iš gyro → MCU). SPI1 RX valid pinai: **8, 12, 24, 28, 40**. GPIO 28 pasirinktas nes: (a) GPIO 8 naudojamas MOTOR3, (b) GPIO 12 naudojamas SERVO1, (c) 28 yra šalia kitų SPI1 pinų (30-31). |
| 29 | PA29 | **GYRO_CS** | GPIO | LSM6DSV Chip Select. Nors SPI1 CSn valid pinai yra 9,13,25,29,41 — CS dažnai valdomas kaip paprastas GPIO (ne hardware CS). GPIO 29 yra tarp SDI(28) ir SCK(30) — kompaktiškas layout. |
| 30 | PA30 | **SPI1_SCK** | SPI1 CLK | LSM6DSV SPI laikrodis. SPI1 SCK valid pinai: **10, 14, 26, 30, 42**. GPIO 30 pasirinktas nes: (a) GPIO 10 naudojamas PIOUART0, (b) GPIO 14 naudojamas SERVO3, (c) 30 yra greta SDI(28) ir SDO(31). |
| 31 | PA31 | **SPI1_SDO** | SPI1 TX | LSM6DSV MOSI (duomenys iš MCU → gyro). SPI1 TX valid pinai: **11, 15, 27, 31, 43**. GPIO 31 pasirinktas nes: (a) GPIO 11 naudojamas PIOUART0, (b) GPIO 15 naudojamas SERVO4, (c) 31 tęsia SPI1 bloką (28-31). |
| 32 | PA32 | **I2C0_SDA** | I2C0 | Vidinė I2C magistralė — BMP580 + BMM150 + INA226. I2C0 SDA valid: GPIO 0,4,8,12,16,20,24,28,**32**,36,40,44. GPIO 32 pasirinktas: (a) visi žemesni valid pinai jau užimti kitomis funkcijomis, (b) 32-33 yra RP2350B papildomi GPIO — tinka vidiniams komponentams. |
| 33 | PA33 | **I2C0_SCL** | I2C0 | Vidinė I2C clock. I2C0 SCL valid: GPIO 1,5,9,13,17,21,25,29,**33**,37,41,45. 33 yra natūrali pora su SDA=32. |
| 34 | PA34 | **SPI0_SCK** | SPI0 CLK | SD kortelės + FRAM SPI laikrodis. SPI0 SCK valid: GPIO 2,6,18,22,**34**,38,46. GPIO 34 pasirinktas: (a) žemesni pinai užimti (2=I2C1, 6=MOTOR1, 18=LED, 22=OSD), (b) 34-37 sudaro kompaktišką SPI0 bloką vidiniams komponentams. |
| 35 | PA35 | **SPI0_SDO** | SPI0 TX | SD kortelės + FRAM MOSI. SPI0 TX valid: GPIO 3,7,19,23,**35**,39,47. GPIO 35: (a) 3=I2C1, 7=MOTOR2, 19=LED/spare, 23=LED_STRIP — visi užimti. |
| 36 | PA36 | **SPI0_SDI** | SPI0 RX | SD kortelės + FRAM MISO. SPI0 RX valid: GPIO 0,4,16,20,**36**,44. GPIO 36: (a) 0=UART0, 4=UART1, 16=PIOUART1, 20=OSD — visi užimti. |
| 37 | PA37 | **SDCARD_CS** | GPIO | SD kortelės Chip Select. Bet kuris laisvas GPIO tinka. Pasirinktas 37 — šalia SPI0 linijų (34-36), trumpas takelis PCB. |
| 38 | PA38 | **FRAM_CS** | GPIO | FRAM Chip Select. Atskirtas nuo SD CS — leidžia adresuoti FRAM nepriklausomai. GPIO 38 šalia SD CS (37) — abi CS linijos greta, patogu trasavimui. |
| 39 | PA39 | **BARO_EOC** | GPIO/EXTI | BMP580 INT (data-ready). Opcionalus, bet rekomenduojamas — leidžia naudoti interrupt-driven barometo nuskaitymą vietoj polling. Pulsed mode, active HIGH, push-pull — nesukelia papildomo triukšmo I2C magistralėje. |
| 40 | PA40 | **ADC_RSSI** | ADC | RSSI analoginis įėjimas. RP2350B ADC kanalai: GPIO 40-47. Pasirinktas 40 — natūrali ADC grupės pradžia. |
| 41 | PA41 | **ADC_CURR** | ADC | Srovės jutiklio analoginis įėjimas. Šalia RSSI (40) — ADC grupė kartu. **Pastaba:** su INA226 šis pinas tampa neprivalomas (INA226 matuoja srovę per I2C). Galima naudoti kaip atsarginį ADC arba ACS712 analogini srovės jutiklį. |
| 42 | PA42 | **ADC_EXT1** | ADC | Papildomas ADC kanalas (laisvas). Galimas panaudojimas: temperatūra, papildoma srovė. |
| 43 | PA43 | **Spare ADC** | ADC/GPIO | Laisvas ADC/GPIO. Rezervuotas būsimam panaudojimui. |
| 44 | PA44 | **ADC_VBAT** | ADC | Baterijos įtampos matavimas per įtampos daliklį. **KRITINIS** — betaflight naudoja VBAT LVC (Low Voltage Cutoff) apsaugai. Įtampos daliklis ant PCB: pvz., 100kΩ/10kΩ (11:1). |
| 45 | PA45 | **LED0** (board) | GPIO | Plokštės būsenos šviesos diodas. Gali būti naudojamas vietoje PA18 arba kartu. Ant aukštų GPIO — arčiau MCU pakuotės. |
| 46 | PA46 | **Spare** | GPIO | Laisvas GPIO. Galimas panaudojimas: papildomas LED strip, test point. |
| 47 | PA47 | **Spare** | GPIO | Laisvas GPIO. Galimas panaudojimas: test point, debug. |

---

## PWM Slice Žemėlapis

RP2350B turi **12 PWM slices** (0-11). Kiekvienas slice turi 2 kanalus (A ir B)
su **vienodu dažniu** bet **nepriklausomu duty cycle**.

Formulė: `slice = (GPIO / 2) % 12`, `channel = GPIO % 2`

### Servo PWM paskirstymas

| Servo | GPIO | PWM Slice | Kanalas | Konfliktas? |
|-------|------|-----------|---------|-------------|
| SERVO1 | PA12 | **Slice 6** | A | Nėra — slice 6 naudojamas tik servo |
| SERVO2 | PA13 | **Slice 6** | B | Nėra — tas pats slice, bet kitas kanalas (OK: abu 50Hz) |
| SERVO3 | PA14 | **Slice 7** | A | Nėra — slice 7 naudojamas tik servo |
| SERVO4 | PA15 | **Slice 7** | B | Nėra — tas pats slice, kitas kanalas (OK: abu 50Hz) |

### Kodėl nekonliktuoja su motorais?

**DSHOT motorai naudoja PIO0, NE hardware PWM.** Todėl motorų GPIO 6-9 PWM
slices (3-4) nėra aktyvuoti — jokio konflikto su servo slices 6-7.

Jei ateityje reikėtų PWM motorų (ne DSHOT), motorai naudotų slices 3-4,
o servo liktų ant slices 6-7 — vis tiek nekonliktuoja.

### Pilnas PWM Slice Naudojimas

| Slice | GPIO A | GPIO B | Funkcija | Konfliktas |
|-------|--------|--------|----------|------------|
| 0 | PA0 (UART0_TX) | PA1 (UART0_RX) | Nepanaudotas (UART) | — |
| 0* | PA24 (PINIO1) | PA25 (PINIO2) | *Wraps — naudoja GPIO, ne PWM | — |
| 1 | PA2 (I2C1_SDA) | PA3 (I2C1_SCL) | Nepanaudotas (I2C) | — |
| 2 | PA4 (UART1_TX) | PA5 (UART1_RX) | Nepanaudotas (UART) | — |
| 3 | PA6 (MOTOR1) | PA7 (MOTOR2) | **Motorams tik jei PWM** (dabar PIO) | — |
| 4 | PA8 (MOTOR3) | PA9 (MOTOR4) | **Motorams tik jei PWM** (dabar PIO) | — |
| 5 | PA10 (PIOUART0) | PA11 (PIOUART0) | Nepanaudotas (PIO UART) | — |
| **6** | **PA12 (SERVO1)** | **PA13 (SERVO2)** | **Servo 50Hz ✓** | **Nėra** |
| **7** | **PA14 (SERVO3)** | **PA15 (SERVO4)** | **Servo 50Hz ✓** | **Nėra** |
| 8 | PA16 (PIOUART1) | PA17 (PIOUART1) | Nepanaudotas (PIO UART) | — |
| 9 | PA18 (LED0) | PA19 (LED1) | Nepanaudotas (GPIO) | — |
| 10 | PA20 (OSD_W) | PA21 (OSD_EN) | Nepanaudotas (PIO2) | — |
| 11 | PA22 (OSD_SYNC) | PA23 (LED_STRIP) | Nepanaudotas (PIO2) | — |

**Išvada:** Servo slices 6-7 yra visiškai izoliuoti — jokio konflikto su jokia
kita funkcija.

---

## PIO Blokų Paskirstymas

### PIO0 — DSHOT Motorai

| SM | GPIO | Funkcija | Pastaba |
|----|------|----------|---------|
| SM0 | PA6 | MOTOR1 DSHOT600 | Bendra PIO programa (13 arba 29 instr.) |
| SM1 | PA7 | MOTOR2 DSHOT600 | |
| SM2 | PA8 | MOTOR3 DSHOT600 | |
| SM3 | PA9 | MOTOR4 DSHOT600 | |

**GPIO Bazė:** 0 (GPIO langas 0-31)  
**Apribojimas:** Visi motorų pinai turi tilpti į 32-GPIO langą. GPIO 6-9 ∈ [0-31] ✓  
**Max motorai:** 4 (vienas PIO blokas = 4 SM)

### PIO1 — Software UART

| SM | GPIO | Funkcija | Pastaba |
|----|------|----------|---------|
| SM0 | PA10 | PIOUART0 TX | |
| SM1 | PA11 | PIOUART0 RX | |
| SM2 | PA16 | PIOUART1 TX | |
| SM3 | PA17 | PIOUART1 RX | |

**GPIO Bazė:** 0 (GPIO langas 0-31)  
**Pastaba:** Visos 4 SM panaudotos — daugiau PIO UART pridėti negalima.

### PIO2 — OSD + LED Strip

| SM | GPIO | Funkcija | Pastaba |
|----|------|----------|---------|
| SM0 | PA20-22 | OSD (W, EN, SYNC) | 3 gretimi pinai privalomi |
| SM1 | PA23 | LED Strip (WS2812) | |
| SM2 | — | Laisva | |
| SM3 | — | Laisva | |

**GPIO Bazė:** 0 arba 16 (abu tinka, nes GPIO 20-23 ∈ [0-31] IR [16-47])

---

## SPI Pin Multipleksavimo Apribojimai

RP2350B SPI pin multiplekseris leidžia tik **specifines GPIO** kombinacijas:

### SPI0 Valid Pinai

| Signalas | Valid GPIO | Šiame projekte |
|----------|-----------|----------------|
| SPI0 SCK | 2, 6, 18, 22, **34**, 38, 46 | **GPIO 34** ✓ |
| SPI0 TX (MOSI) | 3, 7, 19, 23, **35**, 39, 47 | **GPIO 35** ✓ |
| SPI0 RX (MISO) | 0, 4, 16, 20, 32, **36**, 44 | **GPIO 36** ✓ |
| SPI0 CSn | 1, 5, 17, 21, 33, **37**, 45 | **GPIO 37** (SD CS) |

**Kodėl GPIO 34-37?** Visi žemesni valid pinai jau užimti:
- GPIO 2,3: I2C1
- GPIO 6,7: DSHOT motorai
- GPIO 18,19: LED / spare
- GPIO 22,23: OSD / LED strip
- GPIO 0,4,16,20: UART / PIO / OSD
- GPIO 32: I2C0 SDA

### SPI1 Valid Pinai

| Signalas | Valid GPIO | Šiame projekte |
|----------|-----------|----------------|
| SPI1 SCK | 10, 14, 26, **30**, 42 | **GPIO 30** ✓ |
| SPI1 TX (MOSI) | 11, 15, 27, **31**, 43 | **GPIO 31** ✓ |
| SPI1 RX (MISO) | 8, 12, 24, **28**, 40 | **GPIO 28** ✓ |
| SPI1 CSn | 9, 13, 25, **29**, 41 | **GPIO 29** (Gyro CS) |

**Kodėl GPIO 28-31?** Žemesni valid pinai užimti:
- GPIO 8,9: DSHOT motorai
- GPIO 10,11: PIOUART0
- GPIO 12,13: Servo 1-2
- GPIO 14,15: Servo 3-4
- GPIO 24,25: PINIO

---

## I2C Pin Apribojimai

### I2C0 (Vidiniai jutikliai)

| Signalas | Valid GPIO | Šiame projekte |
|----------|-----------|----------------|
| I2C0 SDA | 0, 4, 8, 12, 16, 20, 24, 28, **32**, 36, 40, 44 | **GPIO 32** |
| I2C0 SCL | 1, 5, 9, 13, 17, 21, 25, 29, **33**, 37, 41, 45 | **GPIO 33** |

**Kodėl GPIO 32-33?**
- GPIO 0: UART0_TX
- GPIO 4: UART1_TX
- GPIO 8: MOTOR3
- GPIO 12: SERVO1
- GPIO 16: PIOUART1_TX
- GPIO 20: OSD_W_PIN
- GPIO 24: PINIO1
- GPIO 28: SPI1_SDI (gyro)
- **GPIO 32: PIRMAS LAISVAS VALID PIN** ✓

### I2C1 (Išoriniai jutikliai)

| Signalas | Valid GPIO | Šiame projekte |
|----------|-----------|----------------|
| I2C1 SDA | 2, 6, 10, 14, 18, 22, 26, 30, 34, 38, 42, 46 | **GPIO 2** |
| I2C1 SCL | 3, 7, 11, 15, 19, 23, 27, 31, 35, 39, 43, 47 | **GPIO 3** |

**Kodėl GPIO 2-3?** Pirmieji valid pinai — patogu kraštiniam konektoriui.
Taip pat šalia UART1 (GPIO 4-5) — GPS modulis dažnai naudoja ir UART ir I2C.

---

## UART Pin Apribojimai

### Hardware UART

| UART | Signalas | Valid GPIO | Šiame projekte |
|------|----------|-----------|----------------|
| UART0 | TX | 0, 12, 16, 28, 32, 44 | **GPIO 0** |
| UART0 | RX | 1, 13, 17, 29, 33, 45 | **GPIO 1** |
| UART1 | TX | 4, 8, 20, 24, 36, 40 | **GPIO 4** |
| UART1 | RX | 5, 9, 21, 25, 37, 41 | **GPIO 5** |

**Kodėl GPIO 0-1 ir 4-5?** Pirmieji valid pinai — tinka PCB kraštiniams
konektoriams. UART0 (RC) ir UART1 (GPS) yra du dažniausiai naudojami UART.

### Software UART (PIO)

| UART | Signalas | GPIO | PIO | SM |
|------|----------|------|-----|-----|
| PIOUART0 | TX | **PA10** | PIO1 | SM0 |
| PIOUART0 | RX | **PA11** | PIO1 | SM1 |
| PIOUART1 | TX | **PA16** | PIO1 | SM2 |
| PIOUART1 | RX | **PA17** | PIO1 | SM3 |

**Pastaba:** PIO UART gali naudoti **bet kurį GPIO** — nėra multiplekserio
apribojimų. Pasirinkti GPIO 10-11 ir 16-17 dėl PCB layout patogumui ir
laisvumo nuo kitų funkcijų.

### Bendras UART sąrašas

| # | Tipas | Paskirtis | GPIO TX | GPIO RX |
|---|-------|-----------|---------|---------|
| 1 | HW UART0 | **RC Radio (ELRS/CRSF)** | PA0 | PA1 |
| 2 | HW UART1 | **GPS** | PA4 | PA5 |
| 3 | PIO UART0 | **Telemetrija / DJI / MAVLink** | PA10 | PA11 |
| 4 | PIO UART1 | **ESC telemetrija / papildomas** | PA16 | PA17 |

**Iš viso: 4 UART kanalai** (2 hardware + 2 software)

---

## ADC Kanalai

RP2350B turi ADC kanalus ant GPIO 26-29 ir GPIO 40-47.

| GPIO | ADC Kanalas | Funkcija | Pastaba |
|------|-------------|----------|---------|
| 26 | ADC0 | ~~Laisvas~~ | Naudojamas GYRO_CLKIN — ADC neprieinamas |
| 27 | ADC1 | ~~Laisvas~~ | Naudojamas GYRO_EXTI — ADC neprieinamas |
| 28 | ADC2 | ~~Laisvas~~ | Naudojamas SPI1_SDI — ADC neprieinamas |
| 29 | ADC3 | ~~Laisvas~~ | Naudojamas GYRO_CS — ADC neprieinamas |
| 40 | ADC4 | **ADC_RSSI** | RSSI signalo stiprumas |
| 41 | ADC5 | **ADC_CURR** | Atsarginis srovės jutiklis (INA226 yra pagrindinis per I2C0) |
| 42 | ADC6 | **ADC_EXT1** | Papildomas (temperatūra, antra srovė, ...) |
| 43 | ADC7 | **Spare** | Rezervuotas |
| 44 | ADC8 | **ADC_VBAT** | Baterijos įtampa per daliklį |

**Pastaba:** GPIO 26-29 teoriškai turi ADC, bet šiame projekte naudojami SPI1
giroskopu. ADC poreikiams pilnai pakanka GPIO 40-44.

---

## SD Kortelė + FRAM — Bendra SPI0 Magistralė

```
                     SPI0 magistralė
                     ┌─────────────────────────────────┐
         SCK (PA34) ─┤                                 │
        MOSI (PA35) ─┤        SPI0 Bus                 │
        MISO (PA36) ─┤                                 │
                     └────────┬──────────┬─────────────┘
                              │          │
                         CS (PA37)  CS (PA38)
                              │          │
                         ┌────┴───┐ ┌────┴───┐
                         │ microSD│ │  FRAM  │
                         │  Card  │ │FM25V10 │
                         └────────┘ └────────┘
```

### Veikimo principas

- SCK, MOSI, MISO linijos yra bendros
- Kiekvienas įrenginys turi **atskirą CS** signalą
- Vienu metu aktyvus tik vienas įrenginys (CS = LOW)
- MCU valdo CS per paprastą GPIO (ne hardware SPI CS)

### SD Kortelė

| Parametras | Reikšmė |
|------------|---------|
| SPI režimas | Mode 0 (CPOL=0, CPHA=0) |
| Max dažnis | 25 MHz (SPI mode) |
| CS aktyvus | LOW |
| Paskirtis | Blackbox duomenų įrašymas |

### FRAM (FM25V10 / MB85RS256)

| Parametras | Reikšmė |
|------------|---------|
| SPI režimas | Mode 0 (CPOL=0, CPHA=0) |
| Max dažnis | 40 MHz (FM25V10) |
| CS aktyvus | LOW |
| Talpa | 128 KB (FM25V10) arba 32 KB (MB85RS256) |
| Paskirtis | Greitas konfigūracijos saugojimas |

**FRAM privalumai prieš Flash:**
- Nereikia erase ciklo — rašo tiesiai
- Neriboti rašymo ciklai (10^14 vs Flash ~100,000)
- Greitesnis rašymas (nėra erase delay)
- Idealu parametrų saugojimui kurie dažnai keičiasi

---

## Žinomi Konfliktai ir Sprendimai

### 1. PWM Slice Wrapping (GPIO 24-25)

**Problema:** GPIO 24-25 mapo į tą patį PWM slice kaip GPIO 0-1 (slice 0).  
**Sprendimas:** GPIO 24-25 naudojami kaip PINIO (GPIO režimu), ne PWM. Konflikto
nėra, nes PWM slice 0 neaktyvuotas nei vienai porai.

### 2. PIO2 Instrukcijų Atminties Limitas

**Problema:** PIO2 naudojamas ir OSD, ir LED Strip. Kiekvienas PIO blokas turi
tik 32 instrukcijų vietas.  
**Sprendimas:** OSD programa užima ~15 instrukcijų, LED Strip (WS2812) ~10
instrukcijų. Bendrai ~25/32 — telpa.

### 3. SPI0 Bus Contention (SD + FRAM)

**Problema:** Du SPI slave'ai ant vienos magistralės gali konfliktuoti jei abu
adresuojami vienu metu.  
**Sprendimas:** Programinė apsauga — betaflight bus driver užtikrina kad vienu
metu aktyvus tik vienas CS. SD kortelė naudojama blackbox rašymui (periodiškai),
FRAM — konfigūracijos saugojimui (retai). Laiko konflikto tikimybė minimali.

### 4. ADC Pinai Naudojami SPI1 (GPIO 26-29)

**Problema:** GPIO 26-29 turi ADC funkciją, bet naudojami SPI1 giroskopu.  
**Sprendimas:** ADC perkelta į GPIO 40-44, kur yra pakankamai ADC kanalų.
RP2350B papildomi GPIO (40-47) visi turi ADC funkciją.

### 5. OSD Pinų Seka

**Problema:** OSD PIO programa reikalauja 3 gretimų GPIO (W, EN, SYNC).  
**Sprendimas:** GPIO 20-22 yra gretimi ir laisvi nuo kitų specialių funkcijų.
Jie taip pat yra PIO2 GPIO bazės [0-31] arba [16-47] lange.

---

## Vizualus Pinout Žemėlapis

```
                        RP2350B QFN-80
                    ┌───────────────────┐
         PA0  (UART0_TX  RC) ─┤1              80├─ PA47 (Spare)
         PA1  (UART0_RX  RC) ─┤2              79├─ PA46 (Spare)
         PA2  (I2C1_SDA EXT) ─┤3              78├─ PA45 (LED0 board)
         PA3  (I2C1_SCL EXT) ─┤4              77├─ PA44 (ADC_VBAT)
         PA4  (UART1_TX GPS) ─┤5              76├─ PA43 (Spare ADC)
         PA5  (UART1_RX GPS) ─┤6              75├─ PA42 (ADC_EXT1)
         PA6  (MOTOR1 DSHOT) ─┤7              74├─ PA41 (ADC_CURR)
         PA7  (MOTOR2 DSHOT) ─┤8              73├─ PA40 (ADC_RSSI)
         PA8  (MOTOR3 DSHOT) ─┤9              72├─ PA39 (BARO_EOC)
         PA9  (MOTOR4 DSHOT) ─┤10             71├─ PA38 (FRAM_CS)
        PA10  (PIOUART0_TX ) ─┤11             70├─ PA37 (SD_CS)
        PA11  (PIOUART0_RX ) ─┤12             69├─ PA36 (SPI0_SDI)
        PA12  (SERVO1  PWM6A)─┤13             68├─ PA35 (SPI0_SDO)
        PA13  (SERVO2  PWM6B)─┤14             67├─ PA34 (SPI0_SCK)
        PA14  (SERVO3  PWM7A)─┤15             66├─ PA33 (I2C0_SCL)
        PA15  (SERVO4  PWM7B)─┤16             65├─ PA32 (I2C0_SDA)
        PA16  (PIOUART1_TX ) ─┤17             64├─ PA31 (SPI1_SDO)
        PA17  (PIOUART1_RX ) ─┤18             63├─ PA30 (SPI1_SCK)
        PA18  (LED0 status  ) ─┤19             62├─ PA29 (GYRO_CS)
        PA19  (LED1 / spare ) ─┤20             61├─ PA28 (SPI1_SDI)
        PA20  (OSD_W_PIN   ) ─┤21             60├─ PA27 (GYRO_EXTI)
        PA21  (OSD_EN_PIN  ) ─┤22             59├─ PA26 (GYRO_CLKIN)
        PA22  (OSD_SYNC_PIN) ─┤23             58├─          ...
        PA23  (LED_STRIP   ) ─┤24             57├─
        PA24  (PINIO1      ) ─┤25             56├─
        PA25  (PINIO2      ) ─┤26             55├─
                    ...       ─┤..             ..├─
                               └───────────────────┘

    Pastaba: Tikras RP2350B QFN-80 pinout gali skirtis —
    šis žemėlapis yra funkcinis, ne fizinis.
```

---

## Konfigūracijos Santrauka

| Resursas | Kiekis | GPIO | Magistralė |
|----------|--------|------|------------|
| Hardware UART | 2 | PA0-1, PA4-5 | UART0, UART1 |
| Software UART (PIO) | 2 | PA10-11, PA16-17 | PIO1 |
| **Iš viso UART** | **4** | | |
| DSHOT motorai | 4 | PA6-9 | PIO0 |
| Servo (PWM) | 4 | PA12-15 | PWM slices 6-7 |
| SPI (Gyro) | 1 | PA28-31 | SPI1 |
| SPI (SD+FRAM) | 1 | PA34-37 + PA38 | SPI0 |
| I2C (vidinis) | 1 | PA32-33 | I2C0 |
| I2C (išorinis) | 1 | PA2-3 | I2C1 |
| OSD | 1 | PA20-22 | PIO2 |
| LED Strip | 1 | PA23 | PIO2 |
| ADC | 4+ | PA40-44 | ADC |
| PINIO | 2 | PA24-25 | GPIO |
| LED | 2 | PA18, PA45 | GPIO |
| **Panaudoti GPIO** | **38** | | |
| **Laisvi GPIO** | **10** | PA19, PA26*, PA39*, PA42-43, PA45-47 | |

*PA26 ir PA39 priskirti, bet opcionalūs (CLKIN ir BARO_EOC)

---

## Rekomendacijos PCB Trasavimui

1. **UART konektoriai (PA0-1, PA4-5):** Plokštės krašte, šalia maitinimo pinų
2. **Motorų pinai (PA6-9):** Kuo arčiau krašto, trumpi takeliai prie ESC jungčių
3. **Servo pinai (PA12-15):** Atskiroje eilėje nuo motorų — servo signalai yra 50Hz PWM, ne DSHOT
4. **Gyro SPI (PA26-31):** Kuo trumpesni takeliai, 33Ω serijiniai rezistoriai
5. **I2C0 (PA32-33):** Trumpi takeliai prie BMP580, BMM150 ir INA226, 4.7kΩ pull-up prie 3.3V
6. **INA226 + Šuntas:** Šunto rezistorius (1mΩ) kuo arčiau INA226 IN+/IN- pinų. Kelvin jungimas. VS pinas jungtas prie baterijos (iki 36V), VDD prie 3.3V. 100nF dekuplingas.
6. **SD+FRAM SPI (PA34-38):** Šalia vienas kito, trumpi SCK takeliai
7. **OSD (PA20-22):** Toliau nuo motorų takelių, šalia video jungties
8. **ADC (PA40-44):** Atskirtas nuo skaitmeninių signalų, RC filtras ant VBAT

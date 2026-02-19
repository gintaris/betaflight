# PICO OSD — Budget Build (LM393 Comparator)

**Tikslas:** Kuo pigesnis analoginio OSD sprendimas Madflight FC3 plokštei.  
**Bendra kaina:** ~€0.25 (be video mikserio rezistorių) / ~€0.28 su visais komponentais.

---

## Kaip veikia FB_OSD

Betaflight PICO OSD (FB_OSD) naudoja RP2350 PIO (Programmable I/O) analoginio
vaizdo signalo perdengimui. Reikalingi 3 GPIO kontaktai:

| Signalas | GPIO | Kryptis | Funkcija |
|----------|------|---------|----------|
| OSD_W_PIN | PA22 | Output | Baltas/juodas pikselis (1=baltas, 0=juodas) |
| OSD_EN_PIN | PA23 | Output | Perdengimo aktyvavimas (1=OSD, 0=praleidžia video) |
| OSD_SYNC_PIN | PA24 | Input | Sync signalas: HIGH video metu, LOW sync metu |

**Svarbu:** Šie 3 kontaktai TURI būti iš eilės (PA22, PA23, PA24).

### PIO pikselių kodavimas (2 bitai per pikselį)

| W_PIN | EN_PIN | Rezultatas |
|-------|--------|------------|
| X | 0 | Skaidrus — video praeina be pakeitimų |
| 0 | 1 | Juodas pikselis |
| 1 | 1 | Baltas pikselis |

---

## Sync signalo reikalavimai

PIO programa (`osd_tx.pio`) tikisi:

```asm
wait 1 PIN, 0      ; laukia kol SYNC = HIGH → aktyvus video
wait 0 PIN, 0      ; laukia kol SYNC = LOW  → sync impulsas
```

**PIO tikisi:**
- **HIGH (3.3V)** — aktyvaus video / blanking metu
- **LOW (0V)** — sync impulsų metu

**Kodėl reikia aktyvios grandinės:**
Kompozitinis video signalas yra tik 0–1V, bet RP2350 GPIO HIGH slenkstis (VIH)
yra ~2.3V (0.7 × 3.3V). Video signalas NIEKADA nepasiekia šio slenksčio.
Todėl būtina aktyvi lygio konversijos grandinė.

---

## Komponentų sąrašas (BOM)

| Ref | Komponentas | Vertė | LCSC # | Kaina | Aprašymas |
|-----|-------------|-------|--------|-------|-----------|
| U1 | LM393DR2G | - | C7955 | €0.08 | Dvigubas komparatorius (SOIC-8) |
| R1 | Rezistorius | 1MΩ | C22935 | €0.01 | Įtampos daliklio viršus |
| R2 | Rezistorius | 47kΩ | C25819 | €0.01 | Įtampos daliklio apačia |
| R3 | Rezistorius | 4.7kΩ | C99782 | €0.01 | CSYNC pull-up prie 3.3V |
| R4 | Rezistorius | 470Ω | C114660 | €0.01 | W_PIN → video mikseris |
| R5 | Rezistorius | 1kΩ | C17513 | €0.01 | EN_PIN → video mikseris |
| R6 | Rezistorius | 75Ω | C174231 | €0.01 | Video terminacija |
| C1 | Kondensatorius | 100nF | C1591 | €0.01 | Maitinimo filtravimas |

**Bendra kaina: ~€0.15 (be R4, R5, R6) / ~€0.18 su visais**

---

## Schema

```
        ĮTAMPOS DALIKLIS (referencinė 0.15V)
        ═══════════════════════════════════

                    +3.3V
                      │
                    [1MΩ] R1
                      │
                      ├──── Ref taškas (~0.148V)
                      │
                    [47kΩ] R2
                      │
                     GND

        Vref = 3.3V × 47k / (1000k + 47k) = 0.148V


        LM393 KOMPARATORIUS
        ══════════════════

                        +3.3V
                          │
                        [100nF] C1 (dekuplingas)
                          │
                         GND

                        +3.3V
                          │
                        [4.7kΩ] R3 (pull-up)
                          │
                          ├───────────────────── PA24 (OSD_SYNC_PIN)
                          │
                 ┌────────┴────────────────┐
                 │     LM393DR2G           │
                 │                         │
  Camera Video ──┤ Pin 3 (IN1+)            │
                 │                         │
  Ref 0.148V ────┤ Pin 2 (IN1-)            │
                 │                         │
                 │ Pin 1 (OUT1) ───► R3    │
                 │                         │
        GND ─────┤ Pin 4 (GND)            │
                 │                         │
       +3.3V ───┤ Pin 8 (VCC)            │
                 │                         │
       +3.3V ───┤ Pin 6 (IN2-)  (unused) │
                 │ Pin 5 (IN2+) NC        │
                 │ Pin 7 (OUT2) NC        │
                 │                         │
                 └─────────────────────────┘


        VIDEO MIKSERIS (rezistorinis DAC)
        ═════════════════════════════════

  PA22 (W_PIN) ────[470Ω]───┐
                    R4       │
                             ├──────────────► VIDEO OUT ──► VTX
  PA23 (EN_PIN) ───[1kΩ]────┤
                    R5       │
                             │
  Camera Video ────[75Ω]────┤
  Out (tee)        R6       │
                             │
                            GND
```

---

## Veikimo principas

LM393 yra atviro kolektoriaus komparatorius:

| Sąlyga | IN+ vs IN- | Tranzistorius | Išėjimas |
|--------|-----------|---------------|----------|
| **Aktyvus video** (0.3–1V) | IN+ > IN- | IŠJUNGTAS | Pull-up → **3.3V (HIGH)** ✓ |
| **Sync impulsas** (0V) | IN+ < IN- | ĮJUNGTAS | Traukia žemyn → **0V (LOW)** ✓ |

Tai **tiesiogiai atitinka** PIO lūkesčius — jokios inversijos nereikia!

### Laiko diagrama

```
Kompozitinis video:
  1V  ──────────────── aktyvus video ──────────────────── 1V
 0.3V ─┐    ┌─── blanking ─┘                  └── blanking ─┐    ┌─── 0.3V
  0V   └────┘ sync                                    sync  └────┘ 0V
       |4.7µs|

Ref lygis (0.15V):
  ─────────────────────── 0.148V ─────────────────────────────────

LM393 išėjimas (PA24):
 3.3V ─┐    ┌────────────────── HIGH ───────────────────────┐    ┌── 3.3V
  0V   └────┘ LOW (sync)                         LOW (sync) └────┘ 0V
       ↑                                                    ↑
  PIO: wait 0                                          PIO: wait 0
  (aptinka sync pradžią)                       (kitas sync)
```

---

## Žingsnis po žingsnio pajungimas

### 1 žingsnis: Įtampos daliklis (referencinė)

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| +3.3V kontaktas | R1 vienas galas | Laidas |
| R1 kitas galas | R2 vienas galas | Sujungimas (ref taškas) |
| R2 kitas galas | GND | Laidas |

### 2 žingsnis: LM393 maitinimas

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| +3.3V | LM393 Pin 8 (VCC) | Laidas |
| GND | LM393 Pin 4 (GND) | Laidas |
| Pin 8 | Pin 4 | 100nF kondensatorius (C1) |

### 3 žingsnis: Komparatoriaus įėjimai

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| Camera Video Out | LM393 Pin 3 (IN1+) | Laidas (tiesiogiai!) |
| Ref taškas (0.15V) | LM393 Pin 2 (IN1-) | Laidas |

> **SVARBU:** Video ant **IN+ (pin 3)**, referencinė ant **IN- (pin 2)** —  
> tai užtikrina teisingą poliaritetą!

### 4 žingsnis: Komparatoriaus išėjimas

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| LM393 Pin 1 (OUT1) | PA24 (OSD_SYNC_PIN) | Laidas |
| LM393 Pin 1 (OUT1) | +3.3V | 4.7kΩ rezistorius (R3) |

### 5 žingsnis: Nenaudojamas komparatorius

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| +3.3V | LM393 Pin 6 (IN2-) | Laidas (neleidžia generuoti) |

### 6 žingsnis: Video mikseris

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| PA22 (OSD_W_PIN) | Video sujungimo taškas | 470Ω rezistorius (R4) |
| PA23 (OSD_EN_PIN) | Video sujungimo taškas | 1kΩ rezistorius (R5) |
| Camera Video Out | Video sujungimo taškas | 75Ω rezistorius (R6) |
| Video sujungimo taškas | VTX Video In | Laidas |

### 7 žingsnis: Masė

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| FC GND | Camera GND | Laidas |
| FC GND | VTX GND | Laidas |

> Visos masės turi būti sujungtos kartu (star ground geriausia).

---

## Santrauka — 12 jungčių iš viso

```
 1. +3.3V ──[1MΩ]──┬──[47kΩ]──► GND               (ref daliklis → 0.15V)
 2. +3.3V ─────────────────────► LM393 Pin 8 (VCC)
 3. GND ───────────────────────► LM393 Pin 4 (GND)
 4. Pin 8 ──[100nF]───────────► Pin 4               (dekuplingas)
 5. Camera Video ──────────────► LM393 Pin 3 (IN1+) (video įėjimas)
 6. Ref taškas (0.15V) ───────► LM393 Pin 2 (IN1-) (referencinė)
 7. LM393 Pin 1 ──────────────► PA24                (sync išėjimas)
 8. LM393 Pin 1 ──[4.7kΩ]─────► +3.3V              (pull-up)
 9. +3.3V ─────────────────────► LM393 Pin 6        (nenapudojamas komp.)
10. PA22 ──────────[470Ω]──────► Video taškas        (W mikseris)
11. PA23 ──────────[1kΩ]───────► Video taškas        (EN mikseris)
12. Camera Video ──[75Ω]───────► Video taškas → VTX  (terminacija)
```

---

## Programinė konfigūracija

### config.h (jau sukonfigūruota)

```c
#define USE_FB_OSD
#define OSD_W_PIN        PA22
#define OSD_EN_PIN       PA23
#define OSD_SYNC_PIN     PA24
```

### CLI komandos (po flash'inimo)

```
feature OSD
set vcd_video_system = AUTO
save
```

---

## Video signalas

| Parametras | PAL | NTSC |
|-----------|-----|------|
| Eilutės | 16 eilučių | 13 eilučių |
| Simboliai/eilutė | 30 | 30 |
| Simbolio rezoliucija | 12×18 pikselių | 12×18 pikselių |
| Frame buffer | 92KB ×2 (double-buffered) |

---

## Rezistorių derinimas

| Pakeitimas | Veiksmas |
|------------|----------|
| Ryškesnis baltas | R4: 470Ω → 330Ω |
| Tamsesnis baltas | R4: 470Ω → 680Ω |
| Tamsesnis juodas | R5: 1kΩ → 680Ω |
| Šviesenis juodas | R5: 1kΩ → 1.5kΩ |

---

## Trikčių šalinimas

| Problema | Galima priežastis | Sprendimas |
|----------|-------------------|-----------|
| Nėra OSD | Feature neįjungta | `feature OSD` + `save` |
| Nėra OSD | SYNC neaptinkamas | Patikrinti LM393 išėjimą multimetru |
| Nėra OSD | Poliaritetas neteisingas | Turi būti HIGH video metu, LOW sync metu |
| OSD mirga | Sync triukšmas | Pridėti 100nF prie PA24 |
| OSD mirga | Ground loop | Naudoti vieną masės tašką |
| OSD neteisinga pozicija | PAL/NTSC | `set vcd_video_system = PAL` |
| OSD per blyšku | R4 per didelė | Sumažinti iki 330Ω |
| OSD per ryšku | R4 per maža | Padidinti iki 560Ω |
| Video iškraipytas | Impedanso neatitikimas | Patikrinti 75Ω terminaciją |

---

## Kodėl pasyvi grandinė (100nF + 1MΩ) NEVEIKS

1. **Įtampos lygiai per žemi**: Video signalas yra 0–1V, RP2350 VIH ~2.3V
2. **Nėra lygio konversijos**: Pasyvūs komponentai negali stiprinti signalo
3. **Nėra sync atskyrimo**: Paprastas filtras negali atskirti sync nuo video

---

## KiCad footprints

| Komponentas | Footprint |
|-------------|-----------|
| LM393DR2G | SOIC-8_3.9x4.9mm_P1.27mm |
| 0603 rezistoriai | R_0603_1608Metric |
| 0603 kondensatoriai | C_0603_1608Metric |

---

## Nuorodos

- Betaflight PR #14882: https://github.com/betaflight/betaflight/pull/14882
- LM393 Datasheet: https://www.onsemi.com/pdf/datasheet/lm393-d.pdf
- LM393DR2G (LCSC): https://www.lcsc.com/product-detail/C7955.html
- RP2350 PIO: https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf

---

*Jei LM393 neduoda stabilaus rezultato, žiūrėti PLAN B: [PICO_OSD_PLAN_B.md](PICO_OSD_PLAN_B.md)*

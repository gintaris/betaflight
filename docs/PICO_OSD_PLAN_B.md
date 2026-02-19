# PICO OSD — Plan B (LM1881 Sync Separator)

**Naudoti kai:** LM393 komparatorius neduoda stabilaus sync aptikimo.  
**Bendra kaina:** ~€0.65  
**Privalumas:** Profesionalus sync separatorius su integruotu AGC ir filtru.

---

## Kodėl LM1881?

LM1881 yra specializuotas sync separatorius, kuris:
- Turi integruotą video AGC (automatic gain control)
- Patikimai atskiria sync nuo video turinio
- Veikia su PAL, NTSC, ir nestandardiniais signalais
- Turi back porch ir odd/even išėjimus (neprivalomi)

**Trūkumas:** LM1881 kainuoja ~€0.57 (vs €0.08 už LM393) ir reikalauja 5V maitinimo.

---

## Komponentų sąrašas (BOM)

| Ref | Komponentas | Vertė | LCSC # | Kaina | Aprašymas |
|-----|-------------|-------|--------|-------|-----------|
| U1 | LM1881M/TR | - | C518922 | €0.57 | Sync separatorius (SOIC-8) |
| U1 alt | LM1881N | - | C518923 | €0.59 | Sync separatorius (DIP-8) |
| R1 | Rezistorius | 680kΩ | C103420 | €0.01 | LM1881 laiko filtras |
| R2 | Rezistorius | 4.7kΩ | C99782 | €0.01 | CSYNC pull-up prie 3.3V |
| R3 | Rezistorius | 470Ω | C114660 | €0.01 | W_PIN → video mikseris |
| R4 | Rezistorius | 1kΩ | C17513 | €0.01 | EN_PIN → video mikseris |
| R5 | Rezistorius | 75Ω | C174231 | €0.01 | Video terminacija |
| C1 | Kondensatorius | 100nF | C1591 | €0.01 | LM1881 įėjimo AC sujungimas |
| C2 | Kondensatorius | 100nF | C1591 | €0.01 | Maitinimo dekuplingas |

**Bendra kaina: ~€0.65**

---

## Sync poliaritetas

PIO programa tikisi:
- **HIGH (3.3V)** — aktyvaus video metu
- **LOW (0V)** — sync impulsų metu

LM1881 CSYNC (Pin 7) yra **active LOW** — open-collector:
- **LOW** sync impulsų metu
- **HIGH** (pull-up) aktyvaus video metu

**Tiesioginis atitikimas — jokio inverterio nereikia!**  
Tereikia 4.7kΩ pull-up rezistoriaus **prie 3.3V** (NE 5V!).

---

## Schema

```
                              +5V (iš BEC / kameros maitinimo)
                               │
                       ┌───────┴───────┐
                       │    [100nF]    │
                       │      C2       │
                       │       │       │
                       │      GND      │
                       │               │
                 ┌─────┴───────────────┴─────┐
                 │     LM1881 (SOIC-8)       │
                 │                           │
                 │ Pin 1 GND ────────► GND   │
                 │                           │
  Camera ──┐     │ Pin 2 VIN                 │
  Video    │     │                           │
  Out   [100nF]  │                    VCC Pin 8│──── +5V
          C1     │                           │
          │      │ Pin 3 RSET                │
          └──────┤                           │
                 │   │                 CSYNC Pin 7│──┬──► PA24 (OSD_SYNC)
               [680kΩ]                       │  │
                 R1  │                       │  │
                 │   │ Pin 4 VREF  B.PORCH Pin 6│  [4.7kΩ] R2
                GND  │                       │  │        │
                     │           O/E Pin 5   │  │       +3.3V
                     └───────────────────────┘  │
                                                │
                                                │
              VIDEO MIKSERIS                    │
              ══════════════                    │
                                                │
    PA22 (W_PIN) ────[470Ω]───┐                 │
                      R3      │                 │
                              ├─────────────────┼──► VIDEO OUT ──► VTX
    PA23 (EN_PIN) ───[1kΩ]────┤                 │
                      R4      │                 │
                              │                 │
    Camera Video ────[75Ω]────┤                 │
    Out (tee)         R5      │                 │
                              │                 │
                             GND                │
```

---

## LM1881 pinout (SOIC-8 / DIP-8)

```
        ┌────────────┐
  GND  1│            │8  VCC (+5V)
  VIN  2│   LM1881   │7  CSYNC (open-collector, active LOW)
  RSET 3│            │6  BURST/BACK PORCH
  VREF 4│            │5  ODD/EVEN
        └────────────┘
```

- **Pin 1 (GND):** Masė
- **Pin 2 (VIN):** Video įėjimas (per 100nF AC coupling)
- **Pin 3 (RSET):** 680kΩ → GND (laiko konstanta)
- **Pin 4 (VREF):** Vidinis referencinis (nepajungti)
- **Pin 5 (O/E):** Odd/Even laukų indikatorius (nepajungti)
- **Pin 6 (B.PORCH):** Back porch indikatorius (nepajungti)
- **Pin 7 (CSYNC):** Composite sync išėjimas → PA24
- **Pin 8 (VCC):** +5V maitinimas

---

## Žingsnis po žingsnio pajungimas

### 1 žingsnis: LM1881 maitinimas

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| +5V (BEC) | LM1881 Pin 8 (VCC) | Laidas |
| GND | LM1881 Pin 1 (GND) | Laidas |
| Pin 8 | Pin 1 | 100nF kondensatorius (C2) |

### 2 žingsnis: Video įėjimas

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| Camera Video Out | C1 vienas galas | Laidas |
| C1 kitas galas | LM1881 Pin 2 (VIN) | 100nF kondensatorius (C1) |

### 3 žingsnis: Laiko rezistorius

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| LM1881 Pin 3 (RSET) | GND | 680kΩ rezistorius (R1) |

### 4 žingsnis: CSYNC → RP2350

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| LM1881 Pin 7 (CSYNC) | PA24 (OSD_SYNC_PIN) | Laidas (tiesiogiai!) |
| LM1881 Pin 7 (CSYNC) | +3.3V | 4.7kΩ rezistorius (R2) |

> **SVARBU:** Pull-up prie **3.3V**, NE 5V! Tai užtikrina saugų lygį RP2350 GPIO.

### 5 žingsnis: Video mikseris

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| PA22 (OSD_W_PIN) | Video taškas | 470Ω rezistorius (R3) |
| PA23 (OSD_EN_PIN) | Video taškas | 1kΩ rezistorius (R4) |
| Camera Video Out | Video taškas | 75Ω rezistorius (R5) |
| Video taškas | VTX Video In | Laidas |

### 6 žingsnis: Masės

| Iš kur | Į kur | Komponentas |
|--------|-------|-------------|
| FC GND | Camera GND | Laidas |
| FC GND | VTX GND | Laidas |

---

## Santrauka — 11 jungčių iš viso

```
 1. +5V ──────────────────────► LM1881 Pin 8 (VCC)
 2. GND ──────────────────────► LM1881 Pin 1 (GND)
 3. Pin 8 ──[100nF]──────────► Pin 1                (dekuplingas)
 4. Camera Video ──[100nF]───► LM1881 Pin 2 (VIN)   (AC coupling)
 5. LM1881 Pin 3 ──[680kΩ]──► GND                   (laiko rez.)
 6. LM1881 Pin 7 ────────────► PA24                  (CSYNC tiesiai!)
 7. LM1881 Pin 7 ──[4.7kΩ]──► +3.3V                 (pull-up)
 8. PA22 ──────────[470Ω]────► Video taškas          (W mikseris)
 9. PA23 ──────────[1kΩ]─────► Video taškas          (EN mikseris)
10. Camera Video ──[75Ω]─────► Video taškas → VTX    (terminacija)
11. GND sujungimas ──────────► Camera + VTX          (masė)
```

---

## Laiko diagrama

```
Kompozitinis video:
  1V  ──────────────── aktyvus video ──────────────────── 1V
 0.3V ─┐    ┌─── blanking ─┘                  └── blanking ─┐    ┌─── 0.3V
  0V   └────┘ sync                                    sync  └────┘ 0V
       |4.7µs|

LM1881 CSYNC (su 3.3V pull-up):
 3.3V ─┐    ┌────────────────── HIGH ───────────────────────┐    ┌── 3.3V
  0V   └────┘ LOW (sync impulsas)                LOW (sync) └────┘ 0V

PA24 (OSD_SYNC_PIN) = tas pats signalas:
 HIGH ─┐    ┌────────────── aktyvus video ──────────────────┐    ┌── HIGH
 LOW   └────┘                                               └────┘ LOW
       ↑                                                    ↑
  PIO: wait 0 PIN, 0                                   PIO: wait 0 PIN, 0
```

---

## Programinė konfigūracija

Tokia pati kaip ir Budget variante:

```c
#define USE_FB_OSD
#define OSD_W_PIN        PA22
#define OSD_EN_PIN       PA23
#define OSD_SYNC_PIN     PA24
```

```
feature OSD
set vcd_video_system = AUTO
save
```

---

## Trikčių šalinimas

| Problema | Galima priežastis | Sprendimas |
|----------|-------------------|-----------|
| Nėra OSD | LM1881 negauna 5V | Patikrinti VCC ant Pin 8 |
| Nėra OSD | C1 trūksta | AC coupling būtinas video įėjimui |
| Nėra OSD | Pull-up prie 5V | TURI būti prie 3.3V! |
| OSD mirga | 680kΩ neteisingas | Patikrinti R1 vertę |
| OSD mirga | Ground loop | Naudoti vieną masės tašką |
| Blogai aptinkamas sync | RSET vertė | Bandyti 470kΩ–1MΩ diapazone |

---

## KiCad footprints

| Komponentas | Footprint |
|-------------|-----------|
| LM1881M (SOIC-8) | SOIC-8_3.9x4.9mm_P1.27mm |
| LM1881N (DIP-8) | DIP-8_W7.62mm |
| 0603 rezistoriai | R_0603_1608Metric |
| 0603 kondensatoriai | C_0603_1608Metric |

---

## Nuorodos

- LM1881 Datasheet: https://www.ti.com/product/LM1881
- LM1881M/TR (LCSC): https://www.lcsc.com/product-detail/C518922.html
- LM1881N DIP-8 (LCSC): https://www.lcsc.com/product-detail/C518923.html
- Betaflight PR #14882: https://github.com/betaflight/betaflight/pull/14882

---

*Grįžti prie pigaus varianto: [PICO_OSD_BUDGET.md](PICO_OSD_BUDGET.md)*

# ESP32 Camera — Full Schematic (with LoRa)

## System Block Diagram

```text
                         USB-C (charging only)
                               │
                               ▼
  ┌──────────┐          ┌─────────────┐
  │ 18650 A  │──────────┤             │
  │ 3400mAh  │          │   TP4056    │
  └──────────┘          │  + DW01A    ├──── OUT+ ──┐
       ║    (parallel)  │  protection │            │
  ┌──────────┐          │             ├──── OUT- ──┤
  │ 18650 B  │──────────┤             │            │
  │ 3400mAh  │          └─────────────┘            │
  └──────────┘                                     │
                                             ┌─────┴──────────────────────────┐
                                             │      XIAO ESP32S3 Sense        │
                                             │                                │
                                             │   ┌──────────┐  ┌──────────┐   │
                                             │   │ OV2640   │  │ MicroSD  │   │
                                             │   │ Camera   │  │  Card    │   │
                                             │   │(internal)│  │(internal)│   │
                                             │   └──────────┘  └──────────┘   │
                                             │                                │
                                             └───────────────┬────────────────┘
                                                             │
                                          (8 wires: 3V3, GND, SCK, MISO, MOSI,
                                           NSS, RESET, DIO0 — see "LoRa Module"
                                           section below for exact pin mapping)
                                                             │
                                             ┌───────────────┴────────────────┐
                                             │     HopeRF RFM95W (SX1276)     │
                                             │            LoRa                │
                                             │                                │
                                             │   ANT ─────────────── SMA ─────┼──► Antenna
                                             └────────────────────────────────┘
```

> This block diagram is a high-level overview only. The single line between the
> XIAO and RFM95W boxes represents **8 separate wires** (power + SPI bus +
> control signals), not one connection. For the exact pin-to-pin mapping, see
> [LoRa Module — RFM95W (SX1276)](#lora-module--rfm95w-sx1276) below.

---

## Power Subsystem

```text
  ┌─────────────────────────────────────────────────────────┐
  │                    POWER SECTION                        │
  │                                                         │
  │  [18650 A]──┐                                           │
  │             ├──BAT+──────┬──────────────────────────►   │
  │  [18650 B]──┘            │                              │
  │                          │                              │
  │                    [TP4056 IN BAT+]                     │
  │                                                         │
  │  BAT- ───────────────────┴──────── GND                  │
  │                                                         │
  │  TP4056 PROG pin ── R_prog (1.2kΩ = 1A charge rate)     │
  │  TP4056 VCC  ── USB-C 5V                                │
  │  TP4056 OUT+ ─── D+ (1N5819) ─── XIAO BAT pin           │
  │  TP4056 OUT- ─── GND                                    │
  │                                                         │
  │  C1: 100µF electrolytic across BAT+ / BAT-              │
  │  C2: 100nF ceramic across TP4056 OUT+ / GND             │
  │  C3: 100nF ceramic across XIAO 3V3 / GND                │
  └─────────────────────────────────────────────────────────┘
```

> The `BAT+` rail also feeds the battery voltage divider (R1/R2 → XIAO D0) for
> ADC monitoring — that circuit is drawn separately and correctly below in
> [Battery Voltage Divider (ADC monitoring)](#battery-voltage-divider-adc-monitoring),
> since fitting it inside this box made the connections ambiguous.

### Battery Voltage Divider (ADC monitoring)

```text
  BAT+ ──── R1 (100kΩ) ────┬──── R2 (100kΩ) ──── GND
                           │
                         XIAO D0
                        (GPIO1 / ADC)

  Vout = Vbat × R2 / (R1 + R2) = Vbat × 0.5
  At 4.2V full:  Vout = 2.10V  (within 3.3V ADC range ✓)
  At 3.0V empty: Vout = 1.50V
```

---

## LoRa Module — RFM95W (SX1276)

### Wiring

```text
  XIAO ESP32S3 Sense                 HopeRF RFM95W
  ──────────────────                 ─────────────
  3V3  ──────────────────────────►  VCC
  GND  ──────────────────────────►  GND
  D8   (GPIO7  / SPI SCK)  ──────►  SCK
  D9   (GPIO8  / SPI MISO) ◄──────  MISO
  D10  (GPIO9  / SPI MOSI) ──────►  MOSI
  D3   (GPIO4  / CS)       ──────►  NSS
  D2   (GPIO3)             ──────►  RESET
  D1   (GPIO2)             ◄──────  DIO0  (TX done / RX ready IRQ)
  D0   (GPIO1)             ◄──────  DIO1  (RX timeout, optional)

  RFM95W ANT pin ── 50Ω coax ── SMA connector ── Antenna
```

> **SPI bus sharing:** The XIAO ESP32S3 Sense's internal MicroSD card uses the
> same SPI bus (GPIO7/8/9). This is fine — SPI supports multiple devices via
> separate CS lines. The SD card CS is handled internally by the board (GPIO21).
> Ensure the firmware de-asserts the RFM95W NSS (D3 HIGH) when accessing the SD
> card, and vice versa.

### Antenna

| Band | Region | Antenna length (1/4 wave) |
|------|--------|--------------------------|
| 868 MHz | Europe | ~86mm |
| 915 MHz | USA / AU / NZ | ~82mm |
| 433 MHz | Asia | ~173mm |

Use the matching RFM95W variant for your region. A simple wire soldered to the ANT pad and routed out of the enclosure through the PG7 cable gland works; a tuned rubber duck on an SMA connector is better.

---

## GPIO Allocation

| XIAO Pin | GPIO | Function | Direction | Connected to |
|----------|------|----------|-----------|--------------|
| D0 | GPIO1 | ADC battery monitor | IN | Voltage divider midpoint |
| D1 | GPIO2 | LoRa DIO0 (IRQ) | IN | RFM95W DIO0 |
| D2 | GPIO3 | LoRa RESET | OUT | RFM95W RESET |
| D3 | GPIO4 | LoRa NSS (CS) | OUT | RFM95W NSS |
| D4 | GPIO5 | I2C SDA | I/O | Reserved / expansion |
| D5 | GPIO6 | I2C SCL | OUT | Reserved / expansion |
| D6 | GPIO43 | UART TX | OUT | Debug serial (optional) |
| D7 | GPIO44 | UART RX | IN | Debug serial (optional) |
| D8 | GPIO7 | SPI SCK | OUT | RFM95W SCK + SD card |
| D9 | GPIO8 | SPI MISO | IN | RFM95W MISO + SD card |
| D10 | GPIO9 | SPI MOSI | OUT | RFM95W MOSI + SD card |
| BAT | — | LiPo battery | PWR | TP4056 OUT+ |
| 3V3 | — | 3.3V output | PWR | RFM95W VCC |
| GND | — | Ground | PWR | Common ground |

Internal (not on headers):

- GPIO11–GPIO20: OV2640 camera (DVP, handled by board)
- GPIO21: MicroSD CS (handled internally by Sense board)

---

## Solar Top-Up — CN3791 MPPT Charger

### Solar Wiring

```text
  5V/2W Solar Panel                  CN3791 Module
  ─────────────────                  ─────────────
  Panel (+) ──────────────────────►  VIN
  Panel (-) ──────────────────────►  GND

  CN3791 BAT+ ── D2 (1N5819) ─────► VCC_BATT (battery +)
  CN3791 GND  ─────────────────────► GND

  CN3791 PROG ── R_solar (3kΩ) ───► GND   (sets ~400mA charge current)
```

Both chargers (TP4056 for USB, CN3791 for solar) connect to the same battery rail
through separate Schottky diodes, preventing back-current between them.

```text
  ┌──────────────────────────────────────────────────────────┐
  │               DUAL-SOURCE CHARGING                       │
  │                                                          │
  │  USB-C 5V ──► TP4056 ──► D1 (1N5819) ──┐                 │
  │                                          ├──► VCC_BATT   │
  │  Solar 5-6V ─► CN3791 ─► D2 (1N5819) ──┘                 │
  │                                                          │
  │  VCC_BATT ──► XIAO BAT pin                               │
  └──────────────────────────────────────────────────────────┘
```

### Sizing the Solar Panel

| Condition | Panel output  | Charge current        | Hours to charge 6800mAh |
|-----------|---------------|-----------------------|-------------------------|
| Full sun  | ~333mA @ 6V   | ~280mA (after losses) | ~24h                    |
| Overcast  | ~50–80mA      | ~40–65mA              | — (trickle only)        |

A 1-hour capture cycle draws ~25mAh/day. Even in heavy overcast (~50mA average
for 2h of usable light), the panel delivers ~100mAh — 4× more than the daily draw.
In practice the battery will stay close to full indefinitely.

### CN3791 PROG Resistor Reference

| R_prog | Charge current                     |
|--------|------------------------------------|
| 1.5kΩ  | ~800mA                             |
| 2kΩ    | ~600mA                             |
| 3kΩ    | ~400mA ← recommended for 2W panel  |
| 6kΩ    | ~200mA                             |

Do not exceed ~50% of the panel's short-circuit current as charge current.

### Panel Placement

Mount the panel on the enclosure lid or a separate bracket angled toward the sky.
Run two wires (20AWG minimum) through a second PG7 cable gland. Polarity-protect
with D2 — the CN3791 does not have reverse-voltage protection on VIN.

---

## Full Wiring Netlist

```
Net: VCC_BATT
  18650-A(+) ── 18650-B(+) ── TP4056(BAT+) ── C1(+) ── R1(a)

Net: GND
  18650-A(-) ── 18650-B(-) ── TP4056(BAT-) ── C1(-) ── C2(-) ── C3(-)
  ── RFM95W(GND) ── XIAO(GND)

Net: VBAT_PROTECTED
  TP4056(OUT+) ── 1N5819(anode)

Net: XIAO_BAT
  1N5819(cathode) ── C2(+) ── XIAO(BAT)

Net: VCC_3V3
  XIAO(3V3) ── C3(+) ── RFM95W(VCC)

Net: VBAT_ADC
  R1(b) ── R2(a) ── XIAO(D0/GPIO1)

Net: GND_DIV
  R2(b) ── GND

Net: SPI_SCK
  XIAO(D8/GPIO7) ── RFM95W(SCK)

Net: SPI_MISO
  XIAO(D9/GPIO8) ── RFM95W(MISO)

Net: SPI_MOSI
  XIAO(D10/GPIO9) ── RFM95W(MOSI)

Net: LORA_NSS
  XIAO(D3/GPIO4) ── RFM95W(NSS)

Net: LORA_RESET
  XIAO(D2/GPIO3) ── RFM95W(RESET)

Net: LORA_DIO0
  XIAO(D1/GPIO2) ── RFM95W(DIO0)

Net: LORA_ANT
  RFM95W(ANT) ── SMA_connector(center)
  SMA_connector(shield) ── GND
```

---

## Component Values Summary

| Ref | Value | Package | Purpose |
|-----|-------|---------|---------|
| R1 | 100kΩ 1% | 0603 or axial | Battery divider upper |
| R2 | 100kΩ 1% | 0603 or axial | Battery divider lower |
| R_prog | 1.2kΩ | 0603 or axial | TP4056 charge current set (1A) |
| C1 | 100µF 10V | Electrolytic radial | Bulk decoupling at battery |
| C2 | 100nF 10V | Ceramic 0603 | Decoupling at TP4056 output |
| C3 | 100nF 10V | Ceramic 0603 | Decoupling at XIAO 3V3 |
| D1 | 1N5819 | DO-41 | Schottky reverse protection |

> To set a different TP4056 charge current, change R_prog:
> I_charge = 1200 / R_prog(kΩ) in mA.
> e.g. 1.2kΩ → 1000mA, 2kΩ → 600mA, 10kΩ → 120mA.

---

## Assembly Notes

1. **Wire the two 18650 cells in parallel** (+ to +, − to −) before connecting to TP4056. Both cells must be at the same voltage before paralleling — charge them individually first if new.

2. **TP4056 protection module OUT± vs BAT±**: Use the `OUT` pads (after the DW01A protection IC), not the `BAT` pads, to feed the XIAO. The protection IC handles over-discharge cutoff (~2.4V).

3. **1N5819 diode** sits between TP4056 OUT+ and XIAO BAT pin. Adds ~0.3V drop — acceptable. Prevents backfeed if XIAO 3V3 LDO ever back-powers the battery circuit.

4. **RFM95W RESET**: Pull HIGH via 10kΩ resistor to 3V3 to prevent spurious resets. Drive LOW briefly in firmware to reset the module.

5. **RFM95W antenna**: Never power the module without an antenna connected — the SX1276 PA can be damaged by the reflected RF energy.

6. **Enclosure**: Mount the SMA connector in a hole drilled in the enclosure wall. Run the antenna outside; LoRa RF does not penetrate metal enclosures well. The camera window (clear acrylic) should be sealed with neutral-cure silicone (not acetic-cure, which corrodes copper).

7. **Programming**: The XIAO USB-C port remains accessible via a short USB extension routed through the PG7 cable gland, or a sealed panel-mount USB-C pass-through.

---

## Firmware GPIO Init (Arduino/IDF reference)

```cpp
// LoRa (RadioLib or LoRa.h)
#define LORA_SCK   7
#define LORA_MISO  8
#define LORA_MOSI  9
#define LORA_CS    4   // D3
#define LORA_RST   3   // D2
#define LORA_DIO0  2   // D1

// Battery ADC
#define BAT_ADC_PIN  1  // D0 — reads 0–2.1V for 0–4.2V battery
```

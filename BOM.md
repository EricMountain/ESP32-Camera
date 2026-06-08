# ESP32 Camera — Bill of Materials

## Power Budget

- **Cycle:** wake → capture → transmit → sleep (every hour)
- **Active time:** ~15 seconds @ ~250mA
- **Deep sleep:** ~14µA
- **Consumption/day:** ~25 mAh
- **6 months (180 days):** ~4,500 mAh needed
- **With 85% regulator efficiency + 80% usable battery depth:** target ~6,600 mAh battery

---

## Core

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 1 | **Seeed XIAO ESP32S3 Sense** | Integrated OV2640 camera, 8MB PSRAM, microSD slot, deep sleep ~14µA | $14 |
| 2 | **MicroSD card 32GB Class 10** | Local image storage when WiFi unavailable | $6 |

The XIAO ESP32S3 Sense is the cleanest choice here — camera, SD slot, and ultra-low deep sleep current in one small board.

---

## Power System

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 3 | **2× Panasonic NCR18650B cells** (3400mAh each) | 6800mAh in parallel @ 3.7V — covers 6+ months | $14 |
| 4 | **18650 dual-cell parallel holder** with leads | Keeps cells in parallel (same voltage, doubled capacity) | $3 |
| 5 | **TP4056 + protection module** | Charges via USB-C, includes over-discharge/short protection | $2 |

> **Alternatively**, a single flat **6000mAh LiPo pouch** (e.g. 606090 size) is easier to fit in a slim enclosure — same price range.

---

## LoRa Add-on (optional)

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 6 | **HopeRF RFM95W module** (SX1276) | 868MHz (EU) or 915MHz (US/AU) — check your region | $8 |
| 7 | **Matching 1/4-wave whip antenna + SMA pigtail** | ~8cm wire or rubber duck | $3 |

**Important caveat on LoRa + images:** LoRa maxes out around 37 kbps. A compressed 320×240 JPEG is ~5–15 KB, which transfers in 1–3 seconds — workable, but this requires a LoRa gateway (private or The Things Network) on the receiving end. Full-res images are not practical over LoRa; store them on SD and retrieve via WiFi when available, using LoRa only as a notification channel or for thumbnail-sized images.

---

## Passives & Wiring

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 8 | **Schottky diode 1N5819** | Reverse polarity protection on battery input | $0.50 |
| 9 | **Resistors assortment** (10kΩ, 100kΩ) | Pull-ups for SPI/I2C, voltage divider for battery level sensing | $1 |
| 10 | **Capacitors** 100µF electrolytic + 100nF ceramic | Decoupling near power rails | $1 |
| 11 | **JST 2.0 PH connectors** (×3) | Battery, solar, camera — keyed so no accidental reversal | $2 |
| 12 | **22AWG silicone wire** (0.5m red/black) | Flexible, heat-resistant | $2 |

---

## Enclosure

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 13 | **IP67 ABS waterproof enclosure** (~100×60×35mm) | Hammond 1551 series or generic equivalent | $10 |
| 14 | **25mm clear acrylic disc** (3mm thick) | Camera window — glue inside with silicone sealant | $3 |
| 15 | **PG7 cable gland** (×1) | Weatherproof wire entry for charging/programming port | $1.50 |
| 16 | **M3 stainless standoffs + screws** | Mount PCB inside enclosure | $2 |

---

## Development & Flashing

| # | Part | Notes | Est. Price |
|---|------|-------|-----------|
| 17 | **USB-C cable** | XIAO programs directly via USB-C | $0 (likely have one) |
| 18 | **Small perfboard** (5×7cm) | Or order a custom PCB from JLCPCB once design is validated | $2 |

---

## Cost Summary

| Config | Approx. Total |
|--------|--------------|
| Base (WiFi only) | ~$64 |
| With LoRa | ~$75 |

---

## Optional: Solar Top-Up

If the camera is in a sunny outdoor spot, a small panel eliminates battery replacement entirely:

| Part | Notes | Est. Price |
|------|-------|-----------|
| **5V/2W mini solar panel** (60×110mm) | Enough to trickle-charge between transmissions | $7 |
| **CN3791 MPPT solar charger module** | Handles LiPo charging from variable solar input | $5 |

---

## Key Design Notes

1. **Deep sleep discipline** is what makes 6 months possible. The ESP32S3 must power down WiFi radio, camera, and SD before sleeping. Budget ~15s active per cycle.
2. **WiFi reconnect time** is the biggest power drain — use a static IP and store WiFi credentials to skip DHCP negotiation.
3. **Battery voltage monitoring** via ADC pin (with resistor divider) lets the firmware stop transmitting and go into low-power indefinite sleep when the battery drops below ~3.3V.
4. **LoRa gateway** is a separate purchase/project if you go that route — a RAK7268 indoor gateway is ~$90, or you can use Helium/TTN coverage if available in your area.

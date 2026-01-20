# Off-Grid Wireless Network Box - Parts List

## Project Overview
A rugged, waterproof pelican-style case containing an off-grid mesh network system with:
- LoRa mesh networking (Meshtastic) for long-range encrypted communication
- Local WiFi access point for ~10 users
- Battery power with solar charging capability
- External antenna connection (detachable)
- Waterproof controls

---

## 1. RUGGED ENCLOSURE

### Pelican Case (Recommended: 1450 or 1500 series)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Pelican 1450 Case** | Interior: 14.62" x 10.18" x 6.00", watertight, IP67, Pick N' Pluck foam | [Amazon.ca](https://www.amazon.ca/Pelican-1450-Case-Foam-Black/dp/B00013J86I) | ~$200-250 |
| **Alternative: Any waterproof case** | ~$50-100 |

**Canadian Pelican Dealers:**
- [Production Case](https://productioncase.com/collections/pelican-cases) - Ontario, ships nationwide
- [CasePlace.ca](https://caseplace.ca/) - Official distributor
- [Canada Case Co](https://canadacaseco.com/) - Mississauga, ON
- [Canadian Preparedness](https://canadianpreparedness.com/collections/pelican-cases)

**Note:** In this project we will use 3D printed internal mounts, so get the case **without foam** or with Pick N' Pluck foam you can remove.

---

## 2. COMPUTE & NETWORKING

### Main Computer (Gateway/Server)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Raspberry Pi 5 8GB** | 2.4GHz quad-core, handles 10+ users, runs offline maps | [Amazon.ca](https://www.amazon.ca/Raspberry-Pi-8GB-2023-Processor/dp/B0CK2FCG1K) | ~$120-150 |
| **Active Cooler for Pi 5** | Essential for sustained operation | Amazon.ca (often bundled) | ~$15-25 |
| **NVMe HAT + SSD** | For storing offline maps (OpenStreetMap tiles) | See below | ~$50-100 |

**Recommended Pi 5 Kits (includes cooler, case):**
- [CanaKit Raspberry Pi 5 Starter Kit](https://www.canakit.com/canakit-raspberry-pi-5-desktop-pc-with-nvme.html) - Canadian company
- [GeeekPi Starter Kit 8GB](https://www.amazon.ca/GeeekPi-Raspberry-Starter-Active-Readers/dp/B0CW18N8YR) - Amazon.ca ~$280

### Storage for Offline Maps

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Geekworm X1001 NVMe HAT** | PCIe M.2 adapter for Pi 5 | [Amazon.ca](https://www.amazon.ca/Geekworm-X1001-Peripheral-Raspberry-Computer/dp/B0CPPGGDQT) | ~$25 |
| **NVMe SSD 256GB-512GB** | M.2 2230/2242/2280 | Amazon.ca | ~$40-80 |
| **Alternative: Pimoroni NVMe Base** | Bottom-mount design | [PiShop.ca](https://www.pishop.ca/product/nvme-base-for-raspberry-pi-5-nvme-base/) | ~$30 |

---

## 3. LORA MESH NETWORK (MESHTASTIC)

### LoRa Nodes (Minimum 2 recommended - one for base, one portable)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **LILYGO T-Beam V1.2 915MHz** | ESP32 + LoRa + GPS, pre-flashed Meshtastic | [Rokland Store](https://store.rokland.com/products/lilygo-ttgo-meshtastic-t-beam-v1-1-esp32-lora-915-mhz-wireless-module-wifi-gps-neo-6m-with-oled-display-soldered-for-arduino-q349) | ~$50-60 USD |
| **LILYGO T-Beam Supreme** | Newer model, SMA connector, better GPS | [Rokland Store](https://store.rokland.com/products/lilygo-t-beam-supreme-esp32-s3-lora-development-board-sx1262-915mhz-gps-l76k-or-u-blox) | ~$70-80 USD |
| **RAK WisBlock Starter Kit** | Modular design, excellent for base stations | [Rokland Store](https://store.rokland.com/collections/lilygo-t-beam-series) | ~$50-70 USD |

**Important:** Use **915MHz** version for Canada (ISM band, license-free, encryption permitted)

### LoRa Antennas

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **915MHz 10dBi Whip Antenna** | 17-20cm, SMA Male, for portable use | [Amazon.com](https://www.amazon.com/915MHz-Antenna-10dBi-Meshtastic-Range/dp/B0DB5MJ3CZ) | ~$15-20 USD |
| **ALFA AOA-915-5ACM** | 5dBi outdoor omni, weatherproof | [Rokland Store](https://store.rokland.com/collections/all-helium-antennnas) | ~$40-50 USD |
| **Rokland 10dBi Outdoor** | 45" fiberglass, serious range | Rokland Store | ~$80-100 USD |

---

## 4. POWER SYSTEM

### Battery

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **12V 20Ah LiFePO4 Battery** | Compact, 3000+ cycles, built-in BMS | [Canbat.com](https://canbat.com/product/12v-20ah-lithium-battery/) - **Canadian** | ~$180-220 |
| **Alternative: ECO-WORTHY 12V 20Ah** | Portable with handle | [ECO-WORTHY Canada](https://ca.eco-worthy.com/products/12v-20ah-portable-lifepo4-deep-cycle-rechargeable-battery) | ~$130-150 |
| **Amazon.ca Option** | Various brands available | [Amazon.ca search](https://www.amazon.ca/s?k=12v+20ah+lifepo4) | ~$100-180 |

**Runtime Estimate:** Pi 5 (~5W) + LoRa node (~0.5W) + WiFi AP = ~8-10W total → 20Ah @ 12V = ~24+ hours runtime

### Solar Charging (Optional but recommended)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **MPPT Solar Charge Controller** | 10-20A, LiFePO4 compatible | [Amazon.ca](https://www.amazon.ca/s?k=solar+charge+controller+12v+lifepo4) | ~$35-70 |
| **Portable Solar Panel 50-100W** | Foldable for portability | Amazon.ca | ~$80-150 |

**Recommended Controllers:**
- Victron SmartSolar (premium, Bluetooth monitoring)
- Bioenno Power SC-122420JUD (LiFePO4 optimized)

### DC-DC Converter (12V Battery → 5V for Pi)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **12V to 5V USB-C Buck Converter** | 5A/25W, waterproof | [Amazon.com](https://www.amazon.com/Klnuoxj-Converter-Interface-Waterproof-Compatible/dp/B0CRVW7N2J) | ~$12-20 USD |
| **Alternative: PlusRoc Waterproof** | IP68 rated | [Amazon.com](https://www.amazon.com/PlusRoc-Waterproof-Converter-Compatible-Raspberry/dp/B09DGDQ48H) | ~$10-15 USD |

---

## 5. WATERPROOF CONTROLS & CONNECTORS

### Waterproof Switches with Rubber Caps

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Waterproof Toggle Switch 12V 30A** | SPST ON/OFF with boot cap | [Amazon.com](https://www.amazon.com/DaierTek-Waterproof-Toggle-Aluminum-Housing/dp/B08S34VYX6) | ~$15-20 for 2-pack |
| **Joinfworld Toggle Switch 4-pack** | IP65 with rubber covers | Amazon.com/Amazon.ca | ~$15-20 |
| **Rubber Boot Caps (spare)** | 12mm thread, various colors | Amazon | ~$8-12 for 10-20pc |

### Panel-Mount Antenna Connector (Waterproof Feedthrough)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **SMA Female Bulkhead Waterproof** | Panel mount with O-ring seal | [Amazon.com](https://www.amazon.com/outstanding-SMA-Female-Waterproof-Bulkhead-Connector/dp/B07BXZ2NDV) | ~$10-15 for 2-pack |
| **SMA Pigtail Cable** | SMA Female to U.FL/IPEX | Amazon | ~$8-12 |

**Wiring Note:** Mount SMA bulkhead connector in case wall → connect internal pigtail to LoRa node → external antenna screws onto outside

### Cable Glands (for power cables)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Waterproof Cable Glands** | Various sizes (PG7, PG9, PG11) | Amazon.ca | ~$10-15 for assortment |

---

## 6. SUMMARY & BUDGET

### Essential Components

| Category | Est. Cost (CAD) |
|----------|-----------------|
| Pelican Case 1450/1500 | $200-280 |
| Raspberry Pi 5 8GB + Cooler + Storage | $200-280 |
| LILYGO T-Beam Meshtastic x2 | $100-150 |
| LoRa Antenna (outdoor) | $50-100 |
| 12V 20Ah LiFePO4 Battery | $130-220 |
| DC-DC Converter | $15-25 |
| Waterproof Switches x3 | $20-30 |
| SMA Bulkhead + Cables | $20-30 |
| Cable Glands | $15-20 |
| **TOTAL (Essential)** | **$750-1,135** |

### Optional Additions

| Item | Est. Cost (CAD) |
|------|-----------------|
| Solar Panel 50-100W | $80-150 |
| MPPT Charge Controller | $35-70 |
| Extra LoRa nodes for users | $50-70 each |
| High-gain directional antenna | $80-150 |

---

## 7. CANADIAN RETAILERS SUMMARY

### Electronics & Computing
- **Amazon.ca** - Wide selection, fast shipping
- **Canada Computers** - Physical stores in QC/ON, [canadacomputers.com](https://www.canadacomputers.com)
- **Addison Électronique** - Montreal, [addison-electronique.com](https://www.addison-electronique.com)
- **CanaKit** - Canadian Raspberry Pi specialist, [canakit.com](https://www.canakit.com)
- **PiShop.ca** - Canadian Pi retailer

### Batteries & Solar
- **Canbat** - Canadian LiFePO4 specialist, free shipping, [canbat.com](https://canbat.com)
- **ECO-WORTHY Canada** - Ships from Canadian warehouse

### Pelican Cases
- **Production Case** - Ontario
- **CasePlace.ca** - Official distributor
- **Canadian Preparedness** - Also sells survival/preparedness gear

### LoRa/Meshtastic
- **Rokland Store** - US-based but ships to Canada, best Meshtastic selection
- **LILYGO Official** - Ships from China (longer wait but cheapest)
- **Amazon.com** - Often ships to Canada

---

## 8. NOTES FOR YOUR BUILD

### 3D Printing Considerations
- Design mounts with vibration dampening (rubber grommets)
- Include ventilation slots for heat dissipation
- SMA bulkhead needs ~16mm hole
- Toggle switches typically need 12mm holes

### Software Stack Suggestion
- **Raspberry Pi OS Lite** (headless)
- **Kiwix** for offline Wikipedia (you already know this!)
- **Organic Maps** or **MapTiler** for offline OpenStreetMap
- **Meshtastic Python API** for gateway functionality
- **hostapd** for WiFi access point
- **dnsmasq** for DHCP

### Power Budget
| Device | Power Draw |
|--------|------------|
| Raspberry Pi 5 (loaded) | 5-8W |
| T-Beam LoRa node | 0.3-0.5W |
| WiFi (integrated) | included in Pi |
| **Total** | ~6-10W |

With 20Ah @ 12V (240Wh) battery: **24-40 hours runtime** depending on load

---

*Last updated: January 2026*

# Off-Grid Wireless Network Box - Parts List

## Project Overview
A rugged, waterproof case containing an off-grid mesh network system with:
- LoRa mesh networking (Meshtastic) for long-range encrypted communication
- Local WiFi access point for ~10 users
- Battery power with solar charging capability
- External antenna connection (detachable)
- Waterproof controls

---

## 1. RUGGED ENCLOSURE

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **MEIJIA Portable Waterproof Hard Camera Case With Foam** | Outside: 15.98" x 12.99" x 6.85", watertight, IP67, Pick N' Pluck foam | [Amazon.ca](https://www.amazon.ca/dp/B07RYHL69R) | ~$115 |

**Note:** In this project we will use 3D printed internal mounts, so get the case **without foam** or with Pick N' Pluck foam you can remove.

---

## 2. COMPUTE & NETWORKING

### Main Computer (Gateway/Server)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Raspberry Pi 5 4GB Starter Kit** | 2.4GHz quad-core, with active cooler and storage, runs MeshMonitor for offline map with GPS | [CanaKit](https://www.canakit.com/canakit-raspberry-pi-5-starter-kit-turbine-black.html) | ~$250 |

---

## 3. LORA MESH NETWORK (MESHTASTIC)

### LoRa Nodes (Minimum 2 recommended - one for base, one portable)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **Heltec LoRa 32 V4 Meshtastic GPS ESP32** | Newer model | [Aliexpress](https://www.aliexpress.com/item/1005010017665425.html) | ~$40 |
| **LilyGo T-Echo Meshtastic** | ESP32 + LoRa + GPS, pre-flashed Meshtastic | [LilyGo Store](https://lilygo.cc/products/t-echo-meshtastic?variant=45315790307509) | ~$55 |
| **LilyGo T-Deck Plus Meshtastic** | ESP32-S3FN16R8 Dual-core LX7 microprocessor | [LilyGo Store](https://lilygo.cc/products/t-deck-plus-meshtastic?variant=45315795845301) | ~$80 |

**Important:** Use **915MHz** version for Canada (ISM band, license-free, encryption permitted)

### LoRa Antennas

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **915MHz 10dBi Whip Antenna** | Ø20*900mm, N-Female 868-960MHz (915MHz) | [Aliexpress](https://www.aliexpress.com/item/1005006663842344.html) | ~$65 |
| **UFL SMA Coax Cable SMA Female to U.FL** | Low Loss Extender 12inch (30cm) 5 Pack | [Amazon.ca](https://www.amazon.ca/dp/B098QH631G) | ~$10 |

---

## 4. POWER SYSTEM

### Battery

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **ECO-WORTHY Portable 12V 20Ah LiFePO4 Battery** | Compact, 4000+ cycles, built-in BMS | [Amazon.ca](https://www.amazon.ca/dp/B0CJFP3VW5) | ~$110 |
| **ECO-WORTHY 12V Lithium Battery Charger 5A** | Portable with display | [Amazon.ca](https://www.amazon.ca/dp/B08T95FF8K) | ~$60 |
| **10 Pack 5 Sizes Thermal Circuit Breakers** | 5A 10A 15A 20A 30A Circuit Breakers with Waterproof Button Caps | [Amazon.ca](https://www.amazon.ca/dp/B0DBPWF393) | ~$25 |
| **Joinfworld Toggle Switch 12V DC** | SPST 2 Pin Waterproof Switch | [Amazon.ca](https://www.amazon.ca/dp/B09SCXKKY2) | ~$15 |

**Runtime Estimate:** Pi 5 (~5W) + LoRa node (~0.5W) + WiFi AP = ~8-10W total → 20Ah @ 12V = ~24+ hours runtime

### DC-DC Converter (12V Battery → 5V for Pi)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **2-Pack DC 12V/24V to 5V USB C Buck Converter** | 3A/15W, waterproof | [Amazon.com](https://www.amazon.ca/dp/B0BNQ5JNWZ) | ~$20 |

### Solar Charging (Optional but recommended)

| Item | Description | Source | Est. Price (CAD) |
|------|-------------|--------|------------------|
| **MPPT Solar Charge Controller** | 10-20A, LiFePO4 compatible | [Amazon.ca](https://www.amazon.ca/s?k=solar+charge+controller+12v+lifepo4) | ~$35-70 |
| **Portable Solar Panel 50-100W** | Foldable for portability | Amazon.ca | ~$80-150 |

**Recommended Controllers:**
- Victron SmartSolar (premium, Bluetooth monitoring)
- Bioenno Power SC-122420JUD (LiFePO4 optimized)

---

## 6. SUMMARY & BUDGET

### Essential Components

| Category | Est. Cost (CAD) |
|----------|-----------------|
| Case | ~$115 |
| Computing | ~$250 |
| LoRa | ~$250 |
| Electrical | $230-350 (230$ without solar option) |
| **TOTAL (Essential)** | **$845-965** |

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

### Cases
- **Meijia Cases** - [meijiacase.com](https://www.meijiacase.com/hard-compact-case)
- **CasePlace.ca** - Official distributor
- **Canadian Preparedness** - Also sells survival/preparedness gear

### LoRa/Meshtastic
- **Aliexpress** - Cheap with free delivery
- **Rokland Store** - US-based but ships to Canada, best Meshtastic selection
- **LILYGO Official** - Ships from China (longer wait but cheapest)
- **Amazon.com** - Often ships to Canada

---

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
| LoRa node | 0.3-0.5W |
| WiFi (integrated) | included in Pi |
| **Total** | ~6-10W |

With 20Ah @ 12V (240Wh) battery: **24-40 hours runtime** depending on load

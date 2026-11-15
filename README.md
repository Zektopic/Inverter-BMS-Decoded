<div align="center">

# Inverter & BMS Protocols, Decoded

**A community reference for the communication protocols of battery management systems (BMS) and hybrid inverters:
frame formats, field tables, scaling, captured frames, and links to the original documents.**

![License](https://img.shields.io/badge/license-MIT-green)
![Contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen)

[Protocol index](#protocol-index) · [How to read a page](#how-to-read-a-page) · [Related projects](RELATED_PROJECTS.md) · [Contributing](CONTRIBUTING.md) · [Template](TEMPLATE.md)

</div>

---

Most battery and inverter vendors publish their protocols only as scattered PDFs, spreadsheets or forum posts, often in Chinese,
often with errors. This repository collects them in one consistent format so hobbyists and integrators can read a battery without
reverse-engineering it again. Where a value has been checked against real frames, the page says so; where sources disagree, the
page says that too.

## Protocol index

| Brand | Product / family | Link | Settings | Framing | Status | Page |
|---|---|---|---|---|---|---|
| **Seplos** | EMU10xx / 1101-SP (V2) | RS485 | 19200 8N1 | YD/T1363 ASCII | ✅ Verified | [SEPLOS](SEPLOS/README.md) |
| | Seplos V3 | RS485 | 19200 8N1 | Modbus RTU | 🌐 Community | [SEPLOS/V3](SEPLOS/V3.md) |
| | Seplos V2 / V3 Bluetooth | BLE | `0000ff00` / `0000fff0` | `7E 10` XMODEM / Modbus RTU | ✅ Verified | [SEPLOS/BLE](SEPLOS/BLE.md) |
| **Pylontech** | US2000 / US3000 / US5000 (low voltage) | RS485 | 115200 / 9600 | YD/T1363 ASCII | 📄 Vendor doc | [PYLONTECH](PYLONTECH/README.md#rs485-protocol-v33) |
| | Low-voltage inverter CAN (used by most "inverter-compatible" BMS) | CAN | 500 kbit/s | 11-bit, LE | 📄 Vendor doc | [PYLONTECH](PYLONTECH/README.md#low-voltage-can-protocol) |
| **JK (Jikong)** | JK-PB inverter BMS | RS485 | 115200, Modbus | FC16 trigger → 300-byte frame | ✅ Verified | [JK](JK/README.md#jk-pb-inverter-bms-rs485-modbus) |
| | JK-B / JK-BD smart BMS | UART / BLE | 115200 / `0000ffe0` | `4E 57` / `55 AA EB 90` | ✅ Verified | [JK/BLE](JK/BLE.md) |
| | BMS → display CAN (V1.02, 2019) | CAN | 250 kbit/s | 11-bit, LE | 📄 Vendor doc | [JK](JK/README.md#bms-to-display-can-protocol-v102) |
| **Daly** | Smart BMS | UART / RS485 | 9600 8N1 | `A5`, 13-byte frames | 🌐 Community | [DALY](DALY/README.md) |
| | Smart BMS / 100Balance / Sinowealth | BLE / UART | `0000fff0` / 9600 8N1 | `D2`/`81` Modbus, `A5`, `0A` | ✅ Verified | [DALY/BLE](DALY/BLE.md) |
| **JBD (Jiabaida)** | Smart BMS (Xiaoxiang / Overkill Solar / LLT) | UART / RS485 / BLE | 9600 8N1; BLE `0xFF00` | `DD A5`/`5A` … `77`, BE | 📄 Vendor doc | [JBD](JBD/README.md) |
| **ANT** | ANT BMS (Legacy 2019 & New 2021+) | UART / BLE | 19200 / `0000ffe0` | `AA 55 AA FF` / `7E A1` | ✅ Verified | [ANT](ANT/README.md) |
| **PACE** | P16S100A / P15S (rack batteries) | RS485 / RS232 / CAN | 9600 8N1 | YD/T1363 (`~25…`), Modbus | 📄 Vendor doc | [PACE](PACE/README.md) |
| **NEEY** | NEEY / Heltec / Enerkey Active Balancer | BLE | `0000ffe0` | `AA 55 11` / `55 AA 11` floats | ✅ Verified | [NEEY](NEEY/README.md) |
| **LiTime** | LiTime / Redodo / Power Queen BLE | BLE | `0000ffe0` | 105-byte binary frames, LE | ✅ Verified | [LITIME](LITIME/README.md) |
| **SOK** | 12 V / 24 V BLE (ABC) & 48 V Rack (PACE) | BLE / RS485 | `0000ffe0` / 9600 | `EE`/`CC` 20-byte / YD/T1363 | 🌐 Community | [SOK](SOK/README.md) |
| **ECO-WORTHY** | ECO-WORTHY / DCHOUSE / Creabest / CBT | BLE | `0000fff0` / `0000ffe5` | `A1`/`A2` Modbus / `AA 55` | ✅ Verified | [ECOWORTHY](ECOWORTHY/README.md) |
| **OGT / Others** | Offgridtec / D-powercore / TDT / E&J / RoyPow | BLE | `0000fff0` / `0000ffe0` | XOR ASCII / `7E` / `:`..`~` / `EA D1` | 🌐 Community | [OGT](OGT/README.md) |
| **EG4** | EG4-LL (Modbus) & LifePower4 | RS485 / CAN | 9600 8N1 | Modbus RTU / `7E … 0D` | 📄 Vendor doc | [EG4](EG4/README.md) |
| **Renogy** | Smart Lithium Battery, Rover MPPT & BT-1/BT-2 | RS485 / BLE | 9600 8N1 / `0000ffd0` | Modbus RTU | 📄 Vendor doc | [RENOGY](RENOGY/README.md) |
| **BYD** | Battery-Box Premium HVS / HVM | CAN; RS485 Modbus RTU to Fronius | 500 kbit/s | 11-bit, BE | 🌐 Community | [BYD](BYD/README.md) |
| **Voltronic** | Axpert / MPP Solar / InfiniSolar (PI30, PI18) | RS232 / USB | 2400 8N1 | ASCII + CRC16-XMODEM | 📄 Vendor doc | [VOLTRONIC](VOLTRONIC/README.md) |
| **Deye** | Deye / Sunsynk / Sol-Ark Hybrid | RS485 | 9600 8N1 | Modbus RTU | 📄 Vendor doc | [DEYE](DEYE/README.md) |
| **Luxpower** | LXP-LB / SNA / EG4 18kPV / 6000XP | RS485 / TCP | 19200 8N1 / port `8000` | Modbus RTU / `A1 1A` TCP | 📄 Vendor doc | [LUXPOWER](LUXPOWER/README.md) |
| **Growatt** | SPF / SPH / SPA / MIN / MOD | RS485 / TCP | 9600 8N1 / port `5279` | Modbus RTU / ShineWiFi | 🌐 Community | [GROWATT](GROWATT/README.md) |
| **Huawei** | SUN2000 & LUNA2000 | RS485 / TCP | 9600 8N1 / `502`, `6607` | Modbus RTU / TCP | 📄 Vendor doc | [HUAWEI](HUAWEI/README.md) |
| **Solis** | RHI / S5 / S6 Hybrid | RS485 / CAN | 9600 8N1 / 500 kbit/s | Modbus RTU / Pylontech CAN | 🌐 Community | [SOLIS](SOLIS/README.md) |
| **GoodWe** | ET / EH / ES / EM / BT Hybrid | RS485 / UDP | 9600 8N1 / port `8899` | Modbus RTU / UDP | 🌐 Community | [GOODWE](GOODWE/README.md) |
| **Victron** | SmartShunt / MPPT (VE.Direct) & GX CAN | UART / CAN | 19200 8N1 / 500 kbit/s | VE.Direct Text/HEX & CAN | 📄 Vendor doc | [VICTRON](VICTRON/README.md) |
| **SMA** | Sunny Island (LV) & Sunny Boy Storage (HV) | CAN | 500 kbit/s | 11-bit, LE | 🌐 Community | [SMA](SMA/README.md) |
| **Fronius** | Symo / Primo GEN24 Plus (inverter side) | RS485 to battery; Ethernet | Modbus RTU / TCP, HTTP | SunSpec, Solar API | 📄 Vendor doc | [FRONIUS](FRONIUS/README.md) |
| **MUST** | MUST battery BMS CAN V2.0.2 | CAN | 100 kbit/s (as stated) | 11-bit + 29-bit, LE | 📄 Vendor doc | [MUST](MUST/README.md) |
| **TDT** | TDT battery ("Force_L" / Pylon RS485 dialect) | RS485 | 9600 8N1 | YD/T1363 ASCII, CID2 0x60–0x64 | 🔬 Team decoded | [TDT](TDT/README.md) |
| **SHINEGLE** | Motor controller → instrument cluster (not a BMS) | CAN | 250 kbit/s | 29-bit, J1939-style | 📄 Vendor doc | [SHINEGLE](SHINEGLE/README.md) |

### Status legend

| | Meaning |
|---|---|
| ✅ **Verified** | Field tables checked against frames captured from a real device; the frames are in the page or its sources |
| 🔬 **Team decoded** | Decoded by contributors from their own captures; may be incomplete |
| 📄 **Vendor doc** | Transcribed from the manufacturer's document (linked or stored here); not independently checked |
| 🌐 **Community** | Based on open-source implementations and forum findings; follow the links and treat with care |

## How to read a page

Every protocol page follows [TEMPLATE.md](TEMPLATE.md):

1. **Summary**: product family, physical layer, serial/CAN settings, addressing, who is master.
2. **Framing**: byte layout and checksum algorithm with a worked example.
3. **Commands / frames**: what to send and what comes back, field by field with type, unit and scaling.
4. **Alarms and status bits**.
5. **Captured frames**: real examples to test a decoder against.
6. **Errata and disagreements**: where documents are wrong or sources conflict.
7. **Sources**: links to the original documents and implementations.

Conventions: `LE` / `BE` = little / big-endian; `u16`, `s16`… = unsigned / signed integers; scaling is written as the value of one
LSB (e.g. *0.01 V*); temperatures in *0.1 K* mean °C = (raw − 2731) / 10.

## Safety

These pages describe how to *read* batteries. Several protocols also contain commands that switch MOSFETs, change charge
parameters or calibration, or shut a pack down. Sending them to a live battery can damage it or create a fire risk. Read-only
access is strongly recommended. Nothing here is endorsed by the manufacturers.

## Contributing

Captured frames, corrections and new brands are very welcome. Read [CONTRIBUTING.md](CONTRIBUTING.md), start from
[TEMPLATE.md](TEMPLATE.md), and use the issue templates to report an error or request a protocol.

## Related projects

See [RELATED_PROJECTS.md](RELATED_PROJECTS.md) for the full curated index of open-source BLE, RS485, UART, CAN, and inverter protocol repositories with decoders and captured test frames.

- [patman15/aiobmsble](https://github.com/patman15/aiobmsble) & [patman15/BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA): 15+ BLE BMS decoders and captured notification frames.
- [syssi/esphome-seplos-bms](https://github.com/syssi/esphome-seplos-bms), [syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms), [syssi/esphome-jbd-bms](https://github.com/syssi/esphome-jbd-bms),
  [syssi/esphome-ant-bms](https://github.com/syssi/esphome-ant-bms), [syssi/esphome-neey-bms](https://github.com/syssi/esphome-neey-bms), [syssi/esphome-daly-bms](https://github.com/syssi/esphome-daly-bms): ESPHome components for these BMSs.
- [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery): Victron Venus OS driver for 20+ BMSs.
- [dalathegreat/Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator): EV batteries as home storage, with many inverter protocols.
- [ArminJo/JK-BMSToPylontechCAN](https://github.com/ArminJo/JK-BMSToPylontechCAN) and [NEEY-electronic/JK](https://github.com/NEEY-electronic/JK):
  included as submodules under [JK](JK/).
- Vega Datalogger: ESP32-S3 firmware by the maintainer that reads several of these BMSs over RS485/CAN and publishes them over MQTT.

## License

[MIT](LICENSE) for the text in this repository. Vendor documents stored here remain the property of their owners and are kept for
reference; links to originals are given where available.

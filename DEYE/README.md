# Deye / Sunsynk / Sol-Ark Hybrid Inverters — Modbus RTU Protocol

> Status: 📄 Vendor doc / 🌐 Community — transcribed from Deye's official Modbus RTU specifications (*V118* for single-phase, *V105.4* and *V104* for three-phase LV/HV) and cross-checked against [kellerza/sunsynk](https://github.com/kellerza/sunsynk), [StephanJoubert/home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman), and hardware-tested register notes in [tema-mazy/deye-modbus-test](https://github.com/tema-mazy/deye-modbus-test).

Deye manufactures hybrid inverters sold under its own name (**Deye** `SUN-xK-SG0xLP1` / `SG0xLP3` / `SG0xHP3`) and OEM-branded as **Sunsynk**, **Sol-Ark**, **Turbo Energy**, and others. All share the same Modbus RTU protocol over RS485 (and wrapped in Solarman V5 frames over TCP port `8899` on the Wi-Fi/LAN logger stick), split into two register maps:

1. **[Single-phase low-voltage hybrid](#single-phase-register-map-sun-xk-sg01lp1--sg03lp1--sg04lp1--sg05lp1)** (`SUN-3.6..16K-SG01LP1 / SG03LP1 / SG04LP1 / SG05LP1`, device type `0x0300` or `0x0003`)
2. **[Three-phase low-voltage & high-voltage hybrid](#three-phase-register-map-sg04lp3--sg05lp3--sg01hp3)** (`SUN-5..20K-SG04LP3 / SG05LP3` 48 V LV and `SUN-6..50K-SG01HP3` 160–700 V HV, device types `0x0500` / `0x0600`)

---

## Summary

| | |
|---|---|
| Products | **Single-phase LV (48 V):** Deye `SUN-3.6/5/6/7.6/8/12/16K-SG01LP1`, `SG03LP1`, `SG04LP1`, `SG05LP1`; Sunsynk 3.6 / 5 / 8 / 16 kW; Sol-Ark 8K / 12K / 15K<br>**Three-phase LV (48 V):** Deye `SUN-5/6/8/10/12/15/20K-SG04LP3-EU`, `SG05LP3`; Sunsynk 3P LV<br>**Three-phase HV (160–700 V):** Deye `SUN-6..50K-SG01HP3-EU-AM2`, `SG04HP3`; Sunsynk 3P HV |
| Physical layer | **RS485** on RJ45 socket (`Modbus` / `Modebus` or shared `BMS 485/CAN`); also exposed via RS232/logger port (Solarman V5 over TCP port `8899`) |
| Settings | **9600 8N1** (9600 baud, 8 data bits, no parity, 1 stop bit) |
| Addressing | Default slave ID **`1`** (configurable `1`–`247` on LCD under *Advanced → Multi-Inverter / Modbus SN*) |
| Roles | External controller / logger is **Modbus master**; inverter is **slave**. Only **one** Modbus master can poll a port at a time. Recommended block size ≤ 40 registers with ≥ 50 ms inter-frame gap |
| Byte order | **Big-endian (`BE`)** within each 16-bit register; **little-endian word order (`low word first`)** for 32-bit register pairs; CRC-16 is **little-endian (`LE`)** |
| Battery port | Separate RJ45 `BMS 485/CAN` (or `BMS`) socket where the inverter is **master**:<br>• **CAN** (pins 4 = CAN-H, 5 = CAN-L, 500 kbit/s): [Pylontech low-voltage CAN](../PYLONTECH/README.md#low-voltage-can-protocol) (`Lithium Mode 00`) or HV CAN ([BYD](../BYD/README.md) / Pylon HV)<br>• **RS485** (9600 8N1): [Pylontech RS485](../PYLONTECH/README.md#rs485-protocol-v33) (`Lithium Mode 12` on 1P / `0x05` in spec) and other BMS dialects |

### RJ45 connector pinouts (T-568B)

Depending on hardware revision, the inverter provides either a dedicated **`Modbus`** RJ45 socket alongside the **`BMS 485/CAN`** socket, or a combined **`BMS 485/CAN`** socket that carries both BMS CAN and Modbus RS485 on different pins:

| Pin | T-568B wire colour | Dedicated `Modbus` port | Combined `BMS 485/CAN` port | Notes |
|---|---|---|---|---|
| 1 | White / Orange | `RS485_B` (`sunspe-485_B`) | `RS485_B` | Modbus RTU Data − |
| 2 | Orange | `RS485_A` (`sunspe-485_A`) | `RS485_A` | Modbus RTU Data + |
| 3 | White / Green | `GND` (`GND_sunspe-485`) | — (Reserved) | RS485 ground reference |
| 4 | Blue | — | `CAN-H` | Battery BMS CAN High (500 kbit/s) |
| 5 | White / Blue | — | `CAN-L` | Battery BMS CAN Low (500 kbit/s) |
| 6 | Green | `GND` | `GND_485` | RS485 ground reference |
| 7 | White / Brown | `RS485_A` | `RS485_A` | Duplicated RS485 Data + |
| 8 | Brown | `RS485_B` | `RS485_B` | Duplicated RS485 Data − |

> **Notes on wiring:**
> - Despite the `sunspe-485` pin labels in Deye manuals, the `Modbus` port speaks the **native Deye register map**, not SunSpec.
> - If you split a combined `BMS 485/CAN` port to use a CAN battery (pins 4–5) and an RS485 monitor (pins 1–2 or 7–8) at the same time, crimp **only** pins 4–5 (and ground if needed) on the battery leg so RS485 voltages are not fed into the battery BMS.
> - Do **not** connect a Modbus master to the **`RS485 / Meter`** port: the inverter is master on that bus (polling Eastron/CHINT meters) and pin 3 may carry switched +5 V.

---

## Framing

Standard **Modbus RTU** framing over RS485:

```
[Slave Addr (1B)] [Function Code (1B)] [Data (N bytes)] [CRC-16 (2B, LE)]
```

### Function codes

| FC | Name | Address range | Usage |
|---|---|---|---|
| `0x03` | Read Holding Registers | `0`–`2000` (`10000+` for extended BMS packs) | Reads all identification, settings, and live telemetry registers (some firmwares also accept `0x04` as an alias) |
| `0x10` | Write Multiple Registers | `20`–`499` (1P) / `60`–`499` (3P) | Official write function code for both single-register (`N = 1`) and multi-register writes (`N = 1..123`) |
| `0x06` | Write Single Register | `20`–`499` (1P) / `60`–`499` (3P) | Supported on some firmware versions; `0x10` is preferred by Deye specs |

Exception responses set bit 7 of the function code (`0x83` for read errors, `0x90` for write errors) followed by a 1-byte exception code (`0x01` illegal function, `0x02` illegal data address, `0x03` illegal data value, `0x04` slave device failure).

### Checksum (CRC-16/MODBUS)

- **Polynomial:** `0x8005` (reflected `0xA001`), initial value `0xFFFF`, no final XOR.
- **Byte order:** Appended **little-endian** (`CRC_Lo` first, `CRC_Hi` second).

### Worked examples (verified in Python)

1. **Vendor specification example** (*Deye V118*, pages 2–3, with Slave ID `0x01`): read 3 registers starting at address `107` (`0x006B`, corrected battery capacity and PV energy):

```
Request:   01 03 00 6B 00 03 74 17
           ^^                      Slave address = 1
              ^^                   Function code = 0x03 (Read Holding Registers)
                 ^^^^^             Start address = 0x006B (107)
                       ^^^^^       Register count = 0x0003 (3 registers)
                             ^^^^^ CRC-16/MODBUS = 0x1774 (sent LE: 74 17)

Response:  01 03 06 02 2B 00 00 00 64 05 7A
           ^^                               Slave address = 1
              ^^                            Function code = 0x03
                 ^^                         Byte count = 6 (3 registers × 2 bytes)
                    ^^^^^                   Reg 107 = 0x022B (555 Ah)
                          ^^^^^             Reg 108 = 0x0000 (0.0 kWh Day PV)
                                ^^^^^       Reg 109 = 0x0064 (10.0 V PV1 voltage)
                                      ^^^^^ CRC-16/MODBUS = 0x7A05 (sent LE: 05 7A)
```

2. **Single-phase battery telemetry read** (registers `182`–`184` = `0x00B6`–`0x00B8`: battery temperature, voltage, SOC):

```
Request:   01 03 00 B6 00 03 E4 2D
Response:  01 03 06 04 C4 14 A0 00 4B 94 C5
                    ^^^^^                   Reg 182 = 0x04C4 (1220) -> (1220 - 1000) * 0.1 = 22.0 °C
                          ^^^^^             Reg 183 = 0x14A0 (5280) -> 5280 * 0.01 = 52.80 V
                                ^^^^^       Reg 184 = 0x004B (75)   -> 75 % SOC
```

3. **Three-phase battery telemetry read** (registers `586`–`591` = `0x024A`–`0x024F`):

```
Request:   01 03 02 4A 00 06 E5 A6
```

### Data encoding rules (both single-phase and three-phase)

- **Temperatures (`offset +1000`):** Stored as unsigned `u16` in `0.1 °C` with a `+1000` offset (`1000` = `0.0 °C`, `1220` = `22.0 °C`, `438` = `−56.2 °C`):
  $$\text{Temperature (°C)} = (\text{raw} - 1000) \times 0.1$$
  A raw value of `0` indicates no sensor is installed.
- **32-bit counters (`u32` / `s32`, low word first):** Split across two registers `[reg_low, reg_high]`:
  $$\text{raw}_{32} = \text{reg}_{\text{low}} + (\text{reg}_{\text{high}} \ll 16)$$
- **Sign conventions (`s16` two's complement):**
  - **Battery power & current:** **Negative (`−`) = charging**, **positive (`+`) = discharging**.
  - **Grid power & current:** **Positive (`+`) = importing from grid (BUY)**, **negative (`−`) = exporting to grid (SELL)**.
  - **Inverter power:** **Positive (`+`) = inverting (DC → AC)**, **negative (`−`) = rectifying/charging from AC (AC → DC)**.
- **Time-of-Use (TOU) slot times:** Decimal `HHMM` in a single `u16` (e.g. `0830` = `08:30`, `2359` = `23:59`).
- **System clock (3 registers):** Packed as 2 bytes per register:
  - Reg 1: `Hi` = Year − 2000 (`0`–`255`), `Lo` = Month (`1`–`12`)
  - Reg 2: `Hi` = Day (`1`–`31`), `Lo` = Hour (`0`–`23`)
  - Reg 3: `Hi` = Minute (`0`–`59`), `Lo` = Second (`0`–`59`)

---

## Single-Phase Register Map (`SUN-xK-SG01LP1` / `SG03LP1` / `SG04LP1` / `SG05LP1`)

All addresses below are **decimal** (with hex in parentheses) and read via **FC `0x03`**.

### Identification and status

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `0` | `0x0000` | 1 | u16 | enum | **Device type**: `0x0200` (`2`) String, `0x0300` (or `0x0003` on SG05LP1) Single-phase hybrid, `0x0400` Microinverter, `0x0500` 3P LV hybrid, `0x0600` 3P HV hybrid |
| `1` | `0x0001` | 1 | u16 | `1`–`247` | **Modbus slave address** |
| `2` | `0x0002` | 1 | u16 | `Hi.Lo` | **Communication protocol version** (e.g. `0x0102` = v1.2, `0x0201` = v2.1) |
| `3`–`7` | `0x0003`–`0x0007` | 5 | ASCII | 10 chars | **Serial number** (2 ASCII chars per register, `Hi` byte first, e.g. `"AH12345678"`) |
| `13` | `0x000D` | 1 | u16 | hex | Control board firmware version |
| `14` | `0x000E` | 1 | u16 | hex | Communication board firmware version |
| `16`–`17` | `0x0010`–`0x0011` | 2 | u32 | `0.1 W` | **Rated power** (`16` = low word, `17` = high word) |
| `22`–`24` | `0x0016`–`0x0018` | 3 | u16×3 | `YY/MM`, `DD/hh`, `mm/ss` | **System date & time** (R/W) |
| `59` | `0x003B` | 1 | u16 | enum | **Overall running state**: `0` Standby, `1` Self-check, `2` Normal, `3` Alarm, `4` Fault, `5` Activating |
| `90` | `0x005A` | 1 | u16 | `0.1 °C` (offset +1000) | **DC transformer temperature** |
| `91` | `0x005B` | 1 | u16 | `0.1 °C` (offset +1000) | **Heat sink / IGBT radiator temperature** |
| `92` / `94` | `0x005C` / `0x005E` | 1 | u16 | enum | **SD card status** (`1000` = fault, `2000` = normal; see [Errata](#errata-and-disagreements)) |
| `95` | `0x005F` | 1 | u16 | `0.1 °C` (offset +1000) | **Environment temperature** (on some firmwares, high word of Total Gen energy) |
| `194` | `0x00C2` | 1 | u16 | `0` / `1` | **Grid side relay status** (`0` = Off-grid / disconnected, `1` = On-grid / closed) |

### Battery and BMS telemetry (single-phase)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `107` | `0x006B` | 1 | u16 | `1 Ah` | Corrected battery capacity (`Corrected_AH`) |
| `182` | `0x00B6` | 1 | u16 | `0.1 °C` (offset +1000) | **Battery temperature** (`(raw − 1000) × 0.1 °C`) |
| `183` | `0x00B7` | 1 | u16 | `0.01 V` | **Battery voltage** (e.g. `5280` = `52.80 V`) |
| `184` | `0x00B8` | 1 | u16 | `1 %` | **Battery SOC** (`0`–`100 %`) |
| `185` | `0x00B9` | 1 | u16 | enum | **Battery status** (`0` = Charge, `1` = Stand-by, `2` = Discharge) |
| `190` | `0x00BE` | 1 | s16 | `1 W` | **Battery power** (`−` = charging, `+` = discharging) |
| `191` | `0x00BF` | 1 | s16 | `0.01 A` | **Battery current** (`−` = charging, `+` = discharging) |
| `312` | `0x0138` | 1 | u16 | `0.01 V` | **BMS charge voltage limit** (reported by BMS) |
| `313` | `0x0139` | 1 | u16 | `0.01 V` | **BMS discharge voltage limit** (reported by BMS) |
| `314` | `0x013A` | 1 | u16 | `1 A` | **BMS charge current limit** (dynamic limit from BMS) |
| `315` | `0x013B` | 1 | u16 | `1 A` | **BMS discharge current limit** (dynamic limit from BMS) |
| `316` | `0x013C` | 1 | u16 | `1 %` | BMS real-time SOC |
| `317` | `0x013D` | 1 | u16 | `0.01 V` | BMS real-time voltage |
| `318` | `0x013E` | 1 | s16 | `1 A` | BMS real-time current |
| `319` | `0x013F` | 1 | u16 | `0.1 °C` (offset +1000) | BMS real-time temperature |
| `322` / `323` | `0x0142` / `0x0143` | 1 | u16 | bitmask | BMS alarm flag (`322`) / BMS fault flag (`323`) |
| `600`–`613` | `0x0258`–`0x0265` | 14 | u16 | see note | **Pack 1 BMS details** (repeated every 14 regs up to Pack 15 at `796`–`809`): `+0` Voltage (`0.01 V`), `+1` Current (`0.1 A`), `+2` Temp (`0.1 °C`, offset +1000), `+3` SOC (`1 %` or `0.1 %`), `+4` Remain Cap (`0.1 Ah`), `+5` Total Cap (`0.1 Ah`), `+6` Charge V (`0.01 V`), `+7` Charge I (`0.1 A`), `+8` Discharge I (`0.1 A`), `+9` Max Cell V (`0.01 V`), `+10` Min Cell V (`0.01 V`), `+11` Cycles (`1`), `+12` Warning, `+13` Fault |

### PV, Grid, Inverter, Load, and AUX/Gen telemetry (single-phase)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `79` | `0x004F` | 1 | u16 | `0.01 Hz` | **Grid frequency** |
| `109` / `110` | `0x006D` / `0x006E` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV1 voltage** (`109`) / **PV1 current** (`110`) |
| `111` / `112` | `0x006F` / `0x0070` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV2 voltage** (`111`) / **PV2 current** (`112`) |
| `113` / `114` | `0x0071` / `0x0072` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV3 voltage** (`113`) / **PV3 current** (`114`) (models with 3–4 MPPTs) |
| `115` / `116` | `0x0073` / `0x0074` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV4 voltage** (`115`) / **PV4 current** (`116`) (models with 4 MPPTs) |
| `186`–`189` | `0x00BA`–`0x00BD` | 4 | u16 | `1 W` | **PV1 / PV2 / PV3 / PV4 input power** (`186` = PV1, `187` = PV2, `188` = PV3, `189` = PV4; note `deye_hybrid.yaml` maps `189` as Battery Status on older 2-MPPT units) |
| `150` / `151` | `0x0096` / `0x0097` | 1+1 | u16 | `0.1 V` | **Grid voltage L1-N** (`150`) / **L2-N** (`151`, split-phase) |
| `154` / `155` | `0x009A` / `0x009B` | 1+1 | u16 | `0.1 V` | **Inverter output voltage L1-N** (`154`) / **L2-N** (`155`) |
| `157` / `158` | `0x009D` / `0x009E` | 1+1 | u16 | `0.1 V` | **Load voltage L1** (`157`) / **L2** (`158`) |
| `160` / `161` | `0x00A0` / `0x00A1` | 1+1 | s16 | `0.01 A` | **Grid side current L1** (`160`) / **L2** (`161`) |
| `164` / `165` | `0x00A4` / `0x00A5` | 1+1 | s16 | `0.01 A` | **Inverter output current L1** (`164`) / **L2** (`165`) |
| `166` | `0x00A6` | 1 | s16 | `1 W` | **AUX / Generator / Micro-inverter power** (`+` as SmartLoad output, `−` as micro-inverter input) |
| `167` / `168` | `0x00A7` / `0x00A8` | 1+1 | s16 | `1 W` | **Grid internal CT power L1** (`167`) / **L2** (`168`) |
| `169` | `0x00A9` | 1 | s16 | `1 W` (`10 W` on 16 kW) | **Grid total power** (`+` = BUY / import, `−` = SELL / export) |
| `170` / `171` | `0x00AA` / `0x00AB` | 1+1 | s16 | `1 W` | **Grid external CT power L1** (`170`) / **L2** (`171`) |
| `172` | `0x00AC` | 1 | s16 | `1 W` (`10 W` on 16 kW) | **Grid external CT total power** (`+` = import, `−` = export) |
| `173` / `174` | `0x00AD` / `0x00AE` | 1+1 | s16 | `1 W` | **Inverter output power L1** (`173`) / **L2** (`174`) |
| `175` | `0x00AF` | 1 | s16 | `1 W` | **Inverter output total power** (`+` = inverting, `−` = AC charging) |
| `176` / `177` | `0x00B0` / `0x00B1` | 1+1 | s16 | `1 W` | **Load power L1** (`176`) / **L2** (`177`) |
| `178` | `0x00B2` | 1 | s16 | `1 W` | **Load total power** (total consumption `169 + 175`; see [Errata](#errata-and-disagreements) for reg `179` UPS load power on SG05LP1) |
| `192` | `0x00C0` | 1 | u16 | `0.01 Hz` | **Load frequency** |
| `193` | `0x00C1` | 1 | u16 | `0.01 Hz` | **Inverter output frequency** |

### Energy counters (single-phase)

All 32-bit total counters are stored **low word first** (`[low, high]`).

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `60` | `0x003C` | 1 | s16 | `0.1 kWh` | Day active energy |
| `62` | `0x003E` | 1 | u16 | `0.1 kWh` | Day generator energy |
| `63`–`64` | `0x003F`–`0x0040` | 2 | s32 | `0.1 kWh` | Total active energy |
| `70` | `0x0046` | 1 | u16 | `0.1 kWh` | **Day battery charge energy** |
| `71` | `0x0047` | 1 | u16 | `0.1 kWh` | **Day battery discharge energy** |
| `72`–`73` | `0x0048`–`0x0049` | 2 | u32 | `0.1 kWh` | **Total battery charge energy** |
| `74`–`75` | `0x004A`–`0x004B` | 2 | u32 | `0.1 kWh` | **Total battery discharge energy** |
| `76` | `0x004C` | 1 | u16 | `0.1 kWh` | **Day grid import (buy) energy** |
| `77` | `0x004D` | 1 | u16 | `0.1 kWh` | **Day grid export (sell) energy** |
| `78`, `80` | `0x004E`, `0x0050` | 2 | u32 | `0.1 kWh` | **Total grid import (buy) energy** (**non-adjacent!** `78` = low word, `80` = high word; `79` is Grid frequency) |
| `81`–`82` | `0x0051`–`0x0052` | 2 | u32 | `0.1 kWh` | **Total grid export (sell) energy** |
| `84` | `0x0054` | 1 | u16 | `0.1 kWh` | **Day load energy** |
| `85`–`86` | `0x0055`–`0x0056` | 2 | u32 | `0.1 kWh` | **Total load energy** |
| `92`, `95` | `0x005C`, `0x005F` | 2 | u32 | `0.1 kWh` | Total generator energy (`92` = low word, `95` = high word; added in V117) |
| `96`–`97` | `0x0060`–`0x0061` | 2 | u32 | `0.1 kWh` | **Total PV energy** |
| `108` | `0x006C` | 1 | u16 | `0.1 kWh` | **Day PV energy** |

### Key settings and Time-of-Use (single-phase, R/W)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `200` | `0x00C8` | 1 | u16 | enum | Battery charging mode (`0` = Lead-acid 4-stage, `1` = Lithium BMS) |
| `201` / `202` / `203` | `0x00C9`–`0x00CB` | 1×3 | u16 | `0.01 V` | Battery Equalization (`201`), Absorption (`202`), Float (`203`) voltage (`3800`–`6100`) |
| `204` | `0x00CC` | 1 | u16 | `1 Ah` | Battery nominal capacity (`0`–`2000 Ah`) |
| `206` | `0x00CE` | 1 | s16 | `1 W` | Zero-export / Grid trickle feed power (`−500` to `500 W`, default `20`–`40 W`) |
| `210` / `211` | `0x00D2` / `0x00D3` | 1+1 | u16 | `1 A` | **Battery max charge current** (`210`) / **max discharge current** (`211`) (`0`–`185 A`, or `0`–`290 A` on 16 kW) |
| `213` | `0x00D5` | 1 | u16 | enum | Battery operation mode (`0` = Voltage, `1` = Capacity / SOC %, `2` = No battery) |
| `217` / `218` / `219` | `0x00D9`–`0x00DB` | 1×3 | u16 | `1 %` | **Battery Shutdown** (`217`), **Restart** (`218`), **Low Batt** (`219`) SOC limits |
| `220` / `221` / `222` | `0x00DC`–`0x00DE` | 1×3 | u16 | `0.01 V` | **Battery Shutdown** (`220`), **Restart** (`221`), **Low Batt** (`222`) voltage limits |
| `227` / `230` | `0x00E3` / `0x00E6` | 1+1 | u16 | `1 A` | Generator charge battery current (`227`) / **Grid charge battery current** (`230`) |
| `231` / `232` | `0x00E7` / `0x00E8` | 1+1 | u16 | bit 0 | Generator charge enabled (`231`) / **Grid charge enabled** (`232`) |
| `235` | `0x00EB` | 1 | u16 | enum | AUX / Generator port mode (`0` = Generator input, `1` = SmartLoad output, `2` = Micro-inverter input) |
| `243` | `0x00F3` | 1 | u16 | enum | **Energy management priority** (`0` = Battery First, `1` = Load First) |
| `244` | `0x00F4` | 1 | u16 | enum | **Limit control / Work mode** (`0` = Selling First / Allow Export, `1` = Zero Export to Load / Essentials, `2` = Zero Export to CT / Home) |
| `245` | `0x00F5` | 1 | u16 | `1 W` | **Max sell power** |
| `247` | `0x00F7` | 1 | u16 | `0` / `1` | **Solar sell (export)** (`0` = Don't sell, `1` = Sell solar) |
| `248` | `0x00F8` | 1 | u16 | bitmask | **Time of Use (TOU) enable**: bit 0 = TOU master enable, bits 1–7 = Mon–Sun enable (`0xFF` = enabled all week), bit 8 = Spain mode 3 |
| `250`–`255` | `0x00FA`–`0x00FF` | 6 | u16 | `HHMM` | **TOU Slot 1–6 start time** (e.g. `2359` = `23:59`) |
| `256`–`261` | `0x0100`–`0x0105` | 6 | u16 | `1 W` | **TOU Slot 1–6 max battery power** |
| `262`–`267` | `0x0106`–`0x010B` | 6 | u16 | `0.01 V` | **TOU Slot 1–6 target voltage** |
| `268`–`273` | `0x010C`–`0x0111` | 6 | u16 | `1 %` | **TOU Slot 1–6 target SOC** |
| `274`–`279` | `0x0112`–`0x0117` | 6 | u16 | bitmask | **TOU Slot 1–6 flags**: bit 0 = Grid charge, bit 1 = Gen charge, bit 2 = GM mode, bit 3 = BU mode, bit 4 = CH mode, bit 6 (`0x40`) = Slot sell |
| `280` | `0x0118` | 1 | u16 | nibbles | Special functions / peak shaving: bits 0–3 Micro-inv export cutoff, bits 4–7 Gen peak-shaving, bits 8–11 Grid peak-shaving, bit 12 On-Grid always on |
| `292` / `293` | `0x0124` / `0x0125` | 1+1 | u16 | `1 W` | Gen peak-shaving power (`292`) / Grid peak-shaving power (`293`) |
| `325` | `0x0145` | 1 | u16 | enum | **Lithium BMS protocol selection** (`0` = Pylon CAN / Deye, `12` = Pylon RS485, etc.) |

---

## Three-Phase Register Map (`SG04LP3` / `SG05LP3` / `SG01HP3`)

Three-phase low-voltage (`SG04LP3` / `SG05LP3`, 48 V) and high-voltage (`SG01HP3` / `SG04HP3`, 160–700 V) inverters share the same register addresses (*MODBUS RTU V105.4* / *V104*), with **different scaling on HV models** for battery voltage (`0.1 V` vs `0.01 V`), battery power (`10 W` vs `1 W`), PV power (`10 W` vs `1 W`), and power limits (`10 W` vs `1 W`). Both columns are shown below where they differ.

- **Address zones in the three-phase specification:**
  - `0`–`59`: Read-only identification (`FC 0x03`)
  - `60`–`499`: Read/write configuration and BMS pass-through (`FC 0x03` to read, `FC 0x10` to write)
  - `500`–`2000`: Read-only real-time telemetry (`FC 0x03`)
  - `10000+` (and `15000+` on HV BMS2): Extended Deye battery pack registers (`FC 0x03`)

### Identification and status (three-phase)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `0` | `0x0000` | 1 | u16 | enum | **Device type**: `0x0500` (or `0x0005`) 3P LV Hybrid, `0x0600` / `0x0601` (or `0x0006`) 3P HV Hybrid |
| `1` | `0x0001` | 1 | u16 | `1`–`247` | **Modbus slave address** |
| `2` | `0x0002` | 1 | u16 | `Hi.Lo` | **Communication protocol version** |
| `3`–`7` | `0x0003`–`0x0007` | 5 | ASCII | 10 chars | **Serial number** (10 ASCII chars) |
| `20`–`21` | `0x0014`–`0x0015` | 2 | u32 | `0.1 W` | **Rated power** (`20` = low word, `21` = high word) |
| `22` | `0x0016` | 1 | u16 | `Hi` / `Lo` | MPPT count (`Hi` byte, `1`–`8`) and phase count (`Lo` byte, `1`–`3`, e.g. `0x0203` = 2 MPPT, 3-phase) |
| `62`–`64` | `0x003E`–`0x0040` | 3 | u16×3 | `YY/MM`, `DD/hh`, `mm/ss` | **System date & time** (R/W) |
| `500` | `0x01F4` | 1 | u16 | enum | **Overall running state**: `0` Standby, `1` Self-check, `2` Normal, `3` Alarm, `4` Fault, `5` Activating |
| `540` | `0x021C` | 1 | u16 | `0.1 °C` (offset +1000) | **DC transformer temperature** |
| `541` | `0x021D` | 1 | u16 | `0.1 °C` (offset +1000) | **Heat sink / radiator temperature** |
| `552` | `0x0228` | 1 | u16 | bitmask | **AC relay status**: bit 0 INV relay, bit 2 **Grid relay (Grid connected)**, bit 3 Gen relay, bit 4 Grid power to relay, bit 5/7 Dry contact 1, bit 6/8 Dry contact 2 |

### Battery and BMS telemetry (three-phase)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale (LV / HV) | Field |
|---|---|---|---|---|---|
| `586` | `0x024A` | 1 | u16 | `0.1 °C` (offset +1000) | **Battery 1 temperature** |
| `587` | `0x024B` | 1 | u16 | **LV:** `0.01 V`<br>**HV:** `0.1 V` | **Battery 1 voltage** |
| `588` | `0x024C` | 1 | u16 | `1 %` | **Battery 1 SOC** (`0`–`100 %`) |
| `589` | `0x024D` | 1 | u16 | `1 %` | **Battery 2 SOC** (`0`–`100 %`) |
| `590` | `0x024E` | 1 | s16 | **LV:** `1 W`<br>**HV:** `10 W` | **Battery 1 power** (`−` = charging, `+` = discharging) |
| `591` | `0x024F` | 1 | s16 | `0.01 A` | **Battery 1 current** (`−` = charging, `+` = discharging) |
| `592` | `0x0250` | 1 | u16 | `1 Ah` | Corrected battery capacity (`Corrected_AH`) |
| `593` | `0x0251` | 1 | u16 | **LV:** `0.01 V`<br>**HV:** `0.1 V` | **Battery 2 voltage** |
| `594` | `0x0252` | 1 | s16 | `0.01 A` | **Battery 2 current** (`−` = charging, `+` = discharging) |
| `595` | `0x0253` | 1 | s16 | **LV:** `1 W`<br>**HV:** `10 W` | **Battery 2 power** (`−` = charging, `+` = discharging) |
| `596` | `0x0254` | 1 | u16 | `0.1 °C` (offset +1000) | **Battery 2 temperature** |
| `210` | `0x00D2` | 1 | u16 | **LV:** `0.01 V`<br>**HV:** `0.1 V` | **Battery 1 BMS charging voltage** |
| `211` | `0x00D3` | 1 | u16 | **LV:** `0.01 V`<br>**HV:** `0.1 V` | **Battery 1 BMS discharging voltage** |
| `212` / `213` | `0x00D4` / `0x00D5` | 1+1 | u16 | `1 A` | **Battery 1 BMS charge** (`212`) / **discharge** (`213`) **current limit** |
| `214` | `0x00D6` | 1 | u16 | `1 %` | **Battery 1 BMS SOC** |
| `215` | `0x00D7` | 1 | u16 | **LV:** `0.01 V`<br>**HV:** `0.1 V` | **Battery 1 BMS voltage** |
| `216` | `0x00D8` | 1 | s16 | **LV:** `0.01 A` (or `1 A`)<br>**HV:** `0.1 A` | **Battery 1 BMS current** |
| `217` | `0x00D9` | 1 | u16 | `0.1 °C` (offset +1000) | **Battery 1 BMS temperature** |
| `218` / `219` | `0x00DA` / `0x00DB` | 1+1 | u16 | `1 A` | Battery 1 BMS max charge (`218`) / discharge (`219`) current limit |
| `220` / `221` | `0x00DC` / `0x00DD` | 1+1 | u16 | bitmask | Battery 1 BMS alarm flag (`220`) / fault flag (`221`) |
| `224` / `226` | `0x00E0` / `0x00E2` | 1+1 | u16 | `1 %` / `1 Ah` | Battery 1 BMS SOH (`224`) / rated Ah (`226`) |
| `241`–`258` | `0x00F1`–`0x0102` | 18 | — | same as `210`–`227` | **Battery 2 BMS parameters** (HV dual-battery models) |

### PV, Grid, Inverter, Load, and AUX/Gen telemetry (three-phase)

In protocol revision V104+, 3-phase AC power registers were widened to 32-bit signed values by adding high-word registers at `656`–`671` and `687`–`709` while keeping the original registers as the low 16-bit word.

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale (LV / HV) | Field |
|---|---|---|---|---|---|
| `598` / `599` / `600` | `0x0256`–`0x0258` | 1×3 | u16 | `0.1 V` | **Grid phase voltage L1 / L2 / L3** |
| `604`–`606` (low)<br>`700`–`702` (high) | `0x025C`–`0x025E`<br>`0x02BC`–`0x02BE` | 1 or 2 | s16 / s32 | `1 W` | **Grid inner side power L1 / L2 / L3** |
| `607` (low), `703` (high) | `0x025F`, `0x02BF` | 1 or 2 | s16 / s32 | `1 W` | **Grid inner side total active power** |
| `609` | `0x0261` | 1 | u16 | `0.01 Hz` | **Grid frequency** |
| `610` / `611` / `612` | `0x0262`–`0x0264` | 1×3 | s16 | `0.01 A` | **Grid inner current L1 / L2 / L3** |
| `613` / `614` / `615` | `0x0265`–`0x0267` | 1×3 | s16 | `0.01 A` | **Grid external CT current L1 / L2 / L3** |
| `616`–`618` (low)<br>`705`–`707` (high) | `0x0268`–`0x026A`<br>`0x02C1`–`0x02C3` | 1 or 2 | s16 / s32 | `1 W` | **Grid external CT power L1 / L2 / L3** |
| `619` (low), `708` (high) | `0x026B`, `0x02C4` | 1 or 2 | s16 / s32 | `1 W` | **Grid external CT total active power** |
| `622`–`624` (low)<br>`687`–`689` (high) | `0x026E`–`0x0270`<br>`0x02AF`–`0x02B1` | 1 or 2 | s16 / s32 | `1 W` | **Grid side power L1 / L2 / L3** (follows selected internal/external source) |
| `625` (low), `690` (high) | `0x0271`, `0x02B2` | 1 or 2 | s16 / s32 | `1 W` | **Grid side total active power** (`+` = import, `−` = export) |
| `627` / `628` / `629` | `0x0273`–`0x0275` | 1×3 | u16 | `0.1 V` | **Inverter output phase voltage L1 / L2 / L3** |
| `630` / `631` / `632` | `0x0276`–`0x0278` | 1×3 | s16 | `0.01 A` | **Inverter output phase current L1 / L2 / L3** |
| `633`–`635` (low)<br>`691`–`693` (high) | `0x0279`–`0x027B`<br>`0x02B3`–`0x02B5` | 1 or 2 | s16 / s32 | `1 W` | **Inverter output power L1 / L2 / L3** |
| `636` (low), `694` (high) | `0x027C`, `0x02B6` | 1 or 2 | s16 / s32 | `1 W` | **Inverter output total active power** |
| `638` | `0x027E` | 1 | u16 | `0.01 Hz` | **Inverter frequency** |
| `640`–`642` (low)<br>`696`–`698` (high) | `0x0280`–`0x0282`<br>`0x02B8`–`0x02BA` | 1 or 2 | u16 / u32 | `1 W` | **UPS / backup load power L1 / L2 / L3** |
| `643` (low), `699` (high) | `0x0283`, `0x02BB` | 1 or 2 | u16 / u32 | `1 W` | **UPS / backup load total power** |
| `644` / `645` / `646` | `0x0284`–`0x0286` | 1×3 | u16 | `0.1 V` | **Load phase voltage L1 / L2 / L3** |
| `650`–`652` (low)<br>`656`–`658` (high) | `0x028A`–`0x028C`<br>`0x0290`–`0x0292` | 1 or 2 | s16 / s32 | `1 W` | **Load phase power L1 / L2 / L3** |
| `653` (low), `659` (high) | `0x028D`, `0x0293` | 1 or 2 | s16 / s32 | `1 W` | **Load total active power** |
| `655` | `0x028F` | 1 | u16 | `0.01 Hz` | **Load frequency** |
| `661` / `662` / `663` | `0x0295`–`0x0297` | 1×3 | u16 | `0.1 V` | **Generator / AUX phase voltage L1 / L2 / L3** |
| `664`–`666` (low)<br>`668`–`670` (high) | `0x0298`–`0x029A`<br>`0x029C`–`0x029E` | 1 or 2 | s16 / s32 | `1 W` | **Generator / AUX phase power L1 / L2 / L3** |
| `667` (low), `671` (high) | `0x029B`, `0x029F` | 1 or 2 | s16 / s32 | `1 W` | **Generator / AUX total power** |
| `672`–`675` | `0x02A0`–`0x02A3` | 1×4 | u16 | **LV:** `1 W`<br>**HV:** `10 W` | **PV1 / PV2 / PV3 / PV4 input power** |
| `676` / `677` | `0x02A4` / `0x02A5` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV1 voltage** (`676`) / **PV1 current** (`677`) |
| `678` / `679` | `0x02A6` / `0x02A7` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV2 voltage** (`678`) / **PV2 current** (`679`) |
| `680` / `681` | `0x02A8` / `0x02A9` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV3 voltage** (`680`) / **PV3 current** (`681`) |
| `682` / `683` | `0x02AA` / `0x02AB` | 1+1 | u16 | `0.1 V` / `0.1 A` | **PV4 voltage** (`682`) / **PV4 current** (`683`) |
| `719`–`722` / `723`–`726` / `727`–`730` | `0x02CF`–`0x02DA` | 12 | u16 | `0.1 V` / `0.1 A` / `10 W` | **PV5–PV8 voltage, current, and power** (large HV models) |

### Energy counters (three-phase)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `501` / `502` | `0x01F5` / `0x01F6` | 1 | s16 | `0.1 kWh` / `0.1 kVarh` | Day active / reactive energy (see [Errata](#errata-and-disagreements)) |
| `504`–`505` | `0x01F8`–`0x01F9` | 2 | u32 | `0.1 kWh` | Total active energy (see [Errata](#errata-and-disagreements)) |
| `514` | `0x0202` | 1 | u16 | `0.1 kWh` | **Day battery charge energy** |
| `515` | `0x0203` | 1 | u16 | `0.1 kWh` | **Day battery discharge energy** |
| `516`–`517` | `0x0204`–`0x0205` | 2 | u32 | `0.1 kWh` | **Total battery charge energy** |
| `518`–`519` | `0x0206`–`0x0207` | 2 | u32 | `0.1 kWh` | **Total battery discharge energy** |
| `520` | `0x0208` | 1 | u16 | `0.1 kWh` | **Day grid import (buy) energy** |
| `521` | `0x0209` | 1 | u16 | `0.1 kWh` | **Day grid export (sell) energy** |
| `522`–`523` | `0x020A`–`0x020B` | 2 | u32 | `0.1 kWh` | **Total grid import (buy) energy** |
| `524`–`525` | `0x020C`–`0x020D` | 2 | u32 | `0.1 kWh` | **Total grid export (sell) energy** |
| `526` | `0x020E` | 1 | u16 | `0.1 kWh` | **Day load energy** |
| `527`–`528` | `0x020F`–`0x0210` | 2 | u32 | `0.1 kWh` | **Total load energy** |
| `529` | `0x0211` | 1 | u16 | `0.1 kWh` | **Day PV energy** |
| `534`–`535` | `0x0216`–`0x0217` | 2 | u32 | `0.1 kWh` | **Total PV energy** |
| `536` | `0x0218` | 1 | u16 | `0.1 kWh` | **Day generator energy** |
| `537`–`538` | `0x0219`–`0x021A` | 2 | u32 | `0.1 kWh` | **Total generator energy** |

### Key settings and Time-of-Use (three-phase, R/W)

| Addr (Dec) | Addr (Hex) | Size | Type | Unit / scale (LV / HV) | Field |
|---|---|---|---|---|---|
| `98` | `0x0062` | 1 | u16 | enum | Battery charging mode (`0` = Lead-acid 4-stage, `1` = Lithium BMS) |
| `99` / `100` / `101` | `0x0063`–`0x0065` | 1×3 | u16 | `0.01 V` | Battery Equalization (`99`), Absorption (`100`), Float (`101`) voltage |
| `102` | `0x0066` | 1 | u16 | `1 Ah` | Battery nominal capacity (`0`–`2000 Ah`) |
| `104` | `0x0068` | 1 | s16 | **LV:** `1 W`<br>**HV:** `10 W` | System zero-export power (`−500` to `500 W`) |
| `108` / `109` | `0x006C` / `0x006D` | 1+1 | u16 | `1 A` | **Battery max charge current** (`108`) / **max discharge current** (`109`) (`0`–`240 A`) |
| `111` | `0x006F` | 1 | u16 | enum | **Battery operation mode** (`0` = Use Battery Voltage, `1` = Lithium / Use BMS, `2` = No Battery) |
| `115` / `116` / `117` | `0x0073`–`0x0075` | 1×3 | u16 | `1 %` | **Battery Shutdown** (`115`), **Restart** (`116`), **Low Batt** (`117`) SOC limits |
| `118` / `119` / `120` | `0x0076`–`0x0078` | 1×3 | u16 | `0.01 V` | **Battery Shutdown** (`118`), **Restart** (`119`), **Low Batt** (`120`) voltage limits |
| `125` / `128` | `0x007D` / `0x0080` | 1+1 | u16 | `1 A` | Generator charge battery current (`125`) / **Grid charge battery current** (`128`) |
| `129` / `130` | `0x0081` / `0x0082` | 1+1 | u16 | `0` / `1` | Generator charge enabled (`129`) / **Grid charge enabled** (`130`) |
| `133` | `0x0085` | 1 | u16 | enum | AUX / Generator port usage (`0` = Generator, `1` = SmartLoad output, `2` = Micro-inverter input) |
| `141` | `0x008D` | 1 | u16 | enum / bits | **Energy management priority** (`0` / `0b10` = Battery First, `1` / `0b11` = Load First) |
| `142` | `0x008E` | 1 | u16 | enum | **Limit control / Work mode** (`0` = Allow Export / Selling First, `1` = Built-in / Zero Export to Load, `2` = External / Zero Export to CT) |
| `143` | `0x008F` | 1 | u16 | **LV:** `1 W`<br>**HV:** `10 W` | **Max sell (export limit) power** |
| `145` | `0x0091` | 1 | u16 | `0` / `1` | **Solar sell (export)** (`0` = Don't sell, `1` = Sell solar) |
| `146` | `0x0092` | 1 | u16 | bitmask | **Time of Use (TOU) enable**: bit 0 = TOU enable, bits 1–7 = Mon–Sun enable, bit 8 = Spain mode |
| `148`–`153` | `0x0094`–`0x0099` | 6 | u16 | `HHMM` | **TOU Slot 1–6 start time** |
| `154`–`159` | `0x009A`–`0x009F` | 6 | u16 | **LV:** `1 W`<br>**HV:** `10 W` | **TOU Slot 1–6 max battery power** |
| `160`–`165` | `0x00A0`–`0x00A5` | 6 | u16 | `0.01 V` | **TOU Slot 1–6 target voltage** |
| `166`–`171` | `0x00A6`–`0x00AB` | 6 | u16 | `1 %` | **TOU Slot 1–6 target SOC** |
| `172`–`177` | `0x00AC`–`0x00B1` | 6 | u16 | bitmask | **TOU Slot 1–6 flags**: bit 0 = Grid charge, bit 1 = Gen charge, bits 2–4 = GM/BU/CH mode, bit 5 (`0x20`) = Slot sell |
| `178` | `0x00B2` | 1 | u16 | 2-bit fields | **Special function bits** (`0b10` = Disable, `0b11` = Enable): bits 0–1 Micro-inv export cutoff, bits 2–3 Gen peak-shaving, bits 4–5 Grid peak-shaving, bits 6–7 On-Grid always on, bits 8–9 External relay, bits 10–11 Loss of Li-bat fault, bits 12–13 DRM, bits 14–15 US ground fault |
| `190` / `191` | `0x00BE` / `0x00BF` | 1+1 | u16 | **LV:** `1 W`<br>**HV:** `10 W` | Gen peak-shaving power (`190`) / **Grid peak-shaving power** (`191`) |
| `223` | `0x00DF` | 1 | u16 | enum | **Lithium BMS protocol** (`0` = Pylon/Solax CAN, `1` = Tianbanda RS485 / BYD CAN, `5` = Pylon RS485, etc.) |
| `229` | `0x00E5` | 1 | u16 | enum | **Battery 1 manufacturer** (LV: `0` HereYin, `1` PYLON, `2` SOLAX, `3` DYNESS_L, `12` BYD_LV, `14` DEYE; HV: `1` PYLON_HV, `2` DynessHV, `5` Deye_HV, `6` BYD_HV) |
| `340` | `0x0154` | 1 | u16 | **LV:** `1 W`<br>**HV:** `10 W` | **Max solar power** |

---

## Alarms and status bits

### Warning words (`101`–`102` on 1P; `553`–`554` on 3P)

| Word | Bit | Code | Meaning |
|---|---|---|---|
| Word 1 (`101` / `553`) | 1 | `W02` | `FAN_WARN` — Fan warning / fault |
| Word 1 (`101` / `553`) | 2 | `W03` | `Grid phase wrong` — Grid phase rotation error |
| Word 1 (`101` / `553`) | 3 | `W04` | `meter_Comm_Fail` — External meter communication failure |
| Word 2 (`102` / `554`) | 14 | — | Lithium battery loss warning (3P) |
| Word 2 (`102` / `554`) | 15 | — | Parallel communication quality warning (3P) |

### Fault words (`103`–`106` on 1P; `555`–`558` on 3P)

Four consecutive 16-bit registers form a 64-bit bitmap where bit $b \in [0..15]$ of word $w \in [0..3]$ corresponds to fault code $\text{F}(16w + b + 1)$:

| Fault code | Word : Bit | Single-phase / 3P LV meaning (*V118* / *V104*) | 3P HV overrides (*kellerza/sunsynk*) |
|---|---|---|---|
| `F01`–`F06` | W1 : 0–5 | — | `F01` DC inversed, `F02` DC insulation permanent, `F03` DC leakage, `F04` GFDI, `F05`/`F06` Memory R/W error |
| `F07` | W1 : 6 | `DC/DC_Softstart_Fault` — DC/DC soft-start fault | `DC START Failure` |
| `F10` | W1 : 9 | `AuxPowerBoard_Failure` — Auxiliary power supply failure | Same |
| `F13` | W1 : 12 | `Working mode change` — Inverter work mode changed | Same |
| `F18` | W2 : 1 | `AC over current fault of hardware` | `AC over current TZ` |
| `F20` | W2 : 3 | `DC over current fault of hardware` | `DC over current` |
| `F22` | W2 : 5 | `Tz_EmergStop_Fault` — Emergency stop / remote lock | `Remote Emergency stop` |
| `F23` | W2 : 6 | `AC leakage current is transient over current` | Same |
| `F24` | W2 : 7 | `DC insulation impedance failure` (PV isolation low) | Same |
| `F26` | W2 : 9 | `DC busbar is unbalanced` | Same |
| `F29` | W2 : 12 | `Parallel_CANBus_Fault` — Parallel comms cable fault | Same |
| `F34` | W3 : 1 | — | `AC Overload (backup)` |
| `F35` | W3 : 2 | `No AC grid` — No utility grid | Same |
| `F40` | W3 : 7 | `Batt_Over_Current` | `DC over current` |
| `F41` | W3 : 8 | `Parallel_system_Stop` | Same |
| `F42` | W3 : 9 | `AC line low voltage` | Same |
| `F46` / `F49` | W3 : 13 / W4 : 0 | `Backup_Battery_Fault` | `Battery 1 fault` (`F46`) / `Battery 2 fault` (`F49`) |
| `F47` / `F48` | W3 : 14 / 15 | `AC over frequency` (`F47`) / `AC lower frequency` (`F48`) | Same |
| `F56` | W4 : 7 | `DC busbar voltage is too low` (Battery voltage low) | `battery 1 voltage low` (`F54`/`F55` bat 1/2 high, `F57` bat 2 low) |
| `F58` | W4 : 9 | `BMS communication fault` | `bms communication lost` |
| `F63` | W4 : 14 | `ARC fault` | Same |
| `F64` | W4 : 15 | `Heat sink high temperature failure` | Same |

---

## Control commands (dangerous)

> [!CAUTION]
> **Flash wear and safety warning:**
> - **EEPROM wear:** Writing to configuration registers (`20`–`499` on 1P, `60`–`499` on 3P) commits to non-volatile EEPROM, which has limited write endurance. Never write registers in a fast loop unless using the volatile **Remote Control** block (`1100`–`1121` on 3P V105.4).
> - **Shutdown / Lockout:** Several registers immediately open the AC/load relays, lock the inverter into fault `F22`, or erase calibration tables. Keep integrations read-only unless you explicitly intend to control the inverter.

| Register (1P / 3P) | Name | Danger |
|---|---|---|
| `20` (1P) / `60` (3P) | **Remote Lock** | Writing `0x0002` locks and shuts down the inverter (`0x0000` unlocks). |
| `43` (1P) / `80` (3P) | **Switch On/Off Enable** | `0` = Power Off (opens load relays, cuts backup output), `1` = Power On. |
| `44` (1P) / `81` (3P) | **Factory Reset Enable** | `1` resets parameters to factory defaults; `3` locks the inverter (`Lockedinv`). |
| `54` (1P) / `91`, `92` (3P) | **EEPROM Initialize** | `1` / `2` wipes and re-initializes the MCU or communication board EEPROM; `3` on reg `92` locks Sunsynk inverters. |
| `55` (1P) / `93`, `94` (3P) | **Factory Test Commands** | Toggles relays, fans, LEDs, and lithium interface test modes. |
| `179` (3P) / `416` (1P) | **Force Off-Grid** | `1` forces the inverter to disconnect from the utility grid relay. |
| `1100`–`1121` (3P V105.4) | **Remote Direct Power/Battery Control** | Overrides normal MPPT/BMS logic with direct constant-voltage (`1107`), constant-current (`1108`), constant-power (`1109`), or AC phase power (`1111`–`1114`) setpoints. Always set the watchdog timer in `1101` (`10`–`18000 s`) first so the inverter exits remote mode if communication drops. |

---

## Captured frames

### Vendor documentation examples (*Deye V118*, pages 2–4)

The vendor specification provides two PDU examples (shown below with Slave ID `0x01` and verified CRC-16/MODBUS):

```
# Read 3 holding registers starting at 107 (0x006B):
01 03 00 6B 00 03 74 17  ->  01 03 06 02 2B 00 00 00 64 05 7A
# Exception reply (slave device failure 0x04):
01 03 00 6B 00 03 74 17  ->  01 83 04 40 F3

# Write 2 holding registers starting at 1 (0x0001) with values 0x000A, 0x0102:
01 10 00 01 00 02 04 00 0A 01 02 92 30  ->  01 10 00 01 00 02 10 08
```

### Hardware-verified register dumps (`SUN-6K-SG05LP1-EU-AM2-P`)

From [tema-mazy/deye-modbus-test](https://github.com/tema-mazy/deye-modbus-test) (`README.md`, verified on a live `SUN-6K-SG05LP1-EU-AM2-P`, firmware `MAIN 4388-1516`, protocol `0x0201`, with a 51.2 V Deye Pro-C battery on `Lithium Mode 00`):

| Operating state | Reg `167` (Grid internal port, W) | Reg `175` (Inverter output, W) | Reg `179` (UPS / Load port, W) | Notes |
|---|---|---|---|---|
| Idle | `200` (`0x00C8`) | `54` (`0x0036`) | `254` (`0x00FE`) | Balance: `167 + 175 = 179` |
| Dishwasher load | `2273` (`0x08E1`) | `23` (`0x0017`) | `2296` (`0x08F8`) | `2273 + 23 = 2296 W` |
| Grid charging battery | `2774` (`0x0AD6`) | `−2713` (`0xF567`) | `61` (`0x003D`) | Battery charging at ~47 A; `2774 + (−2713) = 61 W` |
| Heater load | `846` (`0x034E`) | `29` (`0x001D`) | `875` (`0x036B`) | `846 + 29 = 875 W` |

---

## Errata and disagreements

1. **Single-phase Total Grid Import energy (`78` and `80`) is non-contiguous:**
   In *Deye V118* (pages 11–12), Total Grid Buy (`Total_GridBuy_PowerWh`) has its low word at register **`78`** (`0x004E`) and its high word at register **`80`** (`0x0050`), skipping register **`79`** (`0x004F`, Grid frequency). Both `kellerza/sunsynk` (`Sensor((78, 80), "Total grid import")`) and `StephanJoubert/home_assistant_solarman` (`registers: [0x004E, 0x0050]`) confirm this split.
2. **Single-phase Load Power (`178`) vs. UPS Load Power (`179`):**
   *Deye V118* (page 20) labels register `178` as `load side Total power` (W) and `179` as `Load current L1` (`0.01 A`). However, hardware testing on the `SUN-6K-SG05LP1-EU-AM2-P` ([tema-mazy/deye-modbus-test](https://github.com/tema-mazy/deye-modbus-test)) shows that `179` (mirrored at `185`) actually reports the **UPS / backup load port power in W** (`167 + 175 = 179`), whereas `178` reports total home consumption (`169 + 175`).
3. **Single-phase SD card status (`92` vs. `94`) and Generator Total Energy (`92`, `95`):**
   In *Deye V118*, changelog V117 (page 1) repurposed registers `92` and `95` as Generator Total Energy low/high words, while page 13 lists `94` as `SD Card Status` and `kellerza/sunsynk` reads `92` as `SD status` and `95` as `Environment temperature`. Meaning depends on firmware age.
4. **Three-phase Day / Total Active & Reactive Energy (`501`–`507`):**
   In *MODBUS RTU V105.4* (page 29), `501` is Day Active Energy, `502` is Day Reactive Energy, `504`–`505` is Total Active Energy, and `506`–`507` is Total Reactive Energy. In `kellerza/sunsynk` (`three_phase_common.py`), Day Active Energy is mapped to `502` and Total Active Energy to `(506, 507)`. Check against the inverter LCD on your firmware.
5. **Three-phase AC Relay Status (`552`) dry contact bits:**
   *MODBUS RTU V105.4* (page 32) lists Dry Contact 1 & 2 at **bits 7 and 8**, whereas `kellerza/sunsynk` (`three_phase_common.py`) masks **bits 5 and 6** (`1 << 5`, `1 << 6`). Bit 0 (INV relay) and Bit 2 (Grid relay / Grid connected) agree across all sources.
6. **Three-phase Battery Wake-Up (`112`) polarity:**
   *MODBUS RTU V105.4* (page 9) states `0 = enabled, 1 = Disable` for bit 0 of register `112`. Community testing on LV 3-phase inverters (`kellerza/sunsynk` issue #381) found `0 = Disabled, 1 = Enabled` on LV firmware, while HV firmware follows the spec (`0 = Enabled, 1 = Disabled`).
7. **Three-phase Time-of-Use Sell bit (`172`–`177`) vs. Single-phase (`274`–`279`):**
   On single-phase inverters, the per-slot `Sell` bit in registers `274`–`279` is **bit 6 (`0x40`)**. On three-phase inverters, registers `172`–`177` place the per-slot `Sell` bit at **bit 5 (`0x20`)** (confirmed by both *MODBUS RTU V105.4* page 13 and `kellerza/sunsynk`).
8. **16 kW Single-Phase (`SUN-16K-SG01LP1`) Grid Power scaling:**
   On the 16 kW single-phase model, registers `169` (Grid power) and `172` (Grid CT power) use a scale factor of **`10 W`** (`s16 × 10 W`) instead of `1 W` (`kellerza/sunsynk` `single_phase_16kw.py`).

---

## Sources

- **Vendor protocol specifications (stored in [`PDFs/`](PDFs/)):**
  - [`Deye.Modbus.protocol.V118.pdf`](PDFs/Deye.Modbus.protocol.V118.pdf) — *Modbus RTU Protocol V118* (2021-04-08, 43 pages, single-phase hybrid, string, and microinverters; public copy from [Hypfer/deye-microinverter-cloud-free-assets](https://github.com/Hypfer/deye-microinverter-cloud-free-assets/blob/master/Deye_Modbus.-.-.V118.pdf) and [davidrapan/ha-solarman wiki](https://github.com/davidrapan/ha-solarman/wiki/Documentation)).
  - [`MODBUS.RTU.V105.4-20240814.pdf`](PDFs/MODBUS.RTU.V105.4-20240814.pdf) — *MODBUS RTU 三相储能通信规约 V105* (updated through 2025-01-13, 48 pages, three-phase LV & HV hybrid inverters; public copy from [davidrapan/ha-solarman wiki](https://github.com/davidrapan/ha-solarman/wiki/Documentation)).
  - [`MODBUSRTU.V104.3.1.1111_SG01-HP3-AM2.pdf`](PDFs/MODBUSRTU.V104.3.1.1111_SG01-HP3-AM2.pdf) — *MODBUS RTU 三相储能通信规约 V104* (2021-12-22, 52 pages, three-phase HV `SG01-HP3-AM2` including fault code table and BMS1/BMS2 extended maps; public copy from [davidrapan/ha-solarman wiki](https://github.com/davidrapan/ha-solarman/wiki/Documentation)).
- **Open-source implementations & hardware verifications:**
  - [kellerza/sunsynk](https://github.com/kellerza/sunsynk) — `src/sunsynk/definitions/single_phase.py`, `single_phase_16kw.py`, `three_phase_common.py`, `three_phase_lv.py`, `three_phase_hv.py`, `sensors.py`, `rwsensors.py`, `helpers.py`.
  - [StephanJoubert/home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman) — `custom_components/solarman/inverter_definitions/deye_hybrid.yaml` and `deye_sg04lp3.yaml`.
  - [tema-mazy/deye-modbus-test](https://github.com/tema-mazy/deye-modbus-test) — `README.md` and `deye_read.py` (hardware verification, RJ45 `Modbus` socket wiring, and power-balance dumps on a live `SUN-6K-SG05LP1-EU-AM2-P`).
  - [Developer089/deye-modbus-ha](https://github.com/Developer089/deye-modbus-ha) — `custom_components/deye_modbus/maps/sun_3ph_hybrid.yaml` (three-phase `SG04LP3` / `SG05LP3` register map).

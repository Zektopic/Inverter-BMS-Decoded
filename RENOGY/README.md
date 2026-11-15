# Renogy — Smart Lithium Battery BMS, Rover Charge Controllers & BT-1/BT-2 BLE

Renogy devices use **Modbus RTU** over RS485 or RS232 at **9600 8N1**, and expose the exact same Modbus RTU frames (including slave address and CRC-16) transparently over Bluetooth Low Energy via external **BT-1** (RS232) / **BT-2** (RS485) dongles or built-in BLE modules.

| Family | Typical models | Interfaces | Section |
|---|---|---|---|
| Smart Lithium Battery BMS | `RBT100LFP12S`, `RBT100LFP12SH`, `RBT50LFP48S`, `RBT100LFP12-BT`, `RBT12100LFP-BT` (Pro) | RS485 (RJ45 ×2), built-in BLE or BT-2 | [Smart Lithium Battery BMS](#smart-lithium-battery-bms-modbus-rtu) |
| Solar Charge Controllers | Rover, Rover Elite, Wanderer, Adventurer (`RNG-CTRL-*`), DC-DC (`DCC50S` / `RBC50D1S`) | RS232 (RJ12, BT-1) or RS485 (RJ45, BT-2) | [Rover / Wanderer Charge Controller](#rover--wanderer-charge-controller-modbus-rtu) |
| Bluetooth Modules | `BT-1` (RS232), `BT-2` (RS485), built-in `BT-TH-*` / `RNGRBP*` / `RNGC*` | BLE GATT (`0xFFD0` / `0xFFF0`) | [BT-1 / BT-2 BLE Wrapping](#bt-1--bt-2-bluetooth-low-energy-ble-wrapping) |

---

## Smart Lithium Battery BMS (Modbus RTU)

> Status: 📄 Vendor doc — transcribed from *Suzhou RongSi New Energy Technology Co., Ltd. — Lithium Iron Battery BMS Modbus Protocol V1.7* (2020-03-10, [`PDFs/Lithium.Iron.Battery.BMS.Modbus.Protocol.V1.7.zh-CN.en.1.pdf`](PDFs/Lithium.Iron.Battery.BMS.Modbus.Protocol.V1.7.zh-CN.en.1.pdf)), cross-checked against [`cyrils/renogy-bt`](https://github.com/cyrils/renogy-bt), [`dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery), [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble), and [`rosswarren/renogymodbus`](https://github.com/rosswarren/renogymodbus).

### Summary

| | |
|---|---|
| Products | Renogy 12 V / 48 V Smart Lithium Iron Phosphate batteries (OEM: Suzhou RongSi New Energy): `RBT100LFP12S`, `RBT100LFP12SH-G1` (heated), `RBT50LFP48S`, `RBT100LFP12-BT`, `RBT200LFP12-BT`, and Pro series (`RBT12100LFP-BT`, `RBT12200LFP-BT`) |
| Physical layer | Dual RJ45 (8P8C) ports (`UP` RS485 port for host/BT-2; `LINK` port for inter-pack CAN/cascade), plus built-in BLE on `-BT` models |
| Settings | RS485: **9600 8N1** (initial baud rate). Inter-pack CAN port: 500 kbit/s |
| Addressing | Valid slave IDs `0x03`–`0xF7` (3–247), stored in register `5223` (`0x1467`). Standalone factory default is `0xF7` (247) on RS485 packs (`RBT100LFP12SH-G1`) or `0x30` (48) on built-in BLE packs (`RBT100LFP12-BT`). In Renogy Communication Hub mode, batteries use `0x30`, `0x31`, `0x32` (48–50); daisy-chained packs often assign `0x21`, `0x22`, `0x23` (33–35). Broadcast/universal single-device address: `0xFF` (255, changed from `0x00` in protocol V1.3) |
| Roles | Host / BT-2 module is Modbus master and polls the BMS (typically every 1–60 s) |
| Byte order | Register addresses and multi-byte data fields are **big-endian** (`BE`); the 16-bit Modbus CRC is transmitted **little-endian** (`LE`, low byte first) |

#### RJ45 Pinouts (Vendor PDF, Section 2)

| Pin | RS485 port (`UP` / communication) | Inter-pack cascade port (`LINK`, CAN 500 kbit/s) |
|---|---|---|
| 1 | `NC` (do not connect) | `NC` (do not connect) |
| 2 | **RS485-A** | `NC` |
| 3 | **RS485-B** | `NC` |
| 4 | **GND** (power / communication ground) | **GND** (power / communication ground) |
| 5 | `12V-` | CAN cascade auxiliary signal |
| 6 | `O/F` (remote switch) | **CAN-H** |
| 7 | `O/F (12V+)` (200 mA max load) | CAN cascade auxiliary signal |
| 8 | `NC` | **CAN-L** |

---

### Framing

Standard **Modbus RTU** framing with `> 10 ms` idle gap before and after each frame.

| Byte offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 1 | `ADR` | Slave address (`0x01`–`0xF7`, or `0xFF` single-device broadcast; `0x00` broadcast does not reply) |
| 1 | 1 | `FC` | Function code (`0x03` Read, `0x06` Write Single, `0x10` Write Multiple, `0x78` Factory Reset, `0x79` Clear History). Exception reply sets bit 7 (`FC \| 0x80`) followed by a 1-byte exception code (`0x01`–`0x05`) |
| 2… | N | `DATA` | Request (FC `0x03`): `StartAddr` (u16 BE) + `WordCount` (u16 BE, `1`–`125`).<br>Response (FC `0x03`): `ByteCount` (u8, `2 × WordCount`) + register words (BE) |
| N+2 | 2 | `CRC` | **CRC-16/MODBUS** over all preceding bytes (`poly = 0x8005`, reflected `0xA001`, `init = 0xFFFF`, no final XOR), sent **low byte first** |

#### Worked CRC-16 Example (Verified)

Read 7 registers starting at `5042` (`0x13B2`: current, voltage, remaining capacity, total capacity, cycles) from slave `0x30` (48):

- Payload bytes: `30 03 13 B2 00 07`
- CRC-16/MODBUS calculation (`init = 0xFFFF`, `poly = 0xA001` reflected):
  `crc = 0x8AA4` → transmitted low-byte first as `A4 8A`.

```
Request:  30 03 13 B2 00 07 A4 8A
Response: 30 03 0E FF AA 00 88 00 01 7B B0 00 01 86 A0 00 0F E0 1E
```

Decoding the response (`ByteCount = 0x0E` = 14 bytes = 7 words, CRC `E0 1E` = `0x1EE0` valid):
- Words 0 (`5042`): `0xFFAA` (`s16` = `-86`) × 0.01 A = **−0.86 A** (discharging)
- Word 1 (`5043`): `0x0088` (`u16` = `136`) × 0.1 V = **13.6 V**
- Words 2–3 (`5044`–`5045`): `0x00017BB0` (`u32` = `97200`) × 0.001 Ah = **97.200 Ah** remaining
- Words 4–5 (`5046`–`5047`): `0x000186A0` (`u32` = `100000`) × 0.001 Ah = **100.000 Ah** total capacity
- Word 6 (`5048`): `0x000F` (`u16` = `15`) = **15 cycles**

---

### Register Map (Function Code `0x03`)

#### Cell Voltages, Temperatures & Dynamic Telemetry (`5000`–`5052`)

| Register (dec) | Register (hex) | Size (words) | Type | R/W | Unit / scale | Field |
|---|---|---|---|---|---|---|
| `5000` | `0x1388` | 1 | u16 | R | 1 | Cell count `M` (max 16; `4` for 12 V LFP, `15`/`16` for 48 V) |
| `5001`–`5016` | `0x1389`–`0x1398` | 16 × 1 | u16 | R | **0.1 V** | Cell 1 to Cell 16 voltage (note coarse 100 mV resolution; see errata) |
| `5017` | `0x1399` | 1 | u16 | R | 1 | Cell temperature sensor count `N` (max 16) |
| `5018`–`5033` | `0x139A`–`0x13A9` | 16 × 1 | s16 | R | 0.1 °C | Cell temperature 1 to 16 |
| `5034` | `0x13AA` | 1 | u16 | R | — | Reserved |
| `5035` | `0x13AB` | 1 | s16 | R | 0.1 °C | BMS board temperature |
| `5036` | `0x13AC` | 1 | u16 | R | 1 | Environment temperature sensor count (max 2) |
| `5037`–`5038` | `0x13AD`–`0x13AE` | 2 × 1 | s16 | R | 0.1 °C | Environment temperature 1 and 2 |
| `5039` | `0x13AF` | 1 | u16 | R | 1 | Heater temperature sensor count (max 2) |
| `5040`–`5041` | `0x13B0`–`0x13B1` | 2 × 1 | s16 | R | 0.1 °C | Heater temperature 1 and 2 |
| `5042` | `0x13B2` | 1 | s16 | R | **0.01 A** *(0.1 A on Pro)* | Pack current (`+` charge, `−` discharge; **0.1 A** on `RNGRBP*`/`RNGC*` Pro series) |
| `5043` | `0x13B3` | 1 | u16 | R | 0.1 V | Pack (module) voltage |
| `5044`–`5045` | `0x13B4`–`0x13B5` | 2 | u32 | R | 0.001 Ah (mAh) | Remaining capacity (SOC = remaining / total × 100 %) |
| `5046`–`5047` | `0x13B6`–`0x13B7` | 2 | u32 | R | 0.001 Ah (mAh) | Total (nominal) capacity |
| `5048` | `0x13B8` | 1 | u16 | R | 1 | Cycle count (swapped byte order on `RNGC*` Pro batteries) |
| `5049` | `0x13B9` | 1 | u16 | R | 0.1 V | Charge voltage limit |
| `5050` | `0x13BA` | 1 | u16 | R | 0.1 V | Discharge voltage limit |
| `5051` | `0x13BB` | 1 | u16 | R | 0.01 A | Charge current limit |
| `5052` | `0x13BC` | 1 | s16 | R | 0.01 A | Discharge current limit (signed) |

#### Device Identification (`5110`–`5141`)

| Register (dec) | Register (hex) | Size (words) | Type | R/W | Field |
|---|---|---|---|---|---|
| `5110`–`5117` | `0x13F6`–`0x13FD` | 8 (16 B) | ASCII | R | Serial number (`SN Number`, null/space padded) |
| `5118` | `0x13FE` | 1 (2 B) | ASCII | R | Manufacture version |
| `5119`–`5120` | `0x13FF`–`0x1400` | 2 (4 B) | ASCII | R | Main line version |
| `5121` | `0x1401` | 1 (2 B) | ASCII | R | Communication protocol version |
| `5122`–`5129` | `0x1402`–`0x1409` | 8 (16 B) | ASCII | R | Battery model name (e.g. `"RBT100LFP12-BT  "` or `"RBT100LFP12S-G"`) |
| `5130`–`5131` | `0x140A`–`0x140B` | 2 (4 B) | ASCII | R | Software version (e.g. `"0100"` → `v1.00`) |
| `5132`–`5141` | `0x140C`–`0x1415` | 10 (20 B) | ASCII | R | Manufacturer name |

---

### Alarms and Status Bits (`5100`–`5109`)

#### Per-Sensor 2-Bit Alarm Fields (`5100`–`5105`)

Each sensor uses a 2-bit code: `00` = normal, `01` = below lower limit (protection), `10` = above higher limit (protection), `11` = other alarm.

| Register (dec) | Register (hex) | Size | Bit allocation (32-bit big-endian word `Bit31`..`Bit0`) |
|---|---|---|---|
| `5100`–`5101` | `0x13EC`–`0x13ED` | 2 words (u32) | **Cell voltage alarm info**: `Bits 31:30` = Cell 16 … `Bits 1:0` = Cell 1 (`2 × (n − 1)` for Cell `n`) |
| `5102`–`5103` | `0x13EE`–`0x13EF` | 2 words (u32) | **Cell temperature alarm info**: `Bits 31:30` = Temp 16 … `Bits 1:0` = Temp 1 |
| `5104`–`5105` | `0x13F0`–`0x13F1` | 2 words (u32) | **Other alarm info**: `Bits 31:30` BMS board temp, `29:28` Env temp 1, `27:26` Env temp 2, `25:24` Heater temp 1, `23:22` Heater temp 2 *(labelled Heater 1 in PDF)*, `21:20` Charge current, `19:18` Discharge current, `17:0` Reserved |

#### Protection, Warning, MOSFET & Charge Status Bitmaps (`5106`–`5109`)

| Register | Bit | Meaning (`1` = active / on, `0` = normal / off) |
|---|---|---|
| **`5106` (`0x13F2`)**<br>`Status1` (Protections & MOSFETs) | 15 | Module under-voltage protection (`UV`) |
| | 14 | Charge over-temperature protection |
| | 13 | Charge under-temperature protection |
| | 12 | Discharge over-temperature protection |
| | 11 | Discharge under-temperature protection |
| | 10 | Discharge over-current 1 protection (`DOC1`) |
| | 9 | Charge over-current 1 protection (`COC1`) |
| | 8 | Cell over-voltage protection (`COV`) |
| | 7 | Cell under-voltage protection (`CUV`) |
| | 6 | Module over-voltage protection (`MOV`) |
| | 5 | Discharge over-current 2 protection (`DOC2`) |
| | 4 | Charge over-current 2 protection (`COC2`) |
| | 3 | Using battery module power (`1` = using) |
| | 2 | **Discharge MOSFET** (`1` = ON, `0` = OFF) |
| | 1 | **Charge MOSFET** (`1` = ON, `0` = OFF) |
| | 0 | Short-circuit protection (`1` = triggered) |
| **`5107` (`0x13F3`)**<br>`Status2` (Warnings & Heater) | 15 | Effective charge current (`1` = active) |
| | 14 | Effective discharge current (`1` = active) |
| | 13 | **Heater on** (`1` = ON, `0` = OFF) |
| | 12 | Reserved |
| | 11 | Fully charged (`1` = full, `0` = normal) |
| | 10–9 | Reserved |
| | 8 | Buzzer (`1` = ON, `0` = OFF) |
| | 7 | Discharge high-temperature warning |
| | 6 | Discharge low-temperature warning |
| | 5 | Charge high-temperature warning |
| | 4 | Charge low-temperature warning |
| | 3 | Module high-voltage warning |
| | 2 | Module low-voltage warning |
| | 1 | Cell high-voltage warning |
| | 0 | Cell low-voltage warning |
| **`5108` (`0x13F4`)**<br>`Status3` | 15–0 | Cell 16 (bit 15) … Cell 1 (bit 0) voltage error (`1` = error, `0` = normal) |
| **`5109` (`0x13F5`)**<br>`Charge/discharge status` | 7 | Charge enable (`1` = enabled, `0` = request stop charge) |
| | 6 | Discharge enable (`1` = enabled, `0` = request stop discharge) |
| | 5 | Charge immediately 1 (Force charge I; `1` = request) |
| | 4 | Charge immediately 2 (Force charge II; `1` = request) |
| | 3 | Full-charge request (`1` = request) |
| | 2–0 | Reserved |

---

### Control and Configuration Registers (Dangerous)

> [!CAUTION]
> Writing to registers `5200`–`5229` or `61440`–`61442`, or sending function codes `0x78` / `0x79`, modifies LiFePO4 safety protection thresholds, shuts down the pack, or re-addresses the bus. Do not write to these registers during normal monitoring.

| Register (dec) | Register (hex) | R/W | Type / Unit | Description |
|---|---|---|---|---|
| `5200`–`5203` | `0x1450`–`0x1453` | R/W | u16, 0.1 V | Cell over-voltage, high-voltage, low-voltage, and under-voltage limits |
| `5204`–`5207` | `0x1454`–`0x1457` | R/W | u16/s16, 0.1 °C | Charge over-temp, high-temp, low-temp (`s16`), and under-temp (`s16`) limits |
| `5208`–`5210` | `0x1458`–`0x145A` | R/W | u16, 0.01 A | Charge over-current 2, over-current 1, and high-current limits |
| `5211`–`5214` | `0x145B`–`0x145E` | R/W | u16, 0.1 V | Module over-voltage, high-voltage, low-voltage, and under-voltage limits |
| `5215`–`5218` | `0x145F`–`0x1462` | R/W | u16/s16, 0.1 °C | Discharge over-temp, high-temp, low-temp (`s16`), and under-temp (`s16`) limits |
| `5219`–`5221` | `0x1463`–`0x1465` | R/W | s16, 0.01 A | Discharge over-current 2, over-current 1, and high-current limits |
| `5222` | `0x1466` | R/W | u16 | **Shutdown command** (`0` = don't care, `1` = shut down BMS) |
| `5223` | `0x1467` | R/W | u16 | **Device ID** (Modbus slave address, valid range `3`–`247`) |
| `5224` | `0x1468` | W | u16 | **Lock Control** (`0x5A5A` = Lock, `0xA5A5` = Unlock) |
| `5225` | `0x1469` | W | u16 | **Test Ready** (`0x5A5A` = Test begin, `0xA5A5` = Test over) |
| `5226`–`5227` | `0x146A`–`0x146B` | R/W | u32 | Unique identification code (`0xFFFFFFFF`) |
| `5228` / `5229` | `0x146C` / `0x146D` | R/W | u16, % | Charging / Discharging power setting (default `0x0064` = 100 %) |
| `61440`–`61442` | `0xF000`–`0xF002` | W | u16 | `ACP-Broadcast` (`0`), `ACP-Configure` (`1`–`254`), `ACP-Shake` (`1`–`254`) auto-addressing |
| FC `0x78` | — | W | — | **Restore factory defaults** (data `00 00 00 01`) |
| FC `0x79` | — | W | — | **Clear history** (data `00 00 00 01`) |

---

## Rover / Wanderer Charge Controller (Modbus RTU)

> Status: 📄 Vendor doc — transcribed from *Rover 20A/40A Charge Controller — MODBUS Protocol* ([`PDFs/ROVER.MODBUS.pdf`](PDFs/ROVER.MODBUS.pdf)), cross-checked against [`cyrils/renogy-bt/renogybt/RoverClient.py`](https://github.com/cyrils/renogy-bt/blob/main/renogybt/RoverClient.py) and [`rosswarren/renogymodbus/renogymodbus/charge_controller.py`](https://github.com/rosswarren/renogymodbus/blob/main/renogymodbus/charge_controller.py).

### Summary

| | |
|---|---|
| Products | Renogy Rover (`20A`/`30A`/`40A`/`60A`), Rover Elite (`RCC40RVRE`), Wanderer (`RNG-CTRL-WND10`, `WND30`), Adventurer (`RNG-CTRL-ADV30`), DC-DC MPPT Charger (`DCC50S` / `RBC50D1S-G1`), and SRNE-OEM equivalents (SRNE `ML24`/`ML48`, Rich Solar, PowMr) |
| Physical layer | **RS232** on RJ12 (6P6C, used with **BT-1**) or **RS485** on RJ45 (8P8C, used with **BT-2** or Communication Hub) |
| Settings | **9600 8N1**, Modbus RTU |
| Addressing | Default `0x01` (1) on direct serial; `0x10` / `0x11` (16 / 17) when daisy-chained; `0x60` / `0x61` (96 / 97) in Hub mode. Standalone controllers also respond to `0xFF` (255) over BT-1/BT-2 (`0x00` is no-reply broadcast). Stored in register `0x001A` (26) |
| Byte order | Register words are **big-endian** (`BE`); CRC-16/MODBUS is **little-endian** (`LE`) |

---

### Register Map (Function Code `0x03`)

#### System Information (`0x000A`–`0x001A`)

| Register (hex) | Register (dec) | Size (words) | Type | R/W | Unit / scale | Field |
|---|---|---|---|---|---|---|
| `0x000A` | `10` | 1 | u16 | R | High byte: V (`0xFF` = auto)<br>Low byte: A | Max system voltage supported (`12`, `24`, `36`, `48`, `96`, `255` = auto) and rated charging current (`10`, `20`, `30`, `45`, `60` A) |
| `0x000B` | `11` | 1 | u16 | R | High byte: A<br>Low byte: enum | Rated load discharge current (high byte) and product type (low byte: `0x00` controller, `0x01` inverter) |
| `0x000C`–`0x0013` | `12`–`19` | 8 (16 B) | ASCII | R | — | Product model string (leading/trailing spaces, e.g. `"RNG-CTRL-WND10"`, `"MT4830"`) |
| `0x0014`–`0x0015` | `20`–`21` | 2 | u32 | R | BCD/bytes | Software version: bytes `[00, major, minor, patch]` (e.g. `0x00030201` → `V03.02.01`) |
| `0x0016`–`0x0017` | `22`–`23` | 2 | u32 | R | BCD/bytes | Hardware version: bytes `[00, major, minor, patch]` (e.g. `0x00010203` → `V01.02.03`) |
| `0x0018`–`0x0019` | `24`–`25` | 2 | u32 | R | `YYMMNNNN` | Serial number: high word `YY` (year), `MM` (month); low word unit counter (`0x1501FFFF` → Jan 2015, #65535) |
| `0x001A` | `26` | 1 | u16 | R/W | 1 (`1`–`247`) | Controller Modbus device address (low byte) |

#### Dynamic Controller, Solar & Historical Telemetry (`0x0100`–`0x0122`)

`cyrils/renogy-bt` reads the entire 34-word block `0x0100`–`0x0121` (`256`–`289`) in a single FC `0x03` request: `FF 03 01 00 00 22 D1 F1`.

| Register (hex) | Register (dec) | Size (words) | Type | R/W | Unit / scale | Field |
|---|---|---|---|---|---|---|
| `0x0100` | `256` | 1 | u16 | R | 1 % | Battery state of charge (`SOC`, `0`–`100` %) |
| `0x0101` | `257` | 1 | u16 | R | 0.1 V | Battery voltage |
| `0x0102` | `258` | 1 | u16 | R | 0.01 A | Charging current (to battery) |
| `0x0103` | `259` | 1 | 2 × sm8 | R | 1 °C | **High byte**: Controller temperature; **Low byte**: Battery temperature.<br>Each byte uses **sign-magnitude** (`bit 7` = sign: `0` positive, `1` negative; `bits 6:0` = `°C`, e.g. `0x8A` = `−10 °C`) |
| `0x0104` | `260` | 1 | u16 | R | 0.1 V | Street light (DC load) voltage |
| `0x0105` | `261` | 1 | u16 | R | 0.01 A | Street light (DC load) current |
| `0x0106` | `262` | 1 | u16 | R | 1 W | Street light (DC load) power |
| `0x0107` | `263` | 1 | u16 | R | 0.1 V | Solar panel (PV) voltage |
| `0x0108` | `264` | 1 | u16 | R | 0.01 A | Solar panel (PV) current |
| `0x0109` | `265` | 1 | u16 | R | 1 W | Solar charging power |
| `0x010A` | `266` | 1 | u16 | W | bool | **Load On/Off command** (`1` = turn on load, `0` = turn off load) |
| `0x010B` | `267` | 1 | u16 | R | 0.1 V | Battery minimum voltage today |
| `0x010C` | `268` | 1 | u16 | R | 0.1 V | Battery maximum voltage today |
| `0x010D` | `269` | 1 | u16 | R | 0.01 A | Maximum charging current today |
| `0x010E` | `270` | 1 | u16 | R | 0.01 A | Maximum discharging current today |
| `0x010F` | `271` | 1 | u16 | R | 1 W | Maximum charging power today |
| `0x0110` | `272` | 1 | u16 | R | 1 W | Maximum discharging power today |
| `0x0111` | `273` | 1 | u16 | R | 1 Ah | Charging amp-hours today |
| `0x0112` | `274` | 1 | u16 | R | 1 Ah | Discharging amp-hours today |
| `0x0113` | `275` | 1 | u16 | R | **1 Wh** | Power generation today (PDF Table 1 says `kWh / 10000`; actual unit is **Wh**, see errata) |
| `0x0114` | `276` | 1 | u16 | R | **1 Wh** | Power consumption today (actual unit is **Wh**, see errata) |
| `0x0115` | `277` | 1 | u16 | R | 1 day | Total operating days |
| `0x0116` | `278` | 1 | u16 | R | 1 | Total battery over-discharge count |
| `0x0117` | `279` | 1 | u16 | R | 1 | Total battery full-charge count |
| `0x0118`–`0x0119` | `280`–`281` | 2 | u32 | R | 1 Ah | Cumulative battery charging amp-hours |
| `0x011A`–`0x011B` | `282`–`283` | 2 | u32 | R | 1 Ah | Cumulative battery discharging amp-hours |
| `0x011C`–`0x011D` | `284`–`285` | 2 | u32 | R | **1 Wh** | Cumulative power generation (actual unit in `renogy-bt` is **Wh**; see errata) |
| `0x011E`–`0x011F` | `286`–`287` | 2 | u32 | R | **1 Wh** | Cumulative power consumption |
| `0x0120` | `288` | 1 | u16 | R | packed | **High byte**: `bit 7` = load status (`0` off, `1` on), `bits 6:0` = load brightness (`0`–`100` %).<br>**Low byte**: Charging state (`0x00` deactivated, `0x01` activated, `0x02` MPPT, `0x03` equalizing, `0x04` boost, `0x05` floating, `0x06` current limiting / overpower) |
| `0x0121`–`0x0122` | `289`–`290` | 2 | u32 | R | bitmap | Controller fault and warning flags (below) |

#### EEPROM Battery Parameter & Historical Registers (`0xE001`–`0xF006`)

| Register (hex) | Register (dec) | Size (words) | Type | R/W | Unit / scale | Field |
|---|---|---|---|---|---|---|
| `0xE001` | `57345` | 1 | u16 | R/W | 1 % (`0`–`100`) | Street light (load) dimming brightness |
| `0xE002` | `57346` | 1 | u16 | R | 1 Ah | Nominal battery capacity |
| `0xE003` | `57347` | 1 | u16 | R/W | High/low byte: V | High byte: configured system voltage (`12`, `24`, `36`, `48`, `0xFF` = auto); Low byte: recognized voltage |
| `0xE004` | `57348` | 1 | u16 | R/W | enum | Battery type: `1` = open (flooded), `2` = sealed, `3` = gel, `4` = lithium, `5` = custom |
| `0xE005`–`0xE00E` | `57349`–`57358` | 10 × 1 | u16 | R/W | 0.1 V (`70`–`170`) | Voltage thresholds: over-voltage (`E005`), charge limit (`E006`), equalize (`E007`), boost/overcharge (`E008`), float/recovery (`E009`), boost recovery (`E00A`), over-discharge recovery (`E00B`), under-voltage warning (`E00C`), over-discharge (`E00D`), discharge limit (`E00E`) |
| `0xE01D` | `57373` | 1 | u16 | R/W | enum | Load working mode (`0x00` light control, `0x01`–`0x0E` light + 1–14 h timer, `0x0F` manual, `0x10` debug, `0x11` normal on) |
| `0xF000`–`0xF006` | `61440`–`61446` | 10 words per read | 20 B block | R | — | **Daily historical record** for day `0` (`0xF000`, today) through day `6` (`0xF006`, 6 days ago). Reading 10 words returns a 20-byte record where bytes `11:13` (response offset) = daily max power (W), `15:17` = daily charge Ah, `19:21` = daily power generation (Wh) (`RoverHistoryClient.py`) |

---

### Controller Fault and Warning Bits (`0x0121`–`0x0122`)

Registers `0x0121` (high word, bits 31..16) and `0x0122` (low word, bits 15..0) form a 32-bit fault bitmap (`1` = fault active, `0` = normal):

| Bit (in u32) | Bit in `0x0121` | Meaning |
|---|---|---|
| 31 | 15 | Reserved |
| 30 | 14 | Charge MOSFET short circuit |
| 29 | 13 | Anti-reverse MOSFET short circuit |
| 28 | 12 | Solar panel reversely connected |
| 27 | 11 | Solar panel working point over-voltage |
| 26 | 10 | Solar panel counter-current |
| 25 | 9 | Photovoltaic (PV) input side over-voltage |
| 24 | 8 | Photovoltaic (PV) input side short circuit |
| 23 | 7 | Photovoltaic (PV) input overpower |
| 22 | 6 | Ambient temperature too high |
| 21 | 5 | Controller temperature too high |
| 20 | 4 | Load overpower or load over-current |
| 19 | 3 | Load short circuit |
| 18 | 2 | Battery under-voltage warning *(Table 1 p. 11)* / Battery over-voltage *(Section 3.15 p. 20; see errata)* |
| 17 | 1 | Battery over-voltage *(Table 1 p. 11)* / Battery under-voltage *(Section 3.15 p. 20; see errata)* |
| 16 | 0 | Battery over-discharge |
| 15–0 | (`0x0122`) | Reserved (`0x0000`) |

---

### Control Commands (Dangerous)

> [!WARNING]
> Writing to controller registers can switch DC loads on/off, alter battery charging voltages (risking overcharge if mismatched with LiFePO4 limits), or erase historical data.

- **Switch DC load (`0x010A` / `266`)**: FC `0x06`, value `0x0001` (ON) or `0x0000` (OFF).
- **Modify battery charging parameters (`0xE003`–`0xE014`)**: FC `0x06` or `0x10`; changes system voltage, battery chemistry preset, and charge/discharge voltage cutoffs in EEPROM.
- **Factory reset (FC `0x78`) / Clear history (FC `0x79`)**: Non-standard vendor function codes (`01 78 00 00 00 01 60 00` and `01 79 00 00 00 01 5D C0`) that reset all parameters or permanently erase flash history.

---

## BT-1 / BT-2 Bluetooth Low Energy (BLE) Wrapping

> Status: 🌐 Community — documented from [`cyrils/renogy-bt`](https://github.com/cyrils/renogy-bt) (`BLEManager.py`, `BaseClient.py`) and [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) (`renogy_bms.py`, `renogy_pro_bms.py`).

Renogy's **BT-1** (RJ12 RS232 dongle), **BT-2** (RJ45 RS485 dongle), and built-in Bluetooth modules inside `-BT` batteries operate as **transparent BLE GATT-to-Modbus-RTU bridges**:

1. **No BLE framing wrapper**: The GATT client sends the raw Modbus RTU request frame (including the 1-byte Slave ID and 2-byte little-endian CRC-16) directly to the Write characteristic.
2. **GATT Notification replies**: The module forwards the slave's raw Modbus RTU response frame via GATT Notification(s) on the Notify characteristic (`len = ByteCount + 5` for FC `0x03`).

### BLE Discovery and GATT Profile

| Property | Value | Notes |
|---|---|---|
| Advertised name prefixes | `BT-TH-*`<br>`RNGRBP*`, `RNGC*`<br>`BTRIC*`, `RNGRIU*` | Standard BT-1 / BT-2 modules and standard BLE batteries<br>Renogy Pro series batteries (BLE Manufacturer ID `0xE14C` / `57676`)<br>Renogy Inverters |
| Write Service UUID | `0000ffd0-0000-1000-8000-00805f9b34fb` (`0xFFD0`) | Contains the TX (host → device) characteristic |
| **Write Characteristic (TX)** | `0000ffd1-0000-1000-8000-00805f9b34fb` (`0xFFD1`) | Write / Write-Without-Response (raw Modbus RTU request bytes) |
| Notify Service UUID | `0000fff0-0000-1000-8000-00805f9b34fb` (`0xFFF0`) | Contains the RX (device → host) characteristic |
| **Notify Characteristic (RX)** | `0000fff1-0000-1000-8000-00805f9b34fb` (`0xFFF1`) | Notify (raw Modbus RTU response bytes) |

---

## Captured and Example Frames

### Smart Lithium Battery (`RBT100LFP12-BT`, Slave ID `0x30`, from `patman15/aiobmsble` test suite)

```
# 1. Read cell voltages (5000..5016) and cell temperatures (5017..5033): 34 words (0x0022) at 0x1388
30 03 13 88 00 22 45 5C
-> 30 03 44 00 04 00 23 00 21 00 21 00 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 00 AA 00 AA 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 25 74
# Decodes to: 4 cells (3.5 V, 3.3 V, 3.3 V, 3.3 V), 2 temp sensors (17.0 °C, 17.0 °C)

# 2. Read dynamic pack telemetry (5042..5048): 7 words (0x0007) at 0x13B2
30 03 13 B2 00 07 A4 8A
-> 30 03 0E FF AA 00 88 00 01 7B B0 00 01 86 A0 00 0F E0 1E
# Decodes to: -0.86 A, 13.6 V, 97.200 Ah remaining, 100.000 Ah design capacity, 15 cycles

# 3. Read model name (5122..5129): 8 words (0x0008) at 0x1402
30 03 14 02 00 08 E4 1D
-> 30 03 10 52 42 54 31 30 30 4C 46 50 31 32 2D 42 54 20 20 58 BB
# Decodes to: "RBT100LFP12-BT  "
```

### Smart Lithium Battery Pro (`RNGRBP*`, Slave ID `0x30`, from `patman15/aiobmsble` test suite)

```
# Read dynamic pack telemetry (5042..5048): 7 words at 0x13B2 (note 0.1 A current scale on Pro)
30 03 13 B2 00 07 A4 8A
-> 30 03 0E FF F4 00 85 00 03 30 42 00 03 3B DA 00 06 3E 33
# Decodes to: -1.2 A (0xFFF4 = -12 * 0.1 A), 13.3 V, 208.962 Ah remaining, 211.930 Ah capacity, 6 cycles
```

### Rover Charge Controller (from `cyrils/renogy-bt` and `ROVER.MODBUS.pdf`)

```
# Poll all 34 dynamic registers (0x0100..0x0121) using broadcast address 0xFF (cyrils/renogy-bt)
FF 03 01 00 00 22 D1 F1

# Read system voltage/current (0x000A, 1 word) from slave 0x01 (ROVER.MODBUS.pdf 3.1)
01 03 00 0A 00 01 A4 08  ->  01 03 02 18 1E 32 4C       # 24 V system (0x18), 30 A rated (0x1E)

# Read load status, brightness, and charging stage (0x0120, 1 word) (ROVER.MODBUS.pdf 3.14)
01 03 01 20 00 01 84 3C  ->  01 03 02 E4 02 72 85       # Load ON (bit 7), 100 % brightness (0x64), MPPT stage (0x02)
```

---

## Errata and Disagreements

1. **Coarse `0.1 V` cell voltage resolution on Smart Lithium Batteries (`5001`–`5016`)**:
   Unlike most BMS protocols that report cell voltages in millivolts (`0.001 V`), the Renogy/RongSi BMS reports individual cell voltages in **`0.1 V` (100 mV)** increments (e.g. `35` = `3.5 V`). Confirmed by the vendor PDF (p. 9) and all open-source implementations.
2. **Renogy Pro Series (`RNGRBP*` / `RNGC*`) current scaling (`0.1 A` vs `0.01 A`) and cycle byte order**:
   Standard Renogy batteries scale register `5042` (`0x13B2`, pack current) by **`0.01 A`**. However, the **Renogy Pro** series (`RBT12100LFP-BT`, `RNGRBP*`, `RNGC*`) scales register `5042` by **`0.1 A`** (`patman15/aiobmsble` `renogy_pro_bms.py`). Furthermore, `RNGC*` batteries return the cycle count in register `5048` in little-endian byte order instead of big-endian ([`BMS_BLE-HA#596`](https://github.com/patman15/BMS_BLE-HA/issues/596)).
3. **Temperature register mapping in `dbus-serialbattery` (`renogy.py`)**:
   `dbus-serialbattery` labels register `5037` (`0x13AD`) as `command_bms_temperature_1` and `5040` (`0x13B0`) as `command_bms_temperature_2`, reading both as unsigned `u16`. Per the vendor PDF (p. 10), `5035` is the BMS board temperature, `5037`–`5038` are Environment Temperatures 1–2, `5040`–`5041` are Heater Temperatures 1–2, and all temperatures are signed (`s16`, `0.1 °C`).
4. **Battery BMS firmware V1.5 register shift (`5122`–`5141`)**:
   Per the vendor PDF revision history (p. 2), BMS firmware prior to protocol V1.5 (2019-08-23) placed Battery Name at `5122`–`5127` (6 words) and Software Version at `5128`–`5129`. From V1.5 onward, Battery Name is `5122`–`5129` (8 words), Software Version is `5130`–`5131` (2 words), and Manufacturer Name is `5132`–`5141` (10 words).
5. **Typos in `ROVER.MODBUS.pdf` examples and tables**:
   - **Temperature register `0x0103` (Section 3.7 vs Table 1)**: Section 3.7 (p. 17) claims battery and controller temperatures occupy two consecutive registers `0x0102` and `0x0103` and shows an invalid response frame (`01 03 02 0020 0028 73E7`, where `ByteCount = 02` is followed by 4 data bytes). In reality (and per Table 1 p. 9, `cyrils/renogy-bt`, and `renogymodbus`), `0x0102` is battery charging current (`0.01 A`), and **both** temperatures are packed into the single register `0x0103` (high byte = controller temp, low byte = battery temp).
   - **Energy units in `0x0113`, `0x0114`, `0x011C`–`0x011F` (Table 1 vs Section 3.13 vs implementations)**: Table 1 (pp. 9–10) states the unit is `"kilowatt hour / 10000"` (`0.1 Wh`), while Section 3.13 (p. 19) claims `0x000007D0` (`2000`) means `"2000 kilowatt-hours"`. Real controller captures in `cyrils/renogy-bt` show these registers report integer **watt-hours (`1 Wh`)**.
   - **Swapped fault bits `B17` and `B18` (`0x0121`)**: Table 1 (p. 11) defines `B18` as *battery under-voltage warning* and `B17` as *battery over-voltage*, whereas Section 3.15 (p. 20) swaps them (`B18` = *battery over-voltage*, `B17` = *battery under-voltage*).
   - **Write responses in Sections 3.16 and 3.17 (p. 20)**: The example FC `0x06` responses print register address `0100` / `0101` instead of echoing the written register `010A` / `E001`, and the Section 3.19 FC `0x10` request (`01 10 E005 0010 00AA...`) omits the required byte-count byte (`20`) after the word count `0010`.

---

## Sources

- Suzhou RongSi New Energy Technology Co., Ltd., *Lithium Iron Battery BMS Modbus Protocol V1.7*, 2020-03-10 — stored locally as [`PDFs/Lithium.Iron.Battery.BMS.Modbus.Protocol.V1.7.zh-CN.en.1.pdf`](PDFs/Lithium.Iron.Battery.BMS.Modbus.Protocol.V1.7.zh-CN.en.1.pdf) (from [`cyrils/renogy-bt` Discussion #94](https://github.com/cyrils/renogy-bt/discussions/94)).
- Renogy / SRNE, *Rover 20A/40A Charge Controller — MODBUS Protocol* — stored locally as [`PDFs/ROVER.MODBUS.pdf`](PDFs/ROVER.MODBUS.pdf) (from [`cyrils/renogy-bt` Discussion #94](https://github.com/cyrils/renogy-bt/discussions/94)).
- [`cyrils/renogy-bt`](https://github.com/cyrils/renogy-bt) — `renogybt/BaseClient.py`, `renogybt/BLEManager.py`, `renogybt/BatteryClient.py`, `renogybt/RoverClient.py`, `renogybt/RoverHistoryClient.py`, `renogybt/Utils.py`, and `README.md` (BLE GATT UUIDs, Hub/daisy-chain slave IDs, battery and Rover register parsing).
- [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/renogy.py` (RS485 Modbus battery polling and slave addresses `0xF7` / `0x30`).
- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/renogy_bms.py`, `aiobmsble/bms/renogy_pro_bms.py`, `tests/bms/test_renogy_bms.py`, and `tests/bms/test_renogy_pro_bms.py` (BLE BMS implementation, Pro series `0.1 A` current scale, and test frames).
- [`rosswarren/renogymodbus`](https://github.com/rosswarren/renogymodbus) — `renogymodbus/smart_battery.py` and `renogymodbus/charge_controller.py` (register mappings for smart battery and Rover charge controller).

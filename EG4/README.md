# EG4 Electronics — EG4-LL (Modbus RTU) & LifePower4 (RS485)

EG4 server-rack and wall-mount batteries use two distinct RS485 protocols depending on the generation, plus Pylontech/Luxpower-compatible CAN on newer models:

- **EG4-LL / LL-S (Modbus RTU)** — standard Modbus RTU over RS485 (`9600` 8N1, default master ID `0x10`).
- **EG4 LifePower4 (V1)** — proprietary `0x7E … 0x0D` tagged-group binary RS485 protocol.
- **Inverter closed-loop CAN** — EG4-LL and PowerPro/WallMount packs support Pylontech/Luxpower CAN (500 kbit/s, 11-bit IDs); see [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol) and [LUXPOWER](../LUXPOWER/README.md).

---

## EG4-LL / LL-S — Modbus RTU Protocol (V01.06)

> Status: 📄 Vendor doc — transcribed from [EG4-LL-MODBUS-Communication-Protocol.pdf](PDFs/EG4-LL-MODBUS-Communication-Protocol.pdf) (*Version V01.06*, 2017-06-21) and cross-checked against [mr-manuel/venus-os_dbus-serialbattery (`eg4_ll.py`)](https://github.com/mr-manuel/venus-os_dbus-serialbattery/blob/master/dbus-serialbattery/bms/eg4_ll.py).

### Summary

| | |
|---|---|
| Products | EG4-LL 12 V / 24 V / 48 V, EG4-LL-S |
| Physical layer | RS485 (daisy-chain between packs) |
| Settings | `9600` 8N1 (supports `9600`–`19200`; minimum polling interval `100 ms`) |
| Addressing | 4-bit DIP switch (`0x01`–`0x0F` slaves, `0x10` = ID 0 master BMS, all 4 switches ON) |
| Roles | Inverter/host is Modbus master; BMS is Modbus slave |
| Byte order | Big-endian for register values; Modbus CRC-16 is **little-endian** (`CRC Lo`, `CRC Hi`) |

### Framing

Standard Modbus RTU:

- Read holding registers: `<addr> 0x03 <reg_hi> <reg_lo> <count_hi> <count_lo> <crc_lo> <crc_hi>`
- Write multiple registers: `<addr> 0x10 <reg_hi> <reg_lo> <count_hi> <count_lo> <byte_count> <data...> <crc_lo> <crc_hi>`
- Checksum: **CRC-16/Modbus** (polynomial `0xA001` reflected, init `0xFFFF`, appended low byte first).

Worked examples (verified in Python):

```
10 03 00 00 00 27 06 91   # Read 39 live-data registers (0x0000–0x0026) from master BMS 0x10 (CRC = 0x9106 -> 06 91)
01 03 00 69 00 17 D5 D8   # Read 23 device-info registers (0x0069–0x007F) from slave BMS 0x01 (CRC = 0xD8D5 -> D5 D8)
```

### Live-data & device-info registers (`FC 0x03`)

| Register | Count | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0x0000` (0) | 1 | u16 | 10 mV (`0.01 V`) | Pack voltage |
| `0x0001` (1) | 1 | s16 | 10 mA (`0.01 A`) | Pack current (`+` charge, `−` discharge) |
| `0x0002`–`0x0011` (2–17) | 16 | u16 | 1 mV | Cell voltages 1–16 |
| `0x0012` (18) | 1 | s16 | 1 °C | PCB / MOSFET temperature |
| `0x0013` (19) | 1 | s16 | 1 °C | Average cell temperature |
| `0x0014` (20) | 1 | s16 | 1 °C | Max cell temperature |
| `0x0015` (21) | 1 | u16 | 1 Ah *(unverified)* | Remaining capacity (vendor PDF omits unit) |
| `0x0016` (22) | 1 | u16 | 1 A *(unverified)* | Max charging current |
| `0x0017` (23) | 1 | u16 | 1 % | SOH (`0–100`) |
| `0x0018` (24) | 1 | u16 | 1 % | SOC (`0–100`) |
| `0x0019` (25) | 1 | u16 | bitfield | Status & heater flag (below) |
| `0x001A` (26) | 1 | u16 | bitmask | Warning flags (below) |
| `0x001B` (27) | 1 | u16 | bitmask | Protection flags (below) |
| `0x001C` (28) | 1 | u16 | bitmask | Error code: bit 0 voltage error, bit 1 temp error, bit 2 current flow error, bit 4 cell unbalance |
| `0x001D`–`0x001E` (29–30) | 2 | u32 | count | Cycle count (`1–65535`) |
| `0x001F`–`0x0022` (31–34) | 4 | u32 / bytes | mAs (`mAh / 3600`) | Full capacity (`0x0020–0x0021`) / 6-sensor 1-byte temps (`0x0023`–`0x0025` range; see errata) |
| `0x0024` (36) | 1 | u16 | count | Cell count in string |
| `0x0025` (37) | 1 | u16 | 0.1 Ah | Design capacity |
| `0x0026` (38) | 1 | u16 | bitmask | Cell balancing status (bit 0 = cell 1 … bit 15 = cell 16) |
| `0x0069`–`0x0074` (105–116) | 12 | ASCII | — | Model name (24 bytes) |
| `0x0075`–`0x0077` (117–119) | 3 | ASCII | — | Firmware version (6 bytes, e.g. `V03R04`) |
| `0x0078`–`0x007F` (120–127) | 8 | ASCII | — | Serial number (16 bytes) |

### Status, warning, and protection bits

- **Status (`0x0019`)**: high byte `0x00` = heater off, `0x80` = heater on / active. Low nibble: `0x00` standby, `0x01` charging, `0x02` discharging, `0x04` protect, `0x08` charging limit.
- **Warning (`0x001A`) & Protection (`0x001B`)**:
  - bit 0 (`0x0001`): Pack OV
  - bit 1 (`0x0002`): Cell OV
  - bit 2 (`0x0004`): Pack UV
  - bit 3 (`0x0008`): Cell UV
  - bit 4 (`0x0010`): Charge OC
  - bit 5 (`0x0020`): Discharge OC
  - bit 6 (`0x0040`): Abnormal ambient temp
  - bit 7 (`0x0080`): MOS overheating
  - bit 8 (`0x0100`): Charge OT
  - bit 9 (`0x0200`): Discharge OT
  - bit 10 (`0x0400`): Charge UT
  - bit 11 (`0x0800`): Discharge UT
  - bit 12 (`0x1000`): Low capacity
  - bit 13 (`0x2000`): Float stopped (`0x001A`) / Discharge short-circuit (`0x001B`)

### Errata

- In `EG4-LL-MODBUS-Communication-Protocol.pdf` (page 6), the register numbers `0029`–`0035` are printed in decimal (`0x001D`–`0x0023`) with `0031`/`0032` overlapping `Full Capacity` and `Temp 6Byte`, and `0105`–`0127` (`0x0069`–`0x007F`) are likewise decimal register numbers. Always check `eg4_ll.py` (`0x0000` count `0x27` = 39 regs, `0x0069` count `0x17` = 23 regs) for the exact register alignment on your firmware.

---

## EG4 LifePower4 — Binary RS485 Protocol (`0x7E … 0x0D`)

> Status: 🌐 Community — from [mr-manuel/venus-os_dbus-serialbattery (`eg4_lifepower.py`)](https://github.com/mr-manuel/venus-os_dbus-serialbattery/blob/master/dbus-serialbattery/bms/eg4_lifepower.py) and `slim-bean/powermon`.

### Framing

- **Serial**: `9600` 8N1 (RS485).
- **Request frames** (6 bytes: `7E <addr> <cmd> 00 <chk> 0D`):
  - General telemetry (`cmd = 0x01`): e.g. addr `0x01` → `7E 01 01 00 FE 0D`; addr `0x02` → `7E 02 01 00 FC 0D`; addr `0x04` → `7E 04 01 00 F8 0D`.
  - Hardware version (`cmd = 0x42`): `7E <addr> 42 00 FC 0D`.
  - Firmware version (`cmd = 0x33`): `7E <addr> 33 00 <chk> 0D`.
- **Response (`0x01`)**: starts with `7E`, length byte at offset 3, ends with `0x0D`. Starting at offset 4, contains 10 sequential tagged groups formatted as `<group_id> <word_count> <word_count × u16 BE>`:

| Group | Field | Decoding |
|---|---|---|
| `0` | Cell voltages | `(raw & 0x3FFF) / 1000` V (top 2 bits of high byte carry alarm flags) |
| `1` | Current | `(30000 - raw) / 100` A (`+` charge, `−` discharge) |
| `2` | SOC | `raw / 100` % |
| `3` | Full capacity | `raw / 100` Ah |
| `4` | Temperatures (6 sensors) | `(raw & 0xFF) - 50` °C |
| `5` | Protection flags (word 1) | bit 3 over-current, bit 4 OV, bit 5 UV, bit 6 charge OT, bit 7 charge UT |
| `6` | Cycle count | `raw` |
| `7` | Pack voltage | `raw / 100` V |

---

## Sources

- [EG4-LL-MODBUS-Communication-Protocol.pdf](PDFs/EG4-LL-MODBUS-Communication-Protocol.pdf) — EG4-LL Modbus Protocol V01.06 (2017-06-21)
- [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/eg4_ll.py` and `eg4_lifepower.py`

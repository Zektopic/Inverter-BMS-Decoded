# PACE BMS — RS485, RS232, Modbus RTU and CAN Protocols

PACE (Paceic / PeiSi) BMS boards (`P16S100A`, `P16S150A`, `P16S200A`, `P15S100A`, etc.) are installed inside many 48 V / 51.2 V server-rack and wall-mount LiFePO4 batteries, including **Jakiper**, **SOK** (48 V rack), **Gobel Power**, **Rose**, **BSLBATT**, **Revov** (`R100`), **Greenrich** (`U-P5000`), **Easun Powerwall**, **Tewaycell**, **MeritSun**, **PowMr**, **Katbatt**, and **Eenovance / Sunsynk** (`MANA`).

Pick the section matching the port and protocol mode configured on your pack:

| Protocol / interface | Port | Settings | Framing | Status | Section |
|---|---|---|---|---|---|
| **PACE YD/T 1363.3 ASCII (V2.5)** | RS232 console (RJ11) & RS485 daisy-chain (RJ45) | 9600 8N1 | `~25…\r` ASCII hex (`VER = 0x25`, `CID1 = 0x46`) | 📄 Vendor doc | [YD/T 1363.3 ASCII V2.5](#pace-ydt-13633-ascii-protocol-v25--rs485-and-rs232) |
| **PACE Modbus RTU (`PACE_MODBUS`)** | RS485 inverter port (RJ45) | 9600 8N1 | Modbus RTU (`FC 0x03` / `0x10`, CRC-16/MODBUS) | 📄 Vendor doc | [Modbus RTU V1.3](#pace-modbus-rtu-protocol-v13-rs485) |
| **Inverter CAN & RS485 multi-protocol** | CAN (RJ45) & RS485 inverter port (RJ45) | 500 kbit/s (Pylontech CAN) / 250 kbit/s (PACE CAN) | Configurable (Pylontech, Victron, Growatt, GoodWe, …) | 📄 Vendor doc | [Inverter CAN & RS485 selection](#inverter-can-and-rs485-multi-protocol-selection) |

> **Older `VER = 0x20` packs:** Some early boards sold under EG4 (`LIFEPOWER4` / Qualtech `QT-PBMS`), Pylon, or Seplos branding use `VER = 0x20` (`CID1 = 0x46` or `0x4A` on EG4, with `CID2 = 0x47` system parameters, `0x4F` protocol version, `0x51` hardware version, `0x93` serial number, `0x95` shutdown). If your PC tool is **PbmsTools**, your BMS speaks **`VER = 0x25`** documented below; for `0x20` variants see [PYLONTECH](../PYLONTECH/README.md#rs485-protocol-v33) and [SEPLOS](../SEPLOS/README.md).

---

## PACE YD/T 1363.3 ASCII Protocol (V2.5 — RS485 and RS232)

> Status: 📄 Vendor doc (*PACE RS485 Communication Protocol* `2018-06-15` and *PACE RS232 Communication Protocol* `2018-07-05` in [`PDFs/`](PDFs/)), cross-checked against [`nkinnan/esphome-pace-bms`](https://github.com/nkinnan/esphome-pace-bms) (PbmsTools V2.4 captures) and [`mr-manuel/venus-os_dbus-serialbattery` (`bms/pace.py`)](https://github.com/mr-manuel/venus-os_dbus-serialbattery).

### Summary

| | |
|---|---|
| Products | PACE `P16S100A`, `P16S150A`, `P16S200A`, `P15S100A` and rebadged 15S/16S packs managed by `PbmsTools.exe` |
| Physical layer | **RS232** on 6P6C RJ11/RJ12 jack (console); **RS485** on 8P8C RJ45 jacks (inter-pack link / monitoring) |
| Settings | **9600 8N1** (9600 baud, 8 data bits, no parity, 1 stop bit); reply timeout 500 ms |
| Addressing | 4-bit binary DIP switch (`0`–`15`, see table below). Standalone = `0`; multi-pack master = `1`, slaves = `2`–`15` |
| Roles | Host (PC / ESP32 / GX device, or master pack on the inter-pack bus) polls slave BMS |
| Byte order | Big-endian (`BE`) for all multi-byte integers inside the ASCII-hex stream |

#### Connector pinouts and DIP switch addressing

| Port | Connector | Pinout |
|---|---|---|
| **RS232** | RJ11 (6P4C / 6P6C) | Pin 3: **BMS RXD** (connect to host TX) · Pin 4: **BMS TXD** (connect to host RX) · Pin 5: **GND** · Pins 1, 2, 6: NC / unused |
| **RS485** | RJ45 (8P8C) | Pin 1: **B−** · Pin 2: **A+** · Pin 3: **GND** · Pins 4–5: NC · Pin 6: **GND** · Pin 7: **A+** · Pin 8: **B−** *(pins 1–3 and 6–8 areparalleled across the dual RS485 jacks; verify against your battery manual as some brands wire CAN and RS485 on shared jacks)* |

The 4-position DIP switch sets the pack address in binary (`DIP1` = bit 0 / LSB, `DIP4` = bit 3 / MSB):

| Address | Hex | DIP 1 | DIP 2 | DIP 3 | DIP 4 | Typical role |
|---|---|---|---|---|---|---|
| `0` | `0x00` | OFF | OFF | OFF | OFF | Standalone single pack (or PC tool default) |
| `1` | `0x01` | ON | OFF | OFF | OFF | Master pack in parallel daisy-chain (aggregates slaves on RS232 / RS485) |
| `2` | `0x02` | OFF | ON | OFF | OFF | Slave 1 |
| `3` | `0x03` | ON | ON | OFF | OFF | Slave 2 |
| `4` | `0x04` | OFF | OFF | ON | OFF | Slave 3 |
| … | … | … | … | … | … | … |
| `15` | `0x0F` | ON | ON | ON | ON | Slave 14 |

---

### Framing

PACE V2.5 uses the Chinese telecom standard **YD/T 1363.3** ASCII-hex framing (identical in structure to [Seplos V2](../SEPLOS/README.md#frame-format) and [Pylontech RS485](../PYLONTECH/README.md#rs485-protocol-v33)):

```
~   25  01  46  42  E002  01    FD30  \r
│   │   │   │   │   │     │     │     └─ EOI: 0x0D ('\r', 1 raw byte)
│   │   │   │   │   │     │     └─────── CHKSUM: 4 ASCII hex chars (u16 BE)
│   │   │   │   │   │     └───────────── INFO: LENID ASCII hex chars (LENID/2 data bytes)
│   │   │   │   │   └─────────────────── LENGTH: 4 ASCII hex chars = LCHKSUM (4-bit) | LENID (12-bit)
│   │   │   │   └─────────────────────── CID2 (request command) or RTN (response return code)
│   │   │   └─────────────────────────── CID1: 0x46 ("46" = LiFePO4 / ternary lithium BMS; 0x4F seen on some 14S packs)
│   │   └─────────────────────────────── ADR: pack address ("00"–"0F")
│   └─────────────────────────────────── VER: protocol version ("25" = V2.5)
└─────────────────────────────────────── SOI: 0x7E ('~', 1 raw byte)
```

Every field between `SOI` (`0x7E`) and `EOI` (`0x0D`) is transmitted as uppercase hexadecimal ASCII characters (`'0'`–`'9'`, `'A'`–`'F'`), two characters per logical byte.

1. **`LENID`** (12-bit, bits 11..0 of `LENGTH`): number of **ASCII characters** in `INFO` (`2 ×` number of logical `INFO` bytes). When `INFO` is empty, `LENID = 0x000` and `LENGTH = "0000"`.
2. **`LCHKSUM`** (4-bit, bits 15..12 of `LENGTH`): sum the three hex nibbles of `LENID`, modulo 16, bitwise-invert (4-bit), plus 1:
   $$\text{LCHKSUM} = \left(\sim\left((\text{D}_{11..8} + \text{D}_{7..4} + \text{D}_{3..0}) \bmod 16\right) + 1\right) \ \& \ \text{0xF}$$
3. **`CHKSUM`** (16-bit, 4 ASCII hex chars): sum the **ASCII byte values** of all characters after `~` up to the end of `INFO` (excluding `SOI`, `CHKSUM`, and `EOI`), modulo 65536, bitwise-invert (16-bit), plus 1:
   $$\text{CHKSUM} = \left(\sim\left(\sum \text{ord}(c) \bmod 65536\right) + 1\right) \ \& \ \text{0xFFFF}$$

#### Worked checksum example (verified in Python)

Request analog telemetry (`CID2 = 0x42`) from pack `0x01` with `COMMAND = 0x01`:
- `INFO` is 1 byte (`"01"`) $\rightarrow \text{LENID} = 2$ (`0x002`).
- $\text{LCHKSUM} = (\sim(0 + 0 + 2) + 1) \ \& \ \text{0xF} = (\sim 2 + 1) \ \& \ \text{0xF} = \text{0xE} \rightarrow \text{LENGTH} = \texttt{"E002"}$.
- Body between `~` and `CHKSUM` is the 14-character string `"25014642E00201"`:
  ```python
  body = "25014642E00201"
  # ASCII codes: 0x32+0x35+0x30+0x31+0x34+0x36+0x34+0x32+0x45+0x30+0x30+0x32+0x30+0x31 = 0x02D0 (720)
  chksum = ((~sum(ord(c) for c in body)) + 1) & 0xFFFF  # -> 0xFD30 ("FD30")
  ```
- Complete frame: `~25014642E00201FD30\r` (`7E 32 35 30 31 34 36 34 32 45 30 30 32 30 31 46 44 33 30 0D`).

#### Return codes (`RTN` in response `CID2` field)

| `RTN` | Meaning |
|---|---|
| `0x00` | Normal / OK |
| `0x01` | Undefined / `VER` error |
| `0x02` | `CHKSUM` error |
| `0x03` | `LCHKSUM` error |
| `0x04` | `CID2` undefined / invalid command |
| `0x05` / `0x06` | Undefined / invalid format or data |
| `0x09` | Operation or write error (e.g. attempting to turn off both MOSFETs or write out-of-range config) |

---

### Requests and responses (`CID2` commands)

#### Monitoring and identification commands (read-only)

| `CID2` | Name | Request `INFO` | Response `INFO` | Notes |
|---|---|---|---|---|
| `0x90` | Read pack count / confirm address | none (`LENID = 0`) | 1 byte (`u8`) | Returns total connected BMS count on master (`ADR = 1`), or confirms pack presence (`2`–`15`) |
| `0x42` | Read analog information | 1 byte: target pack (`0x00`–`0x0F`, or `0xFF` for all packs) | 61 bytes (16S, 6 temps, `P=3`) per pack | Voltages, temperatures, current, capacities, cycles (table below) |
| `0x44` | Read warn / status information | 1 byte: target pack (`0x00`–`0x0F`, or `0xFF` for all packs) | 38 bytes (16S, 6 temps) per pack | Per-cell/temp warnings + 9 protection/status/fault/balance bitmask bytes (table below) |
| `0xA6` | Read pack capacity | none (`LENID = 0`) | 6 bytes (`3 × u16`) | Remaining capacity (`10 mAh`), full capacity (`10 mAh`), design capacity (`10 mAh`) |
| `0xB1` | Read BMS date/time | none (`LENID = 0`) | 6 bytes (`6 × u8`) | `Year - 2000`, `Month` (1–12), `Day` (1–31), `Hour` (0–23), `Minute` (0–59), `Second` (0–59) |
| `0xC1` | Read software / hardware version | none (`LENID = 0`) | 20 bytes (`LENID = 0x028`) | 20-char ASCII string, space-padded (e.g. `"P16S100A-1812-1.00 "`) |
| `0xC2` | Read product info / serial number | none (`LENID = 0`) | 20 or 40 bytes (`LENID = 0x028` or `0x050`) | 20- or 40-char ASCII serial string, space-padded |
| `0xA1` | Read log history record *(community)* | 2 bytes (`u16` record index `0x0000`–`0x018F`) | 72 bytes (`LENID = 0x090`), or 0 bytes when done | Observed in PbmsTools V2.4 ("Memory Information" tab); up to 400 records (`0` = newest) |

> **Single-pack vs. multi-pack broadcast (`0xFF`):** When querying a master BMS (`ADR = 0x01`) with `COMMAND = 0xFF` in `0x42` or `0x44`, the second byte of the response `INFO` (`Offset 1`) changes from the responding pack's address to the **number of concatenated pack payloads** (`1`–`16`), followed immediately by that many per-pack structures back-to-back. `dbus-serialbattery` sends `~25014642E002FFFD05\r` (or `ADR = 0x00`: `~25004642E002FFFD06\r`).

#### Common read request frames (all checksums verified)

| Target | Command | ASCII frame |
|---|---|---|
| Pack `0x00` (standalone) | `0x42` Analog (`CMD = 0x00`) | `~25004642E00200FD32\r` |
| Pack `0x00` (standalone) | `0x42` Analog (`CMD = 0xFF`) | `~25004642E002FFFD06\r` |
| Pack `0x00` (standalone) | `0x44` Status (`CMD = 0x01`) | `~25004644E00201FD2F\r` |
| Pack `0x00` (standalone) | `0x44` Status (`CMD = 0xFF`) | `~25004644E002FFFD04\r` |
| Pack `0x01` (master) | `0x90` Pack count | `~250146900000FDA5\r` |
| Pack `0x01` (master) | `0x42` Analog (`CMD = 0x01`) | `~25014642E00201FD30\r` |
| Pack `0x01` (master) | `0x44` Status (`CMD = 0x01`) | `~25014644E00201FD2E\r` |
| Pack `0x01` (master) | `0xC1` Hardware version | `~250146C10000FD9A\r` |
| Pack `0x01` (master) | `0xC2` Serial number | `~250146C20000FD99\r` |
| Pack `0x02` (slave 1) | `0x90` Confirm address | `~250246900000FDA4\r` |
| Pack `0x02` (slave 1) | `0x42` Analog (`CMD = 0x02`) | `~25024642E00202FD2E\r` |
| Pack `0x02` (slave 1) | `0x44` Status (`CMD = 0x02`) | `~25024644E00202FD2C\r` |

---

### `0x42` Analog information response (`DATA_INFO`)

For a standard 16-cell, 6-temperature pack with user-defined count `P = 3`, `DATA_INFO` is **61 bytes** (`LENID = 0x07A` = 122 ASCII hex chars, `LENGTH = "F07A"`). All multi-byte fields are big-endian (`BE`).

| Byte offset (in `INFO`) | Size (bytes) | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0` | 1 | `u8` | — | `INFOFLAG` (always `0x00` on wire; see Errata) |
| `1` | 1 | `u8` | — | Pack address (echoes `COMMAND`), or pack count $K$ when `COMMAND = 0xFF` |
| `2` | 1 | `u8` | count | Cell count $M$ (typically `0x10` = 16) |
| `3` | $2 \times M$ | `u16 BE` | `1 mV` (`0.001 V`) | Cell voltages $1 \dots M$ (`32` bytes for 16 cells) |
| `3 + 2M` (`35`) | 1 | `u8` | count | Temperature sensor count $N$ (typically `0x06` = 6; `0x08` on Eenovance/Sunsynk) |
| `4 + 2M` (`36`) | $2 \times N$ | `u16 BE` | `0.1 K` | Temperatures $1 \dots N$: °C $= (\text{raw} - 2730) / 10$. When $N=6$: #1–4 cell groups, #5 MOSFET, #6 ambient/environment |
| `4 + 2M + 2N` (`48`) | 2 | `s16 BE` | `0.01 A` (`10 mA`) | Pack current (**signed**: positive = charging, negative = discharging) |
| `6 + 2M + 2N` (`50`) | 2 | `u16 BE` | `1 mV` (`0.001 V`) | Pack total voltage |
| `8 + 2M + 2N` (`52`) | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) | Pack remaining capacity |
| `10 + 2M + 2N` (`54`) | 1 | `u8` | count | User-defined field count $P$ (`0x03` standard; see Firmware Variants below) |
| `11 + 2M + 2N` (`55`) | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) | Pack full-charge capacity |
| `13 + 2M + 2N` (`57`) | 2 | `u16 BE` | cycles | Cycle count |
| `15 + 2M + 2N` (`59`) | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) | Pack design (nominal) capacity |

- **State of Charge (SOC)** is not sent as a separate field in `0x42`; compute $\text{SOC (\%)} = (\text{Remaining Capacity} / \text{Full Capacity}) \times 100$.
- **State of Health (SOH)** is computed as $\min(100, (\text{Full Capacity} / \text{Design Capacity}) \times 100)$.
- **Known firmware variants (`P` value at offset 54):**
  - `P = 0x03` (`LENGTH = F07A`, 122 hex chars): Standard PACE V2.5 (Jakiper, SOK, Gobel, Easun, Tewaycell).
  - `P = 0x02` (`LENGTH = D07C`, 124 hex chars): Greenrich `U-P5000` and FSP `PS5120E` — appends 1 extra trailing byte (2 hex chars) after Design Capacity.
  - `P = 0x09` (150 hex chars): Wall-mount variant reported in `nkinnan/esphome-pace-bms` — appends 14 extra trailing bytes (28 hex chars).
  - `P = 0x04` (`LENGTH = 909E`, 158 hex chars): Eenovance `MANA` / Sunsynk — reports $N = 8$ temperatures (+4 bytes / 8 hex chars) and appends 14 extra trailing bytes (28 hex chars).

---

### `0x44` Warn / status information response (`DATA_INFO`)

For a 16-cell, 6-temperature pack, `DATA_INFO` is **38 bytes** (`LENID = 0x04C` = 76 ASCII hex chars, `LENGTH = "004C"`).

| Byte offset (in `INFO`) | Size (bytes) | Type | Meaning |
|---|---|---|---|
| `0` | 1 | `u8` | `INFOFLAG` (`0x00`) |
| `1` | 1 | `u8` | Pack address (or pack count $K$ when `COMMAND = 0xFF`) |
| `2` | 1 | `u8` | Cell count $M$ (`0x10` = 16) |
| `3` | $M$ | `u8[M]` | Per-cell warning byte for cells $1 \dots M$ |
| `3 + M` (`19`) | 1 | `u8` | Temperature sensor count $N$ (`0x06` = 6) |
| `4 + M` (`20`) | $N$ | `u8[N]` | Per-temperature warning byte for sensors $1 \dots N$ |
| `4 + M + N` (`26`) | 1 | `u8` | Pack charge current warning byte |
| `5 + M + N` (`27`) | 1 | `u8` | Pack total voltage warning byte |
| `6 + M + N` (`28`) | 1 | `u8` | Pack discharge current warning byte |
| `7 + M + N` (`29`) | 1 | `u8` | **Protect State 1** (bitmask, table below) |
| `8 + M + N` (`30`) | 1 | `u8` | **Protect State 2** (bitmask, table below) |
| `9 + M + N` (`31`) | 1 | `u8` | **Instructions / System State** (bitmask, table below) |
| `10 + M + N` (`32`) | 1 | `u8` | **Control / Configuration State** (bitmask, table below) |
| `11 + M + N` (`33`) | 1 | `u8` | **Fault State** (bitmask, table below) |
| `12 + M + N` (`34`) | 1 | `u8` | **Balance State 1** (bits 0..7 = balancing cells 1..8, or high byte of 16-bit mask; see Errata) |
| `13 + M + N` (`35`) | 1 | `u8` | **Balance State 2** (bits 0..7 = balancing cells 9..16, or low byte of 16-bit mask; see Errata) |
| `14 + M + N` (`36`) | 1 | `u8` | **Warn State 1** (bitmask, table below) |
| `15 + M + N` (`37`) | 1 | `u8` | **Warn State 2** (bitmask, table below) |

**Scalar warning byte values (offsets `3` to `6 + M + N`):**
- `0x00`: Normal
- `0x01`: Below lower limit
- `0x02`: Above upper limit
- `0x80`–`0xEF`: User-defined fault
- `0xF0`: Other fault

#### Status, protection, fault, and warning bitmasks (offsets 29–37 for $M=16, N=6$)

| Byte (offset) | Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
|---|---|---|---|---|---|---|---|---|
| **Protect State 1** (`29`) | Charger high-voltage in protect *(PbmsTools)* | Short-circuit protect | Discharge OC protect | Charge OC protect | Pack UV protect | Pack OV protect | Cell UV protect | Cell OV protect |
| **Protect State 2** (`30`) | **Fully charged** (status flag, not a fault) | Env UT protect | Env OT protect | MOSFET OT protect | Discharge UT protect | Charge UT protect | Discharge OT protect | Charge OT protect |
| **Instructions / System State** (`31`) | Heater active (`Heart indicate`) | AC in (`PbmsTools`) / undefined | Charging (`PbmsTools`) / `ACin` (`PDF`) | Reverse polarity (`Reverse indicate`) | Discharging (`PbmsTools`) / `Pack indicate` (`PDF`) | Discharge MOSFET (`DFET`) ON | Charge MOSFET (`CFET`) ON | Current limiter (`1` = ON in PDF; `1` = OFF in PbmsTools — see Errata) |
| **Control / Config State** (`32`) | Undefined | Static balance enabled *(PbmsTools)* | LED alarm (`1` = disabled in PDF; `1` = enabled in PbmsTools) | Current limiter (`1` = disabled in PDF; `1` = enabled in PbmsTools) | Current limit gear (`1` = low gear, `0` = high gear) | DFET turned off *(observed)* | CFET turned off *(observed)* | Buzzer alarm enabled (`1` = enabled) |
| **Fault State** (`33`) | Heater fault *(PbmsTools)* | CCB fault *(PbmsTools)* | Front-end / VCC sampling fault | Cell fault | Comm fault *(later FW)* | NTC (temp sensor) fault | Discharge MOSFET fault | Charge MOSFET fault |
| **Warn State 1** (`36`) | Undefined | Undefined | Discharge OC warn | Charge OC warn | Pack UV warn | Pack OV warn | Cell UV warn | Cell OV warn |
| **Warn State 2** (`37`) | Low power (low SOC) warn | MOSFET OT warn | Env UT warn | Env OT warn | Discharge UT warn | Charge UT warn | Discharge OT warn | Charge OT warn |

---

### Protection & system parameter commands (`PbmsTools` V2.4 — `VER = 0x25`)

> Status: 🌐 Community — decoded from `PbmsTools.exe` V2.4 traffic captures by [`nkinnan/esphome-pace-bms`](https://github.com/nkinnan/esphome-pace-bms) (`docs/pbmstools-communication.md` and `pace_bms_protocol_v25.h`). Note: `PbmsTools` V2.4 only addresses `ADR = 0x00` when reading/writing parameters, though the BMS firmware accepts the pack's own `ADR`.

All read requests send an empty `INFO` (`LENGTH = "0000"`). Multi-byte values in `INFO` are `u16 BE` / `s16 BE`.

| Read `CID2` | Write `CID2` | Parameter group | `INFO` payload layout (byte offsets in `INFO`) |
|---|---|---|---|
| `0xD1` | `0xD0` | Cell over-voltage | `0`: `0x01` · `1–2`: Alarm (`mV`) · `3–4`: Protect (`mV`) · `5–6`: Release (`mV`) · `7`: Delay (`0.1 s`, e.g. `0x0A` = 1.0 s) |
| `0xD3` | `0xD2` | Cell under-voltage | `0`: `0x01` · `1–2`: Alarm (`mV`) · `3–4`: Protect (`mV`) · `5–6`: Release (`mV`) · `7`: Delay (`0.1 s`) |
| `0xD5` | `0xD4` | Pack over-voltage | `0`: `0x01` · `1–2`: Alarm (`mV`) · `3–4`: Protect (`mV`) · `5–6`: Release (`mV`) · `7`: Delay (`0.1 s`) |
| `0xD7` | `0xD6` | Pack under-voltage | `0`: `0x01` · `1–2`: Alarm (`mV`) · `3–4`: Protect (`mV`) · `5–6`: Release (`mV`) · `7`: Delay (`0.1 s`) |
| `0xD9` | `0xD8` | Charge over-current | `0`: `0x01` · `1–2`: Alarm (`1 A`) · `3–4`: Protect (`1 A`) · `5`: Delay (`0.1 s`) |
| `0xDB` | `0xDA` | Discharge slow OC (OC1) | `0`: `0x01` · `1–2`: Alarm (`1 A`, **read as negative `s16` two's complement, written as positive `u16`!**) · `3–4`: Protect (`1 A`, same sign quirk!) · `5`: Delay (`0.1 s`) |
| `0xE3` | `0xE2` | Discharge fast OC (OC2) | `0`: `0x00` · `1`: Protect (`u8`, `1 A`) · `2`: Delay (`u8`, `25 ms` steps, e.g. `0x04` = 100 ms) *(read reply repeats the 3 bytes twice, 6 bytes total; write sends 3 bytes)* |
| `0xE5` | `0xE4` | Short-circuit protection | `0`: Delay (`u8`, `25 µs` steps, e.g. `0x0C` = 300 µs) |
| `0xB6` | `0xB5` | Cell balancing | `0–1`: Start threshold (`mV`, e.g. `0x0D48` = 3400 mV) · `2–3`: Start delta (`mV`, e.g. `0x001E` = 30 mV) |
| `0xA0` | `0xA8` | Sleep configuration | `0–1`: Cell sleep voltage (`mV`, e.g. `0x0C1C` = 3100 mV) · `2`: `0x00` · `3`: Sleep delay (`u8`, minutes, e.g. `0x05` = 5 min) |
| `0xAF` | `0xAE` | Full-charge & low-SOC | `0–1`: Full-charge voltage (`mV`, e.g. `0xDAC0` = 56000 mV) · `2–3`: Full-charge current (`mA`, e.g. `0x07D0` = 2000 mA) · `4`: Low-SOC alarm (`u8`, `%`, e.g. `0x05` = 5 %) |
| `0xDD` | `0xDC` | Charge/discharge OT | `0`: `0x01` · `1–2`: Chg OT alarm · `3–4`: Chg OT protect · `5–6`: Chg OT release · `7–8`: Dischg OT alarm · `9–10`: Dischg OT protect · `11–12`: Dischg OT release (all `u16` in `0.1 K`: raw $= °\text{C} \times 10 + 2730$) |
| `0xDF` | `0xDE` | Charge/discharge UT | `0`: `0x01` · `1–2`: Chg UT alarm · `3–4`: Chg UT protect · `5–6`: Chg UT release · `7–8`: Dischg UT alarm · `9–10`: Dischg UT protect · `11–12`: Dischg UT release (all `u16` in `0.1 K`) |
| `0xE1` | `0xE0` | MOSFET OT | `0`: `0x01` · `1–2`: Alarm (`0.1 K`) · `3–4`: Protect (`0.1 K`) · `5–6`: Release (`0.1 K`) |
| `0xE7` | `0xE6` | Environment UT & OT | `0`: `0x01` · `1–2`: Env UT alarm · `3–4`: Env UT protect · `5–6`: Env UT release · `7–8`: Env OT alarm · `9–10`: Env OT protect · `11–12`: Env OT release (all `u16` in `0.1 K`) |
| `0xED` | `0xEE` | Charge limiter start current | `0`: `0x00` · `1`: Start current (`u8`, `1 A`, e.g. `0x64` = 100 A) |
| `0xEB` | `0xEC` | Inverter CAN / RS485 protocol | `0`: CAN protocol ID (`u8`) · `1`: RS485 protocol ID (`u8`) · `2`: Type (`0x00` Auto, `0x01` Manual) — see [Inverter selection table](#inverter-can-and-rs485-multi-protocol-selection) |

---

### Control commands (dangerous)

> **Warning:** Do not send these commands unless you intend to switch power MOSFETs, alter battery protection thresholds, or shut down the pack. Disabling protections or switching MOSFETs under load can damage the BMS, inverter, or cells.

1. **`CID2 = 0x99` — Switch control (1-byte `INFO` command code):**
   - `0x06` / `0x07`: LED warning switch (vendor PDF states `0x06` = enable, `0x07` = disable; `PbmsTools` on the wire sends `0x06` = disable, `0x07` = enable — see Errata).
   - `0x08` / `0x09`: Charge current limiter gear (`0x08` = high gear / 10 A, `0x09` = low gear / 5 A).
   - `0x0A` / `0x0B`: Charge current limiter switch (vendor PDF states `0x0A` = enable, `0x0B` = disable; `PbmsTools` sends `0x0A` = disable, `0x0B` = enable).
   - `0x0C` / `0x0D`: Buzzer alarm (`0x0C` = disable, `0x0D` = enable).
2. **`CID2 = 0x9A` — Charge MOSFET control (1-byte `INFO`):**
   - `0x00` = close/turn ON charge MOSFET; `0x01` = open/turn OFF charge MOSFET. (The BMS refuses to turn off both MOSFETs simultaneously and returns `RTN = 0x09`.)
3. **`CID2 = 0x9B` — Discharge MOSFET control (1-byte `INFO`):**
   - `0x00` = close/turn ON discharge MOSFET; `0x01` = open/turn OFF discharge MOSFET.
4. **`CID2 = 0x9C` — Shutdown / reboot (1-byte `INFO = 0x01`):**
   - Shuts down the BMS (or reboots it immediately if charge/discharge current is flowing).
5. **`CID2 = 0xB2` — Write RTC date/time (6-byte `INFO`):**
   - Writes system clock (`Year - 2000`, `Month`, `Day`, `Hour`, `Minute`, `Second`).
6. **`CID2 = 0xA8, 0xAE, 0xB5, 0xD0..0xEE` — Write protection parameters and inverter protocols:**
   - Modifies cell/pack OV/UV, OC, temperature limits, and active CAN/RS485 inverter protocol in non-volatile memory.

---

### Captured frames

#### Example 1 — `0x42` Analog & `0x44` Status (from `nkinnan/esphome-pace-bms`, Jakiper `P16S100A-1812-1.00`, `ADR = 0x01`)

```
# Read Analog Information (CID2 = 0x42, target = 0x01)
~25014642E00201FD30\r
-> ~25014600F07A0001100CC70CC80CC70CC70CC70CC50CC60CC70CC70CC60CC70CC60CC60CC70CC60CC7060B9B0B990B990B990BB30BBCFF1FCCCD12D303286A008C2710E1E4\r
# Decodes to:
#   INFOFLAG = 0x00, Pack = 0x01, Cells = 16 (3269..3272 mV)
#   Temps = 6: 24.1 °C, 23.9 °C, 23.9 °C, 23.9 °C, 26.5 °C (MOS), 27.4 °C (Env)
#   Current = 0xFF1F (-225 -> -2.25 A discharging), Pack voltage = 0xCCCD (52.429 V)
#   Remaining = 0x12D3 (48.19 Ah), P = 0x03, Full = 0x286A (103.46 Ah), Cycles = 0x008C (140), Design = 0x2710 (100.00 Ah)

# Read Status Information (CID2 = 0x44, target = 0x01)
~25014644E00201FD2E\r
-> ~25014600004C000110000000000000000000000000000000000600000000000000000000000E000000000000EF3A\r
# Decodes to:
#   16 cells normal (0x00), 6 temps normal (0x00), current/voltage warnings = 0x00
#   Protect1 = 0x00, Protect2 = 0x00, System State = 0x0E (Discharging + DFET ON + CFET ON), Config/Fault/Balance/Warn = 0x00

# Read Hardware Version (CID2 = 0xC1) and Serial Number (CID2 = 0xC2)
~250146C10000FD9A\r -> ~25014600602850313653313030412D313831322D312E30302000F58E\r   # "P16S100A-1812-1.00 \x00"
~250146C20000FD99\r -> ~25014600B05031383132313031333830333039442020202020202020202020202020202020202020202020202020EE0F\r
```

#### Example 2 — Vendor PDF `0x42` example (`PACE-RS485-MS`, `ADR = 0x02`)

```
~25024642E00202FD2E\r
-> ~25024600F07A0002100D370CE50D080CED0D060CE70D1D0CEB0CF80CFA0CFB0D070CE10CF10CC20D06060BAA0BAC0BA60BA70BA90BB20000CF9406D603138800001388E261\r
# 16 cells (3266..3383 mV), 6 temps (25.2..26.4 °C), 0.00 A, 53.140 V, 17.50 / 50.00 Ah, 0 cycles, CHKSUM = E261
```

---

### Errata and disagreements

1. **Missing `INFOFLAG` byte in the vendor tables (Chart A.13 / A.16 vs. wire format):**
   In both the RS485 (`2018-06-15`) and RS232 (`2018-07-05`) PDFs, the prose above the `0x42` and `0x44` response tables states that `DATA_INFO` consists of `INFOFLAG` (`0x00`) followed by `DATAI`, and the worked examples on page 3 (RS485) and page 16 (RS232) include `30 30` (`0x00`) as the first byte of `DATA_INFO`. However, Charts A.13 and A.16 start numbering `DATAI` at item 1 (`Command` / `PACK number`), omitting `INFOFLAG` from the table rows. Parsers must read `INFOFLAG` at byte offset `0` of `INFO` before reading the pack address at byte offset `1`.
2. **RS485 PDF page 3–4 worked example copy-paste errors:**
   In `PACE-BMS-RS485-communication-protocol-20180615.pdf` (pages 3–4), the raw hex response on line 66 is for address `2` (`30 32` at `ADR` and `COMMAND`) and ends with the valid checksum `45 32 36 31` (`E261`). In the line-by-line breakdown immediately below it (lines 74, 79, 115), the author pasted `30 30` (address `0`) and `45 32 36 35` (`E265`) from a standalone pack example. Both frames (`ADR=02` with `E261` and `ADR=00` with `E265`) have valid checksums for their respective addresses.
3. **RS485 PDF Chart A.16 (`0x44` warn response) row M+N+15 typo:**
   On page 11 of the RS485 PDF, item `M + N + 15` is mislabelled `"Protect state 2 — See at charA.23"`. Chart A.23 is actually **Warn state 2** (matching the RS232 PDF Chart A.18).
4. **RS485 PDF Chart A.25 (`0xC1` software version response) header typo:**
   On page 14 of the RS485 PDF, the response format table for `0xC1` prints `C2H` in column 5 instead of `RTN` (`00H`). Real packs return `RTN = 0x00` (as correctly shown in the RS232 PDF Chart A.29).
5. **Temperature Kelvin offset (`2730` vs. `2731`):**
   Unlike Pylontech and Seplos (which use `2731` = $273.1\text{ K}$ for $0\,^\circ\text{C}$), all PACE vendor documents and `PbmsTools` use **`2730`** ($\text{raw} = \,^\circ\text{C} \times 10 + 2730$). Note that `dbus-serialbattery` (`pace.py`) uses `(raw / 10) - 273`, which agrees with `2730`.
6. **Polarity of LED alarm and Charge Current Limiter bits (`0x44` & `0x99`):**
   Charts A.19, A.20, and A.26 of the vendor PDFs state that in `Control State` bit 5 (LED) and bit 4 (Current limiter), `1 = unenable, 0 = enable`, and that `0x99` command `0x06` enables LED and `0x0A` enables current limiter. `nkinnan/esphome-pace-bms` observed from `PbmsTools` V2.4 wire captures that `0x06` disables LED (`0x07` enables) and `0x0A` disables the current limiter (`0x0B` enables), and that `Instructions State` bit 0 is `1` when the current limiter is disabled.
7. **Discharge Slow Over-Current (`0xDB` / `0xDA`) sign asymmetry:**
   When reading `0xDB`, the BMS reports the discharge OC1 alarm and protection thresholds as **negative** 16-bit two's complement integers (e.g. `-105 A` = `0xFF97`), but when writing `0xDA`, `PbmsTools` sends them as **positive** 16-bit integers (`+105 A` = `0x0069`).
8. **Cell balancing bit order on certain rebadged packs:**
   Standard packs report cells 1–16 across the 16-bit `Balance State` word (or `Balance State 1` = cells 1–8, `Balance State 2` = cells 9–16). On the Easun Powerwall (`P16S100A-32117-1.00`), users reported that the two balancing bytes are swapped relative to physical cell numbers (cells 1–8 and 9–16 exchanged).
9. **`dbus-serialbattery` (`bms/pace.py`) copy-paste bugs:**
   In `dbus-serialbattery`'s `pace.py`, `read_fuses_data()` (lines 306–343) reads `warn_state1` and `warn_state2` from the buffer, but its `if` conditions accidentally test `protect_state1` and `protect_state2` again instead of `warn_state1` and `warn_state2`. Furthermore, in `balance_state2` (line 283), `mask2` is initialized to `0x10` instead of `0x01`, shifting cells 9–12 and losing cells 13–16.

---

## PACE Modbus RTU Protocol V1.3 (RS485)

> Status: 📄 Vendor doc (*PACE BMS Modbus Protocol for RS485 V1.3*, `2017-06-27`, stored in [`PDFs/`](PDFs/)), cross-checked against [`syssi/esphome-pace-bms`](https://github.com/syssi/esphome-pace-bms).

### Summary

| | |
|---|---|
| Products | PACE `P16S100A`, `P16S150A`, `P16S200A` with RS485 Inverter Protocol set to `PACE_MODBUS` (`0x00`) in `PbmsTools.exe` |
| Physical layer | RS485 port adjacent to the CAN port (RJ45: Pin 1/8 `B−`, Pin 2/7 `A+`, Pin 3/6 `GND`) |
| Settings | **9600 8N1**; frame timeout 200 ms; minimum inter-frame interval > 100 ms |
| Addressing | Slave ID `0x01`–`0x0F` from the 4-bit binary DIP switch (`0x00` when all OFF; Modbus spec lists `0x01`–`0x10`) |
| Function codes | `0x03` Read Holding Registers (`1`–`125` registers), `0x10` Write Multiple Registers (`1`–`123` registers) |
| Byte order | Big-endian (`BE`) for 16-bit register payloads; **little-endian (`LE`)** for the 16-bit Modbus CRC |

### Framing and worked CRC-16 example (verified in Python)

Standard Modbus RTU framing:

```
[Slave Addr (1B)] [Func Code (1B)] [Data (N bytes)] [CRC Lo (1B)] [CRC Hi (1B)]
```

The CRC is **CRC-16/MODBUS** (polynomial `0x8005`, reflected `0xA001`, initial value `0xFFFF`, no final XOR), appended **low byte first, high byte second**.

**Worked example:** Read the first 8 holding registers (`0x0000`–`0x0007`: current, voltage, SOC, SOH, capacities, cycles) from slave `0x01` using Function Code `0x03`:
```python
def modbus_crc16(data: bytes) -> int:
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc >> 1) ^ 0xA001) if (crc & 1) else (crc >> 1)
    return crc

pdu = bytes.fromhex("01 03 00 00 00 08")
crc = modbus_crc16(pdu)  # -> 0x0C44 (CRC Lo = 0x44, CRC Hi = 0x0C)
```
Complete request frame: `01 03 00 00 00 08 44 0C` (to read all 37 live telemetry registers `0x0000`–`0x0024` in one transaction: `01 03 00 00 00 25 84 11`).

---

### Holding register map (`FC 0x03` read / `FC 0x10` write)

Every register is 16 bits (2 bytes, `BE`). Where the vendor table specifies `UINT8`, the value is stored in the low byte (`LSB`) with the high byte (`MSB`) set to `0x00`.

#### Telemetry, status, and identification registers (read-only)

| Register (dec) | Register (hex) | Count | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `0` | `0x0000` | 1 | `s16` | `0.01 A` (`10 mA`) | Pack current (**signed**: `+` charging, `−` discharging) |
| `1` | `0x0001` | 1 | `u16` | `0.01 V` (`10 mV`) | Pack total voltage *(note: `10 mV` here vs `1 mV` in YD/T 1363.3!)* |
| `2` | `0x0002` | 1 | `u8` (in `u16`) | `1 %` | State of Charge (SOC, `0`–`100 %`) |
| `3` | `0x0003` | 1 | `u8` (in `u16`) | `1 %` | State of Health (SOH, `0`–`100 %`) |
| `4` | `0x0004` | 1 | `u16` | `0.01 Ah` (`10 mAh`) | Remaining capacity |
| `5` | `0x0005` | 1 | `u16` | `0.01 Ah` (`10 mAh`) | Full-charge capacity |
| `6` | `0x0006` | 1 | `u16` | `0.01 Ah` (`10 mAh`) | Design capacity |
| `7` | `0x0007` | 1 | `u16` | cycles | Battery cycle count |
| `8` | `0x0008` | 1 | — | — | Reserved |
| `9` | `0x0009` | 1 | `u16` | bitmask | **Warning flag** (table below) |
| `10` | `0x000A` | 1 | `u16` | bitmask | **Protection flag** (table below) |
| `11` | `0x000B` | 1 | `u16` | bitmask | **Status / Fault flag** (table below) |
| `12` | `0x000C` | 1 | `u16` | bitmask | **Balance status** (bits 0..15 = cell 1..16 balancing active) |
| `13`–`14` | `0x000D`–`0x000E` | 2 | — | — | Reserved |
| `15`–`30` | `0x000F`–`0x001E` | 16 | `u16[16]` | `1 mV` (`0.001 V`) | Cell voltages 1–16 |
| `31`–`34` | `0x001F`–`0x0022` | 4 | `s16[4]` | `0.1 °C` | Cell temperatures 1–4 *(direct signed `0.1 °C`, **not** Kelvin!)* |
| `35` | `0x0023` | 1 | `s16` | `0.1 °C` | MOSFET temperature (invalid on `BMS-4820`) |
| `36` | `0x0024` | 1 | `s16` | `0.1 °C` | Environment temperature (invalid on `BMS-4820`) |
| `37`–`59` | `0x0025`–`0x003B` | 23 | — | — | Reserved |
| `150`–`159` | `0x0096`–`0x009F` | 10 | `char[20]` | ASCII | Version information (20 bytes, 2 chars per register) |
| `160`–`169` | `0x00A0`–`0x00A9` | 10 | `char[20]` | ASCII (RW) | Model SN (BMS manufacturer serial number, 20 bytes) |
| `170`–`179` | `0x00AA`–`0x00B3` | 10 | `char[20]` | ASCII (RW) | PACK SN (pack manufacturer serial number, 20 bytes) |

#### Protection parameter registers (`60`–`114`, Read/Write — dangerous to write)

| Register (dec) | Hex | Type | Unit / scale | Parameter |
|---|---|---|---|---|
| `60` / `61` / `62` / `63` | `0x003C`–`0x003F` | `u16`×3, `u8` | `1 mV` ×3, `0.1 s` | Pack OV: alarm, protection, release, delay (`1`–`255`) |
| `64` / `65` / `66` / `67` | `0x0040`–`0x0043` | `u16`×3, `u8` | `1 mV` ×3, `0.1 s` | Cell OV: alarm, protection, release, delay (`1`–`255`) |
| `68` / `69` / `70` / `71` | `0x0044`–`0x0047` | `u16`×3, `u8` | `1 mV` ×3, `0.1 s` | Pack UV: alarm, protection, release, delay (`1`–`255`) |
| `72` / `73` / `74` / `75` | `0x0048`–`0x004B` | `u16`×3, `u8` | `1 mV` ×3, `0.1 s` | Cell UV: alarm, protection, release, delay (`1`–`255`) |
| `76` / `77` / `78` | `0x004C`–`0x004E` | `u16`×2, `u8` | `1 A` ×2, `0.1 s` | Charging OC1: alarm, protection, delay (`1`–`255`) |
| `79` / `80` / `81` | `0x004F`–`0x0051` | `u16`×2, `u8` | `1 A` ×2, `0.1 s` | Discharging OC1: alarm, protection, delay (`1`–`255`) |
| `82` / `83` | `0x0052`–`0x0053` | `u16`, `u8` | `1 A`, `0.025 s` | Discharging OC2: protection, delay (`1`–`255` × 25 ms) |
| `84` / `85` / `86` | `0x0054`–`0x0056` | `s16`×3 | `0.1 °C` | Charging OT: alarm, protection, release |
| `87` / `88` / `89` | `0x0057`–`0x0059` | `s16`×3 | `0.1 °C` | Discharging OT: alarm, protection, release |
| `90` / `91` / `92` | `0x005A`–`0x005C` | `s16`×3 | `0.1 °C` | Charging UT: alarm, protection, release |
| `93` / `94` / `95` | `0x005D`–`0x005F` | `s16`×3 | `0.1 °C` | Discharging UT: alarm, protection, release |
| `96` / `97` / `98` | `0x0060`–`0x0062` | `s16`×3 | `0.1 °C` | MOSFET OT: alarm, protection, release |
| `99` / `100` / `101` | `0x0063`–`0x0065` | `s16`×3 | `0.1 °C` | Environment OT: alarm, protection, release |
| `102` / `103` / `104` | `0x0066`–`0x0068` | `s16`×3 | `0.1 °C` | Environment UT: alarm, protection, release |
| `105` / `106` | `0x0069`–`0x006A` | `u16`×2 | `1 mV` | Balance start cell voltage (`105`), balance start delta voltage (`106`) |
| `107` / `108` | `0x006B`–`0x006C` | `u16`×2 | `1 mV`, `1 mA` | Pack full-charge voltage (`107`), full-charge current (`108`) |
| `109` / `110` | `0x006D`–`0x006E` | `u16`×2 | `1 mV`, `1 min` | Cell sleep voltage (`109`), cell sleep delay time (`110`) |
| `111` | `0x006F` | `u8` (in `u16`) | `25 µs` | Short-circuit protection delay time (max `500 µs`) |
| `112` | `0x0070` | `u8` (in `u16`) | `1 %` | SOC low alarm threshold (`0`–`100 %`) |
| `113` / `114` | `0x0071`–`0x0072` | `u16`, `u8` | `1 A`, `0.025 s` | Charging OC2: protection (`113`), delay (`114`, `1`–`255` × 25 ms) |

#### Modbus alarm, protection, and status/fault bitmasks (registers `9`, `10`, `11`)

| Bit | Register `9` (`Warning flag`) | Register `10` (`Protection flag`) | Register `11` (`Status/Fault flag`) |
|---|---|---|---|
| **0** | Cell over-voltage alarm | Cell over-voltage protection | **Fault:** Charging MOSFET fault |
| **1** | Cell under-voltage alarm | Cell under-voltage protection | **Fault:** Discharging MOSFET fault |
| **2** | Pack over-voltage alarm | Pack over-voltage protection | **Fault:** Temperature sensor fault |
| **3** | Pack under-voltage alarm | Pack under-voltage protection | Reserved |
| **4** | Charging over-current alarm | Charging over-current protection | **Fault:** Battery cell fault |
| **5** | Discharging over-current alarm | Discharging over-current protection | **Fault:** Front-end sampling comm fault |
| **6** | Reserved | Short-circuit protection | Reserved |
| **7** | Reserved | Charger over-voltage protection | Reserved |
| **8** | Charging high-temp alarm (cell) | Charging high-temp protection (cell) | **Status:** Charging active (`1` = charging) |
| **9** | Discharging high-temp alarm (cell) | Discharging high-temp protection (cell) | **Status:** Discharging active (`1` = discharging) |
| **10** | Charging low-temp alarm (cell) | Charging low-temp protection (cell) | **Status:** Charging MOSFET is ON |
| **11** | Discharging low-temp alarm (cell) | Discharging low-temp protection (cell) | **Status:** Discharging MOSFET is ON |
| **12** | Environment high-temp alarm | MOSFET high-temp protection | **Status:** Charging current limiter is ON |
| **13** | Environment low-temp alarm | Environment high-temp protection | Reserved |
| **14** | MOSFET high-temp alarm | Environment low-temp protection | **Status:** Charger polarity inversed |
| **15** | SOC low alarm | Reserved (`syssi`: Battery Full) | **Status:** Heater is ON |

*(Note: in Register `83` of [`syssi/esphome-pace-bms` (`esp32-example.yaml`)](https://github.com/syssi/esphome-pace-bms/blob/main/esp32-example.yaml), the scaling lambda multiplies by `0.1` instead of `0.025`, whereas the vendor PDF specifies `0.025S` (`25 ms`), matching register `114`.)*

---

## Inverter CAN and RS485 Multi-Protocol Selection

PACE BMS units store selectable inverter protocol profiles for both the **CAN** port and the **RS485** inverter port. These can be read with `CID2 = 0xEB` (`~250046EB0000FD88\r`) and written with `CID2 = 0xEC` via `PbmsTools.exe` (`System Config -> Inverter Protocol`).

### Selectable protocol IDs (`CID2 = 0xEB` / `0xEC`, from `PbmsTools` V2.4)

| ID (hex / dec) | CAN bus protocol (`Byte 0`) | RS485 bus protocol (`Byte 1`) |
|---|---|---|
| `0xFF` (255) | Disabled / empty | Disabled / empty |
| `0x00` (0) | **PACE** (250 kbit/s, 29-bit J1939; see below) | **PACE Modbus** ([Modbus RTU V1.3](#pace-modbus-rtu-protocol-v13-rs485)) |
| `0x01` (1) | **Pylon** / Deye / CHNT / Livoltek / Megarevo / Sunsynk / Sungrow / Sol-Ark / SolarEdge ([Pylontech CAN](../PYLONTECH/README.md#low-voltage-can-protocol)) | **Pylon** / Deye / Bentterson ([Pylontech RS485](../PYLONTECH/README.md#rs485-protocol-v33)) |
| `0x02` (2) | Growatt / Sacolar | Growatt |
| `0x03` (3) | Victron (500 kbit/s, Pylontech/Victron CAN-bus BMS-Can) | Voltronic / EASun Power / MPP Solar |
| `0x04` (4) | Schneider / SE / SMA | Schneider / SE |
| `0x05` (5) | LuxPower | Phocos |
| `0x06` (6) | Sorotec / SRD | LuxPower |
| `0x07` (7) | SMA / Studer | Solar |
| `0x08` (8) | GoodWe | Lithium / SMARK |
| `0x09` (9) | Studer | EP / MSL |
| `0x0A` (10) | Sofar | RTU04 |
| `0x0B` (11) | [MUST](../MUST/README.md) / PV | LuxPower_V01 |
| `0x0C` (12) | Solis / Jinlang (JL) | LuxPower_V03 |
| `0x0D` (13) | DIDU / TBB | SRNE / WOW |
| `0x0E` (14) | Senergy / Aifu | Leoch |
| `0x0F` (15) | TBB | Pylon_F |
| `0x10` (16) | Pylon_V202 | Afore |
| `0x11` (17) | Growatt_V109 | UPS_AGXN |
| `0x12` (18) | [Must_V202](../MUST/README.md) | Orex_Sunpolo |
| `0x13` (19) | Afore | Xiongtao |
| `0x14` (20) | INVT / YWT | Rongke |
| `0x15` (21) | Fuji | Xinrui |
| `0x16` (22) | Sofar_V21003 | Eltek |
| `0x17` (23) | — | GT |
| `0x18` (24) | — | Leoch_V106 |

- Most hybrid inverters (Victron, Deye, Sunsynk, Sol-Ark, LuxPower, GoodWe, SMA, Solis) use the **Pylontech low-voltage CAN protocol** at **500 kbit/s** (11-bit IDs `0x351`, `0x355`, `0x356`, `0x359`, `0x35C`, `0x35E`, `0x305`, little-endian); see [PYLONTECH — Low-voltage CAN protocol](../PYLONTECH/README.md#low-voltage-can-protocol) for the full frame table.
- Some older SOK firmwares use a custom `CID2 = 0x99` sub-command (`0x0E` = Paceic V2.5, `0x0F` = Pylon/Deye CAN, `0x10` = Growatt CAN, `0x11` = LuxPower 485, `0x12` = Growatt 485, `0x13` = Pylon 485) to switch inverter protocols.

### Native PACE CAN protocol (`can_Pace = 0x00`, V1.0 `2016-12-16`)

> Status: 📄 Vendor doc (*PACEIC BMS-CAN Communication Protocol V1.0*, `2016-12-16`, copy in [`syssi/esphome-pace-bms/docs`](https://github.com/syssi/esphome-pace-bms/blob/main/docs/PACE-BMS-CAN-communication-protocol-20161216.pdf)).

When set to native `PACE` CAN (`0x00`), the BMS uses **CAN 2.0B (29-bit extended identifiers)** at **250 kbit/s** (150 ms timeout), big-endian (`BE`) payload words, polled by a host (`SA = 0x01`, broadcast `0x03`) targeting BMS node addresses `0x11 + DIP_address` (`0x11`, `0x12`, …):
- **29-bit ID layout:** `Bits 28..26` = Priority (3 bits) · `Bit 25` = Reserved (`0`) · `Bits 23..16` = `PF` (message type, 8 bits) · `Bits 15..8` = `PS` (target address, 8 bits) · `Bits 7..0` = `SA` (source address, 8 bits).
- **Message types (`PF`):**
  - `PF = 0x42` (Priority `4`): Cell voltages — `Bytes 0–1`: frame index `n` (`u16 BE`), `Bytes 2–7`: 3 cell voltages `(3n+1)..(3n+3)` in `mV` (`u16 BE` each).
  - `PF = 0x43` (Priority `4`): Temperatures — `Bytes 0–1`: frame index `n` (`u16 BE`), `Bytes 2–7`: 3 temperatures in `0.1 K` (`raw = °C × 10 + 2730`, `0x8000` = undefined; second-to-last is MOSFET, last is environment).
  - `PF = 0x44` (Priority `2`): System summary — `Bytes 0–1`: cell count (`u16 BE`), `Bytes 2–3`: pack voltage (`0.01 V`, `u16 BE`), `Bytes 4–5`: current (`0.1 A`, `s16 BE`), `Bytes 6–7`: cycle count (`u16 BE`).
  - `PF = 0x45` (Priority `1`): Status / alarms — `Byte 0`: Protect State 1, `Byte 1`: Protect State 2, `Byte 2`: Indicate State, `Byte 3`: Control State, `Byte 4`: Fault State, `Byte 5`: Reserved, `Byte 6`: Warn State 1, `Byte 7`: Warn State 2 (same bit definitions as `0x44` in YD/T 1363.3).
  - `PF = 0x46` (Priority `2`): Capacity — `Bytes 0–1`: design capacity (`10 mAh`), `Bytes 2–3`: remaining capacity (`10 mAh`), `Bytes 4–5`: full capacity (`10 mAh`), `Byte 6`: SOC (`%`), `Byte 7`: SOH (`%`).
  - `PF = 0x47` (Priority `5`): Cell balancing — `Bytes 0–1`: frame index `n` (`u16 BE`), `Bytes 2–7`: 48 cell balance bits.
  - `PF = 0x99` (Priority `1`, **dangerous control**): `Byte 0` command (`0x01` open charge MOS, `0x02` close charge MOS, `0x03` open discharge MOS, `0x04` close discharge MOS).

---

## Sources

- **PACE, *PACE RS485 Communication Protocol (`PACE-RS485-MS`)*, V1.0, 2018-06-15** — stored in [`PDFs/PACE-BMS-RS485-communication-protocol-20180615.pdf`](PDFs/PACE-BMS-RS485-communication-protocol-20180615.pdf) (from [`syssi/esphome-pace-bms/docs`](https://github.com/syssi/esphome-pace-bms/tree/main/docs)).
- **PACE, *PACE RS232 Communication Protocol (`PACE-RS232-TY16S`)*, V1.0, 2018-07-05** — stored in [`PDFs/PACE-BMS-RS232-communication-protocol-20180705.pdf`](PDFs/PACE-BMS-RS232-communication-protocol-20180705.pdf) (from [`syssi/esphome-pace-bms/docs`](https://github.com/syssi/esphome-pace-bms/tree/main/docs) and [`nkinnan/esphome-pace-bms/protocol_documentation`](https://github.com/nkinnan/esphome-pace-bms/tree/main/protocol_documentation/paceic/0x25)).
- **PACE, *PACE BMS Modbus Protocol for RS485*, V1.3, 2017-06-27** (prepared by YX.WANG) — stored in [`PDFs/PACE-BMS-Modbus-Protocol-for-RS485-V1.3-20170627.pdf`](PDFs/PACE-BMS-Modbus-Protocol-for-RS485-V1.3-20170627.pdf) (from [`syssi/esphome-pace-bms/docs`](https://github.com/syssi/esphome-pace-bms/tree/main/docs)).
- **PACEIC, *BMS-CAN Communication Protocol*, V1.0, 2016-12-16** — copy in [`syssi/esphome-pace-bms/docs/PACE-BMS-CAN-communication-protocol-20161216.pdf`](https://github.com/syssi/esphome-pace-bms/blob/main/docs/PACE-BMS-CAN-communication-protocol-20161216.pdf).
- **[`nkinnan/esphome-pace-bms`](https://github.com/nkinnan/esphome-pace-bms)** — `README.md`, `components/pace_bms/pace_bms_protocol_v25.h`, `pace_bms_protocol_v25.cpp`, `pace_bms_protocol_v20.h` (`PbmsTools` V2.4 command decodes, captured frames, firmware variants, and inverter protocol list).
- **[`syssi/esphome-pace-bms`](https://github.com/syssi/esphome-pace-bms)** — `README.md`, `esp32-example.yaml`, `docs/pbmstools-communication.md` (Modbus RTU register map, RJ45 pinout, supported battery models).
- **[`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery)** — `dbus-serialbattery/bms/pace.py` (Venus OS driver implementation for PACE RS232/RS485 V2.5).

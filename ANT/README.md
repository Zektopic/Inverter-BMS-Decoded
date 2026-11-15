# ANT BMS

ANT BMS boards (8S–32S smart BMS units) use two distinct binary protocols over their 4-pin TTL UART port, Bluetooth Classic (RFCOMM/SPP on older boards), and BLE (service `0xFFE0`, characteristic `0xFFE1`):

1. **[Legacy protocol (pre-2021 / V1)](#legacy-protocol-pre-2021)** — **big-endian**, 6-byte requests (`5A 5A ...` / `DB DB ...` / `A5 A5 ...`) and a fixed **140-byte** status response starting with `AA 55 AA FF` and ending with a 16-bit byte-sum checksum.
2. **[2021+ protocol (`7E A1` framing)](#2021-protocol-7e-a1-framing)** — **little-endian**, variable-length frames framed by `7E A1 ... AA 55` with a **CRC-16/MODBUS** checksum. (Note: some 2021+ units still respond to legacy `5A 5A` / `DB DB` status requests on UART for backward compatibility.)

---

## Legacy protocol (pre-2021)

> Status: 📄 Vendor doc — transcribed from [`ANT_communication_protocol_EN.1.pdf`](PDFs/ANT_communication_protocol_EN.1.pdf) (`AtyParam.java` / `AtyParam2.java`), cross-checked against [`syssi/esphome-ant-bms`](https://github.com/syssi/esphome-ant-bms) (`ant_bms_old`, `ant_bms_old_ble`) and [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) (`ant.py`, `ant_leg_bms.py`).

### Summary

| | |
|---|---|
| Products | ANT 8S–32S Smart BMS (e.g. ANT 16S 100A, ANT 24S 200A, ANT 32S 300A, BMS-ANT24C; also supported as a fallback on some 2021+ models) |
| Physical layer | TTL UART (3.3 V) on 4-pin JST 1.25 mm `Comm` connector (`VCC`, `GND`, `RXD`, `TXD` — **3.3 V `VCC` must be connected** or the BMS UART optocoupler/transceiver does not respond); Bluetooth Classic SPP or BLE (service `0xFFE0`, RX/TX notify/write characteristic `0xFFE1`, handle `0x10`) |
| Settings | **19200 8N1** in practice (`syssi/esphome-ant-bms`); vendor PDF states **9600 8N1** (see [Errata](#errata-and-disagreements)) |
| Addressing | Point-to-point (no device address in frame) |
| Roles | Polled: host sends a 6-byte request; BMS replies with a 140-byte status frame |
| Byte order | **Big-endian (`BE`)** for all multi-byte values and the 16-bit response checksum |

### Framing

#### Request frame (6 bytes, host → BMS)

| Offset | Size | Value / Type | Description |
|---|---|---|---|
| `0` | 2 | `5A 5A`, `DB DB`, or `A5 A5` | Command header: `5A 5A` = read via UART, `DB DB` = read via BLE (also works on UART), `A5 A5` = write register / control command |
| `2` | 1 | `u8` | Register / command address (`0x00` for status poll) |
| `3` | 2 | `u16 BE` | Data / value (`0x0001` or `0x0000` for status poll) |
| `5` | 1 | `u8` | Checksum: `(byte[2] + byte[3] + byte[4]) & 0xFF` |

**Worked request checksum examples:**

- **BLE / general status poll:** `DB DB 00 00 00 00`
  $$\text{checksum} = (0\text{x}00 + 0\text{x}00 + 0\text{x}00) \bmod 256 = 0\text{x}00$$
- **UART status poll (`esphome-ant-bms`):** `5A 5A 00 00 01 01`
  $$\text{checksum} = (0\text{x}00 + 0\text{x}00 + 0\text{x}01) \bmod 256 = 0\text{x}01$$
- **Write register example (turn Charge MOSFET on, address `0xFA`, value `0x0001`):** `A5 A5 FA 00 01 FB`
  $$\text{checksum} = (0\text{xFA} + 0\text{x}00 + 0\text{x}01) \bmod 256 = 0\text{xFB}$$

#### Status response frame (140 bytes, BMS → host)

| Offset | Size | Value / Type | Description |
|---|---|---|---|
| `0` | 4 | `AA 55 AA FF` | Start of frame (`AA 55 AA`) + frame type `0xFF` (status) |
| `4` | 134 | payload | Status data (offsets `4`..`137`, big-endian) |
| `138` | 2 | `u16 BE` | Checksum: 16-bit unsigned sum of payload bytes `4`..`137` (`sum(frame[4:138]) & 0xFFFF`) |

**Worked response checksum example** (from [`docs/pdus/model2019-req-dbdb00000000.txt`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/pdus/model2019-req-dbdb00000000.txt), 14S pack):

Summing the 134 payload bytes at offsets `4`..`137` (`01 + E8 + 0D + AA + ... + 40 + 01`) yields `5620` (`0x15F4`), stored big-endian at offsets `138`–`139` as `15 F4`.

### Requests and responses

| Command | Direction | Frame | Notes |
|---|---|---|---|
| Poll status (BLE / UART) | Host → BMS | `DB DB 00 00 00 00` | Used by BLE app, `ant_bms_old_ble`, and `dbus-serialbattery` (`ant.py`) |
| Poll status (UART) | Host → BMS | `5A 5A 00 00 01 01` (or `5A 5A 00 00 00 00`) | Used by `ant_bms_old` over UART (`ANT_communication_protocol_EN.1.pdf` lists `5A 5A 00 00 00 00`) |
| Status reply | BMS → Host | `AA 55 AA FF <134 bytes> <CHK_HI> <CHK_LO>` | Always 140 bytes; unused cell slots are `00 00`, disconnected temperature slots are `FF D8` (`-40 °C`) |
| Write parameter / control | Host → BMS | `A5 A5 <ADDR> <VAL_HI> <VAL_LO> <CHK>` | Followed by apply frame `A5 A5 FF 00 00 FF` (see [Control commands](#control-commands-dangerous)) |

### Status frame (`AA 55 AA FF`, 140 bytes)

All multi-byte fields are **big-endian (`BE`)**. The frame always reserves space for 32 cell voltages and 6 temperature sensors regardless of the actual string count in byte `123`.

| Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0` | 4 | `u8[4]` | — | Frame header (`AA 55 AA FF`) |
| `4` | 2 | `u16 BE` | 0.1 V | Total pack voltage |
| `6` | 2 | `u16 BE` | 0.001 V | Cell 1 voltage |
| `8` | 2 | `u16 BE` | 0.001 V | Cell 2 voltage |
| … | … | `u16 BE` | 0.001 V | Cells 3..31 (`offset = 6 + (cell - 1) × 2`) |
| `68` | 2 | `u16 BE` | 0.001 V | Cell 32 voltage (unused cells above byte `123` are `0x0000`) |
| `70` | 4 | `s32 BE` | 0.1 A | Pack current (see [Errata](#errata-and-disagreements) on sign convention) |
| `74` | 1 | `u8` | 1 % | State of charge (SOC, 0–100 %) |
| `75` | 4 | `u32 BE` | 0.000001 Ah (1 µAh) | Physical / nominal battery capacity setting (some BMS report `0`; see [Errata](#errata-and-disagreements)) |
| `79` | 4 | `u32 BE` | 0.000001 Ah (1 µAh) | Remaining battery capacity |
| `83` | 4 | `u32 BE` | 0.001 Ah (1 mAh) | Total accumulated cycle capacity |
| `87` | 4 | `u32 BE` | 1 s | Uptime / accumulated runtime since boot |
| `91` | 2 | `s16 BE` | 1 °C | Temperature sensor 1 (`0xFFD8` = `-40 °C` when disconnected) |
| `93` | 2 | `s16 BE` | 1 °C | Temperature sensor 2 |
| `95` | 2 | `s16 BE` | 1 °C | Temperature sensor 3 |
| `97` | 2 | `s16 BE` | 1 °C | Temperature sensor 4 |
| `99` | 2 | `s16 BE` | 1 °C | Temperature sensor 5 |
| `101` | 2 | `s16 BE` | 1 °C | Temperature sensor 6 |
| `103` | 1 | `u8` | enum | Charge MOSFET status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `104` | 1 | `u8` | enum | Discharge MOSFET status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `105` | 1 | `u8` | enum | Balancer status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `106` | 2 | `u16 BE` | 1 mm | Tire length (e-bike speedometer setting) |
| `108` | 2 | `u16 BE` | 1 | Pulses per wheel revolution (`Pulses number (week)`) |
| `110` | 1 | `u8` | — | Relay switch status |
| `111` | 4 | `s32 BE` | 1 W | Current power |
| `115` | 1 | `u8` | 1-based index | Cell number with highest voltage (`1`..`32`) |
| `116` | 2 | `u16 BE` | 0.001 V | Maximum cell voltage |
| `118` | 1 | `u8` | 1-based index | Cell number with lowest voltage (`1`..`32`) |
| `119` | 2 | `u16 BE` | 0.001 V | Minimum cell voltage |
| `121` | 2 | `u16 BE` | 0.001 V | Average cell voltage |
| `123` | 1 | `u8` | 1 S | Effective cell count (number of battery strings, max 32) |
| `124` | 2 | `u16 BE` | 0.1 V | Discharge MOSFET drain–source (`D-S`) voltage |
| `126` | 2 | `u16 BE` | 0.1 V | Discharge MOSFET gate drive voltage |
| `128` | 2 | `u16 BE` | 0.1 V | Charge MOSFET gate drive voltage |
| `130` | 2 | `u16 BE` | raw | Current-sense comparator initial value at 0 A |
| `132` | 4 | `u32 BE` | bitmask | Cell balancing bitmask (bit 0 = Cell 1 … bit 31 = Cell 32; `1` = balancing active) |
| `136` | 2 | `u16 BE` | bitfield | System log word: bits `0..4` = MOSFET status, bits `5..9` = cell number, bits `10..14` = sequence number, bit `15` = direction (`1` = discharge, `0` = charge) |
| `138` | 2 | `u16 BE` | — | Checksum: `sum(byte[4..137]) & 0xFFFF` |

---

## 2021+ protocol (`7E A1` framing)

> Status: 🌐 Community — decoded in [`syssi/esphome-ant-bms`](https://github.com/syssi/esphome-ant-bms) (`ant_bms`, `ant_bms_ble`) and [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) (`ext/aiobmsble/bms/ant_bms.py`) from BLE HCI snoop logs (`ANT-BLE16ZMUB`, `ANT-BLE24BHUB`) and UART captures.

### Summary

| | |
|---|---|
| Products | 2021+ ANT Smart BMS: `ANT-BLE16ZMUB` (`16ZMUB00-220501A`, `16ZMB0TB120A`), `ANT-BLE24BHUB` (`24BHUB00-211026A`), `ANT-BLE04DMUB` (4S 120A), `ANT@BLE22AAUB` (22S 130A), `16ZMB-TB-7-16S-300A`, `24AHA-TB-24S-200A` |
| Physical layer | TTL UART (3.3 V on 4-pin JST 1.25 mm `Comm` connector; connect `VCC` 3.3 V, `GND`, `RXD`, `TXD`) and BLE (service UUID `0xFFE0`, characteristic UUID `0xFFE1` on handle `0x10` for notify + write-without-response) |
| Settings | **19200 8N1** (UART) |
| Addressing | Point-to-point (`0xA1` fixed second preamble byte) |
| Roles | Polled: host sends a 10-byte read request; BMS replies with a variable-length frame |
| Byte order | **Little-endian (`LE`)** for all addresses, multi-byte fields, and the 16-bit CRC-16/MODBUS |

### Framing

Every request and response uses the same envelope:

| Offset | Size | Value / Type | Description |
|---|---|---|---|
| `0` | 2 | `7E A1` | Start of frame (`0x7E` SOF + `0xA1` protocol/address byte) |
| `2` | 1 | `u8` | Function / command code (responses typically set bit 4 or bit 5 relative to the request: `0x01 → 0x11`, `0x02 → 0x12`, `0x04 → 0x14`, `0x22 → 0x42`, `0x23 → 0x43`, `0x51 → 0x61`) |
| `3` | 2 | `u16 LE` | Register / start address (little-endian) |
| `5` | 1 | `u8` | Data length $N$ (in **read requests**, the requested byte count; in **write requests** and **responses**, the payload byte count $N$ that follows; in **command `0x51`**, `0x00` with 0 payload bytes) |
| `6` | $N$ | `u8[N]` | Payload (0 bytes for read requests and `0x51` control commands; $N$ bytes for write requests and responses) |
| `6 + N` | 2 | `u16 LE` | **CRC-16/MODBUS** (poly `0x8005` reflected as `0xA001`, init `0xFFFF`, no final XOR) computed over **bytes `1` through `5 + N`** (i.e. starting at `0xA1`, excluding byte `0` (`0x7E`), the CRC itself, and the trailer) |
| `8 + N` | 2 | `AA 55` | End of frame (for `0x12` and `0x42` BLE responses, 6 extra trailer bytes appear between the CRC and `AA 55` — see [Errata](#errata-and-disagreements)) |

**Worked CRC-16/MODBUS examples (verified in Python):**

1. **Status request (`0x01`):** `7E A1 01 00 00 BE 18 55 AA 55`
   - Bytes covered by CRC (`offset 1..5`): `A1 01 00 00 BE`
   - `CRC-16/MODBUS("A1 01 00 00 BE") = 0x5518` → transmitted little-endian as `18 55`, followed by `AA 55`.
2. **Device info request (`0x02`, address `0x026C`, length `0x20`):** `7E A1 02 6C 02 20 58 C4 AA 55`
   - Bytes covered by CRC (`offset 1..5`): `A1 02 6C 02 20`
   - `CRC-16/MODBUS("A1 02 6C 02 20") = 0xC458` → transmitted little-endian as `58 C4`, followed by `AA 55`.
3. **Read setting `CellOvervoltageProtection` (`0x02`, address `0x0000`, length `0x02`):**
   - Request: `7E A1 02 00 00 02 19 A0 AA 55` (`CRC-16/MODBUS("A1 02 00 00 02") = 0xA019` → `19 A0`)
   - Response: `7E A1 12 00 00 02 36 10 1F 14 AA 55` (`0x1036` = `4150` → `4.150 V`; `CRC-16/MODBUS("A1 12 00 00 02 36 10") = 0x141F` → `1F 14`).

### Commands and frame types

| Request `cmd` | Response `func` | Address (`u16 LE`) | `data_len` | Description |
|---|---|---|---|---|
| `0x01` | `0x11` | `0x0000` | Req: `0xBE` / Resp: `102 + 2×(cells + temps)` | Read live status frame (e.g. `data_len = 0x8E` (142 payload bytes, 152 bytes total) when `cells + temps = 18`) |
| `0x02` | `0x12` | `0x026C` | `0x20` (32) | Read device info (hardware model + software version strings) |
| `0x02` | `0x12` | `0x0000`..`0x017E` | `0x02`, `0x04`, or `0x38` | Read one settings register (`2` or `4` bytes) or a 56-byte settings block (`0x38`) |
| `0x04` | `0x14` | `0x0000` | `0x0C` (12) | Read permission / challenge data |
| `0x22` | `0x42` | `0x0000`..`0x017E` | `0x02` or `0x04` | Write settings register (followed by command `0x51` address `0x0007` to commit) |
| `0x23` | `0x43` | `0x016A` (`6A 01`) | `0x0C` (or `len(pwd)`) | Authenticate with ASCII password (default `"123456789abc"`) |
| `0x51` | `0x61` | `0x0001`..`0x002C` | Req: `0x00` / Resp: `0x02` | Execute control / switch / button command |

### Status response (`0x11`, variable length)

In the `0x11` status frame, the number of temperature sensors $T$ (byte `8`, typically 2–4, max 6) and the number of cells $C$ (byte `9`, max 32) determine the dynamic byte offset:

$$\text{offset} = 2 \times (C + T)$$

All multi-byte values are **little-endian (`LE`)**.

| Byte offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0` | 2 | `u8[2]` | — | Start of frame (`7E A1`) |
| `2` | 1 | `u8` | — | Function (`0x11`) |
| `3` | 2 | `u16 LE` | — | Address (`0x0000`) |
| `5` | 1 | `u8` | bytes | Payload length (`102 + offset`; e.g. `0x8E` = 142 bytes when $C + T = 18$) |
| `6` | 1 | `u8` | — | Permissions level (e.g. `0x05`) |
| `7` | 1 | `u8` | enum | Battery status: `0` Unknown, `1` Idle, `2` Charge, `3` Discharge, `4` Standby, `5` Error |
| `8` | 1 | `u8` | count | Number of external temperature sensors $T$ (max 6) |
| `9` | 1 | `u8` | count | Number of cells $C$ (battery strings, max 32) |
| `10` | 8 | `u64 LE` | bitmask | Protection bitmask |
| `18` | 8 | `u64 LE` | bitmask | Warning bitmask |
| `26` | 8 | `u64 LE` | bitmask | Balancing / status bitmask *(unverified)* |
| `34` | $2 \times C$ | `u16 LE[C]` | 0.001 V | Cell voltages `1`..$C$ (`34 + 2 × (i - 1)`) |
| `34 + 2C` | $2 \times T$ | `s16 LE[T]` | 1 °C | External temperature sensors `1`..$T$ (`0xFFD8` = `-40 °C` when disconnected) |
| `34 + offset` | 2 | `s16 LE` | 1 °C | MOSFET temperature |
| `36 + offset` | 2 | `s16 LE` | 1 °C | Balancer temperature |
| `38 + offset` | 2 | `u16 LE` | 0.01 V | Total pack voltage |
| `40 + offset` | 2 | `s16 LE` | 0.1 A | Pack current |
| `42 + offset` | 2 | `u16 LE` | 1 % | State of charge (SOC) |
| `44 + offset` | 2 | `u16 LE` | 1 % | State of health (SOH) |
| `46 + offset` | 1 | `u8` | enum | Charge MOSFET status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `47 + offset` | 1 | `u8` | enum | Discharge MOSFET status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `48 + offset` | 1 | `u8` | enum | Balancer status code (see [Alarms and status codes](#alarms-and-status-codes)) |
| `49 + offset` | 1 | `u8` | — | Reserved (`0x00`) |
| `50 + offset` | 4 | `u32 LE` | 0.000001 Ah (1 µAh) | Nominal / design battery capacity setting |
| `54 + offset` | 4 | `u32 LE` | 0.000001 Ah (1 µAh) | Remaining battery capacity |
| `58 + offset` | 4 | `u32 LE` | 0.001 Ah (1 mAh) | Total battery cycle capacity |
| `62 + offset` | 4 | `s32 LE` | 1 W | Current power |
| `66 + offset` | 4 | `u32 LE` | 1 s | Total runtime / uptime |
| `70 + offset` | 4 | `u32 LE` | bitmask | Balanced cell bitmask (bit 0 = Cell 1 … bit 31 = Cell 32) |
| `74 + offset` | 2 | `u16 LE` | 0.001 V | Maximum cell voltage |
| `76 + offset` | 2 | `u16 LE` | 1-based index | Cell number with maximum voltage |
| `78 + offset` | 2 | `u16 LE` | 0.001 V | Minimum cell voltage |
| `80 + offset` | 2 | `u16 LE` | 1-based index | Cell number with minimum voltage |
| `82 + offset` | 2 | `u16 LE` | 0.001 V | Delta cell voltage (max − min) |
| `84 + offset` | 2 | `u16 LE` | 0.001 V | Average cell voltage |
| `86 + offset` | 2 | `u16 LE` | 0.1 V *(unverified)* | Discharge MOSFET drain–source (`D-S`) voltage |
| `88 + offset` | 2 | `u16 LE` | 0.1 V *(unverified)* | Discharge MOSFET drive voltage |
| `90 + offset` | 2 | `u16 LE` | 0.1 V *(unverified)* | Charge MOSFET drive voltage |
| `92 + offset` | 2 | `u16 LE` | raw | Comparator reference (`F40com`) |
| `94 + offset` | 2 | `u16 LE` | enum | Battery chemistry type: `0xFAF1` Ternary Lithium, `0xFAF2` LiFePO4, `0xFAF3` Lithium Titanate (LTO), `0xFAF4` Custom |
| `96 + offset` | 4 | `u32 LE` | 0.001 Ah (1 mAh) | Total accumulated discharging capacity |
| `100 + offset` | 4 | `u32 LE` | 0.001 Ah (1 mAh) | Total accumulated charging capacity |
| `104 + offset` | 4 | `u32 LE` | 1 s | Total accumulated discharging time |
| `108 + offset` | 4 | `u32 LE` | 1 s | Total accumulated charging time |
| `112 + offset` | 2 | `u16 LE` | — | CRC-16/MODBUS over bytes `1`..`111 + offset` |
| `114 + offset` | 2 | `u8[2]` | — | End of frame (`AA 55`) |

### Device info response (`0x12`, address `0x026C`)

Requested with `7E A1 02 6C 02 20 58 C4 AA 55`:

| Byte offset | Size | Type | Description |
|---|---|---|---|
| `0` | 6 | header | `7E A1 12 6C 02 20` (`func = 0x12`, `addr = 0x026C`, `data_len = 0x20`) |
| `6` | 16 | `char[16]` | Hardware version / model, null-padded ASCII (e.g. `"16ZM"`) |
| `22` | 16 | `char[16]` | Software version, null-padded ASCII (e.g. `"16ZMUB00-211026A"`) |
| `38` | 2 | `u16 LE` | CRC-16/MODBUS over bytes `1`..`37` |
| `40` | 6 | `u8[6]` | Extra trailer present on BLE (`FF 0B 00 00 41 F2`; omitted on standard 42-byte frames — see [Errata](#errata-and-disagreements)) |
| `46` (or `40`) | 2 | `u8[2]` | End of frame (`AA 55`) |

### Settings registers (command `0x02` read / `0x22` write)

All settings registers are `u16 LE` (`data_len = 0x02`), except `0x00A2`, `0x00A6`, and `0x00AA` which are `u32 LE` (`data_len = 0x04`):

| Address | Size | Type | Unit / scale | Register name (`esphome-ant-bms`) |
|---|---|---|---|---|
| `0x0000` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageProtection` |
| `0x0002` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageRecovery` |
| `0x0004` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageProtectionL2` |
| `0x0006` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageRecoveryL2` |
| `0x0008` | 2 | `u16 LE` | 0.1 V | `PackOvervoltageProtection` |
| `0x000A` | 2 | `u16 LE` | 0.1 V | `PackOvervoltageRecovery` |
| `0x000C` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageProtection` |
| `0x000E` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageRecovery` |
| `0x0010` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageProtectionL2` |
| `0x0012` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageRecoveryL2` |
| `0x0014` | 2 | `u16 LE` | 0.1 V | `PackUndervoltageProtection` |
| `0x0016` | 2 | `u16 LE` | 0.1 V | `PackUndervoltageRecovery` |
| `0x0018` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceProtection` |
| `0x001A` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceRecovery` |
| `0x0020` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageWarning` |
| `0x0022` | 2 | `u16 LE` | 0.001 V | `CellOvervoltageWarningRecovery` |
| `0x0024` | 2 | `u16 LE` | 0.1 V | `PackOvervoltageWarning` |
| `0x0026` | 2 | `u16 LE` | 0.1 V | `PackOvervoltageWarningRecovery` |
| `0x0028` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageWarning` |
| `0x002A` | 2 | `u16 LE` | 0.001 V | `CellUndervoltageWarningRecovery` |
| `0x002C` | 2 | `u16 LE` | 0.1 V | `PackUndervoltageWarning` |
| `0x002E` | 2 | `u16 LE` | 0.1 V | `PackUndervoltageWarningRecovery` |
| `0x0030` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceWarning` |
| `0x0032` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceWarningRecovery` |
| `0x0068` | 2 | `u16 LE` | 0.1 A | `ChargeOvercurrentProtection` |
| `0x006A` | 2 | `u16 LE` | 1 s | `ChargeOvercurrentProtectionDelay` |
| `0x006C` | 2 | `u16 LE` | 0.1 A | `DischargeOvercurrentProtection` |
| `0x006E` | 2 | `u16 LE` | 1 s | `DischargeOvercurrentProtectionDelay` |
| `0x0070` | 2 | `u16 LE` | 0.1 A | `DischargeOvercurrentProtectionL2` |
| `0x0072` | 2 | `u16 LE` | 1 ms | `DischargeOvercurrentProtectionDelayL2` |
| `0x0074` | 2 | `u16 LE` | 1 A | `ShortCircuitProtection` |
| `0x0076` | 2 | `u16 LE` | 1 µs | `ShortCircuitProtectionDelay` |
| `0x007C` | 2 | `u16 LE` | 0.1 A | `ChargeOvercurrentWarning` |
| `0x007E` | 2 | `u16 LE` | 0.1 A | `ChargeOvercurrentWarningRecovery` |
| `0x0080` | 2 | `u16 LE` | 0.1 A | `DischargeOvercurrentWarning` |
| `0x0082` | 2 | `u16 LE` | 0.1 A | `DischargeOvercurrentWarningRecovery` |
| `0x0084` | 2 | `u16 LE` | 1 % | `SOCLowLevel1Warning` |
| `0x0086` | 2 | `u16 LE` | 1 % | `SOCLowLevel2Warning` |
| `0x008C` | 2 | `u16 LE` | 0.001 V | `CellBalancingVoltage` |
| `0x008E` | 2 | `u16 LE` | 0.001 V | `CellBalancingStartVoltage` |
| `0x0090` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceBalancingOn` |
| `0x0092` | 2 | `u16 LE` | 0.001 V | `CellVoltageDifferenceBalancingOff` |
| `0x0094` | 2 | `u16 LE` | 1 mA | `BalancingCurrent` |
| `0x0096` | 2 | `u16 LE` | 0.1 A | `BalancingChargingCurrent` |
| `0x0098` | 2 | `u16 LE` | enum | `CellType` |
| `0x009A` | 2 | `u16 LE` | 1 S | `CellNumber` |
| `0x009C` | 2 | `u16 LE` | 1 mΩ | `CellInternalResistanceCalibration` |
| `0x009E` | 2 | `u16 LE` | 0.001 V | `ShutdownVoltage` |
| `0x00A0` | 2 | `u16 LE` | 0.1 A | `RequestChargeCurrent` |
| `0x00A2` | 4 | `u32 LE` | 1 Ah *(or µAh)* | `NominalCapacity` |
| `0x00A6` | 4 | `u32 LE` | 1 Ah *(or µAh)* | `RemainingCapacity` |
| `0x00AA` | 4 | `u32 LE` | 1 Ah *(or mAh)* | `TotalCycleCapacity` |
| `0x00C4` | 2 | `u16 LE` | enum | `StateOfChargeMethod` |
| `0x017A` | 2 | `u16 LE` | 1 mm | `TireLength` |
| `0x017C` | 2 | `u16 LE` | 1 | `PulseValue` |
| `0x017E` | 2 | `u16 LE` | 1 | `SecondaryModuleNum` |

---

## Alarms and status codes

Both the Legacy protocol (bytes `103`–`105`) and the 2021+ protocol (bytes `46 + offset` to `48 + offset`) report MOSFET and balancer states as single-byte enumeration codes. Values `0x00`..`0x0F` are defined in the vendor PDF; values `0x10`..`0x14` were added in the 2021+ protocol (`esphome-ant-bms`).

| Code | Charge MOSFET status (`103` / `46+off`) | Discharge MOSFET status (`104` / `47+off`) | Balancer status (`105` / `48+off`) |
|---|---|---|---|
| `0x00` | Off | Off | Off |
| `0x01` | On | On | Exceeding limit trigger balance |
| `0x02` | Cell overcharge protection | Cell overdischarge protection | Charge voltage-difference balance |
| `0x03` | Overcurrent protection | Overcurrent protection | Balance over-temperature |
| `0x04` | Battery full | Level-2 overcurrent *(2021+)* | Automatic balance |
| `0x05` | Pack overvoltage | Pack undervoltage | — |
| `0x06` | Battery over-temperature | Battery over-temperature | — |
| `0x07` | MOSFET over-temperature | MOSFET over-temperature | — |
| `0x08` | Abnormal current | Abnormal current | — |
| `0x09` | Balanced string dropped (cell wire open) | Balanced string dropped (cell wire open) | — |
| `0x0A` | Motherboard over-temperature | Motherboard over-temperature | Motherboard over-temperature |
| `0x0B` | Reserved | Charge MOSFET on | — |
| `0x0C` | Open failed *(2021+)* | Short-circuit protection | — |
| `0x0D` | Discharge MOSFET abnormality | Discharge MOSFET abnormality | — |
| `0x0E` | Waiting *(2021+)* | Start exception *(legacy)* / Open failed *(2021+)* | — |
| `0x0F` | Manually turned off | Manually turned off | — |
| `0x10` | Level-2 overvoltage *(2021+)* | Level-2 undervoltage *(2021+)* | — |
| `0x11` | Low-temperature protection *(2021+)* | Low-temperature protection *(2021+)* | — |
| `0x12` | Voltage difference exceeded *(2021+)* | Voltage difference exceeded *(2021+)* | — |
| `0x13` | Reserved *(2021+)* | Self-detect error *(2021+)* | — |
| `0x14` | Self-detect error *(2021+)* | — | — |

---

## Control commands (dangerous)

> [!CAUTION]
> Writing parameters or sending control commands can switch high-current charge/discharge MOSFETs, zero the current calibration while current is flowing, reset protection thresholds to defaults for a different cell chemistry (e.g. LiFePO4 vs NMC/LTO), or power down the BMS completely. Always prefer read-only monitoring.

### Legacy protocol (`A5 A5 <ADDR> <VAL_HI> <VAL_LO> <CHK>`)

Write operations send an optional 8-character ASCII password Across registers `0xF1`–`0xF4` (2 characters per register), followed by the target register write, and finally `A5 A5 FF 00 00 FF` (`0xFF` = Apply):

- **Control addresses** (`ANT_communication_protocol_EN.1.pdf` p. 3, `klotztech/VBMS`, `esphome-ant-bms/components/ant_bms_old`):
  - `0xEF` (239): Request firmware upgrade (`value = 0x1A01`)
  - `0xF0` (240): Apply Lithium Titanate (LTO) default settings (`0x0000`)
  - `0xF6` (246): Change/randomize Bluetooth MAC address (`0x0000`)
  - `0xF7` (247): **BMS power off / shutdown** (`0x0000`)
  - `0xF8` (248): **Current zero calibration** (`0x0000`)
  - `0xF9` (249): **Discharge MOSFET switch** (`0x0001` = on, `0x0000` = off)
  - `0xFA` (250): **Charge MOSFET switch** (`0x0001` = on, `0x0000` = off)
  - `0xFB` (251): **Apply LiFePO4 default settings** (`0x0000`)
  - `0xFC` (252): Toggle automatic cell balancing (`0x0000`)
  - `0xFD` (253): **Factory reset** (`0x0000`)
  - `0xFE` (254): **Reboot BMS** (`0x0000`)
  - `0xFF` (255): **Apply / commit settings** (`0x0000`)
- **Parameter addresses `0x01`–`0x64` (1–100):** cell/pack voltage thresholds (`1`–`8`), overcurrent protection and delays (`9`–`12`), balancing limits (`13`–`16`), current sensor range and short-circuit settings (`18`–`21`), string count (`24`), temperature protections (`25`–`30`), capacities (`31`–`36`). See [`ANT_communication_protocol_EN.1.pdf`](PDFs/ANT_communication_protocol_EN.1.pdf) page 4 for the full list.

### 2021+ protocol (command `0x51`, `7E A1 51 <ADDR_LO> <ADDR_HI> 00 <CRC_LO> <CRC_HI> AA 55`)

Before sending a `0x51` control command or `0x22` register write, the host authenticates with command `0x23` at address `0x016A` (`6A 01`) carrying the 12-byte ASCII password (default `"123456789abc"`). Control sub-addresses for command `0x51` (`esphome-ant-bms`):

| Address | Action | Address | Action |
|---|---|---|---|
| `0x0001` | Discharge MOSFET **off** | `0x0010` | Bluetooth initialization |
| `0x0003` | Discharge MOSFET **on** | `0x001C` | Bluetooth **off** |
| `0x0004` | Charge MOSFET **off** | `0x001D` | Bluetooth **on** |
| `0x0006` | Charge MOSFET **on** | `0x001E` | Buzzer **on** |
| `0x0007` | Apply / commit written register (`0x22`) | `0x001F` | Buzzer **off** |
| `0x0008` | **Current zero calibration** | `0x0020` | Clear discharge cycle Ah |
| `0x0009` | **Restart BMS** | `0x0021` | Clear charge cycle Ah |
| `0x000B` | **Shutdown / power off BMS** | `0x0022` | Clear discharge time |
| `0x000C` | **Factory reset** | `0x0023` | Clear charge time |
| `0x000D` | Manual balancer **on** | `0x0024` | Clear running time |
| `0x000E` | Manual balancer **off** | `0x0025` | Clear protect time |
| `0x000F` | Clear system log | `0x002A` / `0x002C` | Reset hardware / Save customer data |

---

## Captured frames

### Legacy 140-byte status frame (14S, 170 Ah pack)

Source: [`syssi/esphome-ant-bms/docs/pdus/model2019-req-dbdb00000000.txt`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/pdus/model2019-req-dbdb00000000.txt) (also used in `tests/components/ant_bms_old_ble/frames_14s.h`).

```
Request:
DB DB 00 00 00 00

Response (140 bytes):
AA 55 AA FF 01 E8 0D AA 0D 9C 0D A4 0D 8E 0D 9C 0D 90 0D B4 0D 97 0D B5 0D B5
0D A8 0D 91 0D 9E 0D 8C 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 50 29 0A 21 FE
80 04 19 58 93 00 A9 84 0F 00 FA 08 31 00 16 00 15 00 15 00 15 00 15 00 15 01
01 00 03 E8 00 17 00 00 00 01 86 09 0D B5 0E 0D 8C 0D 9F 0E 00 00 00 70 00 6B
02 AC 00 00 00 00 40 01 15 F4
```

Decoded highlights:
- Total voltage: `0x01E8` = `48.8 V`; Cell count (`123`): `0x0E` = `14S`
- Cells 1–14: `3.498 V`, `3.484 V`, `3.492 V`, `3.470 V`, `3.484 V`, `3.472 V`, `3.508 V`, `3.479 V`, `3.509 V`, `3.509 V`, `3.496 V`, `3.473 V`, `3.486 V`, `3.468 V` (Max: cell 9 at `3.509 V`, Min: cell 14 at `3.468 V`, Avg: `3.487 V`)
- Current (`70..73`): `0x00000050` (`8.0 A`), Power (`111..114`): `0x00000186` (`390 W`), SOC (`74`): `0x29` (`41 %`)
- Nominal capacity (`75..78`): `0x0A21FE80` = `170.000000 Ah`; Remaining (`79..82`): `0x04195893` = `68.770963 Ah`; Cycle capacity (`83..86`): `0x00A9840F` = `11109.391 Ah`
- Temperatures 1–6 (`91..102`): `22 °C`, `21 °C`, `21 °C`, `21 °C`, `21 °C`, `21 °C`
- Charge MOSFET (`103`): `0x01` (On), Discharge MOSFET (`104`): `0x01` (On), Balancer (`105`): `0x00` (Off)
- Checksum (`138..139`): `0x15F4`

### 2021+ status and device info frames (`ANT-BLE16ZMUB`, 16S 280 Ah LiFePO4)

Source: [`syssi/esphome-ant-bms/docs/pdus/btsnoop_hci_ANT-BLE16ZMUB-reduced.txt`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/pdus/btsnoop_hci_ANT-BLE16ZMUB-reduced.txt) (also in `tests/components/ant_bms/frames_16s_status.h`).

```
Device info request -> response (48 bytes on BLE):
7E A1 02 6C 02 20 58 C4 AA 55
-> 7E A1 12 6C 02 20 31 36 5A 4D 00 00 00 00 00 00 00 00 00 00 00 00 31 36 5A 4D
   55 42 30 30 2D 32 31 31 30 32 36 41 72 08 FF 0B 00 00 41 F2 AA 55
   # Hardware model: "16ZM", Software version: "16ZMUB00-211026A", CRC: 0x0872 (72 08)

Status request -> response (152 bytes, 16 cells, 2 temp sensors, offset = 36):
7E A1 01 00 00 BE 18 55 AA 55
-> 7E A1 11 00 00 8E 05 01 02 10 00 00 00 00 00 00 00 00 80 00 80 01 00 00 00 00
   00 00 00 00 00 00 00 00 E4 0C E4 0C E5 0C E5 0C E8 0C E7 0C E7 0C E6 0C E8 0C
   E7 0C E7 0C E7 0C E7 0C E7 0C E6 0C E9 0C 01 00 02 00 02 00 07 00 A4 14 03 00
   5B 00 64 00 01 01 00 00 00 76 B0 10 D5 67 0E 0F BA 32 4A 00 0F 00 00 00 10 58
   2E 02 00 00 00 00 E9 0C 10 00 E4 0C 01 00 05 00 E6 0C 00 00 80 00 7A 00 0F 02
   F2 FA B9 8C 3B 00 BB D8 58 00 DA 2D 43 00 E8 B6 49 00 05 43 AA 55
```

Decoded highlights (status frame):
- `data_len` (`5`): `0x8E` (142 bytes); Permissions (`6`): `5`; Battery status (`7`): `1` (Idle); Temp sensors (`8`): `2`; Cells (`9`): `16` (`offset = 2 × (16 + 2) = 36`)
- Cells 1–16 (`34..65`): `3.300 V` (`0x0CE4`) to `3.305 V` (`0x0CE9`)
- Temperatures (`66..73`): T1 = `1 °C`, T2 = `2 °C`, MOSFET = `2 °C`, Balancer = `7 °C`
- Total voltage (`74..75`): `0x14A4` = `52.84 V`; Current (`76..77`): `0x0003` = `0.3 A`; SOC (`78..79`): `91 %`; SOH (`80..81`): `100 %`
- Nominal capacity (`86..89`): `0x10B07600` = `280.000000 Ah`; Remaining (`90..93`): `0x0F0E67D5` = `252.602325 Ah`; Cycle capacity (`94..97`): `0x004A32BA` = `4862.650 Ah`
- Chemistry (`130..131`): `0xFAF2` (LiFePO4); CRC-16/MODBUS (`148..149`): `0x4305` (`05 43`)

---

## Errata and disagreements

1. **Legacy UART baud rate (9600 vs 19200):**
   [`ANT_communication_protocol_EN.1.pdf`](PDFs/ANT_communication_protocol_EN.1.pdf) (page 1) specifies `9600 baud`, 8N1. In practice, [`syssi/esphome-ant-bms`](https://github.com/syssi/esphome-ant-bms) (`esp32-old-example.yaml`, `esp32-example.yaml`) communicates at **19200 8N1** on the 4-pin TTL UART port for both legacy and 2021+ hardware. If a pack does not answer at 19200, test 9600 (and verify that the `3.3V` pin on the 4-pin connector is powered).
2. **Current and power sign convention:**
   In [`syssi/esphome-ant-bms`](https://github.com/syssi/esphome-ant-bms) (`ant_bms_old.cpp`, `ant_bms.cpp`) and `aiobmsble/bms/ant_bms.py`, raw current (`s32 BE` in legacy, `s16 LE` in 2021+) is multiplied by `+0.1 A` directly. In [`dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) (`bms/ant.py` line 79 and `ext/aiobmsble/bms/ant_leg_bms.py` line 39), legacy current is divided by **`-10`** (`raw / -10`). In the captured sequences [`model2019-req-dbdb00000000.txt`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/pdus/model2019-req-dbdb00000000.txt) and [`btsnoop_hci_ANT-BLE16ZMUB-reduced.txt`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/pdus/btsnoop_hci_ANT-BLE16ZMUB-reduced.txt), positive raw current (`+8.0 A` / `+12.0 A` and `+0.3 A` / `+1.0 A`) and positive power coincide with decreasing remaining capacity and increasing accumulated discharge capacity/time, whereas in `tests/components/ant_bms_old/frames_8s.h` discharging is negative (`-10.7 A`, `-285 W`). Verify the sign under a known load on your firmware.
3. **Temperature offset bug in `dbus-serialbattery/bms/ant.py`:**
   In `dbus-serialbattery/bms/ant.py` (line 101), `unpack_from(">bxb", status_data, 96)` reads the low bytes of **Temperature 3** (offset `95..96`) and **Temperature 4** (offset `97..98`) instead of Temperature 1 (`91..92`) and Temperature 2 (`93..94`).
4. **Zero nominal capacity on some legacy BMS units:**
   As noted in `aiobmsble/bms/ant_leg_bms.py` (line 160) and seen in `tests/components/ant_bms_old/frames_8s.h`, some legacy BMS firmwares report `0x00000000` for the nominal capacity setting at offset `75..78` while still reporting valid remaining capacity (`79..82`) and SOC (`74`).
5. **6-byte extra trailer on 2021+ `0x12` and `0x42` BLE responses:**
   In BLE captures (`btsnoop_hci_ANT-BLE16ZMUB-reduced.txt`), responses to command `0x02` (`func = 0x12`, e.g. device info or block settings read) and command `0x22` (`func = 0x42`, register write ACK) place their valid CRC-16/MODBUS at offset `6 + data_len`, followed by an extra 6-byte sub-block (`FF 0B 00 00 41 F2` or `FF 0A 00 00 10 32`, which carries its own CRC-16/MODBUS) before the `AA 55` terminator. Both `esphome-ant-bms` (`ant_bms_ble.cpp` lines 303–318) and `aiobmsble` (`ant_bms.py` lines 147–164) therefore compute the expected frame length from `data[5]` (`6 + data_len + 4`) and read the CRC at `6 + data_len` rather than `buffer.size() - 4`.
6. **Example table checksum typo in `syssi/esphome-ant-bms/README.md`:**
   The illustrative 13S legacy frame table in `syssi/esphome-ant-bms/README.md` lists `0x0B 0x00` at offset `138..139`, whereas the sum of the example bytes shown in that table (`4..137`) is `0x0B24`. All raw capture files in `docs/pdus/` match `sum(byte[4..137]) & 0xFFFF` exactly.

---

## Sources

- [ANT_communication_protocol_EN.1.pdf](PDFs/ANT_communication_protocol_EN.1.pdf) — original vendor protocol note (`AtyParam.java` / `AtyParam2.java`, 4 pages, retrieved from [`syssi/esphome-ant-bms/docs/ANT_communication_protocol_EN.1.pdf`](https://github.com/syssi/esphome-ant-bms/blob/main/docs/ANT_communication_protocol_EN.1.pdf))
- [`syssi/esphome-ant-bms`](https://github.com/syssi/esphome-ant-bms) — ESPHome components and capture logs:
  - `components/ant_bms/ant_bms.h`, `components/ant_bms/ant_bms.cpp`, `components/ant_bms/button/__init__.py`, `components/ant_bms/switch/__init__.py`, `components/ant_bms/select/__init__.py` (2021+ UART protocol)
  - `components/ant_bms_ble/ant_bms_ble.h`, `components/ant_bms_ble/ant_bms_ble.cpp` (2021+ BLE protocol)
  - `components/ant_bms_old/ant_bms_old.h`, `components/ant_bms_old/ant_bms_old.cpp`, `components/ant_bms_old/button/__init__.py`, `components/ant_bms_old/switch/__init__.py` (Legacy UART protocol)
  - `components/ant_bms_old_ble/ant_bms_old_ble.h`, `components/ant_bms_old_ble/ant_bms_old_ble.cpp` (Legacy BLE protocol)
  - `docs/pdus/model2019-req-dbdb00000000.txt`, `docs/pdus/model2021-req-5a5a00000101.txt`, `docs/pdus/model2021-req-dbdb00000000.txt`, `docs/pdus/model2021-req-7ea1010000be1855aa55.txt`, `docs/pdus/btsnoop_hci_ANT-BLE16ZMUB-reduced.txt` (captured frames)
- [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/ant.py` (Legacy UART driver), `dbus-serialbattery/ext/aiobmsble/bms/ant_bms.py` (2021+ BLE driver), and `dbus-serialbattery/ext/aiobmsble/bms/ant_leg_bms.py` (Legacy BLE driver)
- [`klotztech/VBMS` Wiki — Serial protocol](https://github.com/klotztech/VBMS/wiki/Serial-protocol) — community reverse-engineering of the legacy Android app (`AtyMain.java`, `AtyParam.java`, `AtyParam2.java`, `BcService.java`)

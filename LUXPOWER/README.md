# Luxpower / EG4 — Modbus RTU, Wi-Fi Dongle TCP & Battery CAN

> Status: 📄 Vendor doc / 🌐 Community — transcribed from the Luxpower / EG4 Modbus RTU protocol documents and cross-checked against [celsworth/lxp-bridge](https://github.com/celsworth/lxp-bridge), [ant0nkr/luxpower-ha-integration](https://github.com/ant0nkr/luxpower-ha-integration), and [jefflaplante/lux](https://github.com/jefflaplante/lux).

Luxpower hybrid and off-grid storage inverters — and their OEM equivalents sold by EG4 Electronics — share a common register map exposed over two transports:
1. **RS485 ("Modbus RTU")** on the inverter's `INV485` / Modbus pins, using a **proprietary Luxpower PDU** that embeds the 10-byte ASCII inverter serial number after the function code and encodes all 16-bit fields in **little-endian** byte order.
2. **Wi-Fi / LAN Dongle TCP (port `8000`)**, which wraps that same Luxpower Modbus PDU inside an `A1 1A` framing header.

Towards a lithium battery BMS on the **CAN** port, Luxpower inverters use the [Pylontech low-voltage CAN protocol](../PYLONTECH/README.md#low-voltage-can-protocol) at 500 kbit/s with optional BYD/Luxpower extensions (`0x373` cell extremes and `0x379` total capacity).

---

## Summary

| | |
|---|---|
| Products | Luxpower LXP-LB / Hybrid 3–6K, ACS, SNA 3000–6000, LXP-12K; EG4 18kPV-12LV, 6000XP, 12000XP |
| Physical layer | **Inverter RS485**: `INV485` RJ45 or shared dongle port; **Dongle TCP**: Wi-Fi/LAN datalogger on TCP port `8000`; **Battery CAN**: RJ45 CAN port |
| Settings | **RS485**: `19200 8N1` (some older manuals state `9600 8N1`); **TCP**: port `8000`; **Battery CAN**: `500 kbit/s`, 11-bit IDs |
| Addressing | 1-byte address (`0x00` client → inverter, `0x01` inverter → client on TCP; configurable `0–150` in Hold reg 15) **plus** 10-byte ASCII inverter serial number in every frame |
| Roles | RS485: host polls inverter (`≥ 1 s` interval; `> 60 s` if sharing the bus with a cloud dongle). TCP/8000: dongle pushes 40-register batches unsolicited every ~2 s and accepts interleaved read/write requests. Battery CAN: BMS broadcasts every 1 s, inverter sends `0x305` keep-alive |
| Byte order | **Little-endian (`LE`)** for register addresses, register counts, 16-bit words, 32-bit word pairs (`L` word first, `H` word second), and CRC-16/MODBUS |

---

## Framing

### 1. Luxpower "Modbus RTU" PDU (RS485 and inner TCP payload)

Despite being called "Modbus RTU" in vendor manuals, the wire frame differs from standard Modbus RTU in four ways:
- **Embedded 10-byte Serial Number (`SN[0..9]`)**: Every request and response carries the 10-byte ASCII inverter serial number immediately after the 1-byte Function Code.
- **Little-endian fields**: Starting register address `A`, register count `N`, 16-bit register values, and 32-bit values (`[b0, b1, b2, b3]` = `b0 | (b1 << 8) | (b2 << 16) | (b3 << 24)`) are all transmitted **low byte first**.
- **Echoed start address in read replies**: Normal responses to `0x03` and `0x04` echo the 2-byte starting register address `A` before the byte count and register values.
- **40-register group alignment**: On standard firmware, a single query can read at most **40 registers**, aligned to 40-register boundaries (`0–39`, `40–79`, `80–119`, `120–159`, `160–199`). Queries that cross a boundary (such as `38–40`) are rejected. Newer firmware (TCP protocol `5`) also supports 127-register reads starting at `0` (`254` data bytes).

**Checksum**: Standard **CRC-16/MODBUS** (polynomial `0x8005`, reflected `0xA001`, init `0xFFFF`, no final XOR) computed over the RTU frame from `Address` through the last data byte, transmitted **low byte first** (little-endian).

### 2. Wi-Fi / LAN Dongle TCP Wrapper (Port `8000`)

The Wi-Fi/LAN dongle wraps the RTU frame above inside a header starting with `0xA1 0x1A`:

| Offset | Size | Type | Field | Description |
|---|---|---|---|---|
| 0 | 2 | `u8[2]` | Prefix | Always `A1 1A` (`[161, 26]`) |
| 2 | 2 | `u16 LE` | Protocol | `1`, `2`, or `5` (requests typically use `1` for `0x03`/`0x04`/`0x06` and `2` for `0x10`) |
| 4 | 2 | `u16 LE` | Frame Length | Byte count following byte 5 (`total_packet_len - 6`) |
| 6 | 1 | `u8` | Unknown | Always `0x01` |
| 7 | 1 | `u8` | TCP Function | `0xC1` (193) Heartbeat, `0xC2` (194) TranslatedData, `0xC3` (195) ReadParam, `0xC4` (196) WriteParam |
| 8 | 10 | `char[10]` | Datalog Serial | 10-byte ASCII serial number of the Wi-Fi/LAN dongle |
| 18 | 2 | `u16 LE` | Data Length | (For `0xC2` TranslatedData) Length of the following RTU data frame **excluding** the 2-byte CRC |
| 20 | `Data Length` | `u8[...]` | RTU Data Frame | `Address` (1 B) + `Device Function` (1 B) + `Inverter SN` (10 B) + `Register` (2 B LE) + payload |
| `20 + Data Length` | 2 | `u16 LE` | CRC-16/MODBUS | CRC-16/MODBUS of `packet[20 .. 20 + Data Length]` (the RTU Data Frame only) |

#### Value Length Byte (VLB) rule in `0xC2` TranslatedData

Whether the 1-byte byte-count (`Value Length Byte`) is present at offset `34` (`RTU Data Frame[14]`) depends on the TCP `Protocol` field, direction, and function code:

| TCP Protocol | Device Function | Direction | Value Length Byte (offset 34) |
|---|---|---|---|
| `1` | Any | Any | **Omitted** (except some protocol-1 read replies that include an even byte count matching remaining payload) |
| `2` or `5` | `0x03` ReadHold, `0x04` ReadInput | Client → Inverter (request) | **Omitted** (requests always carry 2-byte `Register Count` LE) |
| `2` or `5` | `0x03` ReadHold, `0x04` ReadInput | Inverter → Client (reply) | **Present** (`2 × N` bytes) |
| Any | `0x06` WriteSingle | Any | **Omitted** (always 2-byte register value LE) |
| `2` | `0x10` WriteMulti | Client → Inverter (request) | **Present** (after 2-byte `Register Count` LE) |

### Worked example (verified)

Requesting **40 input registers (`0x04`) starting at register `0`** from inverter `"5555555555"` via dongle `"2222222222"` (from [`celsworth/lxp-bridge/tests/test_packet_tcp_frames.rs`](https://github.com/celsworth/lxp-bridge/blob/master/tests/test_packet_tcp_frames.rs)):

```
A1 1A 01 00 20 00 01 C2 32 32 32 32 32 32 32 32 32 32 12 00 00 04 35 35 35 35 35 35 35 35 35 35 00 00 28 00 2A 84
```

- `A1 1A` — TCP prefix
- `01 00` — TCP protocol `1` (`u16 LE`)
- `20 00` — Frame length `32` bytes (`38 - 6`)
- `01` — Always `1`
- `C2` — TCP function `194` (`TranslatedData`)
- `32 32 32 32 32 32 32 32 32 32` — Dongle serial `"2222222222"`
- `12 00` — Data length `18` bytes (`u16 LE`, bytes `20..35`)
- `00` — Address (`0` = client → inverter)
- `04` — Device function `0x04` (`ReadInput`)
- `35 35 35 35 35 35 35 35 35 35` — Inverter serial `"5555555555"`
- `00 00` — Start register `0` (`u16 LE`)
- `28 00` — Register count `40` (`0x0028`, `u16 LE`)
- `2A 84` — `CRC-16/MODBUS("00 04 35 35 35 35 35 35 35 35 35 35 00 00 28 00")` = `0x842A`, sent little-endian as `2A 84`.

---

## Requests and responses

### TCP Functions (offset 7)

| Code | Name | Direction | Payload (offset 18+) |
|---|---|---|---|
| `0xC1` (193) | `Heartbeat` | Both | 1 byte (`0x00`), no CRC; total frame is 19 bytes (`Frame Length = 13`) |
| `0xC2` (194) | `TranslatedData` | Both | `Data Length` (2 B LE) + RTU Data Frame + CRC-16/MODBUS (2 B LE) |
| `0xC3` (195) | `ReadParam` | Both |Dongle parameter read (2-byte register LE; reply in protocol 2 adds 2-byte length LE + values, no CRC) |
| `0xC4` (196) | `WriteParam` | Both |Dongle parameter write (2-byte register LE + 2-byte length LE + values, no CRC) |

### RTU Device Function Codes (offset 21 on TCP, offset 1 on RS485)

| Function | Normal Reply | Exception Reply | Description |
|---|---|---|---|
| `0x03` | `0x03` | `0x83` | Read Holding Registers (read/write configuration) |
| `0x04` | `0x04` | `0x84` | Read Input Registers (read-only live telemetry) |
| `0x06` | `0x06` | `0x86` | Write Single Holding Register |
| `0x10` | `0x10` | `0x90` | Write Multiple Holding Registers |

Exception codes (1 byte after `Start Address` in an exception reply):
- `0x01`: Illegal function code
- `0x02`: Illegal data address (or cross-group query)
- `0x03`: Illegal data value (out of range or wrong register count)
- `0x04`: Slave read/write failure
- `0x06`: Slave busy

### RTU Data Frame Layouts

Offsets below are relative to the start of the RTU Data Frame (byte `0` on RS485, or byte `20` inside a TCP `0xC2` packet).

#### Read Holding / Input Registers (`0x03` / `0x04`)

| RTU Offset | Size | Type | Request | Normal Response | Exception Response |
|---|---|---|---|---|---|
| 0 | 1 | `u8` | Address (`0x00` on TCP) | Address (`0x01` on TCP) | Address |
| 1 | 1 | `u8` | `0x03` or `0x04` | `0x03` or `0x04` | `0x83` or `0x84` |
| 2 | 10 | `char[10]` | Inverter `SN[0..9]` | Inverter `SN[0..9]` | Inverter `SN[0..9]` |
| 12 | 2 | `u16 LE` | Start register `A` | Start register `A` | Start register `A` |
| 14 | 2 / 1 | `u16 LE` / `u8` | Register count `N` (`u16 LE`) | Byte count `2 × N` (`u8`) | Exception code (`u8`) |
| 15 | `2 × N` | `u16 LE[N]` | — | Register values `A .. A+N-1` | — |
| End | 2 | `u16 LE` | CRC-16/MODBUS (at offset 16) | CRC-16/MODBUS (at offset `15 + 2N`) | CRC-16/MODBUS (at offset 15) |

#### Write Single Holding Register (`0x06`)

| RTU Offset | Size | Type | Request | Normal Response |
|---|---|---|---|---|
| 0 | 1 | `u8` | Address (`0x00` on TCP) | Address (`0x01` on TCP) |
| 1 | 1 | `u8` | `0x06` | `0x06` |
| 2 | 10 | `char[10]` | Inverter `SN[0..9]` | Inverter `SN[0..9]` |
| 12 | 2 | `u16 LE` | Register address `A` | Register address `A` |
| 14 | 2 | `u16 LE` | Register value | Register value |
| 16 | 2 | `u16 LE` | CRC-16/MODBUS | CRC-16/MODBUS |

#### Write Multiple Holding Registers (`0x10`)

| RTU Offset | Size | Type | Request | Normal Response |
|---|---|---|---|---|
| 0 | 1 | `u8` | Address (`0x00` on TCP) | Address (`0x01` on TCP) |
| 1 | 1 | `u8` | `0x10` | `0x10` |
| 2 | 10 | `char[10]` | Inverter `SN[0..9]` | Inverter `SN[0..9]` |
| 12 | 2 | `u16 LE` | Start register `A` | Start register `A` |
| 14 | 2 | `u16 LE` | Register count `N` | Register count `N` |
| 16 | 1 | `u8` | Byte count `2 × N` | — |
| 17 | `2 × N` | `u16 LE[N]` | Register values `A .. A+N-1` | — |
| End | 2 | `u16 LE` | CRC-16/MODBUS (at `17 + 2N`) | CRC-16/MODBUS (at offset 16) |

---

## Input registers (`0x04`, read-only)

### Group 1: Registers `0–39` (Status, PV, Battery, Grid, EPS, Daily Energy)

| Reg | Size | Type | Unit / scale | Field | Description |
|---|---|---|---|---|---|
| 0 | 1 | `u16` | enum | `State` | Operating mode (see [Operating mode values](#operating-mode-values-input-register-0)) |
| 1 | 1 | `u16` | 0.1 V | `Vpv1` | PV1 string voltage |
| 2 | 1 | `u16` | 0.1 V | `Vpv2` | PV2 string voltage |
| 3 | 1 | `u16` | 0.1 V | `Vpv3` | PV3 string voltage |
| 4 | 1 | `u16` | 0.1 V | `Vbat` | Battery pack voltage |
| 5 | 1 | `u8` + `u8` | 1 % | `SOC` / `SOH` | **Low byte**: Battery State of Charge (`0–100 %`); **High byte**: State of Health (`0–100 %`) |
| 6 | 1 | `u16` | bitfield | `InternalFault` | Internal fault code |
| 7 | 1 | `u16` | 1 W | `Ppv1` | PV1 input power |
| 8 | 1 | `u16` | 1 W | `Ppv2` | PV2 input power |
| 9 | 1 | `u16` | 1 W | `Ppv3` | PV3 input power (or total PV power on some AC-storage models) |
| 10 | 1 | `u16` | 1 W | `Pcharge` | Battery charging power (into battery) |
| 11 | 1 | `u16` | 1 W | `Pdischarge` | Battery discharging power (out of battery) |
| 12 | 1 | `u16` | 0.1 V | `VacR` | Grid R-phase (or single-phase) voltage |
| 13 | 1 | `u16` | 0.1 V | `VacS` | Grid S-phase voltage (floats on single-phase units) |
| 14 | 1 | `u16` | 0.1 V | `VacT` | Grid T-phase voltage (floats on single-phase units) |
| 15 | 1 | `u16` | 0.01 Hz | `Fac` | Grid frequency |
| 16 | 1 | `u16` | 1 W | `Pinv` | On-grid inverter output power (R-phase on 3-phase models) |
| 17 | 1 | `u16` | 1 W | `Prec` | AC charging rectified power (R-phase on 3-phase models) |
| 18 | 1 | `u16` | 0.01 A | `IinvRMS` | Inverter RMS output current |
| 19 | 1 | `u16` | 0.001 | `PF` | Power factor (`0..1000` → `x / 1000`; `1000..2000` → `(1000 - x) / 1000`) |
| 20 | 1 | `u16` | 0.1 V | `VepsR` | Off-grid (EPS) R-phase output voltage |
| 21 | 1 | `u16` | 0.1 V | `VepsS` | Off-grid (EPS) S-phase output voltage |
| 22 | 1 | `u16` | 0.1 V | `VepsT` | Off-grid (EPS) T-phase output voltage |
| 23 | 1 | `u16` | 0.01 Hz | `Feps` | Off-grid (EPS) output frequency |
| 24 | 1 | `u16` | 1 W | `Peps` | Off-grid (EPS) active power |
| 25 | 1 | `u16` | 1 VA | `Seps` | Off-grid (EPS) apparent power |
| 26 | 1 | `u16` | 1 W | `Ptogrid` | Power exported to grid |
| 27 | 1 | `u16` | 1 W | `Ptouser` | Power imported from grid |
| 28 | 1 | `u16` | 0.1 kWh | `Epv1_day` | PV1 energy generated today |
| 29 | 1 | `u16` | 0.1 kWh | `Epv2_day` | PV2 energy generated today |
| 30 | 1 | `u16` | 0.1 kWh | `Epv3_day` | PV3 energy generated today |
| 31 | 1 | `u16` | 0.1 kWh | `Einv_day` | Inverter grid-connected output energy today |
| 32 | 1 | `u16` | 0.1 kWh | `Erec_day` | AC charging rectified energy today |
| 33 | 1 | `u16` | 0.1 kWh | `Echg_day` | Battery charge energy today |
| 34 | 1 | `u16` | 0.1 kWh | `Edischg_day` | Battery discharge energy today |
| 35 | 1 | `u16` | 0.1 kWh | `Eeps_day` | Off-grid (EPS) output energy today |
| 36 | 1 | `u16` | 0.1 kWh | `Etogrid_day` | Energy exported to grid today |
| 37 | 1 | `u16` | 0.1 kWh | `Etouser_day` | Energy imported from grid today |
| 38 | 1 | `u16` | 0.1 V | `Vbus1` | Internal DC Bus 1 voltage |
| 39 | 1 | `u16` | 0.1 V | `Vbus2` | Internal DC Bus 2 voltage |

### Group 2: Registers `40–79` (Cumulative Energy, Faults/Warnings, Temperatures, Runtime)

All 32-bit fields occupy two consecutive registers: low word at `reg`, high word at `reg + 1` (`u32 LE`).

| Reg | Size | Type | Unit / scale | Field | Description |
|---|---|---|---|---|---|
| 40–41 | 2 | `u32` | 0.1 kWh | `Epv1_all` | Cumulative PV1 energy (`40` L, `41` H) |
| 42–43 | 2 | `u32` | 0.1 kWh | `Epv2_all` | Cumulative PV2 energy (`42` L, `43` H) |
| 44–45 | 2 | `u32` | 0.1 kWh | `Epv3_all` | Cumulative PV3 energy (`44` L, `45` H) |
| 46–47 | 2 | `u32` | 0.1 kWh | `Einv_all` | Cumulative inverter output energy (`46` L, `47` H) |
| 48–49 | 2 | `u32` | 0.1 kWh | `Erec_all` | Cumulative AC charging rectified energy (`48` L, `49` H) |
| 50–51 | 2 | `u32` | 0.1 kWh | `Echg_all` | Cumulative battery charge energy (`50` L, `51` H) |
| 52–53 | 2 | `u32` | 0.1 kWh | `Edischg_all` | Cumulative battery discharge energy (`52` L, `53` H) |
| 54–55 | 2 | `u32` | 0.1 kWh | `Eeps_all` | Cumulative off-grid (EPS) output energy (`54` L, `55` H) |
| 56–57 | 2 | `u32` | 0.1 kWh | `Etogrid_all` | Cumulative grid export energy (`56` L, `57` H) |
| 58–59 | 2 | `u32` | 0.1 kWh | `Etouser_all` | Cumulative grid import energy (`58` L, `59` H) |
| 60–61 | 2 | `u32` | bitfield | `FaultCode` | Inverter fault bitmask (`E000`–`E031`; see [Alarms and status bits](#alarms-and-status-bits)) |
| 62–63 | 2 | `u32` | bitfield | `WarningCode` | Inverter warning bitmask (`W000`–`W031`; see [Alarms and status bits](#alarms-and-status-bits)) |
| 64 | 1 | `s16` | 1 °C | `Tinner` | Internal inverter temperature |
| 65 | 1 | `s16` | 1 °C | `Tradiator1` | Heatsink / radiator 1 temperature |
| 66 | 1 | `s16` | 1 °C | `Tradiator2` | Heatsink / radiator 2 temperature |
| 67 | 1 | `s16` | 1 °C | `Tbat` | Lead-acid NTC battery temperature sensor |
| 69–70 | 2 | `u32` | 1 s | `RunningTime` | Total cumulative inverter runtime in seconds (`69` L, `70` H) |
| 71 | 1 | `u16` | bitfield | `AutoTest` | Bits 0–3: start (`0` off, `1` on); Bits 4–7: status (`0` wait, `1` testing, `2` fail, `3` V OK, `4` F OK, `5` pass); Bits 8–11: step (`1..8`) |
| 77 | 1 | `u16` | bitfield | `ACInputType` | Bit 0: `0` Grid, `1` Generator; Bit 1: AC couple flow; Bit 2: AC couple enable |

### Group 3: Registers `80–119` (BMS Telemetry, Parallel Status, Serial Number)

| Reg | Size | Type | Unit / scale | Field | Description |
|---|---|---|---|---|---|
| 80 | 1 | `u16` | packed | `BatTypeAndBrand` | Battery brand/model and BMS link type (`0` = CAN, `1` = RS485) |
| 81 | 1 | `u16` | 0.01 A | `MaxChgCurr` | BMS charge current limit (see [Errata](#errata-and-disagreements)) |
| 82 | 1 | `u16` | 0.01 A | `MaxDischgCurr` | BMS discharge current limit (see [Errata](#errata-and-disagreements)) |
| 83 | 1 | `u16` | 0.1 V | `ChargeVoltRef` | BMS recommended charge voltage |
| 84 | 1 | `u16` | 0.1 V | `DischgCutVolt` | BMS discharge cut-off voltage |
| 85–94 | 10 | `u16[10]` | raw | `BatStatus0_BMS`..`9` | Raw BMS status registers forwarded by inverter |
| 95 | 1 | `u16` | bitfield | `BatStatus_INV` | Inverter-summarized lithium battery status |
| 96 | 1 | `u16` | count | `BatParallelNum` | Number of battery modules in parallel |
| 97 | 1 | `u16` | 1 Ah | `BatCapacity` | Installed battery bank capacity (populated over CAN via `0x379`) |
| 98 | 1 | `s16` | 0.01 A | `BatCurrent_BMS` | BMS battery current (**signed**: `+` charging, `−` discharging; see [Errata](#errata-and-disagreements)) |
| 99 | 1 | `u16` | bitfield | `FaultCode_BMS` | BMS fault code |
| 100 | 1 | `u16` | bitfield | `WarningCode_BMS` | BMS warning code |
| 101 | 1 | `u16` | 0.001 V | `MaxCellVolt_BMS` | Maximum cell voltage |
| 102 | 1 | `u16` | 0.001 V | `MinCellVolt_BMS` | Minimum cell voltage |
| 103 | 1 | `s16` | 0.1 °C | `MaxCellTemp_BMS` | Maximum cell temperature (signed) |
| 104 | 1 | `s16` | 0.1 °C | `MinCellTemp_BMS` | Minimum cell temperature (signed) |
| 105 | 1 | `u16` | enum | `BMSFWUpdateState` | `1` = upgrading, `2` = succeeded, `3` = failed |
| 106 | 1 | `u16` | cycles | `CycleCnt_BMS` | BMS charge/discharge cycle count |
| 107 | 1 | `u16` | 0.1 V | `BatVoltSample_INV` | Battery terminal voltage sampled by the inverter |
| 108 | 1 | `s16` | 0.1 °C | `T1` | 12K battery terminal temperature (`109–112` `T2..T5` reserved) |
| 113 | 1 | `u16` | bitfield | `ParallelStatus` | Bits 0–1: `1` Master, `2` Slave; Bits 2–3: Phase (`1` R, `2` S, `3` T); Bits 8–15: Parallel unit count (`1–255`) |
| 114 | 1 | `u16` | 1 W | `OnGridLoadPower` | On-grid load power (12K models) |
| 115–119 | 5 | `char[10]` | ASCII | `SN[0..9]` | 10-character ASCII inverter serial number (`SN[0]` in low byte of 115, `SN[9]` in high byte of 119) |

### Group 4 & Extended Input Registers (`120–232` and `5000+`)

| Reg | Size | Type | Unit / scale | Field | Description |
|---|---|---|---|---|---|
| 120 | 1 | `u16` | 0.1 V | `VBusP` | Half-bus voltage |
| 121 | 1 | `u16` | 0.1 V | `GenVolt` | Generator AC voltage |
| 122 | 1 | `u16` | 0.01 Hz | `GenFreq` | Generator AC frequency |
| 123 | 1 | `u16` | 1 W | `GenPower` | Generator active input power |
| 124 | 1 | `u16` | 0.1 kWh | `Egen_day` | Generator energy today |
| 125–126 | 2 | `u32` | 0.1 kWh | `Egen_all` | Cumulative generator energy (`125` L, `126` H) |
| 127 | 1 | `u16` | 0.1 V | `EPSVoltL1N` | Split-phase EPS L1–N voltage (or 3-phase S-phase generator V) |
| 128 | 1 | `u16` | 0.1 V | `EPSVoltL2N` | Split-phase EPS L2–N voltage (or 3-phase T-phase generator V) |
| 129–130 | 2 | `u16[2]` | 1 W | `Peps_L1N` / `L2N` | Split-phase EPS L1–N and L2–N active power |
| 131–132 | 2 | `u16[2]` | 1 VA | `Seps_L1N` / `L2N` | Split-phase EPS L1–N and L2–N apparent power |
| 133–134 | 2 | `u16[2]` | 0.1 kWh | `EepsL1N_day` / `L2N` | Split-phase EPS L1–N and L2–N daily energy |
| 135–138 | 4 | `u32[2]` | 0.1 kWh | `EepsL1N_all` / `L2N` | Split-phase EPS L1–N (`135–136`) and L2–N (`137–138`) total energy |
| 140–143 | 4 | `u16[4]` | 1 mA | `AFCI_CurrCH1..4` | AFCI current channels 1–4 |
| 144 | 1 | `u16` | bitfield | `AFCIFlag` | Bits 0–3: CH1–4 arc alarm (`1` alarm); Bits 4–7: CH1–4 self-test fail (`1` fail) |
| 153 | 1 | `s16` | 1 W | `ACCouplePower` | AC-coupled solar inverter power |
| 170 | 1 | `u16` | 1 W | `Pload` | On-grid load power consumption |
| 171 | 1 | `u16` | 0.1 kWh | `Eload_day` | Daily load energy consumption |
| 172–173 | 2 | `u32` | 0.1 kWh | `Eload_all` | Cumulative load energy consumption (`172` L, `173` H) |
| 174 | 1 | `u16` | bitfield | `SwitchState` | Bits 0–4: 5-position safety DIP switch status (`0x00–0x1F`) |
| 180–193 | 14 | `u16` | various | `Pinv_S/T`, `Prec_S/T`… | Three-phase S/T phase powers (`180–189`, W), currents (`190–191`, 0.01 A), PF (`192–193`) |
| 193–204 | 12 | `u16` | 0.1 V / 1 W | `GridVolt_L1N/L2N`… | US split-phase L1–N / L2–N grid & generator voltages (`193–196`) and powers (`197–204`) |

#### Per-Battery Module Blocks (Registers `5000+`, community decoded)

When smart batteries communicate with the inverter, individual pack telemetry can be read starting at register `5000`. Up to 4 battery blocks (30 registers each: `5000–5029`, `5030–5059`, `5060–5089`, `5090–5119`) are returned per read in [`ant0nkr/luxpower-ha-integration`](https://github.com/ant0nkr/luxpower-ha-integration):
- `+3`: Capacity (Ah)
- `+5` / `+6`: Max charge / discharge current
- `+8` / `+9`: Pack voltage / current
- `+10`: SOH / SOC
- `+11`: Cycle count
- `+12` / `+13`: Max / min cell temperature
- `+14` / `+15`: Max / min cell voltage
- `+18`: Firmware version
- `+19..+26`: Zero-terminated UTF-8 battery serial number (14 bytes)

---

## Holding registers (`0x03`, `0x06`, `0x10`, read/write)

Packed time registers store **Hour (`0–23`) in the low byte** and **Minute (`0–59`) in the high byte** (`value = hour | (minute << 8)`).

| Reg | Size | Type | Unit / scale | Field | Description |
|---|---|---|---|---|---|
| 2–6 | 5 | `char[10]` | ASCII | `SN[0..9]` | Inverter 10-byte ASCII serial number |
| 7–8 | 2 | `char[4]` | ASCII | `FWCode0..3` | Firmware model, derived model, ODM, and region ASCII codes |
| 9 | 1 | `u8` + `u8` | raw | `SlaveVer` / `ComVer` | Low byte: Redundant CPU version; High byte: Communication CPU version |
| 10 | 1 | `u8` + `u8` | raw | `CntlVer` / `FWVer` | Low byte: Control CPU version; High byte: External FW version |
| 11 | 1 | `u16` | bitfield | `ResetSetting` | System reset and reboot command bits (see [Control commands](#control-commands-dangerous)) |
| 12 | 1 | `u8` + `u8` | year / month | `Time_Year` / `Month` | Low byte: Year − 2000 (`17–255`); High byte: Month (`1–12`) |
| 13 | 1 | `u8` + `u8` | day / hour | `Time_Date` / `Hour` | Low byte: Day (`1–31`); High byte: Hour (`0–23`) |
| 14 | 1 | `u8` + `u8` | min / sec | `Time_Minute` / `Second` | Low byte: Minute (`0–59`); High byte: Second (`0–59`) |
| 15 | 1 | `u16` | `0–150` | `ComAddr` | Modbus slave address |
| 16 | 1 | `u16` | `0–1` | `Language` | `0` = English, `1` = German (high bits hold Device Type Code on some models) |
| 20 | 1 | `u16` | enum | `PVInputModel` | Standard: `0` None, `1` PV1, `2` PV2, `3` PV1‖PV2, `4` PV1+PV2 separate; 12K: `0` None .. `7` PV1&2&3 |
| 21 | 1 | `u16` | bitfield | `FuncEn` | Primary operating mode & feature enable bitmask (see table below) |
| 22 | 1 | `u16` | 0.1 V | `StartPVVolt` | PV start-up voltage (`900–5000` = `90.0–500.0 V`) |
| 23 | 1 | `u16` | 1 s | `ConnectTime` | Grid connection wait time (`30–600 s`) |
| 24 | 1 | `u16` | 1 s | `ReconnectTime` | Grid reconnection wait time (`0–900 s`) |
| 25–28 | 4 | `u16[4]` | 0.1 V / 0.01 Hz | `GridVolt/FreqConn` | Grid connection voltage (`25–26`) and frequency (`27–28`) low/high limits |
| 60 | 1 | `u16` | 1 % | `ActivePowerPercentCMD` | Active power limit (`0–100 %`; see reg `143` for `0.1 %` resolution) |
| 64 | 1 | `u16` | 1 % | `ChargePowerPercentCMD` | System battery charge power rate (`0–100 %`; see reg `138` for `0.1 %`) |
| 65 | 1 | `u16` | 1 % | `DischgPowerPercentCMD` | System battery discharge power rate (`0–100 %`; see reg `139` for `0.1 %`) |
| 66 | 1 | `u16` | 1 % | `ACChgPowerCMD` | Grid (AC) charge power rate (`0–100 %`; see reg `140` for `0.1 %`) |
| 67 | 1 | `u16` | 1 % | `ACChgSOCLimit` | Grid (AC) charge target SOC limit (`0–100 %`) |
| 68–73 | 6 | `u8` + `u8` | hour / min | `ACChgStart/End` | AC charge time windows 0 (`68–69`), 1 (`70–71`), and 2 (`72–73`) |
| 74 | 1 | `u16` | 1 % | `ChgFirstPowerCMD` | Charge-priority power rate (`0–100 %`; see reg `141` for `0.1 %`) |
| 75 | 1 | `u16` | 1 % | `ChgFirstSOCLimit` | Charge-priority SOC limit (`0–100 %`) |
| 76–81 | 6 | `u8` + `u8` | hour / min | `ChgFirstStart/End` | Charge-priority time windows 0 (`76–77`), 1 (`78–79`), and 2 (`80–81`) |
| 82 | 1 | `u16` | 1 % | `ForcedDischgPowerCMD` | Forced-discharge power rate (`0–100 %`; see reg `142` for `0.1 %`) |
| 83 | 1 | `u16` | 1 % | `ForcedDischgSOCLimit` | Forced-discharge SOC stop limit (`0–100 %`) |
| 84–89 | 6 | `u8` + `u8` | hour / min | `ForcedDischgStart/End` | Forced-discharge time windows 0 (`84–85`), 1 (`86–87`), and 2 (`88–89`) |
| 90 | 1 | `u16` | 1 V | `EPSVoltageSet` | Off-grid nominal voltage (`208`, `220`, `230`, `240`, `277 V`) |
| 91 | 1 | `u16` | 1 Hz | `EPSFreqSet` | Off-grid nominal frequency (`50` or `60 Hz`) |
| 99 | 1 | `u16` | 0.1 V | `ChargeVoltRef` | Lead-acid / manual absorption charge voltage (`500–590` = `50.0–59.0 V`) |
| 100 | 1 | `u16` | 0.1 V | `CutVoltForDischg` | Lead-acid / manual discharge cut-off voltage (`400–520` = `40.0–52.0 V`) |
| 101 | 1 | `u16` | 1 A | `ChargeCurr` | Maximum battery charge current (`0–140 A`) |
| 102 | 1 | `u16` | 1 A | `DischgCurr` | Maximum battery discharge current (`0–140 A`) |
| 103 | 1 | `u16` | 1 % | `MaxBackFlow` | Maximum grid export power percentage (`0–100 %`) |
| 105 | 1 | `u16` | 1 % | `EOD` | End-of-discharge (cut-off) SOC (`10–90 %`) |
| 106–109 | 4 | `s16[4]` | 0.1 °C | `TemprLimitDischg/Chg` | Lead-acid discharge (`106` low, `107` high) & charge (`108` low, `109` high) temp limits (signed; see [Errata](#errata-and-disagreements)) |
| 110 | 1 | `u16` | bitfield | `FunctionEn1` | Secondary function enable bitmask (see table below) |
| 112 | 1 | `u16` | `0–3` | `SetSystemType` | `0` Single, `1` 1P parallel master, `2` Slave, `3` 3P parallel master |
| 113 | 1 | `u16` | `1–3` | `SetComposedPhase` | Phase assignment in parallel system (`1` R, `2` S, `3` T) |
| 116 | 1 | `u16` | 1 W | `PtoUserStartdischg` | Grid import threshold to start battery discharge (default `50 W`) |
| 119 | 1 | `s16` | 1 W | `wCT_PowerOffset` | Signed CT power calibration offset (`±1000 W`, import positive) |
| 120 | 1 | `u16` | bitfield | `stSysEnable` | Bit 0: half-hour AC charge; Bits 1–3: `ACChargeType` (`0` off, `1` time, `2` V, `3` SOC, `4` V+time, `5` SOC+time); Bits 4–5: `DischgCtrlType` (`0` V, `1` SOC, `2` both); Bit 6: `OnGridEODType` (`0` V, `1` SOC); Bit 7: `GenChargeType` (`0` V, `1` SOC) |
| 125 | 1 | `u16` | 1 % | `SOCLowLimitForEPSDischg` | Off-grid (EPS) discharge cut-off SOC (`0–EOD %`) |
| 138–143 | 6 | `u16[6]` | 0.1 % | `*PowerPercentCMD` | High-resolution (`0–1000` = `0.0–100.0 %`) charge (`138`), discharge (`139`), AC charge (`140`), charge priority (`141`), forced discharge (`142`), and active power (`143`) commands |
| 144 | 1 | `u16` | 0.1 V | `FloatChargeVolt` | Lead-acid float charge voltage (`500–560` = `50.0–56.0 V`) |
| 145 | 1 | `u16` | `0–2` | `OutputPrioConfig` | Off-grid output priority: `0` Battery first, `1` PV first, `2` AC first |
| 146 | 1 | `u16` | `0–2` | `LineMode` | Transfer time mode: `0` APL (`90–280 V`, 20 ms), `1` UPS (`170–280 V`, 10 ms), `2` GEN |
| 147 | 1 | `u16` | 1 Ah | `BatteryCapacity` | Nominal capacity for un-managed / lead-acid batteries (`0–10000 Ah`) |
| 148 | 1 | `u16` | 0.1 V | `BatteryNominalVoltage` | Nominal voltage for un-managed batteries (`400–590` = `40.0–59.0 V`) |
| 152–157 | 6 | `u8` + `u8` | hour / min | `ACFirstStart/End` | AC-first (utility priority) time windows 0 (`152–153`), 1 (`154–155`), and 2 (`156–157`) |
| 158–159 | 2 | `u16[2]` | 0.1 V | `ACChgStart/EndVolt` | Voltage-based AC charge start (`385–520`) and stop (`480–590`) voltages |
| 160–161 | 2 | `u16[2]` | 1 % | `ACChgStart/EndSOC` | SOC-based AC charge start (`0–90 %`) and stop (`20–100 %`) thresholds |
| 162–165 | 4 | `u16[4]` | 0.1 V / 1 % | `BatLow*` | Battery low alarm & recovery thresholds in V (`162–163`) and SOC (`164–165`) |
| 166–167 | 2 | `u16[2]` | 0.1 V / 1 % | `BatLowtoUtility*` | Off-grid battery-to-utility transfer voltage (`166`, `444–514`) and SOC (`167`, `0–100 %`) |
| 168 | 1 | `u16` | 1 A | `ACChargeBatCurrent` | Maximum battery charge current from AC grid (`0–140 A`) |
| 169 | 1 | `u16` | 0.1 V | `OngridEOD_Voltage` | On-grid end-of-discharge battery voltage (`400–560` = `40.0–56.0 V`) |
| 179 | 1 | `u16` | bitfield | `uFunctionEn2` | Bit 0: AC CT reversed; Bit 1: PV CT reversed; Bit 2: AFCI alarm clear; Bit 3: Battery wakeup / PV sell first; Bit 4: Volt-Watt; Bit 6: ActivePowerCMD enable; Bit 7: Grid peak shaving; Bit 8: Gen peak shaving; Bit 9: Bat charge control (`0` SOC, `1` V); Bit 10: Bat discharge control (`0` SOC, `1` V); Bit 11: AC coupling; Bit 13: Smart Load (`0` Gen, `1` Smart Load); Bit 14: RSD disable (`0` enable, `1` disable) |
| 194–198 | 5 | `u16[5]` | 0.1 V / % / A | `GenChg*` | Generator charge start/end voltage (`194–195`), start/end SOC (`196–197`), max current (`198`, A) |
| 206–212 | 7 | `u16` | various | `GridPeakShaving*` | Grid peak-shaving power (`206`), SOC (`207`), voltage (`208`), and time windows (`209–212`) |

---

## Alarms and status bits

### Operating mode values (Input Register `0`)

| Value | Mode | Description |
|---|---|---|
| `0x00` (0) | Standby | Inverter in standby |
| `0x01` (1) | Fault | Inverter fault state |
| `0x02` (2) | Programming | Firmware update in progress |
| `0x04` (4) | PV On-grid | Solar power feeding load / exporting to grid |
| `0x08` (8) | PV Charge | Solar power charging battery |
| `0x0C` (12) | PV Charge + On-grid | Solar power charging battery and feeding load / grid |
| `0x10` (16) | Battery On-grid | Battery discharging to load / grid |
| `0x11` (17) | Bypass | Off-grid AC bypass *(from `lxp-bridge`)* |
| `0x14` (20) | PV + Battery On-grid | Solar and battery supplying load / grid |
| `0x19` (25) | PV Charge + Bypass | Solar charging battery while AC is in bypass *(from `lxp-bridge`)* |
| `0x20` (32) | AC Charge | Grid power charging battery |
| `0x28` (40) | PV + AC Charge | Solar and grid power charging battery together |
| `0x40` (64) | Battery Off-grid | Off-grid backup supplied by battery |
| `0x60` (96) | Off-grid + Battery Charging | Off-grid mode with AC-coupled solar charging battery |
| `0x80` (128) | PV Off-grid | Off-grid backup from PV only (without battery) |
| `0x88` (136) | PV Charge + Off-grid | Off-grid backup from PV while excess PV charges battery |
| `0xC0` (192) | PV + Battery Off-grid | Off-grid backup supplied by PV and battery |

### Fault and Warning Bitmasks (Input Registers `60–61` and `62–63`)

Input registers `60–61` (`FaultCode`, 32-bit LE) and `62–63` (`WarningCode`, 32-bit LE) use one bit per code (`Bit 0`..`Bit 31`):

| Bit | Fault (`E0xx`, Regs `60–61`) | Warning (`W0xx`, Regs `62–63`) |
|---|---|---|
| 0 | `E000`: Internal communication fault 1 (DSP & M3) | `W000`: Battery communication failure |
| 1 | `E001`: Model fault (safety DIP switch mismatch) | `W001`: AFCI communication failure |
| 2 | `E002`: Reserved (`BatOnMosFail` in `lxp-bridge`) | `W002`: AFCI high / Battery low temperature |
| 3 | `E003`: Reserved (`CT Fail` in `lxp-bridge`) | `W003`: Meter communication failure |
| 4 | `E004`: Reserved | `W004`: Battery failure / charge & discharge forbidden by BMS |
| 5 | `E005`: Reserved | `W005`: Auto-test failed |
| 6 | `E006`: Reserved | `W006`: RSD active (or reserved) |
| 7 | `E007`: Reserved | `W007`: LCD communication failure |
| 8 | `E008`: Parallel CAN communication lost | `W008`: Firmware version mismatch |
| 9 | `E009`: Master unit lost in parallel system | `W009`: Fan stuck |
| 10 | `E010`: Multiple masters or rated power mismatch in parallel | `W010`: Duplicate parallel address / reserved |
| 11 | `E011`: AC input or safety setting inconsistent in parallel | `W011`: Parallel unit count out of range |
| 12 | `E012`: UPS / EPS output short circuit | `W012`: BatOnMos or parallel phase loss |
| 13 | `E013`: Reverse current (backfeed) on UPS / EPS output | `W013`: Overtemperature (NTC high) or no primary set in parallel |
| 14 | `E014`: DC Bus short circuit | `W014`: Multiple primary units set in parallel system |
| 15 | `E015`: Phase error in 3-phase parallel system | `W015`: Battery reverse polarity connection |
| 16 | `E016`: Relay check fault | `W016`: Grid power outage / no AC connection |
| 17 | `E017`: Internal communication fault 2 (DSP & M8) | `W017`: Grid voltage out of range |
| 18 | `E018`: Internal communication fault 3 (DSP & M3) | `W018`: Grid frequency out of range |
| 19 | `E019`: DC Bus voltage high | `W019`: Parallel AC inconsistent / reserved |
| 20 | `E020`: EPS connection fault | `W020`: PV insulation resistance low |
| 21 | `E021`: PV voltage high | `W021`: Leakage current (GFCI) high |
| 22 | `E022`: Overcurrent protection | `W022`: DC injection (DCI) high |
| 23 | `E023`: Neutral fault | `W023`: PV short circuit |
| 24 | `E024`: PV short circuit | `W024`: Reserved |
| 25 | `E025`: Heatsink / radiator temperature out of range | `W025`: Battery voltage high |
| 26 | `E026`: Internal fault (bus sample) | `W026`: Battery voltage low |
| 27 | `E027`: Sampling inconsistent between main and redundant CPU | `W027`: Battery open circuit |
| 28 | `E028`: Parallel sync signal lost / generator connection inconsistent | `W028`: Off-grid (EPS) overload |
| 29 | `E029`: Parallel synchronization trigger signal lost | `W029`: Off-grid (EPS) voltage high |
| 30 | `E030`: Reserved | `W030`: Meter reverse connection |
| 31 | `E031`: Internal communication fault 4 (DSP & M8) | `W031`: EPS DCV high |

### Hold Register `21` (`FuncEn`) and Hold Register `110` (`FunctionEn1`) Bits

Always read the current 16-bit value first and modify individual bits before writing back.

| Bit | Hold Register `21` (`FuncEn`) | Hold Register `110` (`FunctionEn1`) |
|---|---|---|
| 0 | `EPSEn` — Off-grid (EPS) output enable | `ubPVGridOffEn` — PV grid-off enable |
| 1 | `OVFLoadDerateEn` — Over-frequency load derating | `ubFastZeroExport` — Fast zero-export / run without grid |
| 2 | `DRMSEn` — DRMS enable | `ubMicroGridEn` — Micro-grid enable |
| 3 | `LVRTEn` — Low-voltage ride-through enable | `ubBatShared` — Shared battery pack across parallel inverters |
| 4 | `AntiIslandEn` — Anti-islanding enable | `ubChgLastEn` — Charge-last enable (supply load/export before charging) |
| 5 | `NeutralDetectEn` — Ground/neutral detection enable | `CTSampleRatio` (bits 5–6): `0` = 1:1000, `1` = 1:3000 |
| 6 | `GridOnPowerSSEn` — Grid-connected power soft-start | *(part of `CTSampleRatio`)* |
| 7 | `ACChargeEn` — **Grid (AC) battery charging enable** | `BuzzerEn` (off-grid) / `DrycontactorCntlEn` (12K) |
| 8 | `SWSeamlesslyEn` — Seamless off-grid transfer enable | `PVCTSampleType` (bits 8–9): `0` = PV power, `1` = SpecLoad |
| 9 | `SetToStandby` — `0` = **Standby**, `1` = **Power on (Normal)** | *(part of `PVCTSampleType`)* |
| 10 | `ForcedDischgEn` — **Forced discharge enable** | `TakeLoadTogether` — Supply load from grid + battery together |
| 11 | `ForcedChgEn` — **Charge priority (PV charge first) enable** | `OnGridWorkingMode` — `0` = Self-consumption, `1` = Charge first |
| 12 | `ISOEn` — PV isolation check enable | `PVCTSampleRatio` (bits 12–13): `0` = 1:1000, `1` = 1:3000 |
| 13 | `GFCIEn` — Ground-fault current interrupter enable | *(part of `PVCTSampleRatio`)* |
| 14 | `DCIEn` — DC injection protection enable | `GreenModeEn` — Off-grid green mode |
| 15 | `FeedInGridEn` — **Grid export (feed-in) enable** | `EcoModeEn` — Off-grid eco mode |

---

## Battery CAN expectations (Inverter ↔ BMS)

On its battery CAN port (`500 kbit/s`, 11-bit standard CAN IDs, little-endian), a Luxpower / EG4 inverter configured for lithium CAN communication (`Lithium` → Pylon / standard CAN) expects the [Pylontech low-voltage CAN protocol](../PYLONTECH/README.md#low-voltage-can-protocol) broadcast by the BMS every ~1 s, and transmits `0x305` as a keep-alive:

| CAN ID | Dir | DLC | Payload (LE) | Mapped Luxpower Input Registers |
|---|---|---|---|---|
| `0x305` | Inv → BMS | 8 | Keep-alive (`00 00 00 00 00 00 00 00`) | — |
| `0x351` | BMS → Inv | 8 | Charge V limit (`u16`, 0.1 V), max charge I (`s16`, 0.1 A), max discharge I (`s16`, 0.1 A), discharge cut-off V (`u16`, 0.1 V) | Regs `83` (`ChargeVoltRef`), `81` (`MaxChgCurr`), `82` (`MaxDischgCurr`), `84` (`DischgCutVolt`) |
| `0x355` | BMS → Inv | 4 | SOC (`u16`, 1 %), SOH (`u16`, 1 %) | Reg `5` (`SOC` low byte, `SOH` high byte) |
| `0x356` | BMS → Inv | 6 | Pack voltage (`s16`, 0.01 V), pack current (`s16`, 0.1 A, `+` charge), pack temp (`s16`, 0.1 °C) | Regs `4` (`Vbat`), `98` (`BatCurrent_BMS`) |
| `0x359` | BMS → Inv | 7 | Protection flags (bytes 0–1), alarm flags (bytes 2–3), module count (`u8`, byte 4), `"PN"` (`0x50 0x4E`, bytes 5–6) | Regs `96` (`BatParallelNum`), `99` (`FaultCode_BMS`), `100` (`WarningCode_BMS`) |
| `0x35C` | BMS → Inv | 2 | Byte 0: bit 7 charge enable, bit 6 discharge enable, bit 5 force charge I, bit 4 force charge II, bit 3 full charge request | Reg `95` (`BatStatus_INV`) |
| `0x35E` | BMS → Inv | 8 | ASCII manufacturer string (e.g. `"PYLON   "`) | — |

### Luxpower / BYD CAN Extensions (`0x373` and `0x379`)

Standard Pylontech CAN (`0x351`–`0x35E`) does not report total pack capacity in Ah or individual cell voltage/temperature extremes. Luxpower inverters (such as the SNA and LXP series) additionally parse two extended CAN frames documented in [`ArminJo/JK-BMSToPylontechCAN` (`Pylontech_CAN.h`)](https://github.com/ArminJo/JK-BMSToPylontechCAN/blob/master/JK-BMSToPylontechCAN/Pylontech_CAN.h) and [`dfch/BydCanProtocol`](https://github.com/dfch/BydCanProtocol):

| CAN ID | Dir | DLC | Offset | Size | Type | Unit / scale | Field | Mapped Input Reg |
|---|---|---|---|---|---|---|---|---|
| `0x373` | BMS → Inv | 8 | 0 | 2 | `u16 LE` | 0.001 V (mV) | Minimum cell voltage | Reg `102` (`MinCellVolt_BMS`) |
| | | | 2 | 2 | `u16 LE` | 0.001 V (mV) | Maximum cell voltage | Reg `101` (`MaxCellVolt_BMS`) |
| | | | 4 | 2 | `u16 LE` | 1 K (°C + 273) | Minimum cell temperature | Reg `104` (`MinCellTemp_BMS`) |
| | | | 6 | 2 | `u16 LE` | 1 K (°C + 273) | Maximum cell temperature | Reg `103` (`MaxCellTemp_BMS`) |
| `0x379` | BMS → Inv | 8 | 0 | 2 | `u16 LE` | 1 Ah | Installed battery bank capacity | Reg `97` (`BatCapacity`) |
| | | | 2 | 6 | `u8[6]` | — | Reserved (`00 00 00 00 00 00`) | — |

---

## Control commands (dangerous)

> [!CAUTION]
> Writing holding registers (`0x06` / `0x10`) modifies live inverter operating state, grid protection relays, and battery charging limits. Frequent writes to holding registers can also wear out the inverter's non-volatile EEPROM/flash memory.

1. **System Reset,BMS Switch Override, and Reboot — Hold Register `11` (`ResetSetting`)**:
   - Bit 0 (`EnergyRecordClr`): Clears cumulative energy totals and runtime counters.
   - Bit 1 (`AlltoDefault`): **Factory-resets all system settings to defaults.**
   - Bit 2 (`AdjRatioClr`): Resets calibration correction coefficients to `1`.
   - Bit 3 (`FaultRecordClr`): Clears historical fault records.
   - Bit 4 (`MonitorData`): Clears offline monitoring data logs.
   - Bit 5 (`BMSChgSWOn`) / Bit 6 (`BMSDischgSWOn`): Turns on BMS charge / discharge switches (`EG4-18KPV-12LV-Modbus-Protocol.pdf`; see [Errata](#errata-and-disagreements) for `InvReboot` bit numbering).
   - Bit 7 (`InvReboot`): **Restarts the inverter immediately.**
2. **Power / Grid-Tie / Safety Toggles — Hold Register `21` (`FuncEn`)**:
   - Clearing bit 9 (`SetToStandby = 0`) places the inverter into **Standby**, dropping output.
   - Modifying bit 4 (`AntiIslandEn`), bit 5 (`NeutralDetectEn`), bit 12 (`ISOEn`), bit 13 (`GFCIEn`), bit 14 (`DCIEn`), or bit 15 (`FeedInGridEn`) alters grid-code safety protections or enables grid export.
3. **Grid Protection Trip Limits — Hold Registers `25–53`**:
   - Controls grid under/over-voltage and under/over-frequency trip points and trip times. Do not modify without utility authorization.
4. **Battery Voltage and Current Cut-offs — Hold Registers `99–102`, `132`, `144`, `148–149`, `158–169`, `227–228`**:
   - Setting `ChargeVoltRef` (`99`), `FloatChargeVolt` (`144`), `EqualizationVolt` (`149`), or `ACChgEndVolt` (`159`) above the battery pack's maximum safe voltage can overcharge an un-managed battery.

---

## Captured frames

From [`celsworth/lxp-bridge/tests/test_packet_tcp_frames.rs`](https://github.com/celsworth/lxp-bridge/blob/master/tests/test_packet_tcp_frames.rs) and [`integration_tests/fixtures/read_inputs_0_to_39_ok.packet.json`](https://github.com/celsworth/lxp-bridge/blob/master/integration_tests/fixtures/read_inputs_0_to_39_ok.packet.json) (dongle serial anonymized to `"2222222222"`, inverter serial `"5555555555"`):

```
# 1. Dongle TCP Heartbeat (TCP function 0xC1 = 193, 19 bytes, no CRC)
A1 1A 02 00 0D 00 01 C1 32 32 32 32 32 32 32 32 32 32 00

# 2. Read Holding Registers 12..14 (system time, 3 registers)
Request:  A1 1A 01 00 20 00 01 C2 32 32 32 32 32 32 32 32 32 32 12 00 00 03 35 35 35 35 35 35 35 35 35 35 0C 00 03 00 70 26
Response: A1 1A 02 00 25 00 01 C2 32 32 32 32 32 32 32 32 32 32 17 00 01 03 35 35 35 35 35 35 35 35 35 35 0C 00 06 16 06 14 05 10 39 5D 87
# Decoded values: reg 12 = [0x16, 0x06] (2022-06), reg 13 = [0x14, 0x05] (20th, 05h), reg 14 = [0x10, 0x39] (16m 57s) -> 2022-06-20 05:16:57

# 3. Read Input Registers 0..39 (40 registers, 80 payload bytes)
Request:  A1 1A 01 00 20 00 01 C2 32 32 32 32 32 32 32 32 32 32 12 00 00 04 35 35 35 35 35 35 35 35 35 35 00 00 28 00 2A 84
Response: A1 1A 02 00 6F 00 01 C2 32 32 32 32 32 32 32 32 32 32 61 00 01 04 35 35 35 35 35 35 35 35 35 35 00 00 50 10 00 00 00 00 00 02 00 10 02 53 00 00 2F 00 00 00 00 00 00 00 00 00 02 25 09 01 01 04 00 8C 13 F6 01 00 00 E7 00 E8 03 25 09 00 00 40 00 8C 13 00 00 00 00 00 00 00 00 00 00 00 00 00 00 16 00 16 00 15 00 16 00 00 00 00 00 DE 00 42 0E 81 0C 24 90
# Decoded values: State = 0x0010 (Battery On-grid), Vbat = 52.8 V (0x0210), SOC = 83 % (0x53), SOH = 0 %,
# Pdischarge = 512 W (0x0200), VacR = 234.1 V (0x0925), Fac = 50.04 Hz (0x138C), Pinv = 502 W (0x01F6),
# PF = 1.000 (0x03E8), VepsR = 234.1 V, Feps = 50.04 Hz, Echg_day = 2.1 kWh, Edischg_day = 2.2 kWh,
# Etouser_day = 22.2 kWh (0x00DE), Vbus1 = 365.0 V (0x0E42), Vbus2 = 320.1 V (0x0C81).
```

---

## Errata and disagreements

- **Input Registers `81` (`MaxChgCurr`) and `82` (`MaxDischgCurr`) scaling (`0.01 A` vs `0.1 A`)**: Both vendor PDFs specify `0.01 A` per LSB, and `celsworth/lxp-bridge` uses `/ 100.0` in `ReadInput3` (e.g. raw `6600` = `66.0 A` in `read_inputs_80_to_119_ok.decoded.json`). However, `ReadInputAll` in [`celsworth/lxp-bridge/src/lxp/packet.rs`](https://github.com/celsworth/lxp-bridge/blob/master/src/lxp/packet.rs) (used when protocol `5` returns 127 registers at once) mistakenly uses `le_u16_div10`, over-reporting BMS current limits by a factor of 10 (e.g. raw `5000` reported as `500.0 A` instead of `50.00 A`).
- **Input Register `98` (`BatCurrent_BMS`) signedness**: Both vendor PDFs state that `BatCurrent_BMS` is a **signed** 16-bit integer (`0.01 A`). In the protocol `5` capture in `test_packet_tcp_frames.rs`, register `98` contains `[0x4C, 0xFF]` (`-180` = `-1.80 A`), but `celsworth/lxp-bridge` parses it with `le_u16_div100` (unsigned), producing `653.56 A`.
- **Hold Register `11` (`ResetSetting`) bit positions**: `EG4-18KPV-12LV-Modbus-Protocol.pdf` (page 13) lists `BMSChgSWOn` at bit 5, `BMSDischgSWOn` at bit 6, and `InvReboot` at bit 7. In `6kXP-Modbus.updated.on.2023.10.28.pdf` (page 18), bits 4–6 are blank (`...`) and the next row (`InvReboot`) is misprinted as `Bit5` followed by `Bit6..Bit13` reserved. `ant0nkr/luxpower-ha-integration` confirms `InvReboot` is **Bit 7**.
- **Hold Registers `106–109` (`TemprLowerLimitDischg` etc.) signedness**: Both vendor PDFs print the range as `0–65536` (`0.1 °C`), but live hardware dumps in [`ant0nkr/luxpower-ha-integration/docs/protocol-registers.md`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/docs/protocol-registers.md) show raw `65336` (`0xFF38` = `-200` signed = `-20.0 °C`), confirming these registers are signed `s16`.
- **Register numbers in `jefflaplante/lux/PROTOCOL.md`**: The "Safety-Critical Registers" table at the bottom of `jefflaplante/lux/PROTOCOL.md` lists Hold registers `41–45`, `60–61`, and `96` as battery charge/discharge cut-offs and `WorkMode`. In the vendor Luxpower/EG4 Modbus register maps (and `lxp-bridge`), Hold registers `41–45` are grid voltage/frequency protection limits, `60–61` are active/reactive power percentages, and `96` is `DelayTimeForQVCurve`.
- **Input Register `193` (`PF_T` vs `GridVolt_L1N`)**: `6kXP-Modbus.updated.on.2023.10.28.pdf` lists three-phase `PF_T` at Input Register `193` (followed by a duplicate `192 PF_S` row), whereas US split-phase firmware places `GridVolt_L1N` at `193` (`193–204` split-phase metrics) and moves `PF_T` to `205`.

---

## Sources

- [Shenzhen Lux Power Technology Co., Ltd — *Modbus RTU Protocol* (V58, 2023-10-28)](PDFs/6kXP-Modbus.updated.on.2023.10.28.pdf) ([source attachment in `celsworth/lxp-bridge` discussion #273](https://github.com/user-attachments/files/16330919/6kXP-Modbus.updated.on.2023.10.28.pdf)) — 42-page official register map, RTU frame tables, operating modes, and fault/warning codes.
- [*Modbus RTU Protocol — EG4 18KPV-12LV* (2023-07)](PDFs/EG4-18KPV-12LV-Modbus-Protocol.pdf) ([mirror at `dth.net`](https://www.dth.net/solar/luxpower/modbus/EG4-18KPV-12LV-Modbus-Protocol.pdf)) — 28-page protocol manual for the EG4 18kPV-12LV / Luxpower 12K hybrid platform.
- [`celsworth/lxp-bridge`](https://github.com/celsworth/lxp-bridge) — [`src/lxp/packet.rs`](https://github.com/celsworth/lxp-bridge/blob/master/src/lxp/packet.rs), [`src/lxp/packet_decoder.rs`](https://github.com/celsworth/lxp-bridge/blob/master/src/lxp/packet_decoder.rs), [`doc/LXP_REGISTERS.txt`](https://github.com/celsworth/lxp-bridge/blob/master/doc/LXP_REGISTERS.txt), [`doc/inputs.md`](https://github.com/celsworth/lxp-bridge/blob/master/doc/inputs.md), [`tests/test_packet_tcp_frames.rs`](https://github.com/celsworth/lxp-bridge/blob/master/tests/test_packet_tcp_frames.rs), and [`integration_tests/fixtures/`](https://github.com/celsworth/lxp-bridge/tree/master/integration_tests/fixtures) (dongle TCP framing, VLB rules, and captured test packets).
- [`ant0nkr/luxpower-ha-integration`](https://github.com/ant0nkr/luxpower-ha-integration) — [`docs/protocol-registers.md`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/docs/protocol-registers.md), [`custom_components/lxp_modbus/constants/input_registers.py`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/custom_components/lxp_modbus/constants/input_registers.py), [`hold_registers.py`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/custom_components/lxp_modbus/constants/hold_registers.py), [`battery_registers.py`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/custom_components/lxp_modbus/constants/battery_registers.py), and [`classes/lxp_response.py`](https://github.com/ant0nkr/luxpower-ha-integration/blob/main/custom_components/lxp_modbus/classes/lxp_response.py) (extended registers, 5000+ battery blocks, and live hardware dump verification).
- [`jefflaplante/lux`](https://github.com/jefflaplante/lux) — [`PROTOCOL.md`](https://github.com/jefflaplante/lux/blob/main/PROTOCOL.md) (TCP/8000 unsolicited broadcast cycle on EG4 18kPV).
- [`ArminJo/JK-BMSToPylontechCAN`](https://github.com/ArminJo/JK-BMSToPylontechCAN) (`Pylontech_CAN.h`), [`dfch/BydCanProtocol`](https://github.com/dfch/BydCanProtocol) (`BatterySize.h`, `MessageId.h`), and [`dalathegreat/Battery-Emulator`](https://github.com/dalathegreat/Battery-Emulator) (`PYLON-LV-CAN.cpp`) — Pylontech low-voltage CAN frames and `0x373` / `0x379` Luxpower/BYD CAN extensions.

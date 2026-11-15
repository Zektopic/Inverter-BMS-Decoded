# JBD (Jiabaida / Xiaoxiang / Overkill Solar) Smart BMS — UART / RS485 / BLE Protocol

> Status: 📄 Vendor doc — transcribed from Jiabaida *General Protocol V4* and *Software Board Common Protocol V8 / V11*, cross-checked against [FurTrader/OverkillSolarBMS](https://github.com/FurTrader/OverkillSolarBMS), [syssi/esphome-jbd-bms](https://github.com/syssi/esphome-jbd-bms), and [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery).

## Summary

| | |
|---|---|
| Products | JBD (Dongguan Jiabaida Electronic Technology Co., Ltd.) smart BMS boards (`JBD-SP04S…`, `JBD-DP04S…`, `JBD-AP20S…`, `JBD-SP16S…`, etc.), sold under **JBD**, **LLT Power**, **Overkill Solar**, **Xiaoxiang BMS**, and inside many drop-in LiFePO4 batteries (Eco-Worthy, LionTron, Vatrer, Supervolt, LANPWR, CHINS) |
| Physical layer | **UART** (3.3 V / 5 V TTL, 4-pin JST-XH: `GND`, `RXD`, `TXD`, `V+`), isolated **RS485** / **RS232** adapter boxes, or **BLE** UART bridge module |
| Settings | **9600 8N1** (9600 bps, 8 data bits, no parity, 1 stop bit; custom baud rates optional) |
| BLE GATT | Service UUID `0xFF00` (`0000ff00-0000-1000-8000-00805f9b34fb`); **Notify/RX** characteristic `0xFF01` (handle `0x0011`), **Write/TX** characteristic `0xFF02` (handle `0x0015`, Write or Write Without Response). Frames longer than the BLE MTU (such as `0x03` basic info) arrive split across 2 consecutive `0xFF01` notifications |
| Addressing | None in standard V4/V11 frame format (point-to-point master/slave; `dbus-serialbattery` probes on pseudo-address `0x00`) |
| Roles | Host is master and polls the BMS; the BMS never transmits unsolicited frames. If the BMS is asleep, the first frame may be ignored and must be resent. Wait ≥ 50 ms between consecutive requests |
| Byte order | **Big-endian (`BE`)** for all multi-byte integers and the 16-bit checksum |

---

## Framing

Every frame begins with `0xDD` and ends with `0x77`. The second byte indicates direction (`0xA5` read, `0x5A` write) in host-to-BMS requests, and echoes the register/command code in BMS-to-host responses.

### Host → BMS request

| Offset | Size | Value | Field |
|---|---|---|---|
| `0` | 1 | `0xDD` | Start of frame (`SOF`) |
| `1` | 1 | `0xA5` = Read, `0x5A` = Write | Direction / status byte |
| `2` | 1 | `cmd` | Register address / command code |
| `3` | 1 | `len` (`N`) | Data payload length in bytes (`0x00` for standard read requests) |
| `4` | `N` | … | Data payload (omitted when `len == 0`) |
| `4 + N` | 2 | `u16 BE` | Checksum over bytes `2 .. 3 + N` (`cmd`, `len`, and `data`) |
| `6 + N` | 1 | `0x77` | End of frame (`EOF`) |

### BMS → Host response

| Offset | Size | Value | Field |
|---|---|---|---|
| `0` | 1 | `0xDD` | Start of frame (`SOF`) |
| `1` | 1 | `cmd` | Register address / command code echoed from request |
| `2` | 1 | `status` | `0x00` = OK; non-zero = error (`0x80` unsupported/failed, `0x81` invalid operation / not in factory mode, `0x82` verify error, `0x83` password error) |
| `3` | 1 | `len` (`N`) | Data payload length in bytes (`0x00` on write acknowledgement or error) |
| `4` | `N` | … | Data payload (omitted when `len == 0`) |
| `4 + N` | 2 | `u16 BE` | Checksum over bytes `2 .. 3 + N` (`status`, `len`, and `data`) |
| `6 + N` | 1 | `0x77` | End of frame (`EOF`) |

> **Callback ID extension (Protocol V3+):** The V8/V11 vendor specification notes that up to 4 optional `CALLBACK_ID` bytes may be appended after `0x77` by the host and echoed back by the slave. Standard implementations (`esphome-jbd-bms`, `dbus-serialbattery`, Overkill Solar) omit this.

### Checksum algorithm and worked examples

In both directions, the 16-bit big-endian checksum is the **16-bit two's complement** of the sum of all bytes from **frame offset 2** up to the end of the data payload (i.e. excluding `0xDD` at byte 0 and the direction/command byte at byte 1):

$$\text{checksum} = \left( \text{0x10000} - \sum_{i=2}^{3+N} \text{frame}[i] \right) \bmod \text{0x10000}$$

```python
def jbd_checksum(frame_bytes_from_offset_2_to_end_of_data: bytes) -> int:
    return (0x10000 - sum(frame_bytes_from_offset_2_to_end_of_data)) & 0xFFFF
```

#### Worked example 1 — Read basic info (`0x03`) request

```
DD A5 03 00 FF FD 77
```

- Bytes covered by checksum (offset 2..3): `03 00`
- Sum: `0x03 + 0x00 = 0x0003`
- Two's complement: `0x10000 - 0x0003 = 0xFFFD` → transmitted big-endian as `FF FD`.

#### Worked example 2 — Read cell voltages (`0x04`) 4S response (from `syssi/esphome-jbd-bms`)

```
DD 04 00 08 0F 45 0F 3D 0F 37 0F 3D FE C6 77
```

- Byte 0: `DD` (start), Byte 1: `04` (command `0x04`, **not** included in response checksum)
- Bytes covered by checksum (offset 2..11): `00 08 0F 45 0F 3D 0F 37 0F 3D`
- Sum: `0x00 + 0x08 + 0x0F + 0x45 + 0x0F + 0x3D + 0x0F + 0x37 + 0x0F + 0x3D = 0x013A` (314)
- Two's complement: `0x10000 - 0x013A = 0xFEC6` → transmitted big-endian as `FE C6`.

---

## Requests and responses / frame IDs

| Command / Reg | Direction | Request frame (read) | Payload | Notes |
|---|---|---|---|---|
| `0x03` | Read (`0xA5`) | `DD A5 03 00 FF FD 77` | `23 + 2×N` bytes (+ optional 9 bytes in V8+) | Basic pack info, SOC, balancing, protections, FET status, NTC temperatures |
| `0x04` | Read (`0xA5`) | `DD A5 04 00 FF FC 77` | `2 × cells` bytes | Individual cell voltages in mV |
| `0x05` | Read (`0xA5`) | `DD A5 05 00 FF FB 77` | `1..31` bytes | Hardware version / device model ASCII string |
| `0xAA` | Read (`0xA5`) | `DD A5 AA 00 FF 56 77` | `22` or `24` bytes | Historical protection event counters (`11 × u16` or `12 × u16`) |
| `0xF6` | R/W (`0xA5`/`0x5A`) | `DD A5 F6 00 FF 0A 77` | `2 × N` bytes (`30 × s16` max) | Cell internal resistance compensation in `0.1 mΩ` *(V8/V11; not supported on all firmware)* |
| `0x00` | Read (`0xA5`) | `DD A5 00 00 FF 00 77` | `2` bytes (`0x00 XX`) | Read AFE chip type (`0` TI BQ769x0, `1` O2Micro OZ7717, `2` Panasonic AN49502, `3` SinoWealth SH367309, `4` SH367303, `5` Vasense DC10xx, `6` OZ3714) *(V4+)* |
| `0x00` | Write (`0x5A`) | — | `56 78` (2 bytes) | **Enter factory (EEPROM) mode** |
| `0x01` | Write (`0x5A`) | — | `00 00` or `28 28` (2 bytes) | **Exit factory mode** (`0x0000` discard / no NVM reset; `0x2828` save EEPROM & zero `0xAA` counters) |
| `0x10`–`0xDF` | R/W (`0xA5`/`0x5A`) | `DD A5 <reg> 00 <crc> 77` | 2 bytes (`u16`/`s16`) or string | EEPROM configuration & calibration registers (require factory mode `0x00` first) |
| `0xFA` | R/W (`0xA5`/`0x5A`) | `DD A5 FA 03 <id_hi> <id_lo> <cnt> <crc> 77` | `3 + 2×cnt` bytes | Bulk parameter block read/write by parameter index (`0`–`183`) *(V8/V11)* |
| `0xE0` | Write (`0x5A`) | — | 2 bytes (`u16`) | **Control:** Force remaining capacity |
| `0xE1` | Write (`0x5A`) | — | 2 bytes (`0x0000`–`0x0003`) | **Control:** Software MOSFET lock/unlock |
| `0xE2` | Write (`0x5A`) | — | 2 bytes (`0x0001`–`0x0003`) | **Control:** Balancer test control (odd / even / off) |
| `0x0A` | Write (`0x5A`) | — | 2 bytes (`0x0100`–`0x0C00`) | **Control:** Action command (reset SOC, clear history, MCU reset, sleep, power-down, auto-balance, storage mode, force heating) *(V8/V11)* |
| `0xFB` | Write (`0x5A`) | — | 2 bytes (`YY XX`) | **Control:** Alternative MOSFET control command *(V11)* |
| `0xFC` | Write (`0x5A`) | — | 5 bytes | **Control:** Heating schedule / trigger *(V5/V11)* |

---

### `0x03` Basic info and status (`JBD_CMD_BASICINFO`)

Request: `DD A5 03 00 FF FD 77`.
Response header: `DD 03 00 <len>`, followed by `<len>` data bytes (minimum `23 + 2 × ntc_cnt` bytes):

| Data offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0x00` (0) | 2 | `u16 BE` | `0.01 V` (`10 mV`) | Total pack voltage (`pack_mv`) |
| `0x02` (2) | 2 | `s16 BE` | `0.01 A` (`10 mA`) or `0.1 A`¹ | Pack current (`+` = charging, `−` = discharging) |
| `0x04` (4) | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) or `0.1 Ah`¹ | Remaining (residual) capacity (`cycle_cap`) |
| `0x06` (6) | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) or `0.1 Ah`¹ | Nominal (full/design) capacity (`design_cap`) |
| `0x08` (8) | 2 | `u16 BE` | `1 cycle` | Charge/discharge cycle count (`cycle_cnt`) |
| `0x0A` (10) | 2 | `u16 BE` | bitfield | Production date: bits `15:9` = `year − 2000`, bits `8:5` = `month` (1–12), bits `4:0` = `day` (1–31) |
| `0x0C` (12) | 2 | `u16 BE` | bitmask | Balance status low (`bal_low`): bit 0 = cell 1 … bit 15 = cell 16 (`1` = balancing active) |
| `0x0E` (14) | 2 | `u16 BE` | bitmask | Balance status high (`bal_high`): bit 0 = cell 17 … bit 15 = cell 32 (`1` = balancing active) |
| `0x10` (16) | 2 | `u16 BE` | bitmask | Protection / error status bits (see [Alarms and status bits](#alarms-and-status-bits)) |
| `0x12` (18) | 1 | `u8` | BCD nibbles | Software version: `(byte >> 4) + (byte & 0x0F) × 0.1` (e.g. `0x10` = `1.0`, `0x80` = `8.0`) |
| `0x13` (19) | 1 | `u8` | `1 %` | Remaining state of charge (`RSOC`, 0–100 %) |
| `0x14` (20) | 1 | `u8` | bitmask | FET / operation status (see [FET and operation status byte](#fet-and-operation-status-byte-0x03-offset-0x14)) |
| `0x15` (21) | 1 | `u8` | `1 cell` | Number of series cells (`cell_cnt`, 1–32) |
| `0x16` (22) | 1 | `u8` | `1 sensor` | Number of NTC temperature sensors (`N`) |
| `0x17` (23) | `2 × N` | `N × u16 BE` | `0.1 K` | NTC temperatures `1..N`: `°C = (raw − 2731) / 10` (e.g. `2981` = `25.0 °C`) |

¹ **High-current / high-capacity unit scaling (Protocol V9+):** When **bit 7 (`0x80`) of data byte `0x14` (offset 20, FET status)** is `1` (corresponding to `func_config` register `0x2D` bit 12), the LSB unit for current (`0x02`), remaining capacity (`0x04`), and nominal capacity (`0x06`) changes from **`10 mA` / `10 mAh` (`0.01`)** to **`100 mA` / `100 mAh` (`0.1`)** so packs above 655.35 Ah / 327.67 A do not overflow 16-bit integers.

#### Extended `0x03` trailing fields (Protocol V8 / V11 firmware)

Some newer firmware versions append 9 bytes immediately after the `2 × N` NTC temperature bytes (starting at data offset `23 + 2 × N`):

| Data offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `23 + 2×N` | 1 | `u8` | `1 %` | Humidity |
| `24 + 2×N` | 2 | `u16 BE` | bitmask | Alarm status (pre-protection warnings; see below) |
| `26 + 2×N` | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) | Full charge capacity |
| `28 + 2×N` | 2 | `u16 BE` | `0.01 Ah` (`10 mAh`) | Remaining capacity |
| `30 + 2×N` | 2 | `u16 BE` | `1 mA` | Balancing current |

---

### `0x04` Cell voltages (`JBD_CMD_CELLINFO`)

Request: `DD A5 04 00 FF FC 77`.
Response header: `DD 04 00 <len>`, where `<len>` = `2 × cell_cnt`.

| Data offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0x00` (0) | 2 | `u16 BE` | `0.001 V` (`1 mV`) | Cell 1 voltage (e.g. `0x0F66` = `3942 mV` = `3.942 V`) |
| `0x02` (2) | 2 | `u16 BE` | `0.001 V` (`1 mV`) | Cell 2 voltage |
| `2 × (k − 1)` | 2 | `u16 BE` | `0.001 V` (`1 mV`) | Cell `k` voltage (`k = 1 .. cell_cnt`) |

---

### `0x05` Hardware version / device model (`JBD_CMD_HWVER`)

Request: `DD A5 05 00 FF FB 77`.
Response header: `DD 05 00 <len>`, where `<len>` (`1..31`) is the string length.

| Data offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0x00` (0) | `len` | ASCII | — | Hardware version / model string (e.g. `"JBD-SP04S034-L4S-200A-B-U"`) |

---

### `0xAA` Historical protection counts (`JBD_CMD_ERROR_COUNTS`)

Request: `DD A5 AA 00 FF 56 77`.
Response header: `DD AA 00 <len>`, where `<len>` is `0x16` (22 bytes, older firmware) or `0x18` (24 bytes, V8/V11 firmware). Can be read without entering factory mode.

| Data offset | Size | Type | Field |
|---|---|---|---|
| `0x00` (0) | 2 | `u16 BE` | Short-circuit protection count (`sc_err_cnt`) |
| `0x02` (2) | 2 | `u16 BE` | Charge over-current protection count (`chgoc_err_cnt`) |
| `0x04` (4) | 2 | `u16 BE` | Discharge over-current protection count (`dsgoc_err_cnt`) |
| `0x06` (6) | 2 | `u16 BE` | Cell over-voltage protection count (`covp_err_cnt`) |
| `0x08` (8) | 2 | `u16 BE` | Cell under-voltage protection count (`cuvp_err_cnt`) |
| `0x0A` (10) | 2 | `u16 BE` | Charge over-temperature protection count (`chgot_err_cnt`) |
| `0x0C` (12) | 2 | `u16 BE` | Charge under-temperature protection count (`chgut_err_cnt`) |
| `0x0E` (14) | 2 | `u16 BE` | Discharge over-temperature protection count (`dsgot_err_cnt`) |
| `0x10` (16) | 2 | `u16 BE` | Discharge under-temperature protection count (`dsgut_err_cnt`) |
| `0x12` (18) | 2 | `u16 BE` | Pack over-voltage protection count (`povp_err_cnt`) |
| `0x14` (20) | 2 | `u16 BE` | Pack under-voltage protection count (`puvp_err_cnt`) |
| `0x16` (22) | 2 | `u16 BE` | System restart count *(present only when `len == 24`)* |

---

### EEPROM register map (requires Factory Mode)

To read or write EEPROM configuration registers (`0x10`–`0xDF`), the host must first enter **Factory Mode** by writing `0x5678` to register `0x00`, perform the register reads (`0xA5`) or writes (`0x5A`), and then exit Factory Mode by writing `0x0000` (discard / read-only exit) or `0x2828` (commit to EEPROM and reset `0xAA` error counters) to register `0x01`.

| Reg | Name | Type | Unit / scale | Description |
|---|---|---|---|---|
| `0x10` | `design_cap` | `u16 BE` | `10 mAh` (`100 mAh`¹) | Nominal / design pack capacity |
| `0x11` | `cycle_cap` | `u16 BE` | `10 mAh` (`100 mAh`¹) | Cycle capacity |
| `0x12` | `cap_100` | `u16 BE` | `1 mV` | Full-charge cell voltage (100 % SOC reference) |
| `0x13` | `cap_0` | `u16 BE` | `1 mV` | End-of-discharge cell voltage (0 % SOC reference) |
| `0x14` | `dsg_rate` | `u16 BE` | `0.1 %` | Cell self-discharge rate |
| `0x15` | `mfg_date` | `u16 BE` | bitfield | Manufacture date (`year−2000` in bits `15:9`, `month` `8:5`, `day` `4:0`) |
| `0x16` | `serial_num` | `u16 BE` | — | Serial number |
| `0x17` | `cycle_cnt` | `u16 BE` | `1 cycle` | Cycle count |
| `0x18` / `0x19` | `chgot` / `chgot_rel` | `u16 BE` | `0.1 K` | Charge over-temperature trip / release threshold (`°C = (raw − 2731)/10`) |
| `0x1A` / `0x1B` | `chgut` / `chgut_rel` | `u16 BE` | `0.1 K` | Charge under-temperature trip / release threshold |
| `0x1C` / `0x1D` | `dsgot` / `dsgot_rel` | `u16 BE` | `0.1 K` | Discharge over-temperature trip / release threshold |
| `0x1E` / `0x1F` | `dsgut` / `dsgut_rel` | `u16 BE` | `0.1 K` | Discharge under-temperature trip / release threshold |
| `0x20` / `0x21` | `povp` / `povp_rel` | `u16 BE` | `10 mV` | Pack over-voltage trip / release threshold |
| `0x22` / `0x23` | `puvp` / `puvp_rel` | `u16 BE` | `10 mV` | Pack under-voltage trip / release threshold |
| `0x24` / `0x25` | `covp` / `covp_rel` | `u16 BE` | `1 mV` | Cell over-voltage trip / release threshold |
| `0x26` / `0x27` | `cuvp` / `cuvp_rel` | `u16 BE` | `1 mV` | Cell under-voltage trip / release threshold |
| `0x28` | `chgoc` | `s16 BE` | `10 mA` (`100 mA`¹) | Charge over-current threshold (positive number) |
| `0x29` | `dsgoc` | `s16 BE` | `10 mA` (`100 mA`¹) | Discharge over-current threshold (**negative** two's complement number) |
| `0x2A` | `bal_start` | `u16 BE` | `1 mV` | Cell balancing start voltage |
| `0x2B` | `bal_window` | `u16 BE` | `1 mV` | Cell balancing differential threshold (`delta mV`) |
| `0x2C` | `shunt_res` | `u16 BE` | `0.1 mΩ` (`0.01 mΩ`¹) | Current shunt resistance |
| `0x2D` | `func_config` | `u16 BE` | bitmask | Function configuration bits (see table below) |
| `0x2E` | `ntc_config` | `u16 BE` | bitmask | Enabled NTC thermistors (bit 0 = NTC 1 … bit 7 = NTC 8) |
| `0x2F` | `cell_cnt` | `u16 BE` | `1 cell` | Number of series cells |
| `0x30` / `0x31` | `fet_time` / `led_time` | `u16 BE` | `1 s` | Switch control time / LED timer |
| `0x32`–`0x35` | `cap_80`..`cap_20` | `u16 BE` | `1 mV` | Cell voltage for 80 % (`0x32`), 60 % (`0x33`), 40 % (`0x34`), 20 % (`0x35`) SOC |
| `0x36` / `0x37` | `covp_high` / `cuvp_high` | `u16 BE` | `1 mV` | Hardware (secondary AFE) cell over-voltage / under-voltage threshold |
| `0x38` | `sc_dsgoc2` | `2 × u8` | AFE-specific | Hardware short-circuit & secondary discharge over-current (TI BQ769x0 or AFE lookup table) |
| `0x39` | `cxvp_high_delay_sc_rel` | `2 × u8` | bitfield / `1 s` | Byte 0: hardware UVP/OVP delays; Byte 1: short-circuit release time (s) |
| `0x3A` | `chg_t_delays` | `2 × u8` | `1 s` | Byte 0: charge under-temp delay; Byte 1: charge over-temp delay |
| `0x3B` | `dsg_t_delays` | `2 × u8` | `1 s` | Byte 0: discharge under-temp delay; Byte 1: discharge over-temp delay |
| `0x3C` | `pack_v_delays` | `2 × u8` | `1 s` | Byte 0: pack under-voltage delay; Byte 1: pack over-voltage delay |
| `0x3D` | `cell_v_delays` | `2 × u8` | `1 s` | Byte 0: cell under-voltage delay; Byte 1: cell over-voltage delay |
| `0x3E` | `chgoc_delays` | `2 × u8` | `1 s` | Byte 0: charge over-current delay; Byte 1: charge over-current release time |
| `0x3F` | `dsgoc_delays` | `2 × u8` | `1 s` | Byte 0: discharge over-current delay; Byte 1: discharge over-current release time |
| `0x40` / `0x41` | `gps_off` / `gps_off_time` | `u16 BE` | `1 mV` / `1 s` | GPS cutoff voltage and delay |
| `0x42`–`0x47` | `cap_90`..`cap_10`, `cap_100_v2` | `u16 BE` | `1 mV` | Cell voltage for 90 % (`0x42`), 70 % (`0x43`), 50 % (`0x44`), 30 % (`0x45`), 10 % (`0x46`), 100 % (`0x47`, FW ≥ `0x20`) |
| `0xA0` / `0xA1` / `0xA2` | `mfg_name` / `device_name` / `barcode` | string | ASCII | Length-prefixed ASCII string (`byte 0` = string length `n`, `bytes 1..n` = characters) |
| `0xAD` / `0xAE` / `0xAF` | `cal_cur_idle` / `chg` / `dsg` | `u16 BE` | — / `10 mA` | Current calibration: idle (`0x0000`), charge actual current, discharge actual current |
| `0xB0`–`0xCF` | `cal_v_cell_01`..`32` | `u16 BE` | `1 mV` | Cell 1–32 voltage calibration (write actual measured cell mV) |
| `0xD0`–`0xD7` | `cal_t_ntc_0`..`7` | `u16 BE` | `0.1 K` | NTC 1–8 temperature calibration (write actual measured `0.1 K`) |

¹ Scaled by 10× (`100 mAh` / `100 mA` / `0.01 mΩ`) when `func_config` (`0x2D`) bit 12 is `1` (or bit 15 of `0x2C` is `1` in V8/V11). For the complete V8/V11 `0xFA` parameter index table (indices `0`–`183`), see [`PDFs/Translated_to_english_JBD_comm_ptotocol_20230321_V11.pdf`](PDFs/Translated_to_english_JBD_comm_ptotocol_20230321_V11.pdf) (pages 13–18).

---

## Alarms and status bits

### Protection status (`0x03` data offset `0x10`, `u16 BE`)

Bit value `0` = normal, `1` = protection active:

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x0001` | Cell over-voltage protection (`covp_err`) |
| 1 | `0x0002` | Cell under-voltage protection (`cuvp_err`) |
| 2 | `0x0004` | Pack over-voltage protection (`povp_err`) |
| 3 | `0x0008` | Pack under-voltage protection (`puvp_err`) |
| 4 | `0x0010` | Charge over-temperature protection (`chgot_err`) |
| 5 | `0x0020` | Charge under-temperature protection (`chgut_err`) |
| 6 | `0x0040` | Discharge over-temperature protection (`dsgot_err`) |
| 7 | `0x0080` | Discharge under-temperature protection (`dsgut_err`) |
| 8 | `0x0100` | Charge over-current protection (`chgoc_err`) |
| 9 | `0x0200` | Discharge over-current protection (`dsgoc_err`) |
| 10 | `0x0400` | Short-circuit protection (`sc_err`) |
| 11 | `0x0800` | Front-end detection IC (AFE) error (`afe_err`) |
| 12 | `0x1000` | Software MOSFET lock active (`software_err`, see `0xE1`) |
| 13 | `0x2000` | Charge timeout close (`esphome-jbd-bms`) / Charge MOSFET breakdown flag (V11 doc) |
| 14 | `0x4000` | Discharge MOSFET breakdown flag (V11 doc; reserved in V4) |
| 15 | `0x8000` | Reserved |

### FET and operation status byte (`0x03` data offset `0x14`, `u8`)

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x01` | Charge MOSFET state (`1` = ON / conducting, `0` = OFF / open) |
| 1 | `0x02` | Discharge MOSFET state (`1` = ON / conducting, `0` = OFF / open) |
| 2 | `0x04` | Current-limiting module active (`1` = ON, V8/V11 doc) |
| 3 | `0x08` | Heating active (`1` = heating ON, V8/V11 doc & `esphome-jbd-bms`) |
| 4 | `0x10` | Heating active (`esphome-jbd-bms` checks `0x18` = bit 3 or bit 4) |
| 6 | `0x40` | Factory mode active (`esphome-jbd-bms`) |
| 7 | `0x80` | Capacity / current unit flag (`0` = `10 mA` / `10 mAh`, `1` = `100 mA` / `100 mAh`) |

### Function configuration bits (`0x2D` `func_config`, `u16 BE`)

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x0001` | Weak-current switch (`switch`) |
| 1 | `0x0002` | Short-circuit load release lock (`scrl` / `LOAD_EN`) |
| 2 | `0x0004` | Balancer enable (`balance_en`) |
| 3 | `0x0008` | Charge-only balancing (`chg_balance_en`; requires `balance_en = 1`) |
| 4 | `0x0010` | LED capacity indicator enable (`led_en`) |
| 5 | `0x0020` | LED mode (`led_num`) |
| 6 | `0x0040` | FCC learning restriction (`V8/V11` doc; `dbus-serialbattery` maps bit 6/7 as `RTC` / `EDV`) |
| 7 | `0x0080` | RTC / history recording enable |
| 8 | `0x0100` | Charging handshake enable (`V8/V11` doc) / GPS board enable (`dbus-serialbattery`) |
| 9 | `0x0200` | GPS function (`V8/V11` doc) / Buzzer enable (`dbus-serialbattery`) |
| 10 | `0x0400` | Buzzer function (`V8/V11` doc) |
| 11 | `0x0800` | Starter battery mode (`V8/V11` doc) |
| 12 | `0x1000` | High current/capacity unit multiplier (`0` = `10 mA`/`10 mAh`, `1` = `100 mA`/`100 mAh`) |

---

## Control commands (dangerous)

> **Warning:** Writing to these registers switches main pack MOSFETs under load, alters protection thresholds in EEPROM, resets state-of-charge, or puts the BMS into deep sleep requiring a charger to wake. Do not send write commands (`0x5A`) to a live pack unless you explicitly intend to control it.

### 1. Factory / EEPROM mode (`0x00` and `0x01`)

- **Enter Factory Mode (`0x00`):** Write `0x5678` (`DD 5A 00 02 56 78 FF 30 77`). Required before reading or writing EEPROM registers (`0x10`–`0xDF`) or `0xFA`.
- **Exit Factory Mode without saving (`0x01`):** Write `0x0000` (`DD 5A 01 02 00 00 FF FD 77`).
- **Exit Factory Mode, save EEPROM, and clear `0xAA` history (`0x01`):** Write `0x2828` (`DD 5A 01 02 28 28 FF AD 77`). Repeatedly saving EEPROM wears non-volatile memory.

### 2. Software MOSFET control (`0xE1` and V11 `0xFB`)

Register `0xE1` controls the charge and discharge MOSFETs directly (without entering factory mode, though `syssi/esphome-jbd-bms` captures show the official app also sends `0x01` exit-factory `0x0000` afterwards).
**Inverted bit logic:** In `0xE1`, **bit set (`1`) = force MOSFET OFF (disabled)**, whereas in `0x03` offset `0x14` bit set (`1`) = MOSFET ON.

| Value (`u16 BE`) | Bit 1 (Discharge) | Bit 0 (Charge) | Action | Verified frame |
|---|---|---|---|---|
| `0x0000` | `0` (ON) | `0` (ON) | Both charge & discharge MOSFETs enabled (normal) | `DD 5A E1 02 00 00 FF 1D 77` |
| `0x0001` | `0` (ON) | `1` (OFF) | Software-lock **charge MOSFET OFF**, discharge ON | `DD 5A E1 02 00 01 FF 1C 77` |
| `0x0002` | `1` (OFF) | `0` (ON) | Software-lock **discharge MOSFET OFF**, charge ON | `DD 5A E1 02 00 02 FF 1B 77` |
| `0x0003` | `1` (OFF) | `1` (OFF) | Software-lock **both MOSFETs OFF** | `DD 5A E1 02 00 03 FF 1A 77` |

 BMS acknowledgment: `DD E1 00 00 00 00 77` (note: see [Errata](#errata-and-disagreements) regarding `00 00` checksum on write acknowledgments).
- **V11 `0xFB` alternative MOSFET command:** Payload `YY XX` (2 bytes), where `YY` selects the target (`0x00` discharge MOS, `0x01` charge MOS, `0x03` pre-discharge MOS, `0x0A` both) and `XX` is `0x00` (unlock/ON) or `0x01` (software close/OFF).

### 3. Capacity, balancer, heating, and action commands (`0xE0`, `0xE2`, `0x0A`, `0xFC`)

- **`0xE0` Set remaining capacity:** Write `u16 BE` remaining capacity in `10 mAh` (`100 mAh` if unit flag set).
- **`0xE2` Balancer test mode:** Write `0x0001` (balance odd cells), `0x0002` (balance even cells), or `0x0003` (close all cells). Enter and exit factory mode to return to automatic balancing.
- **`0x0A` System action command (V8/V11):** Write `u16 BE` sub-command:
  - `0x0100`: Reset capacity / re-estimate SOC from cell voltages (`DD 5A 0A 02 01 00 FF F3 77`)
  - `0x0200`: Clear historical protection counters
  - `0x0300`: Reset BMS microcontroller
  - `0x0400`: Clear active protection and alarm status (`DD 5A 0A 02 04 00 FF F0 77`)
  - `0x0500`: Enter sleep mode
  - `0x0600`: Enter ultra-low-power shutdown mode (requires charging to wake)
  - `0x0700`: Trigger automatic balancing (`DD 5A 0A 02 07 00 FF ED 77`)
  - `0x0800`: Enter storage and transportation mode
  - `0x0C00`: Force heating (stops automatically at 15 °C)
- **`0xFC` Heating control (V5/V11):** Write 5 bytes `XX HH MM ZZ WW` (`XX`: `0x01` start, `0x02` stop; `HH`: hours `0..255`; `MM`: minutes `0..60`; `ZZ`: start temp `s8` °C; `WW`: stop temp `s8` °C).

### 4. Password and BLE module commands (`0x06`–`0x09` and `FF AA`)

- **UART/EEPROM password registers (firmware ≥ `0x16`):** `0x06` (`use_password`, payload `06` + 6 ASCII bytes), `0x07` (`set_password`, payload `0C` + 6 old + 6 new ASCII bytes), `0x09` (`clear_password`, master reset payload `06` + `"J1B2D4"`).
- **BLE module `FF AA` commands:** Frames starting with `FF AA <cmd> <len> <data...> <sum8>` (where `<sum8>` is `(cmd + len + sum(data)) & 0xFF`) are intercepted by the Bluetooth module or BLE authentication layer:
  - `0x07`: Rename BLE advertising name (e.g. `FF AA 07 03 58 59 5A 15` sets name to `"XYZ"`).
  - `0x15` / `0x17` / `0x18` / `0x1D`: BLE connection authentication flow documented in `syssi/esphome-jbd-bms` (`components/jbd_bms_ble/jbd_bms_ble.cpp`).

---

## Captured frames

### 1. Real hardware captures (`JBD-SP04S034-L4S-200A-B-U`, 4S 5 Ah pack)

From [`syssi/esphome-jbd-bms` `docs/pdus/JBD-SP04S034-L4S-200A-B-U.txt`](https://github.com/syssi/esphome-jbd-bms/blob/main/docs/pdus/JBD-SP04S034-L4S-200A-B-U.txt) and [`JBD-SP04S034-L4S-200A-B-U_MOS.txt`](https://github.com/syssi/esphome-jbd-bms/blob/main/docs/pdus/JBD-SP04S034-L4S-200A-B-U_MOS.txt):

```text
# 0x03 Basic info: 15.60 V, 0.00 A, 4.98 / 5.00 Ah, 0 cycles, date 2022-03-28 (0x2C7C),
# balance 0, errors 0, SW v8.0 (0x80), SOC 100 % (0x64), FET 0x03 (Chg+Dsg ON),
# 4 cells, 3 NTCs: 22.4 °C (0x0B8B), 22.3 °C (0x0B8A), 21.7 °C (0x0B84)
DD A5 03 00 FF FD 77  ->  DD 03 00 1D 06 18 00 00 01 F2 01 F4 00 00 2C 7C 00 00 00 00 00 00 80 64 03 04 03 0B 8B 0B 8A 0B 84 FA 8D 77

# 0x04 Cell voltages (4 cells): 3.909 V (0x0F45), 3.901 V (0x0F3D), 3.895 V (0x0F37), 3.901 V (0x0F3D)
DD A5 04 00 FF FC 77  ->  DD 04 00 08 0F 45 0F 3D 0F 37 0F 3D FE C6 77

# 0x05 Hardware version (25 ASCII bytes): "JBD-SP04S034-L4S-200A-B-U"
DD A5 05 00 FF FB 77  ->  DD 05 00 19 4A 42 44 2D 53 50 30 34 53 30 33 34 2D 4C 34 53 2D 32 30 30 41 2D 42 2D 55 FA 08 77

# 0xAA Error counts (24 bytes): cell OVP = 122 (0x007A), cell UVP = 2 (0x0002), system restarts = 1 (0x0001)
DD A5 AA 00 FF 56 77  ->  DD AA 00 18 00 00 00 00 00 00 00 7A 00 02 00 00 00 00 00 00 00 00 00 00 00 00 00 01 FF 6B 77

# 0xE1 MOSFET control: turn charging OFF, keep discharging ON -> BMS ACK
DD 5A E1 02 00 01 FF 1C 77  ->  DD E1 00 00 00 00 77
```

### 2. Vendor protocol document examples (15S and 17S packs)

From *General Protocol V4* ([`PDFs/Jiabaida.communication.protocol.pdf`](PDFs/Jiabaida.communication.protocol.pdf), pages 2–8):

```text
# 0x03 Basic info (15S pack, p. 2): 58.88 V (0x1700), 0.00 A, 7.20 / 10.00 Ah, 0 cycles,
# date 2016-03-24 (0x2078), SW v1.0 (0x10), SOC 72 % (0x48), FET 0x03, 15 cells, 2 NTCs (20.3 °C, 21.5 °C)
DD A5 03 00 FF FD 77  ->  DD 03 00 1B 17 00 00 00 02 D0 03 E8 00 00 20 78 00 00 00 00 00 00 10 48 03 0F 02 0B 76 0B 82 FB FF 77

# 0x03 Basic info (17S pack discharging, p. 8): 66.23 V (0x19DF), -20.12 A (0xF824), 34.93 / 40.00 Ah,
# 2 cycles, date 2018-04-17 (0x2491), SW v1.2 (0x12), SOC 87 % (0x57), FET 0x03, 17 cells,
# 4 NTCs (24.7 °C, 26.4 °C, 24.5 °C, 24.6 °C)
DD A5 03 00 FF FD 77  ->  DD 03 00 1F 19 DF F8 24 0D A5 0F A0 00 02 24 91 00 00 00 00 00 00 12 57 03 11 04 0B 98 0B A9 0B 96 0B 97 F8 9A 77
```

---

## Errata and disagreements

- **Response checksum scope vs. V4 table wording:** Page 1 of the V4 vendor document states that the checksum is calculated over *"data segment content + length byte + command code byte"*. That description applies **only to host requests** (`[0xDD, 0xA5/0x5A, cmd, len, ...]`). In **BMS responses** (`[0xDD, cmd, status, len, ...]`), byte 1 (`cmd`) is **excluded** and the checksum covers `status + len + data` (bytes `2 .. 3+N`). All worked examples in the V4 PDF and the explicit response table on page 3 of the V8/V11 PDF confirm this.
- **Write-acknowledgment checksum `00 00`:** When acknowledging write commands such as `0xE1` (`DD E1 00 00 00 00 77`) or `0x01` (`DD 01 00 00 00 00 77`) with `status = 0x00` and `len = 0x00`, the sum of bytes `2..3` (`00 00`) is `0`, and `0x10000 - 0` wraps to `0x0000` in 16-bit arithmetic. Some non-standard error replies on older firmware also omit a valid checksum (noted in Overkill Solar's `BLE _bluetooth_protocol.md`).
- **Corrupted `0x03` example in the V8/V11 vendor PDF:** On page 4 of `JBD_comm_ptotocol_20230321_V11.pdf`, the vendor inserted three dummy bytes (`AA BB CC`) into the V4 `0x03` example (`... 0B 76 0B 82 AA BB CC FB FF 77`) without updating the length (`1B` = 27) or checksum (`FB FF`), and wrote `=0x48=80%` for `RSOC` (`0x48` is 72 %).
- **Typo in V4 PDF production date and cell voltage examples:** On page 2 of `Jiabaida.communication.protocol.pdf`, the production date bitmask example mixes `0x2068` and `0x2028` while the frame contains `0x2078` (2016-03-24). On page 7, cell 2 (`0x0EC8` = 3784 mV) is mislabelled `3744`.
- **Typo in Overkill Solar `BLE _bluetooth_protocol.md`:** Lines 98 and 107–108 write `0x1000` instead of `0x10000` and compute `0x10000 - 0x03 = 0xffdd` (the actual checksum is `0xFFFD`, as correctly shown on line 52 of the same file).
- **`0x05` payload layout in `JBD_REGISTER_MAP.md`:** Overkill Solar's `JBD_REGISTER_MAP.md` lists data offset `0x00` of register `0x05` as a 1-byte string length followed by the ASCII string. In fact, that length byte is byte 3 of the frame header (`DD 05 00 <len> <ASCII...> <crc> 77`); the data payload starting at frame byte 4 is purely the ASCII characters (unlike EEPROM string registers `0xA0`–`0xA2`, which do carry an extra length byte at data offset `0x00`).
- **Software version decoding (`0x12` in `0x03`):** The vendor documents and `esphome-jbd-bms` / `dbus-serialbattery` decode the version byte as two nibbles `(ver >> 4) + (ver & 0x0F) * 0.1` (`0x80` → `8.0`), whereas `jbdtool` divides the raw byte by 10 (`0x80` = 128 → `12.8`).

---

## Sources

- Jiabaida, *软件版通用协议 V4 / General Protocol V4* — stored in [`PDFs/Jiabaida.communication.protocol.pdf`](PDFs/Jiabaida.communication.protocol.pdf) (from [`syssi/esphome-jbd-bms/docs/Jiabaida.communication.protocol.pdf`](https://github.com/syssi/esphome-jbd-bms/blob/main/docs/Jiabaida.communication.protocol.pdf)) and English-only copy [`PDFs/JBD Protocol English version.pdf`](PDFs/JBD%20Protocol%20English%20version.pdf) (from [`FurTrader/OverkillSolarBMS`](https://github.com/FurTrader/OverkillSolarBMS/blob/master/Comm_Protocol_Documentation/JBD%20Protocol%20English%20version.pdf)).
- Jiabaida, *Software board common protocol V8 (rev V11, 2023-03-21)* — English translation stored in [`PDFs/Translated_to_english_JBD_comm_ptotocol_20230321_V11.pdf`](PDFs/Translated_to_english_JBD_comm_ptotocol_20230321_V11.pdf) (from [`FurTrader/OverkillSolarBMS/Comm_Protocol_Documentation/`](https://github.com/FurTrader/OverkillSolarBMS/tree/master/Comm_Protocol_Documentation)).
- Overkill Solar / Eric Poulsen & Steve Tecza, [`JBD_REGISTER_MAP.md`](https://github.com/FurTrader/OverkillSolarBMS/blob/master/Comm_Protocol_Documentation/JBD_REGISTER_MAP.md), [`BLE _bluetooth_protocol.md`](https://github.com/FurTrader/OverkillSolarBMS/blob/master/Comm_Protocol_Documentation/BLE%20_bluetooth_protocol.md), and [`Capacity_parameter_registers.md`](https://github.com/FurTrader/OverkillSolarBMS/blob/master/Comm_Protocol_Documentation/Capacity_parameter_registers.md) — EEPROM register map, password registers, and BLE GATT details.
- [syssi/esphome-jbd-bms](https://github.com/syssi/esphome-jbd-bms) — `components/jbd_bms/jbd_bms.cpp`, `components/jbd_bms_ble/jbd_bms_ble.cpp`, `tests/components/jbd_bms/frames.h`, `tests/components/jbd_bms/write_command_test.cpp`, and `docs/pdus/JBD-SP04S034-L4S-200A-B-U*.txt` (captured frames and BLE authentication).
- [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/lltjbd.py` and `dbus-serialbattery/ext/aiobmsble/test_data/jbd_bms.json` (register addresses, EEPROM context manager, BLE advertisements).

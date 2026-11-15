# JK (Jikong) Smart BMS — Bluetooth (BLE) Protocol

> Status: ✅ Verified — GATT UUIDs, 20-byte `AA 55 90 EB` commands, 300-byte `55 AA EB 90` notification offsets, and sum-8 checksums verified against captured frames in [`patman15/aiobmsble` (`tests/bms/test_jikong_bms.py`)](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_jikong_bms.py) and [`syssi/esphome-jk-bms`](https://github.com/syssi/esphome-jk-bms).

## Summary

| | |
|---|---|
| Products | JK-B / JK-BD smart BMS (HW v10/v11+) and JK-PB inverter BMS over BLE |
| GATT Service UUID | `0000ffe0-0000-1000-8000-00805f9b34fb` |
| Characteristic (Notify + Write) | `0000ffe1-0000-1000-8000-00805f9b34fb` |
| Manufacturer IDs (adv) | `0x0B65`, `0x4B4A` |
| Roles | Host sends 20-byte request; BMS streams 300-byte notifications (often chunked into ~20-byte BLE packets) |
| Byte order | **Little-endian** |

---

## Framing & Checksum

### Host → BMS command (20 bytes)

```
AA 55 90 EB  <cmd>  <len>  <value (up to 13 bytes, zero-padded)>  <crc8>
```

- **Checksum (`crc8`)**: 8-bit sum of bytes `0..18` (`sum(frame[:19]) & 0xFF`).
- **Worked examples (verified in Python):**
  - Request device info (`0x97`): `AA 55 90 EB 97 00 00 00 00 00 00 00 00 00 00 00 00 00 00 11` (`sum = 0x211 → 0x11`)
  - Request cell info (`0x96`): `AA 55 90 EB 96 00 00 00 00 00 00 00 00 00 00 00 00 00 00 10` (`sum = 0x210 → 0x10`)
  - BMS ready ACK (`0xC8`, sent by BMS): `AA 55 90 EB C8 01 01 00 00 00 00 00 00 00 00 00 00 00 00 44`

### BMS → Host notification (300 bytes)

```
55 AA EB 90  <type>  <counter>  <payload (293 bytes)>  <crc8>
```

- `type = 0x01`: Settings frame
- `type = 0x02`: Cell & live telemetry frame
- `type = 0x03`: Device info frame
- **Reassembly note**: BLE notifications arrive in chunks (often 20 or 128 bytes). Strip any `AT\r\n` (`41 54 0D 0A`) injected by the BLE-UART bridge, accumulate from `55 AA EB 90` until 300 bytes are received, and verify `sum(frame[:299]) & 0xFF == frame[299]`.

---

## `0x02` Cell & Live Data Frame (300 bytes)

Offsets below are for **JK02_32S** (firmware `V11.x+`). For **JK02_24S** (firmware `< V11`, e.g. `10.08`), apply `offs = -32` to the main telemetry block (`150+offs` .. `236+offs`) and `offs >> 1` (`-16`) to the cell-mask/delta block (`70..77`).

| Offset (`JK02_32S`) | Offset (`JK02_24S`) | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `6` | `6` | 32×2 / 24×2 | u16 | 1 mV | Cell voltages 1–32 (or 1–24) |
| `70` | `54` | 4 | u32 | bitmask | Cells present |
| `76` | `60` | 2 | u16 | 1 mV | Max cell voltage delta |
| `144` | `134` | 2 | s16 | 0.1 °C | MOSFET temperature (`-2000` = absent) |
| `150` | `118` | 4 | u32 | 1 mV | Pack voltage |
| `158` | `126` | 4 | s32 | 1 mA | Pack current (`+` charge, `−` discharge) |
| `162`, `164` | `130`, `132` | 2 × 2 | s16 | 0.1 °C | Battery temperatures 1 & 2 |
| `166` | `134` | 4 | u32 | bitmask | Alarms / problem code (`>> 16` on `JK02_24S`, `& 0xFFFF` on `JK02_32S`) |
| `170` | `138` | 2 | s16 | 1 mA | Balancing current |
| `172` | `140` | 1 | u8 | bool | Balancing active |
| `173` | `141` | 1 | u8 | 1 % | State of Charge (SOC) |
| `174` | `142` | 4 | u32 | 1 mAh | Remaining capacity |
| `178` | `146` | 4 | u32 | 1 mAh | Design / full capacity |
| `182` | `150` | 4 | u32 | count | Cycle count |
| `190` | `158` | 1 | u8 | 1 % | State of Health (SOH) |
| `198` | `166` | 1 | u8 | bool | Charge MOSFET on |
| `199` | `167` | 1 | u8 | bool | Discharge MOSFET on |
| `214` | `182` | 1 | u8 | bitmask | Temperature sensor presence mask |
| `215` | `183` | 1 | u8 | bool | Heater active |
| `236` | `204` | 2 | s16 | 1 mA | Heater current |
| `254`, `256`, `258` | — | 3 × 2 | s16 | 0.1 °C | Extra temperatures (SW `>= 14`: MOS2, T4, T5) |
| `280` | — | 1 | u8 | enum | Battery mode (SW `>= 15`) |

---

## `0x03` Device Info Frame (300 bytes)

| Offset | Size | Type | Field |
|---|---|---|---|
| `6` | 16 | ASCII | Model / Vendor ID (e.g. `JK_B2A8S20P`) |
| `22` | 8 | ASCII | Hardware version (e.g. `11.XA`) |
| `30` | 8 | ASCII | Software version (e.g. `11.48`) |
| `46` | 16 | ASCII | Device name |
| `62` | 16 | ASCII | **Bluetooth passcode (never publish)** |
| `86` | 8+ | ASCII | Serial number |

---

## Control Commands (Dangerous)

Writing settings or toggling MOSFETs uses `AA 55 90 EB <reg> 04 <u32_LE> ... <crc8>` (e.g. charge/discharge/balance enables in the `0x1D`–`0x1F` range). Do not send write commands without verifying the exact firmware version in [`syssi/esphome-jk-bms`](https://github.com/syssi/esphome-jk-bms).

---

## Captured Frame Example (`JK02_24S`, from `test_jikong_bms.py`)

```
Device Info (0x03, SW 10.08):
55 AA EB 90 03 79 4A 4B 2D 42 32 41 32 30 53 32 30 50 00 00 00 00 31 30 2E 58 47 ... 93
# Model: JK-B2A20S20P, HW: 10.XG, SW: 10.08, CRC: 0x93
```

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/jikong_bms.py` and `tests/bms/test_jikong_bms.py`
- [`syssi/esphome-jk-bms`](https://github.com/syssi/esphome-jk-bms) — `components/jk_bms_ble/`

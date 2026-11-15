# ECO-WORTHY / DCHOUSE / Creabest / CBT Power — BLE Protocols

> Status: ✅ Verified — GATT UUIDs, framing, and checksums/CRCs verified against [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) (`ecoworthy_bms.py`, `cbtpwr_bms.py`, and their captured BLE test frames).

Two common BLE protocols appear in ECO-WORTHY, DCHOUSE, and Creabest LiFePO4 batteries:

1. **ECO-WORTHY / DCHOUSE (`0xA1` / `0xA2` Modbus-CRC BLE)** — `0000fff0` service, `fff1` notify, `fff2` write.
2. **CBT Power / Creabest (`AA 55 … 0D 0A` BLE)** — `0000ffe5` (or `ffe0`) service, `ffe4` notify, `ffe9` write.

---

## 1. ECO-WORTHY / DCHOUSE (`0xA1` / `0xA2`)

### Summary

| | |
|---|---|
| Adv local names | `ECO-WORTHY*`, `ECO-WORTHY 02_*`, `DCHOUSE*` |
| GATT Service / RX / TX | `0000fff0-…` / `0000fff1-…` (notify) / `0000fff2-…` (write) |
| Integrity | **CRC-16/Modbus** (little-endian, last 2 bytes of frame) |
| Byte order | **Big-endian** |

### Framing & Fields

- Initialization commands sent to `fff2`:
  - `FF 08 02 00 0B 01 00 64 01 FF FF FF FF FF FF FF 00 2D`
  - `FF 08 02 00 0B 01 00 14 01 FF FF FF FF FF FF FF 65 EF`
- Notifications begin with `A1` or `A2` (or `<6-byte MAC> A1/A2` on newer `V2` firmware; shift offsets by +2 if MAC prefix is absent):
  - **`0xA1` frame**: SOC (`16`, u16, 1 %), SOH (`18`, u16, 1 %), voltage (`20`, u16, `0.01 V`), current (`22`, s16, `0.01 A` on V1 / `0.1 A` on V2), design capacity (`26`, u16, `0.01 Ah`), problem code (`51`, u16).
  - **`0xA2` frame**: cell count (`14`, u16), temperature count (`80`, u16), followed by cell voltages and temperatures.

---

## 2. CBT Power / Creabest (`AA 55 … 0D 0A`)

### Summary

| | |
|---|---|
| Adv / Models | CBT Power smart BMS, Creabest (`???[CR]??????`) |
| GATT Service / RX / TX | `0000ffe5-…` (or `ffe0`) / `0000ffe4-…` (notify) / `0000ffe9-…` (write) |
| Frame format | Request: `AA 55 <cmd> <len> [data] <sum8> 0A 0D`<br>Reply: `AA 55 <cmd> <len> [data] <sum8> 0D 0A` |
| Checksum | 8-bit sum of `<cmd> <len> [data]` (`sum(frame[2:-3]) & 0xFF`) |
| Byte order | **Little-endian** |

Worked request example (verified in Python):

```
AA 55 0B 00 0B 0A 0D   # Request pack voltage & current (cmd 0x0B, len 0x00, sum8 = 0x0B)
```

### Commands

| Cmd | Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `0x05`–`0x08` | `4` | u16 × N | mV | Cell voltages (groups) |
| `0x09` | `4` | 2 | s16 | 1 °C | Temperature |
| `0x0A` | `4` | 1 | u8 | 1 % | SOC |
| `0x0B` | `4` / `8` | 4 / 4 | u32 / s32 | 1 mV / 1 mA | Pack voltage / pack current |
| `0x0C` | `14` | 2 | u16 | 0.01 h | Remaining runtime |
| `0x15` | `4` / `6` | 2 / 2 | u16 | 1 Ah / count | Design capacity / cycle count |
| `0x21` | `4` | 4 | u32 | bitmask | Problem / alarm code |

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/ecoworthy_bms.py`, `cbtpwr_bms.py`, `tests/bms/test_ecoworthy_bms.py`, `test_cbtpwr_bms.py`

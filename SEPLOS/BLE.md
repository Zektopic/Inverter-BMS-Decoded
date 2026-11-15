# Seplos BMS — Bluetooth (BLE) Protocols (V2 & V3)

> Status: ✅ Verified — GATT UUIDs, framing, CRCs, and register offsets verified against [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) (`seplos_bms.py`, `seplos_v2_bms.py`, and their captured BLE test frames).

Seplos V2 and Seplos V3 use **different GATT UUIDs and different framing** over Bluetooth Low Energy:

| Generation | Adv Name Patterns | Service UUID | RX (Notify) | TX (Write) | Framing |
|---|---|---|---|---|---|
| **Seplos V3** | `SP??B*`, `SP1??B*`, `XZHX*`, `CSY*`, `ST*` | `0000fff0-…` | `0000fff1-…` | `0000fff2-…` | Modbus RTU (`FC 0x04` / `0x01`, CRC-16/Modbus) |
| **Seplos V2** | `BP0*`, `BP1*`, `BP2*` | `0000ff00-…` | `0000ff01-…` | `0000ff02-…` | Binary `7E 10 … 0D` (CRC-16/XMODEM) |

*(Full 128-bit base UUID: `-0000-1000-8000-00805f9b34fb`.)*

---

## 1. Seplos V3 BLE (Modbus RTU over BLE)

Seplos V3 sends standard **Modbus RTU** frames over `fff2` (write) and receives fragmented responses on `fff1` (notify). Reassemble chunks until `len == data[2] + 5` (3-byte header + `byte_count` + 2-byte CRC-16/Modbus LE).

Worked request examples (verified in Python):

```
00 04 20 00 00 1A 7B D0   # EIA: Read 26 input regs (0x2000) from pack 0 (combined system telemetry)
01 04 10 00 00 11 34 C6   # PIA: Read 17 input regs (0x1000) from pack 1 (individual pack telemetry)
```

### Query blocks

| Block | FC | Start reg | Count | Content |
|---|---|---|---|---|
| `EIA` | `0x04` | `0x2000` | `0x1A` (26) | Combined pack voltage (`swap32(u32)/100` V), current (`swap32(s32)/10` A), remaining Ah (`swap32/100`), design Ah (`swap32//100`), pack count (offset 44), cycles (46), SOC (`0.1 %`, 48), SOH (`0.1 %`, 50) |
| `EIB` | `0x04` | `0x2100` | `0x16` (22) | Temperatures (starts at frame offset 35) |
| `EIC` | `0x01` | `0x2200` | `0x05` (5) | Coils: problem/alarm bits (bytes `1..9`, masked `0xFFFF00FF00FF0000FF`), FET flags at byte 7 (bit 0 discharge, bit 1 charge, bit 2 current-limit/balance, bit 3 heater) |
| `PIA` | `0x04` | `0x1000` | `0x11` (17) | Per-pack (slave `1..N`): voltage (`0.01 V`), current (`0.01 A` signed), remaining (`0.01 Ah`), design (`0.01 Ah`), SOC (`0.1 %`), SOH (`0.1 %`), cycles |
| `PIB` | `0x04` | `0x1100` | `0x1A` (26) | Per-pack cell voltages (16 × u16 mV) and temperatures |
| `SPA` | `0x04` | `0x1300` / `0x1335` | `0x35` | Protection limits (OVP, UVP, OCP, OTP, UTP, MOS OTP) |

*(Note: `swap32` swaps the two 16-bit big-endian words of a 32-bit value.)*

---

## 2. Seplos V2 BLE (`7E … 0D` Binary Protocol)

Unlike V2 RS485 ([README.md](README.md), which is YD/T 1363 hex-ASCII), V2 BLE uses a **compact binary frame** on `0000ff00`:

```
7E  <ver>  <addr>  <cmd>  <len>  [payload...]  <crc_hi> <crc_lo>  0D
```

- `ver`: `0x10` for requests, `0x14` for responses.
- **Checksum**: **CRC-16/XMODEM** (polynomial `0x1021`, init `0x0000`, big-endian) computed over `7E <ver> <addr> <cmd> <len> [payload]`, followed by `0x0D`.
- Worked request example (verified in Python):
  ```
  7E 10 00 51 00 F4 07 0D   # Request device info (cmd 0x51) from addr 0x00 (CRC-XMODEM = 0xF407)
  ```

### Key commands

| Cmd | Payload | Response content |
|---|---|---|
| `0x51` | `""` | Device info: model (`[26:36]` ASCII), software version (`frame[37].frame[38]`) |
| `0x61` | `0x00` | Single-machine live data: cell voltages (from offset 9), pack voltage (`0.01 V`), current (`0.01 A` signed), remaining Ah (`0.01 Ah`), design Ah (`0.01 Ah`), SOC (`0.1 %`), SOH (`0.1 %`), cycles, temperatures, and 8 alarm event bytes |

---

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/seplos_bms.py`, `aiobmsble/bms/seplos_v2_bms.py`, `tests/bms/test_seplos_bms.py`, `tests/bms/test_seplos_v2_bms.py`

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
00 04 21 00 00 16 7A 29   # EIB: Read 22 input regs (0x2100) from pack 0
00 01 22 00 00 50 37 9F   # EIC: Read 80 coils (0x2200) from pack 0 (reply carries 10 data bytes)
01 04 10 00 00 11 34 C6   # PIA: Read 17 input regs (0x1000) from pack 1 (individual pack telemetry)
01 04 11 00 00 1A 74 FD   # PIB: Read 26 input regs (0x1100) from pack 1 (cells and temperatures)
```

### Query blocks

Offsets in this table are **data offsets**: bytes from the start of the register data, which begins at frame offset 3
(`aiobmsble` decodes with `start=_HEAD_LEN`). Add 3 for the offset inside the notification frame.

| Block | FC | Start reg | Count | Content |
|---|---|---|---|---|
| `EIA` | `0x04` | `0x2000` | `0x1A` (26) | Combined pack voltage (`swap32(u32)/100` V, data `0`), current (`swap32(s32)/10` A, `4`), remaining Ah (`swap32/100`, `8`), design Ah (`swap32//100`, `20`), pack count (`44`), cycles (`46`), SOC (`0.1 %`, `48`), SOH (`0.1 %`, `50`) |
| `EIB` | `0x04` | `0x2100` | `0x16` (22) | Requested by `aiobmsble` on every update, but none of its fields are decoded there. Contents *(unverified)* |
| `EIC` | `0x01` | `0x2200` | `0x50` (80 coils) | Coils: problem/alarm bits (data bytes `1..9`, masked `0xFFFF00FF00FF0000FF`), FET flags at data byte `7` (bit 0 discharge, bit 1 charge, bit 2 current-limit/balance, bit 3 heater). `aiobmsble` writes the count as `5 × 0x10` |
| `PIA` | `0x04` | `0x1000` | `0x11` (17) | Per-pack (slave `1..N`): voltage (`0.01 V`, data `0`), current (`0.01 A` signed, `2`), remaining (`0.01 Ah`, `4`), design (`0.01 Ah`, `6`), SOC (`0.1 %`, `10`), SOH (`0.1 %`, `12`), cycles (`14`). Data `8` (register `0x1004`) is skipped by `aiobmsble` *(meaning unverified)* |
| `PIB` | `0x04` | `0x1100` | `0x1A` (26) | Per-pack cell voltages (16 × u16 mV, data `0`); cell temperatures 1–4 (`0.1 K`, `°C = (raw − 2731) / 10`) at data `32` (frame offset `35`); four reserved words; two more temperatures at data `48` (frame `51`, registers `0x1118` / `0x1119`) |
| `SPA` | `0x04` | `0x1300` / `0x1335` | `0x35` | Protection limits (OVP, UVP, OCP, OTP, UTP, MOS OTP) |

*(Note: `swap32` swaps the two 16-bit big-endian words of a 32-bit value.)*

`aiobmsble` reads the pack count from `EIA`, then asks every pack `1..N` (at most 16) for `PIA` and `PIB`, all over the one
Bluetooth connection to the master. A logger that only needs one pack can skip `EIA`/`EIB`/`EIC` and poll `PIA` + `PIB` at
address 1.

### Captured frames (pack 1, from `test_seplos_bms.py`)

```
Request:  01 04 10 00 00 11 34 C6
Response: 01 04 22 14 72 FD 30 34 64 6D 60 00 D5 01 DF 03 E7 00 09 0C C7 0B 9F 0C C8 0C C6 0B A5 0B 99 00 00 00 B4 00 B4 6F F3
# PIA, 39 bytes: 52.34 V (0x1472), -7.20 A (0xFD30), 134.12 Ah remaining (0x3464), 280.00 Ah full (0x6D60),
#                register 0x1004 = 0x00D5, SOC 47.9 % (0x01DF), SOH 99.9 % (0x03E7), 9 cycles

Request:  01 04 11 00 00 1A 74 FD
Response: 01 04 34 0C C8 0C C8 0C C8 0C C7 0C C7 0C C7 0C C7 0C C6 0C C6 0C C7 0C C7 0C C7 0C C7 0C C8 0C C8 0C C8
          0B A5 0B 99 0B 9A 0B A4 0A AB 0A AB 0A AB 0A AB 0B C4 0B B5 97 1F
# PIB, 57 bytes: 16 cells 3.270-3.272 V; cell temperatures 25.0, 23.8, 23.9, 24.9 °C (0x0BA5 ...);
#                four reserved words 0x0AAB (2731, i.e. 0.0 °C placeholders); 0x1118 = 28.1 °C, 0x1119 = 26.6 °C
```

Both CRCs and the `len == data[2] + 5` rule were checked in Python. Which of `0x1118` / `0x1119` is the MOSFET sensor and
which is ambient is disputed between sources: see [V3.md](V3.md).

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

## Errata and disagreements

- **Temperatures are in `PIB`, not `EIB` (fixed 2026-11)**: an earlier version of this page put "temperatures, starting at
  frame offset 35" in the `EIB` row. Frame offset 35 is `_TEMP_START` in `aiobmsble`, and it is applied to the per-pack
  `PIB` reply. The captured `PIB` frame above decodes to plausible cell temperatures there.
- **`EIC` count**: the block is often described as "count 5", but the Modbus quantity on the wire is `0x0050` coils.

See the cross-brand [Bluetooth guide](../BLUETOOTH.md) for the connection sequence. Note that Seplos V2 shares its GATT
UUIDs (`0xFF00` / `0xFF01` / `0xFF02`) with JBD boards, and Seplos V3 shares `0xFFF0` / `0xFFF1` / `0xFFF2` with Daly and
ECO-WORTHY, so match on the advertised name as well as the service.

---

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/seplos_bms.py`, `aiobmsble/bms/seplos_v2_bms.py`, `tests/bms/test_seplos_bms.py`, `tests/bms/test_seplos_v2_bms.py`

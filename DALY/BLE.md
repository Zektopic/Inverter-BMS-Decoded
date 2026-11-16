# Daly Smart BMS — Bluetooth (BLE) & Sinowealth Variants

> Status: ✅ Verified (BLE `0xD2` / `0x81` Modbus-over-BLE) / 🌐 Community (UART-over-BLE `0xA5` & Sinowealth `0x0A`) — BLE frames and CRC-16 verified against captured notifications in [`patman15/aiobmsble` (`tests/bms/test_daly_bms.py`)](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_daly_bms.py).

Daly Smart BMS boards ship with three different protocols depending on hardware generation and Bluetooth dongle:

1. **Newer Daly / 100Balance Modbus-over-BLE (`0xD2` and `0x81` / `0x51`)** — used on current Daly Smart BMS & 100Balance Bluetooth boards.
2. **Legacy `0xA5` UART-over-BLE** — the Bluetooth dongle bridges the 13-byte `0xA5` UART frames ([PROTOCOL.md](PROTOCOL.md)) over BLE (`0000fff0`).
3. **Sinowealth chip variant (`0x0A`)** — some Daly-branded boards use a Sinowealth SH36730x AFE with a completely different UART protocol.

---

## 1. BLE GATT Service & Characteristics

| | |
|---|---|
| Local name / advertisement | `DL-*` or manufacturer IDs `0x0102`, `0x0104`, `0x0107`, `0x0302`, `0x0303`, `0x0402` |
| GATT Service UUID | `0000fff0-0000-1000-8000-00805f9b34fb` |
| Notify / Read Characteristic (RX) | `0000fff1-0000-1000-8000-00805f9b34fb` |
| Write Characteristic (TX) | `0000fff2-0000-1000-8000-00805f9b34fb` |
| Byte order | Big-endian for register data; **CRC-16/Modbus is little-endian** (`CRC_Lo`, `CRC_Hi`) |

---

## 2. Modbus-over-BLE (`0xD2` and `0x81` / `0x51` Protocols)

### Framing & Checksum

Requests use standard 8-byte Modbus RTU Read Holding Registers (`FC 0x03`) with slave address `0xD2` (standard Daly BLE) or `0x81` (48S/newer variant, which replies with slave byte `0x51`):

```
<slave> 03 <reg_hi> <reg_lo> <count_hi> <count_lo> <crc_lo> <crc_hi>
```

Worked request examples (verified in Python):

```
D2 03 00 00 00 3E D7 B9   # Read 62 registers (0x0000) — cells, temps, V, I, SOC, alarms
D2 03 00 3E 00 09 F7 A3   # Read 9 registers (0x003E) — MOSFET temperature (if supported)
D2 03 00 A9 00 20 87 91   # Read 32 registers (0x00A9) — hardware/software/model ASCII strings
81 03 00 00 00 40 5B FA   # Variant 0x81: read 64 registers (0x0000), replies with 0x51 0x03 0x80 ...
```

Notifications may arrive split across multiple BLE packets; reassemble until `len == 3 + byte_count + 2` (`byte_count = frame[2]`), then verify CRC-16/Modbus (`0xA001`, init `0xFFFF`).

### `0xD2` Response (`0x0000`, 62 registers = 124 data bytes, 129 bytes total)

Header is `D2 03 7C` (3 bytes). The **Frame** column is the byte offset from the start of the frame; the **Data** column is
the offset inside the register data (frame offset − 3), which is what
[`aiobmsble`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/daly_bms.py) uses in its field table
(it decodes with `start=_HEAD_LEN`):

| Frame | Data | Size | Type | Unit / scale | Field |
|---|---|---|---|---|---|
| `3` | `0` | 64 | 32 × u16 | 1 mV | Cell voltages 1–32 (unused cells = `0x0000`) |
| `67` | `64` | 16 | 8 × u16 | 1 °C (`raw - 40`) | Temperatures 1–8. Unused slots are `0x0000`, which would decode as −40 °C: read only as many as the sensor count at frame `103` |
| `83` | `80` | 2 | u16 | 0.1 V | Pack voltage |
| `85` | `82` | 2 | u16 | `(raw - 30000) × 0.1 A` | Pack current (`+` charge, `−` discharge) |
| `87` | `84` | 2 | u16 | 0.1 % | State of Charge (SOC) |
| `99` | `96` | 2 | u16 | 0.1 Ah | Remaining capacity (`cycle_charge`) |
| `101` | `98` | 2 | u16 | count | Active cell count (max 32) |
| `103` | `100` | 2 | u16 | count | Active temperature sensor count (max 8) |
| `105` | `102` | 2 | u16 | count | Cycle count |
| `107` | `104` | 2 | u16 | bool / state | Balancer active |
| `109` | `106` | 2 | u16 | bool | Charge MOSFET on |
| `111` | `108` | 2 | u16 | bool | Discharge MOSFET on |
| `115` | `112` | 2 | u16 | 1 mV | Max cell voltage delta |
| `119` | `116` | 8 | u64 | bitmask | Problem / alarm code (bit meanings not documented in any source used here) |

This block has no full or design capacity, so remaining capacity cannot be turned into a second SOC figure.

- **MOSFET temperature (`0x003E`, 9 registers)**: big-endian `u16` at frame offset `11` (`_HEAD_LEN + 8`, data offset `8`),
  `(raw - 40)` °C. `0x0000` or `0xFFFF` means the board has no MOSFET sensor; `aiobmsble` stops asking after the first such
  reply or a timeout.
- **`0x81` / `0x51` variant**: uses two reads (`0x0000` count `64` and `0x0041` count `62`) and supports up to 48 cells. The
  offsets below are **data offsets** (add 3 for the frame offset), from the `aiobmsble` field table:
  - first reply: cell voltages from `0` (48 × u16), temperatures from `96`, `voltage` at `112`, `current` at `114`, `SOC` at
    `116`, `cell_count` at `120`, `temp_count` at `122`;
  - second reply: `capacity` at `20`, `cycles` at `22`, `balancer` at `24`, `chrg_mosfet` at `34`, `dischrg_mosfet` at `36`,
    `heater` at `40`, `problem_code` at `88` (4 bytes).
  - device info is read from `0x0178` (74 registers) instead of `0x00A9`; there is no MOSFET temperature read.

### Captured BLE Frames (`0xD2`, from `test_daly_bms.py`)

```
Request:  D2 03 00 00 00 3E D7 B9
Response: D2 03 7C 10 1F 10 29 10 33 10 3D 00 00 ... 00 3C 00 3D 00 3E 00 3F ... 00 8C 75 4E 03 84 ... A0 DF
# 129 bytes, CRC-16/Modbus 0xDFA0 sent as A0 DF.
# Decodes: 4 cells (4.127 V, 4.137 V, 4.147 V, 4.157 V) at frame 3; 4 temps (20, 21, 22, 23 °C) at frame 67;
#          pack 14.0 V (0x008C at 83), current +3.0 A ((0x754E - 30000)/10 at 85), SOC 90.0 % (0x0384 at 87),
#          remaining 345.6 Ah (at 99), 4 cells (101), 4 sensors (103), 57 cycles (105), balancer 1 (107),
#          charge MOSFET 0 (109), discharge MOSFET 1 (111), delta 0.321 V (115), problem code 0 (119)

Request:  D2 03 00 3E 00 09 F7 A3
Response: D2 03 12 00 00 00 00 75 30 00 00 00 4E FF FF FF FF FF FF FF FF 0B 4E
# MOSFET temperature: 0x004E at frame 11 = 78 - 40 = 38 °C
```

Reading the same frame with the data offsets as if they were frame offsets gives nonsense (pack 0.0 V, current −3000 A),
which is a quick way to notice the 3-byte slip.

*(unverified)* In this capture, frame offsets `89` and `91` hold `0x103D` and `0x101F` (4157 and 4127, the highest and lowest
cell voltage) and `113` holds `0x102E` (4142, the average of the four cells). No source used here names these registers. The
delta field (`0.321 V`) does not equal highest − lowest (`0.030 V`) in the same frame, so do not rely on it without checking
on your own board.

---

## 3. Legacy `0xA5` UART-over-BLE

Older Daly Bluetooth modules forward the 13-byte `0xA5` UART frames ([PROTOCOL.md](PROTOCOL.md)) directly over `0000fff2` (write) and `0000fff1` (notify), using host address `0x80` (Bluetooth app) instead of `0x40` (UART):

```
A5 80 90 08 00 00 00 00 00 00 00 00 BD   # Request voltage, current, SOC (sum8 = 0xBD)
```

---

## 4. Sinowealth Chip Variant (`0x0A` UART)

> Status: 🌐 Community — from [`mr-manuel/venus-os_dbus-serialbattery` (`sinowealth.py`)](https://github.com/mr-manuel/venus-os_dbus-serialbattery/blob/master/dbus-serialbattery/bms/sinowealth.py).

Some Daly-branded boards use a Sinowealth AFE chip on UART (`9600` 8N1) with 4-byte requests `[0x0A] [<cmd>] [<resp_len>] [<checksum>]`:

- Commands: `0x01`+ cell voltages, `0x0B` pack voltage, `0x0C`–`0x0F` temperatures, `0x10` current, `0x11` design capacity, `0x12` remaining capacity, `0x13` SOC, `0x14` cycles, `0x15` status, `0x16` battery status, `0x17` pack config.

---

## Implementation notes

- **Detecting the variant**: `aiobmsble` sends the `0xD2` main request first and, on a timeout, tries `0x81`. Whichever
  answers is used for the rest of the connection.
- **Polling order** (read-only): main block (`0x0000`), then the MOSFET block (`0x003E`) only while it keeps returning a
  valid value.
- **Framing**: a reply is complete at `3 + frame[2] + 2` bytes; it can arrive in several notifications. Check that
  `frame[0]` is the expected slave byte (`0xD2`, or `0x51` for the `0x81` variant) and `frame[1] == 0x03` before trusting
  `frame[2]` as a length.
- See the cross-brand [Bluetooth guide](../BLUETOOTH.md) for the connection sequence and for telling a Daly board apart
  from the other brands that use service `0xFFF0`.

## Errata and disagreements

- **Offset base (fixed 2026-11)**: an earlier version of this page listed pack voltage and every later field at the
  `aiobmsble` data offsets (`80`, `82`, `84` …) under a heading that called them frame offsets. Frame offsets are 3 higher
  (`83`, `85`, `87` …). The table above now gives both, checked against the captured frame.

---

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/daly_bms.py` and `tests/bms/test_daly_bms.py` (captured BLE frames)
- [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/daly.py` and `sinowealth.py`
- [`syssi/esphome-daly-bms`](https://github.com/syssi/esphome-daly-bms) — ESPHome UART/BLE implementation

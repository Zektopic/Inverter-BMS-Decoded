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

Header is `D2 03 7C` (3 bytes). All offsets below are byte offsets from start of frame:

| Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `3` | 64 | 32 × u16 | 1 mV | Cell voltages 1–32 (unused cells = `0x0000`) |
| `67` | 16 | 8 × u16 | 1 °C (`raw - 40`) | Temperatures 1–8 (`0x0000` if unused) |
| `80` | 2 | u16 | 0.1 V | Pack voltage |
| `82` | 2 | u16 | `(raw - 30000) × 0.1 A` | Pack current (`+` charge, `−` discharge) |
| `84` | 2 | u16 | 0.1 % | State of Charge (SOC) |
| `96` | 2 | u16 | 0.1 Ah | Remaining capacity (`cycle_charge`) |
| `98` | 2 | u16 | count | Active cell count (max 32) |
| `100` | 2 | u16 | count | Active temperature sensor count (max 8) |
| `102` | 2 | u16 | count | Cycle count |
| `104` | 2 | u16 | bool / state | Balancer active |
| `106` | 2 | u16 | bool | Charge MOSFET on |
| `108` | 2 | u16 | bool | Discharge MOSFET on |
| `112` | 2 | u16 | 1 mV | Max cell voltage delta |
| `116` | 8 | u64 | bitmask | Problem / alarm code |

- **MOSFET temperature (`0x003E`, 9 registers)**: `s16` at frame offset `11` (`_HEAD_LEN + 8`), `(raw - 40)` °C.
- **`0x81` / `0x51` variant**: uses two reads (`0x0000` count `64` and `0x0041` count `62`), supporting up to 48 cells (`voltage` at offset `112`, `current` at `114`, `SOC` at `116`, `cell_count` at `120` in the first reply; `capacity` at `20`, `cycles` at `22`, `chrg_mosfet` at `34`, `dischrg_mosfet` at `36`, `heater` at `40`, `problem_code` at `88` in the second reply).

### Captured BLE Frame (`0xD2`, from `test_daly_bms.py`)

```
Request:  D2 03 00 00 00 3E D7 B9
Response: D2 03 7C 10 1F 10 29 10 33 10 3D 00 00 ... 00 3C 00 3D 00 3E 00 3F ... 00 8C 75 4E 03 84 ... A0 DF
# Decodes: 4 cells (4.127 V, 4.137 V, 4.147 V, 4.157 V), 4 temps (20, 21, 22, 23 °C),
#          pack 14.0 V (0x008C), current +3.0 A ((0x754E - 30000)/10), SOC 90.0 % (0x0384), 57 cycles
```

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

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/daly_bms.py` and `tests/bms/test_daly_bms.py` (captured BLE frames)
- [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery) — `dbus-serialbattery/bms/daly.py` and `sinowealth.py`
- [`syssi/esphome-daly-bms`](https://github.com/syssi/esphome-daly-bms) — ESPHome UART/BLE implementation

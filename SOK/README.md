# SOK Battery — Smart BLE (12 V / 24 V) & Rack RS485/CAN (48 V)

SOK batteries use two completely different BMS platforms:

1. **12 V / 24 V Smart Bluetooth batteries (`SOK-*` / `ABC-*`)** — uses the **ABC-BMS** BLE protocol (`0000ffe0`, `0xEE` requests / `0xCC` 20-byte responses with CRC-8).
2. **48 V 100 Ah Server-Rack batteries** — uses a **PACE BMS** inside (RS232/RS485 YD/T 1363.3 ASCII `~25…` and Pylontech/Victron CAN). See [PACE](../PACE/README.md) and [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol).

---

## 12 V / 24 V Smart BLE Protocol (ABC-BMS)

> Status: 🌐 Community — from [`patman15/aiobmsble` (`abc_bms.py` & `tests/bms/test_abc_bms.py`)](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/abc_bms.py).

### Summary

| | |
|---|---|
| Adv local names | `SOK-*`, `ABC-*`, `NB-*`, `Hoover` (adv service `0000fff0-…`) |
| GATT Service UUID | `0000ffe0-0000-1000-8000-00805f9b34fb` |
| RX (Notify) / TX (Write) | `0000ffe1-…` (notify) / `0000ffe2-…` (write) |
| Frame size | Fixed **20 bytes** per request & response |
| Byte order | **Little-endian** |

### Framing & Commands

- **Host → BMS**: `EE <cmd> 00 ... <crc8>` (20 bytes).
- **BMS → Host**: `CC <type> <payload (17 bytes)> <crc8>` (20 bytes).

| Command (`EE …`) | Expected reply `CC <type>` | Decoded fields |
|---|---|---|
| `0xC0` | `0xF1` | Model name (`[2:-1]` ASCII) |
| `0xC1` | `0xF0`, `0xF2` | `0xF0`: voltage (`[2:5]` u24, 1 mV), current (`[5:8]` s24, 1 mA), design cap (`[8:11]` u24, 1 mAh), remaining cap (`[11:14]` u24, 1 mAh), cycles (`[14:16]` u16), SOC (`[16]` u8, %)<br>`0xF2`: charge MOSFET (`[2]` bool), discharge MOSFET (`[3]` bool), temp count (`[4]` u8) |
| `0xC2` | `0xF0`, `0xF3`, `0xF4` | `0xF3`: heater (`[8]` bool), balancer (`[10]` u8)<br>`0xF4`: cell voltages (4 cells per `0xF4` frame, part index at `[2]`) |
| `0xC3` | `0xF5`–`0xFA` | Extended temperature / cell groups |
| `0xC4` | `0xF9` | Problem / alarm bytes (`[2:18]`, bit 0 of each byte = active flag) |

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/abc_bms.py` and `tests/bms/test_abc_bms.py`
- [`PACE/README.md`](../PACE/README.md) — SOK 48 V server-rack RS485/RS232/CAN protocol

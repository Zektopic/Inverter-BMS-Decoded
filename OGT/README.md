# Offgridtec / D-powercore / TDT / E&J / RoyPow — Smart Battery BLE Protocols

> Status: 🌐 Community — transcribed from [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) (`ogt_bms.py`, `dpwrcore_bms.py`, `tdt_bms.py`, `ej_bms.py`, `roypow_bms.py`).

Many white-label LiFePO4 smart batteries (Offgridtec, Liontron, Topband, D-powercore, RoyPow, Chins, Volthium, Lithtech, TDT) share five compact BLE protocols documented below.

---

## 1. Offgridtec Smart Pro (`SmartBat-A*` / `SmartBat-B*`)

- **GATT**: Service `0000fff0-…`, RX `0000fff4-…` (notify), TX `0000fff6-…` (write); XOR-obfuscated ASCII AT-style commands (`+RAA<reg>` for Type A, `+R16<reg>` for Type B).
- **Obfuscation**: Every character is XORed with a per-device key derived from the 4-digit hex ID in the BLE name (`SmartBat-Axxxx` / `SmartBat-Bxxxx`) using lookup table `_CRY_SEQ = (2, 5, 4, 3, 1, 4, 1, 6, 8, 3, 7, 2, 5, 8, 9, 3)` plus offset `5` (Type A) or `8` (Type B).
- **Key registers (Type A / Type B)**: SOC (`2` / `13`, 1 %), remaining Ah (`4` / `15`, mAh), pack voltage (`8` / `9`, mV), temperature (`12` / `8`, 0.1 K), current (`16` / `10`, `0.01 A` / `0.001 A`), runtime (`24` / `18`, min), cycles (`44` / `23`), design capacity (`60` / `24`, mAh).

---

## 2. D-powercore (`DXB-*` / `TBA-*`)

- **GATT**: Service `0000fff0-…`, RX `0000fff4-…` (notify), TX `0000fff3-…` *(verify TX in `dpwrcore_bms.py`)*.
- **Flow**: Requires an unlock handshake (`0x64` → `0x65` / `0x32` accept / `0x33` reject) before polling `0x60` (`LEGINFO1`), `0x61` (`LEGINFO2`), `0x62` (`CELLVOLT`), and `0xA0` (`GETINFO`).
- **Key fields**: `0x60`: voltage (`0.1 V`, offset 6), current (`0.1 A` signed, 8), remaining Ah (`0.001 Ah`, 12), SOC (`%`, 14), problem code (15). `0x61`: cycles (offset 8), temperature (`0.1 K`, offset 12). `0x62`: cell count (offset 6) + cell voltages.

---

## 3. TDT BLE (`WTDH*`, `WT???AZ*`, `HS*`, `WT???BG*`)

- **GATT**: Service `0000fff0-…` or `0000ffe1-…`; config characteristic `0000fffa-…`.
- **Framing**: `7E` (or `1E`) `<ver> ... <crc16_modbus> 0D`. Note: some firmware has a known CRC bug on `0x8D` (cell voltage) frames.
- **Key fields (`0x8C` reply)**: current (`(raw & 0x3FFF) / 10 * (-1 if raw >> 15 else 1)`, offset 0), voltage (`0.01 V`, 2), remaining (`0.1 Ah`, 4), cycles (8), design (`0.1 Ah`, 10), SOC (`%`, 12), SOH (`%`, 14).

---

## 4. E&J Technology (`libatt*`, `SV12V*`, `L-12V*`, `LT-12*`, `V-12V*`)

- **Framing**: ASCII-hex / binary frames delimited by `:` (`0x3A`) start and `~` (`0x7E`) end, with 8-bit sum checksum (ignored on `libattU`).
- **Commands**: `0x02` (`RT` real-time telemetry: current at offset 44 as `(u16_hi - u16_lo) / 100` A, temp at 48 `raw - 40` °C, MOSFET/alarm bits at 52, balancer at 55, cycles at 57, SOC at 61, design cap at 66) and `0x10` (`CAP`: remaining Ah at offset 7 `0.1 Ah`).

---

## 5. RoyPow (` [BS]12*`, ` [BS]24*`, ` UT*`)

- **GATT**: Service `0000ffe0-…`, RX/TX `0000ffe1-…` (strips `AT+STAT\r\n` injected by BLE bridge).
- **Framing**: Header `EA D1 01`, footer `0xF5`.
- **Key fields**:
  - `0x03`: current (`0.01 A`, sign bit 16, offset 6), problem code (9), temp count (13), charge/discharge MOSFET flags (byte 24 bits 2/1).
  - `0x04`: SOC (`%`, offset 7), cycles (9), remaining Ah (offset 24, mixed-endian 32-bit `/ 1000`), runtime (`min`, offset 30), pack voltage (`0.01 V`, offset 47).

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `ogt_bms.py`, `dpwrcore_bms.py`, `tdt_bms.py`, `ej_bms.py`, `roypow_bms.py`

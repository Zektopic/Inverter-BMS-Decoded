# JK (Jikong) BMS

JK makes two BMS families with different protocols, plus a display CAN protocol. Pick the section for your hardware.

| Family | Typical models | Interfaces | Section |
|---|---|---|---|
| Inverter BMS ("PB") | JK-PB1A16S…, JK-PB2A16S… | RS485 ×2, CAN, Bluetooth | [JK-PB](#jk-pb-inverter-bms-rs485-modbus) |
| Smart BMS ("B"/"BD") | JK-B2A24S…, JK-BD6A20S… | 4-pin UART ("GPS" port), Bluetooth | [JK-B / JK-BD](#jk-b--jk-bd-smart-bms-uart-and-bluetooth) |
| BMS → display | JK BMS with a CAN display | CAN 250 kbit/s | [Display CAN](#bms-to-display-can-protocol-v102) |

For inverters, JK-PB also speaks the Pylontech low-voltage CAN protocol on its inverter port: see
[PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol).

---

## JK-PB inverter BMS: RS485 Modbus

> Status: ✅ Verified. Offsets checked against two 300-byte frames captured from a JK-PB2A16S20P (firmware 15.41), published in
> the [dbus-serialbattery test suite](https://github.com/mr-manuel/venus-os_dbus-serialbattery), and against the official register table.

| | |
|---|---|
| Port | the RS485 port for parallel packs / PC ("Modbus"), not the inverter RS485/CAN port |
| Serial | 115200 8N1 |
| Address | DIP switch, 1–15 |
| Official document | *Jikong BMS RS485 Modbus Universal Protocol V1.0 / V1.1* (copies in [phinix-org/Multiple-JK-BMS-by-Modbus-RS485](https://github.com/phinix-org/Multiple-JK-BMS-by-Modbus-RS485/tree/main/docs)) |

### Two ways to read it

1. **Modbus registers.** Blocks start at `0x1000` (settings, RW), `0x1200` (live data, R), `0x1400` (device info, R) and
   `0x1600` (commands, W). The document lists each field as *base + byte offset*. Reading a block from its base address returns
   its memory from offset 0, which avoids any ambiguity about how later register numbers map to offsets.
2. **Trigger frames** (the method dbus-serialbattery uses). Write `0x0000` with FC16 to `0x1620`; the BMS acknowledges
   (`<addr> 10 16 20 00 01 <crc>`) and sends a **300-byte frame** `55 AA EB 90 02 …` that contains the `0x1200` block,
   **little-endian**, starting at frame offset 6, with a sum-8 checksum in the last byte. `0x161E` returns the settings frame
   (type `01`), `0x161C` the device-info frame (type `03`).

Request for address 3 (captured): `03 10 16 20 00 01 02 00 00 CF 91`.

### Live data (`0x1200` block, byte offsets; frame offset = 6 + offset)

| Offset | Type | Unit | Field |
|---|---|---|---|
| `0x00` | 32 × u16 | mV | cell voltages 1–32 |
| `0x40` | u32 | bitmask | cells present |
| `0x44` / `0x46` | u16 | mV | average cell voltage / max difference |
| `0x48` / `0x49` | u8 | | cell number with max / min voltage |
| `0x4A` | 32 × u16 | mΩ | balance-wire resistance per cell |
| `0x8A` | s16 | 0.1 °C | MOSFET (power board) temperature |
| `0x8C` | u32 | bitmask | balance-wire resistance alarm per cell |
| `0x90` | u32 | mV | pack voltage |
| `0x94` | u32 | mW | pack power |
| `0x98` | s32 | mA | current (+ charge) |
| `0x9C` / `0x9E` | s16 | 0.1 °C | battery temperature 1 / 2 |
| `0xA0` | u32 | bits | alarms (below) |
| `0xA4` | s16 | mA | balancing current |
| `0xA6` / `0xA7` | u8 | — / % | balancing state (0 off, 1 charge, 2 discharge) / SOC |
| `0xA8` | s32 | mAh | remaining capacity |
| `0xAC` | u32 | mAh | full-charge capacity |
| `0xB0` | u32 | | cycle count |
| `0xB4` | u32 | mAh | total cycled capacity |
| `0xB8` / `0xB9` | u8 | % / bool | SOH / precharge on |
| `0xBC` | u32 | s | run time |
| `0xC0` / `0xC1` | u8 | bool | charge / discharge MOSFET on |
| `0xD0` | u8 | bits | bit 0 MOS sensor present, bits 1–5 battery sensors 1–5 present; heating flag |
| `0xE4` | u16 | 0.01 V | pack voltage (second copy) |
| `0xF8` / `0xFA` / `0xFC` | s16 | 0.1 °C | battery temperature 3 / 4 / 5 |

Alarm bits (`0xA0`): 0 balance-wire resistance, 1 MOS over-temperature, 2 cell count mismatch, 3 current sensor, 4 cell OVP,
5 pack OVP, 6 charge OCP, 7 charge short circuit, 8 charge OTP, 9 charge UTP, 10 internal communication, 11 cell UVP, 12 pack UVP,
13 discharge OCP, 14 discharge short circuit, 15 discharge OTP, 16 charge MOS fault, 17 discharge MOS fault, 18 GPS disconnected,
19 change-password reminder, 20 discharge start failure, 21 battery over-temperature alarm, 22 temperature sensor fault,
23 parallel module fault.

Device-info frame (`0x161C`, type `03`): model at offset 6 (16 ASCII, e.g. `JK_PB2A16S20P`), hardware version 22 (8), software
version 30 (8, e.g. `15.41`), run time 38, power-on count 42, device name 46, **Bluetooth passcode 62 (never publish it)**,
manufacturing date 78, serial number 86.

Settings (`0x1000`, u32/s32 each): smart-sleep voltage, cell UVP/UVPR/OVP/OVPR, balance trigger, SOC 100 %/0 % voltages, recommended
charge and float voltages, shutdown voltage, charge/discharge over-current and delays, maximum balance current, charge/discharge/MOS
temperature limits, cell count, charge/discharge/balance switches, design capacity, and more (see the official document).
**Writing them changes battery protection.**

---

## JK-B / JK-BD smart BMS: UART and Bluetooth

> Status: 🌐 Community, from [syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms) (`jk_bms` / `jk_modbus` for UART,
> `jk_bms_ble` for Bluetooth) and the *JK-Jiabaida communication protocol v4.0* PDF in that repository's `docs/`.

**UART ("GPS" port, 115200 8N1)**: frames start with `4E 57`.

| Bytes | Field |
|---|---|
| 0–1 | `4E 57` start |
| 2–3 | length (big-endian) |
| 4–7 | BMS terminal number |
| 8 | command: `01` activate, `02` write, `03` read, `05` password, `06` read all |
| 9 | frame source: `00` BMS, `01` Bluetooth, `02` GPS, `03` PC |
| 10 | frame type: `00` read, `01` reply, `02` active upload |
| 11… | register id and data; read-all replies contain a sequence of `id value` items (cell voltages, temperatures, voltage, current, SOC, …) |
| end | record number (4 bytes), `68` end marker, checksum (4 bytes; 16-bit sum of all previous bytes in the last two) |

**Bluetooth**: frames `55 AA EB 90 <type> <counter> …`, type `01` settings, `02` cell info, `03` device info. Newer firmware
(JK02_32S) uses the same layout as the JK-PB frames above. See [BLE.md](BLE.md) for the full GATT UUIDs (`0000ffe0`/`0000ffe1`),
20-byte `AA 55 90 EB` commands, and byte-offset tables for `JK02_24S` and `JK02_32S`.

---

## BMS to display CAN protocol (V1.02)

> Status: 📄 Vendor doc: [`PDF/1_CAN英文.pdf`](PDF/) (Jikong, V1.02, 2019-04-28).

| | |
|---|---|
| Bus | CAN 2.0A, 11-bit ids, **250 kbit/s**, point-to-point BMS → display |
| Byte order | little-endian |
| Id | `FUNC << 8 \| 0xF4` (source address 0xF4 = BMS) |
| Current sign | **positive = discharge**, negative = charge |

| Id | Name | Period | Fields (start bit, length, scale, offset) |
|---|---|---|---|
| `0x02F4` | battery status | 20 ms | voltage (0, 16, 0.1 V); current (16, 16, 0.1 A, −400 A); SOC (32, 8, %); discharge time (48, 16, h) |
| `0x04F4` | cell voltage | 100 ms | max cell (0, 16, mV); its number (16, 8); min cell (24, 16, mV); its number (40, 8) |
| `0x05F4` | cell temperature | 100 ms | max (0, 8, °C, −50); its number (8, 8); min (16, 8, −50); its number (24, 8); average (32, 8, −50) |
| `0x07F4` | alarms | on event | 15 alarms × 2 bits (level 0 none, 1 serious, 2 important, 3 general): cell over/under voltage, pack over/under voltage, cell difference, discharge/charge over-current, temperatures, SOC low, insulation, … |

Examples from the document:

```
02F4 13 01 D7 11 33 XX 64 00   -> 27.5 V, 56.7 A, SOC 51 %, discharge time 100 h
04F4 8C 0A 05 92 09 08 XX XX   -> max cell 2700 mV (#5), min cell 2450 mV (#8)
05F4 48 06 2F 01 3F XX XX XX   -> max 22 °C (#6), min −3 °C (#1), average 13 °C
```

---

## Submodules

- [`JK/`](JK) → [NEEY-electronic/JK](https://github.com/NEEY-electronic/JK): JK smart BMS instructions (PDF).
- [`JK-BMSToPylontechCAN/`](JK-BMSToPylontechCAN) → [ArminJo/JK-BMSToPylontechCAN](https://github.com/ArminJo/JK-BMSToPylontechCAN):
  Arduino bridge from JK-B UART to Pylontech CAN; its `Pylontech_CAN.h` documents the CAN frames with sample data.

Clone with `git clone --recurse-submodules` to get them.

## Sources

- Jikong, *BMS RS485 Modbus Universal Protocol V1.0/V1.1*: [phinix-org/Multiple-JK-BMS-by-Modbus-RS485/docs](https://github.com/phinix-org/Multiple-JK-BMS-by-Modbus-RS485/tree/main/docs)
- [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery): `bms/jkbms_pb.py`, `tests/bms/test_jkbms_pb.py` (captured frames)
- [syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms): `components/jk_bms_ble`, `components/jk_modbus`
- Jikong, *BMS-CAN communication protocol V1.02*: [`PDF/1_CAN英文.pdf`](PDF/)

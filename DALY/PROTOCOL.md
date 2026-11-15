# Daly smart BMS — UART / RS485 protocol

> Status: 🌐 Community. Built from [maland16/daly-bms-uart](https://github.com/maland16/daly-bms-uart) and
> [dbus-serialbattery `bms/daly.py`](https://github.com/mr-manuel/venus-os_dbus-serialbattery). No captures in this repository yet.

| | |
|---|---|
| Port | UART (3.3 V TTL) or the RS485 module |
| Serial | 9600 8N1 |
| Frame | 13 bytes: `A5`, address, command, `08`, 8 data bytes, checksum |
| Checksum | sum of bytes 0–11, low 8 bits |
| Address | host → BMS: `0x40` + board number − 1 (board 1 = `0x40`); BMS → host: the board number (`0x01`) |
| Byte order | big-endian |

Request (board 1, command `0x90`): `A5 40 90 08 00 00 00 00 00 00 00 00 7D`.

## Commands

| Cmd | Reply data (8 bytes) |
|---|---|
| `0x90` | pack voltage (u16, 0.1 V), acquisition voltage (u16), current (u16, `(raw − 30000) × 0.1 A`), SOC (u16, 0.1 %) |
| `0x91` | max cell voltage (mV) + cell number, min cell voltage (mV) + cell number |
| `0x92` | max temperature (°C + 40) + sensor number, min temperature + sensor number |
| `0x93` | state (0 idle, 1 charge, 2 discharge), charge MOS, discharge MOS, BMS life, remaining capacity (u32, mAh) |
| `0x94` | cell count, temperature sensor count, charger connected, load connected, DIO states, cycle count (u16) |
| `0x95` | cell voltages: ⌈cells / 3⌉ reply frames, each `frame number` + 3 × u16 mV |
| `0x96` | temperatures: ⌈sensors / 7⌉ reply frames, each `frame number` + 7 × (°C + 40) |
| `0x97` | balancing state, one bit per cell (byte 0 bit 0 = cell 1) |
| `0x98` | alarms (below) |
| `0x50` | rated capacity (u32, mAh) and nominal cell voltage (u32) |
| `0x53` / `0x57` | production details / battery code (several frames) |

Control, **do not send to a live battery** unless you mean it: `0x21` set SOC, `0xD9` discharge MOS on/off, `0xDA` charge MOS on/off.

## Alarm bits (`0x98`)

Each condition has a level-1 (warning) bit and a level-2 (alarm) bit.

| Byte | Bits |
|---|---|
| 0 | cell high L1/L2, cell low L1/L2, pack high L1/L2, pack low L1/L2 |
| 1 | charge temp high L1/L2, charge temp low L1/L2, discharge temp high L1/L2, discharge temp low L1/L2 |
| 2 | charge over-current L1/L2, discharge over-current L1/L2, SOC high L1/L2, SOC low L1/L2 |
| 3 | cell voltage difference L1/L2, temperature difference L1/L2 |
| 4 | charge / discharge MOS over-temperature, MOS temperature sensor faults, MOS adhesion faults, MOS open-circuit faults |
| 5 | AFE chip, voltage acquisition, cell temperature sensor, EEPROM, RTC, precharge, communication, internal communication faults |
| 6 | current module, pack voltage detection, short-circuit protection, low-voltage charge forbidden |
| 7 | fault code |

## Open questions

- The sign of the `0x90` current differs between implementations (some invert it). Check against the Daly app.
- Multi-board RS485 addressing beyond board 1 is not confirmed by a capture.

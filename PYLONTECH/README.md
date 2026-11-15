# Pylontech (low voltage)

Two protocols matter: **RS485** for monitoring (polling) and **CAN** towards an inverter (broadcast). The CAN protocol has become a
de-facto standard: JK-PB, Seplos, Daly, MUST and many other BMSs offer it as "Pylontech CAN".

---

## RS485 protocol V3.3

> Status: 📄 Vendor doc: *PYLON low voltage Protocol RS485 V3.3 (2018-08-21)*, copy in
> [syssi/esphome-seplos-bms/docs](https://github.com/syssi/esphome-seplos-bms/tree/main/docs). Its example frame was checked; see errata.

| | |
|---|---|
| Models | US2000, US2000B/C, US3000, US5000 and compatible BMS |
| Serial | 115200 8N1 (default); 9600 with DIP 1 up (restart needed) |
| Framing | YD/T 1363 ASCII, same as [Seplos V2](../SEPLOS/README.md#frame-format): `~` VER ADR CID1 CID2 LENGTH INFO CHKSUM `CR`, VER `0x20`, CID1 `0x46` |
| Address | **starts at 2** (master battery); multi-group `ADR = 0x0n + 0x10 × group` (group from DIP 2–4 of the master) |

### Commands (INFO = 1 byte, the address, unless noted)

| CID2 | Reply |
|---|---|
| `0x4F` | protocol version (no INFO) |
| `0x51` | manufacturer info: device name (10 ASCII), software version (2), manufacturer (20 ASCII) — no INFO |
| `0x42` | analog values (below) |
| `0x44` | alarm info (below) |
| `0x47` | system parameters: cell high / low / under-voltage limits, charge / discharge temperature limits, charge current limit, module voltage limits… (no INFO) |
| `0x92` | charge / discharge management: charge voltage limit (mV), discharge voltage limit (mV), charge current limit (0.1 A), discharge current limit (0.1 A), status (bit 7 charge enable, 6 discharge enable, 5/4 force charge I/II, 3 full-charge request) |
| `0x93` | serial number (16 ASCII) |
| `0x96` | software version |
| `0x94` | **set** charge / discharge management — write |
| `0x95` | **turn off** — write |

### `0x42` analog reply

| Field | Size | Unit |
|---|---|---|
| INFO flag, command value | 1 + 1 | |
| cell count M | 1 | |
| cell voltages | 2 × M | mV |
| temperature count N | 1 | |
| temperatures (#1 BMS board, #2–5 cell groups, …) | 2 × N | 0.1 K (°C = (raw − 2731) / 10) |
| current | 2, signed | **100 mA** (+ charge) |
| module voltage | 2 | mV |
| remaining capacity 1 | 2 | mAh (`FFFF` if > 65 Ah) |
| user-defined count P | 1 | 2, or 4 for > 65 Ah packs |
| total capacity 1 | 2 | mAh (`FFFF` if > 65 Ah) |
| cycle count | 2 | |
| remaining capacity 2 (P = 4) | 3 | mAh |
| total capacity 2 (P = 4) | 3 | mAh |

There is no SOC field: SOC = remaining / total.

### `0x44` alarm reply

Per-cell, per-temperature, charge current, module voltage and discharge current bytes (`00` normal, `01` below limit, `02` above
limit, `F0` other; both limit values "act as protection"), then:

| Byte | Bits |
|---|---|
| Status 1 | 7 module under-voltage, 6 charge over-temperature, 5 discharge over-temperature, 4 discharge over-current, 2 charge over-current, 1 cell under-voltage, 0 module over-voltage |
| Status 2 | 3 using battery power, 2 discharge MOSFET on, 1 charge MOSFET on, 0 pre-MOSFET |
| Status 3 | 7 effective charge current, 6 effective discharge current, 5 heater, 3 fully charged, 0 buzzer |
| Status 4 / 5 | cell 1–8 / 9–16 voltage error (> 4.2 V or < 1 V) |

### Errata

The analog-reply example in the V3.3 document (page 24) is printed with CHKSUM `E1A2`. All fields decode correctly
(15 cells, 50.981 V, 51.8 / 74.0 Ah via the 24-bit fields, 2 cycles), but the correct checksum is `E27A`:

```
~20024600F07A11020F0D450D440D450D440D450D440D3E0D450D4A0D4A0D4B0D4A0D4A0D4A0D4A050BC30BC30BC30BCD0BCD0000C725FFFF04FFFF000200CA58012110E27A
```

Request for that pack (from the same document): `~20024642E00202FD33`.

### Variant: "Force_L" commands `0x60`–`0x64`

Some batteries (for example the [TDT](../TDT/README.md) pack, whose basic-info reply names `Force_L` / `Pylon`) answer CID2
`0x60` basic info, `0x61` analog data, `0x62` alarms, `0x63` charge/discharge management and `0x64` shutdown instead.

---

## Low-voltage CAN protocol

> Status: 📄 Vendor doc (*Pylontech CAN-Bus protocol, low voltage, V1.2*), cross-checked with the sample frames in
> [ArminJo/JK-BMSToPylontechCAN `Pylontech_CAN.h`](https://github.com/ArminJo/JK-BMSToPylontechCAN).

| | |
|---|---|
| Bus | CAN 2.0A, 11-bit ids, 500 kbit/s |
| Byte order | little-endian |
| Roles | the BMS broadcasts about once per second; the inverter sends `0x305` (keep-alive) |

| Id | Content | Example (from the source) |
|---|---|---|
| `0x305` | inverter → BMS keep-alive, 8 bytes | |
| `0x351` | charge voltage limit (0.1 V), charge current limit (0.1 A, signed), discharge current limit (0.1 A, signed), discharge voltage limit (0.1 V) | `28 02 F4 01 20 03 E0 01` → 55.2 V, 50.0 A, 80.0 A, 48.0 V |
| `0x355` | SOC (%), SOH (%) — u16 each | `3C 00 64 00` → 60 %, 100 % |
| `0x356` | voltage (0.01 V), current (0.1 A, signed, + charge), temperature (0.1 °C) | `6C 14 04 00 DC 00` → 52.28 V, 0.4 A, 22.0 °C |
| `0x359` | protections: byte 0 bit 1 cell over-voltage, 2 cell under-voltage, 3 over-temperature, 4 under-temperature, 7 discharge over-current; byte 1 bit 0 charge over-current, 7 system error. Warnings: bytes 2–3, same bits. Byte 4 module count, bytes 5–6 `"PN"` | `00 00 00 00 01 50 4E` → no alarms, 1 module |
| `0x35C` | request flags: bit 7 charge enable, 6 discharge enable, 5 force charge I, 4 force charge II, 3 full-charge request | `C0 00` → charge and discharge enabled |
| `0x35E` | manufacturer name, 8 ASCII | `50 59 4C 4F 4E 20 20 20` → `PYLON` |

Some BMSs send the discharge current limit as a negative number. Extensions used by some inverters: `0x35F` capacity (SMA Sunny
Island), `0x373` cell min/max (BYD-style), `0x379` capacity (Luxpower); see [MUST](../MUST/README.md) for another extended set.

## Sources

- Pylontech, *PYLON low voltage Protocol RS485 V3.3*, 2018-08-21 ([copy](https://github.com/syssi/esphome-seplos-bms/tree/main/docs))
- Pylontech, *CAN-Bus protocol PYLON low voltage V1.2*
- [ArminJo/JK-BMSToPylontechCAN](https://github.com/ArminJo/JK-BMSToPylontechCAN), `Pylontech_CAN.h` (frame layout and samples)

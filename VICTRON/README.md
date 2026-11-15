# Victron Energy — VE.Direct & GX CAN-Bus BMS Protocols

> Status: 📄 Vendor doc / 🌐 Community — VE.Direct serial framing from Victron *VE.Direct Protocol* specification; GX CAN-bus BMS protocol cross-checked with [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol) and [`mr-manuel/venus-os_dbus-serialbattery`](https://github.com/mr-manuel/venus-os_dbus-serialbattery).

## 1. VE.Direct Text & HEX Protocol (Shunts, MPPTs, Inverters)

### Summary

| | |
|---|---|
| Products | SmartShunt, BMV-700/712, BlueSolar/SmartSolar MPPT, Phoenix VE.Direct inverters |
| Physical layer | 4-pin JST-PH 2.0 mm (`GND`, `RX`, `TX`, `VCC`), 3.3 V / 5 V TTL UART |
| Settings | **`19200` 8N1**, no flow control |
| Roles | Device broadcasts a Text-mode block every `1 s`; host can send HEX commands (`:...`) |

### Text-Mode Framing & Checksum

Each block consists of `\r\n<Label>\t<Value>` pairs, ending with `\r\nChecksum\t<byte>`.

- **Checksum**: Modulo-256 sum of **all bytes in the block** (including `\r`, `\n`, `\t`, and the `Checksum` byte itself) must equal **`0x00`** (`sum(block) % 256 == 0`).
- **Worked example (verified in Python):**
  ```
  \r\nPID\t0xA389\r\nV\t13250\r\nI\t-1250\r\nSOC\t850\r\nChecksum\t\xb2
  # Sum of bytes before 0xB2 = 0x104E; (-0x104E) & 0xFF = 0xB2; (0x104E + 0xB2) & 0xFF = 0
  ```

### Common VE.Direct Text Fields

| Label | Unit / scale | Meaning |
|---|---|---|
| `PID` | hex string | Product ID (e.g. `0xA389` = SmartShunt 500 A) |
| `V` | mV | Main battery voltage |
| `I` | mA (signed) | Battery current (`+` charge, `−` discharge) |
| `P` | W (signed) | Instantaneous power |
| `SOC` | 0.1 % (per mille) | State of charge (`1000` = 100.0 %) |
| `TTG` | min | Time-to-go (`-1` = infinite) |
| `CE` | mAh | Consumed Ah |
| `CS` | enum | State of operation (`0` Off, `3` Bulk, `4` Absorption, `5` Float, `9` Inverting) |
| `ERR` | enum | Error code (`0` = No error) |
| `H1`–`H18` | various | Historical statistics (deepest discharge, cycles, min/max V) |

### HEX-Mode Framing

Commands start with `:` (`0x3A`), ASCII hex payload, 8-bit two's-complement checksum (`(0x55 - sum(bytes)) & 0xFF` for `:1` ping, `:7` get, `:8` set), terminated by `\n`.

---

## 2. Victron GX CAN-Bus BMS Protocol (`VE.Can` / `BMS-Can`)

| | |
|---|---|
| Bus | CAN 2.0A, 11-bit IDs, **500 kbit/s** (`BMS-Can` port; `250 kbit/s` on `VE.Can`) |
| Byte order | **Little-endian** |
| Compatibility | Superset of [Pylontech low-voltage CAN](../PYLONTECH/README.md#low-voltage-can-protocol) |

| CAN ID | Period | Content |
|---|---|---|
| `0x305` | 1 s | GX → BMS keep-alive |
| `0x351` | 1 s | Charge voltage limit (`0.1 V`), max charge current (`0.1 A`), max discharge current (`0.1 A`), discharge voltage limit (`0.1 V`) |
| `0x355` | 1 s | SOC (`%`, u16), SOH (`%`, u16), high-res SOC (`0.01 %`, u16 at bytes 4–5, optional) |
| `0x356` | 1 s | Pack voltage (`0.01 V`, s16), current (`0.1 A`, s16), temperature (`0.1 °C`, s16) |
| `0x35A` | 1 s | Victron 8-byte alarm/warning bitfield (4 bytes alarms + 4 bytes warnings, 2 bits per item: `00` no, `01`/ `10` active) |
| `0x35E` | 1 s | Manufacturer name (8 ASCII) |
| `0x35F` | 1 s | Battery model / firmware version / available capacity (Ah) |
| `0x370`–`0x379` | 1–5 s | Extended Victron / BYD / Luxpower info: `0x372` modules online/blocking, `0x373` min/max cell V (mV) & temp (K), `0x374`–`0x377` min/max cell ID strings, `0x379` installed capacity (Ah) |

## Sources

- Victron Energy, *VE.Direct Protocol* specification ([Victron Live: VE.Direct Protocol](https://www.victronenergy.com/live/vedirect_protocol:start))
- Victron Energy, *GX Device CAN-bus BMS* & [`Sleeper85/esphome-yambms`](https://github.com/Sleeper85/esphome-yambms)

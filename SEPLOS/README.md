# Seplos BMS — protocol V2.0 (RS485, ASCII)

> Status: ✅ Verified. Field tables checked against frames captured from a real Seplos pack (see [captures](captures/seplos_v2_frames.txt))
> and against public frames from [syssi/esphome-seplos-bms](https://github.com/syssi/esphome-seplos-bms).

| | |
|---|---|
| BMS family | Seplos EMU10xx / 1101-SPxx (16S LiFePO4, 48 V) |
| Port | RS485 (the reader is the master); also RS232 on some boards |
| Serial | 19200 baud, 8N1 |
| Framing | YD/T 1363.3 ASCII: `~` … `\r`, hex-ASCII payload |
| Pack address | 0–15, set by the DIP switch on the pack |
| Not supported by this protocol | Seplos **V3** BMS ([see V3](V3.md)) (reported to answer Modbus RTU and ignore these frames; [HA community](https://community.home-assistant.io/t/seplos-bms-v3-mason-10e-all-16-cell-voltages-with-plain-modbus-yaml-no-custom-component-no-esp-no-mqtt/1017847)). For Bluetooth (BLE) on V2 and V3, see [BLE.md](BLE.md). |

## Sources

- Seplos, *MODBUS-ASCII Communication Protocol* for EMU10XX (vendor PDF; public copy on the
  [ioBroker forum](https://forum.iobroker.net/assets/uploads/files/1631599969177-seplos-48v-100a-bms-rs485-protocol.pdf)).
  Despite its title this is not Modbus; it is the YD/T 1363.3 ASCII format.
- Request/response frames captured from a real pack by the Vega team, with hand decodes of an alarm frame, serial numbers and a
  history record: [captures/seplos_v2_frames.txt](captures/seplos_v2_frames.txt) (serial numbers redacted).
- [syssi/esphome-seplos-bms](https://github.com/syssi/esphome-seplos-bms): public telemetry and device-info frames.

## Frame format

```
~  20  00  46  42  E002  00   FD37  \r
│  │   │   │   │   │     │    │     └ EOI 0x0D
│  │   │   │   │   │     │    └ CHKSUM (4 hex chars)
│  │   │   │   │   │     └ INFO (LENID hex chars)
│  │   │   │   │   └ LENGTH = LCHKSUM(4 bit) | LENID(12 bit)
│  │   │   │   └ CID2: command (request) or return code (reply)
│  │   │   └ CID1 0x46 = LiFePO4 battery BMS
│  │   └ ADR: pack address
│  └ VER 0x20 = protocol 2.0
└ SOI 0x7E '~'
```

Every field except SOI/EOI is sent as two uppercase hex ASCII characters per byte.

- **LENID** = number of ASCII characters in INFO (so INFO bytes = LENID / 2).
- **LCHKSUM** = sum of the three LENID nibbles, mod 16, inverted, plus 1.
  Example from the spec: LENID 18 → 0+1+2 = 3 → LCHKSUM 0xD → LENGTH `D012`.
- **CHKSUM** = sum of the ASCII codes of every character between SOI and CHKSUM, mod 65536, inverted, plus 1.
  Example: `~20004642E00200` sums to 0x02C9 → CHKSUM `FD37`.

### Return codes (CID2 of a reply)

| RTN | Meaning |
|---|---|
| 0x00 | OK |
| 0x01 | VER error |
| 0x02 | CHKSUM error |
| 0x03 | LCHKSUM error |
| 0x04 | invalid CID2 |
| 0x05 | invalid command format |
| 0x06 | invalid data |
| 0x07 | no data (end of history) |
| 0xE1 | invalid CID1 |
| 0xE2 | command execution failed |
| 0xE3 | equipment fault |
| 0xE4 | no permission |

## Commands

| CID2 | Name | Request INFO | Read-only? |
|---|---|---|---|
| 0x42 | Telemetry (analog values) | 1 byte: pack address | yes |
| 0x44 | Telesignalization (alarms, status) | 1 byte: pack address | yes |
| 0x45 | Remote control (FETs, shutdown, reset) | 3 bytes | **no: control** |
| 0x47 | System parameters | none | yes |
| 0x4B | History record | 2 bytes: address, `02` first / `01` next | yes |
| 0x4D | Get time | none | yes |
| 0x4E | Set time | 7 bytes | no: writes the clock |
| 0x4F | Protocol version | none | yes |
| 0x51 | Device info (model, firmware, manufacturer) | none | yes |
| 0xA0 | Production calibration | — | **no: control** |
| 0xA2 | Pack serial number (observed) | none | yes |
| 0xA4 | BMS serial number (observed) | none | yes |

The vendor table lists 0xA2 as "timed recording"; on the captured pack it returns the pack serial
(e.g. `EMU1101xxx-XXXX-2021xxxx-xxxxx`), and 0xA4 returns the BMS serial (e.g. `SEPLOS-21xxxxxxxx`). Treat these two as firmware
dependent.

> **Do not send 0x45 or 0xA0 to a live battery.** 0x45 switches the charge/discharge MOSFETs, current limiter, heater,
> shuts the BMS down or factory-resets it; 0xA0 rewrites the voltage/current calibration. The captured control frames are
> listed in the captures file for reference only.

### Request frames for every pack (computed, all checksums verified)

Address 0 examples (all checksums verified):

| Purpose | Frame |
|---|---|
| Telemetry | `~20004642E00200FD37\r` |
| Alarms | `~20004644E00200FD35\r` |
| History, first record | `~2000464BC0040002FCC5\r` |
| History, next record | `~2000464BC0040001FCC6\r` |
| Get time | `~2000464D0000FD9C\r` |
| Protocol version | `~2000464F0000FD9A\r` |
| Device info | `~200046510000FDAE\r` |
| Pack serial | `~200046A20000FDA1\r` |
| BMS serial | `~200046A40000FD9F\r` |

Alarm requests for addresses 1–15: `~2001…FD33`, `~2002…FD31`, `~2003…FD2F`, `~2004…FD2D`, `~2005…FD2B`, `~2006…FD29`,
`~2007…FD27`, `~2008…FD25`, `~2009…FD23`, `~200A…FD13`, `~200B…FD11`, `~200C…FD0F`, `~200D…FD0D`, `~200E…FD0B`,
`~200F…FD09`. Build them from the framing rules rather than copying tables.

## 0x42 telemetry reply (75 INFO bytes for 16 cells)

| # | Field | Bytes | Unit / conversion |
|---|---|---|---|
| 1 | Data flag | 1 | — |
| 2 | Command group (address) | 1 | — |
| 3 | Cell count M | 1 | 16 on EMU10xx (8–16 seen) |
| 4 | Cell voltage 1…M | 2×M | mV |
| 5 | Temperature count N | 1 | 6 |
| 6 | Cell temp 1–4, ambient, MOSFET | 2×N | 0.1 K: °C = (raw − 2731) / 10 |
| 7 | Current | 2 | **signed**, 0.01 A, + = charging |
| 8 | Pack voltage | 2 | 0.01 V |
| 9 | Remaining capacity | 2 | 0.01 Ah |
| 10 | Custom count P | 1 | 10 (number of following words) |
| 11 | Full-charge capacity | 2 | 0.01 Ah (spec says Ah; real packs send 0.01 Ah) |
| 12 | SOC | 2 | per mille |
| 13 | Rated capacity | 2 | 0.01 Ah |
| 14 | Cycles | 2 | count |
| 15 | SOH | 2 | per mille |
| 16 | Port voltage | 2 | 0.01 V |
| 17–20 | Reserved | 2×4 | — |

Some BMS firmware stops after field 13. **Current is signed**: parsing it as unsigned hides discharge.

## 0x44 alarm / status reply (49 INFO bytes for 16 cells)

| # | Field | Bytes |
|---|---|---|
| 1 | Data flag | 1 |
| 2 | Command group | 1 |
| 3 | Cell count M | 1 |
| 4 | Cell voltage warning 1…M | M |
| 5 | Temperature count N | 1 |
| 6 | Temp warning: cell 1–4, ambient, MOSFET | N |
| 7 | Current warning | 1 |
| 8 | Pack voltage warning | 1 |
| 9 | Bit-warning count P | 1 (20) |
| 10 | Bit warnings (table below) | P |

Byte warnings: `0x00` normal, `0x01` below lower limit, `0x02` above upper limit, `0xF0` other.

| Byte | Bit 0 | Bit 1 | Bit 2 | Bit 3 | Bit 4 | Bit 5 | Bit 6 | Bit 7 |
|---|---|---|---|---|---|---|---|---|
| Warning 1 | voltage sensing fail | temp sensing fail | current sensing fail | power switch fail | cell voltage difference | charge switch fail | discharge switch fail | current-limit switch fail |
| Warning 2 | cell OV warn | cell OV prot | cell UV warn | cell UV prot | pack OV warn | pack OV prot | pack UV warn | pack UV prot |
| Warning 3 | charge OT warn | charge OT prot | charge UT warn | charge UT prot | discharge OT warn | discharge OT prot | discharge UT warn | discharge UT prot |
| Warning 4 | ambient OT warn | ambient OT prot | ambient UT warn | ambient UT prot | MOSFET OT warn | MOSFET OT prot | heating | reserved |
| Warning 5 | charge OC warn | charge OC prot | discharge OC warn | discharge OC prot | transient OC prot | short-circuit prot | transient OC lock | short-circuit lock |
| Warning 6 | charge high-voltage prot | intermittent recharge wait | remaining capacity warn | remaining capacity prot | cell low voltage, charge forbidden | output reverse prot | output connection fail | internal |
| Power status | discharge FET on | charge FET on | current limiter on | heater on | — | — | — | — |
| Equalization 1 | balancing cell 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| Equalization 2 | balancing cell 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
| System status | discharging | charging | float charge | reserved | standby | power off | reserved | reserved |
| Disconnection 1 | cell 1 wire open | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| Disconnection 2 | cell 9 wire open | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
| Warning 7 | internal | internal | internal | internal | auto charge wait | manual charge wait | internal | internal |
| Warning 8 | EEPROM fail | RTC fail | voltage not calibrated | current not calibrated | zero point not calibrated | internal | internal | internal |
| 15–20 | reserved | | | | | | | |

Captured example (pack 0): cell 11 below its lower limit, "cell voltage difference" set, both FETs on, system standby.

## 0x4B history record (97 INFO bytes)

Decoded from the captured frames (not in the vendor PDF):

| # | Field | Bytes | Notes |
|---|---|---|---|
| 1 | Group | 1 | |
| 2 | Command type | 1 | |
| 3 | Year | 2 | e.g. `07E6` = 2022 |
| 4 | Month, day, hour, minute, second | 5 | |
| 5 | System status | 1 | same bits as the 0x44 system status (`01` discharging, `10` standby) |
| 6 | Warning count K | 1 | 15 |
| 7 | Warning bytes | K | **mapping not yet known** |
| 8 | Value count D | 1 | 35 |
| 9 | Values | 2×D | 16 cells, 6 temps, current, voltage, remaining, full, SOC, rated, cycles, SOH, port voltage, 4 reserved (same units as 0x42) |

Reading the log: send `…4B…0002` (first), then `…4B…0001` (next) until the reply's RTN is 0x07 (no data). The spreadsheet's
"500" row suggests up to about 500 records. The history sheet's first record is dated 2000-12-03, probably before the
pack's clock was set.

## Errata in the original capture spreadsheet

Found while checking every frame's checksums:

- Sheet "ALARM REQ & RES", **ASCII** column, packs 7, 8, 9 and 11: the address digits were not updated
  (for example pack 7 reads `~20064644E00206FD27`; the correct frame is `~20074644E00207FD27`). The **HEX** column is correct.
- Row 63 (history init, pack 2): ASCII and HEX columns disagree (`…0200FCC3` vs `…0201FCC2`).
- The init/next history codes are only consistent for pack 0 (`02` then `01`); the rows for other packs look hand-edited.
  Use the pack 0 behaviour (`02` first, then `01`) for every address.
- Sheet "Decoded History Message": the field labels from "Hour" onwards sit one column to the left of their data.
- Pack 0's 0x44 reply reports command group `01` although the request asked for `00`; packs 2–6 report their own
  address. Trust the ADR field of the header, not the command group.

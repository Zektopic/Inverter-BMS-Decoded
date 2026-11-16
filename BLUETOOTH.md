# Reading a BMS over Bluetooth LE — Cross-Brand Guide

> Status: 🌐 Community — a summary of the Bluetooth pages in this repository, plus notes from writing a BLE central for five
> of them (JK, LiTime, Daly, JBD, Seplos V3) on an ESP32-S3. The decoders for those five are unit-tested against the captured
> frames cited on their pages. That implementation has **not yet been connected to a real battery**, so the connection advice
> below comes from the upstream implementations and from bench tests without a battery. Treat it as a checklist, not as proof.

Every brand page describes its own frames. This page covers what they have in common: which GATT service to open, how to tell
the brands apart when they share a service, how to turn notifications back into frames, and what usually goes wrong.

## GATT quick reference

All UUIDs are 16-bit short forms of `0000xxxx-0000-1000-8000-00805f9b34fb`.

| Brand | Page | Service | Notify | Write | Advertising hints | Frame is complete when |
|---|---|---|---|---|---|---|
| JK-B / JK-BD / JK-PB | [JK/BLE](JK/BLE.md) | `FFE0` | `FFE1` | `FFE1` (same characteristic) | manufacturer id `0x0B65`, `0x4B4A` | 300 bytes from `55 AA EB 90`, sum-8 |
| LiTime / Redodo / Power Queen | [LITIME](LITIME/README.md) | `FFE0` | `FFE1` | `FFE2` | manufacturer id `0x585A`; names `L-`, `LT-`, `R-`, `RO-`, `P-`, `PQ-`, `S-`, `SS-` | `frame[2] + 4` bytes from `00 00`, sum-8 |
| ANT | [ANT](ANT/README.md) | `FFE0` | `FFE1` | `FFE1` | see page | see page |
| NEEY / Heltec balancer | [NEEY](NEEY/README.md) | `FFE0` | `FFE1` | `FFE1` | advertises service `FEE7` | see page |
| SOK (ABC) | [SOK](SOK/README.md) | `FFE0` | `FFE1` | `FFE2` | names `SOK-*`, `ABC-*`, `NB-*`, `Hoover`; advertises service `FFF0` | see page |
| Daly / 100Balance | [DALY/BLE](DALY/BLE.md) | `FFF0` | `FFF1` | `FFF2` | name `DL-*`; manufacturer id `0x0102`, `0x0104`, `0x0107`, `0x0302`, `0x0303`, `0x0402` | `3 + frame[2] + 2` bytes, CRC-16/Modbus |
| Seplos V3 | [SEPLOS/BLE](SEPLOS/BLE.md) | `FFF0` | `FFF1` | `FFF2` | names `SP??B*`, `SP1??B*`, `XZHX*`, `CSY*`, `ST*` | `frame[2] + 5` bytes, CRC-16/Modbus |
| ECO-WORTHY / DCHOUSE | [ECOWORTHY](ECOWORTHY/README.md) | `FFF0` | `FFF1` | `FFF2` | names `ECO-WORTHY*`, `DCHOUSE*` | see page |
| Offgridtec | [OGT](OGT/README.md) | `FFF0` | `FFF4` | `FFF6` | see page; the other brands on that page use different characteristics | see page |
| CBT Power / Creabest | [ECOWORTHY](ECOWORTHY/README.md) | `FFE5` (or `FFE0`) | `FFE4` | `FFE9` | see page | see page |
| JBD / Xiaoxiang / Overkill | [JBD](JBD/README.md) | `FF00` | `FF01` | `FF02` | name is set by the reseller and can be changed by the owner | `frame[3] + 7` bytes from `DD`, ends `77` |
| Seplos V2 | [SEPLOS/BLE](SEPLOS/BLE.md) | `FF00` | `FF01` | `FF02` | names `BP0*`, `BP1*`, `BP2*` | see page (`7E … 0D`, CRC-16/XMODEM) |
| Renogy BT-1 / BT-2 | [RENOGY](RENOGY/README.md) | write `FFD0`, notify `FFF0` | `FFF1` | `FFD1` | names `BT-TH-*`, `RNGRBP*`, `RNGC*` | `frame[2] + 5` bytes, CRC-16/Modbus |

## Telling the brands apart

Most of these boards use one of three generic UART-bridge services, so **the service UUID alone does not identify a brand**:

- `FFE0` / `FFE1`: JK, ANT, NEEY, LiTime and SOK.
- `FFF0` / `FFF1` / `FFF2`: Daly, Seplos V3 and ECO-WORTHY.
- `FF00` / `FF01` / `FF02`: JBD and Seplos V2.

Writing the wrong brand's request to a battery is harmless only as long as every request you can send is a read. Match in
this order, and let the user override the result:

1. **Manufacturer id** in the advertisement, where one is known (JK, LiTime, Daly). This is the strongest hint.
2. **Advertised name prefix.** Names come in the scan response, so use an active scan. Owners can rename some boards (JBD has
   a rename command), so a missing prefix proves nothing.
3. **Service UUID**, only to narrow the list.

## Connection sequence

1. **Scan** and keep each device's address **type** (public or random) with its address. A connect with the wrong type fails.
2. **Connect**, then ask for a larger ATT MTU (247 is a common choice). It is optional: it only reduces how many
   notifications a frame is split into.
3. **Discover** the service by UUID, then all of its characteristics. Pick the notify and write characteristics by UUID
   *and* by property; on JK they are the same characteristic.
4. **Subscribe**: find the Client Characteristic Configuration descriptor (`0x2902`) of the notify characteristic and write
   `01 00` to it.
5. **Brand handshake**, if there is one. JK needs the device-info request first ([JK/BLE](JK/BLE.md#connection-sequence));
   some JBD Bluetooth modules have an application-level authentication ([JBD](JBD/README.md)).
6. **Poll**: write one request, collect notifications until a full frame has arrived, check it, decode it.

## Notifications are a byte stream

A notification is not a frame. A 300-byte JK frame arrives as many small notifications; a 105-byte LiTime frame may arrive
whole or split, depending on the MTU; a Bluetooth module can insert its own bytes (`AT\r\n` on JK). Treat the notify
characteristic exactly like a serial port:

- Append every notification to one buffer.
- Search the buffer for the frame header, then wait until the length rule in the table above is satisfied.
- Verify the checksum or CRC. If it fails, drop one byte and search again, so that a header-like pattern inside the data
  cannot stall the stream.
- Give up after a timeout and clear the buffer before the next request. The implementation behind this page waits 2 s for
  the request/response brands and 4 s for JK, where another 300-byte frame can come first; neither figure has been tuned
  on a real battery.

A decoder written this way for RS485 works unchanged over Bluetooth where the frames are the same: JBD (`DD … 77`),
Seplos V3 and Daly `0xD2` (Modbus RTU), and the JK-PB 300-byte frame.

## Read-only polling requests

These are the only requests needed for live values. Every checksum below was recomputed in Python.

| Brand | Request | Reply |
|---|---|---|
| JK | `AA 55 90 EB 97 00 00 00 00 00 00 00 00 00 00 00 00 00 00 11` | `0x03` device info (once per connection) |
| JK | `AA 55 90 EB 96 00 00 00 00 00 00 00 00 00 00 00 00 00 00 10` | `0x02` cells and live values |
| LiTime | `00 00 04 01 13 55 AA 17` | 105-byte status (`0x93`) |
| Daly (`0xD2`) | `D2 03 00 00 00 3E D7 B9` | 62 registers: cells, temperatures, pack values |
| Daly (`0xD2`) | `D2 03 00 3E 00 09 F7 A3` | MOSFET temperature, if fitted |
| JBD | `DD A5 03 00 FF FD 77` | basic info and status |
| JBD | `DD A5 04 00 FF FC 77` | cell voltages |
| Seplos V3 | `01 04 10 00 00 11 34 C6` | `PIA`: pack 1 voltage, current, capacity, SOC |
| Seplos V3 | `01 04 11 00 00 1A 74 FD` | `PIB`: pack 1 cells and temperatures |

Everything else on the brand pages that switches MOSFETs, writes settings, changes SOC or puts the BMS to sleep is listed
there under *Control commands (dangerous)*. A monitor has no reason to send any of it.

## What usually goes wrong

- **The phone app is still connected.** Most of these batteries accept one central at a time. While the vendor app is
  connected the logger cannot connect, and the reverse. This is the first thing to check when a battery shows up in a scan
  but will not connect.
- **An absent battery stalls everything.** Put a timeout on every step (connect, MTU, discovery, subscribe, each request)
  and retry with a growing delay, so one unreachable battery does not hold up the others. A 5 s to 5 min back-off worked on
  the bench: with a non-existent address the retries came at 5, 10, 20 and 40 s and nothing else was delayed.
- **Late callbacks.** A connect or discovery can complete after you have given up on it. Give every operation a sequence
  number and ignore results that carry an old one; if a connection completes late, disconnect it rather than leaving it
  open and unused.
- **A sleeping BMS ignores the first request.** JBD documents this; resend once before reporting an error.
- **Wrong layout, valid checksum.** JK firmware before 11.x uses different offsets in a frame whose checksum still passes.
  Check the version before decoding ([JK/BLE](JK/BLE.md)).
- **Offsets counted from the wrong place.** Several upstream field tables count from the start of the register data, not
  from the start of the frame. For Modbus replies that is a 3-byte difference ([DALY/BLE](DALY/BLE.md),
  [SEPLOS/BLE](SEPLOS/BLE.md)).
- **Unused slots read as real values.** A Daly temperature slot that is not fitted reads `0x0000`, which decodes to −40 °C;
  a JK sensor that is not fitted reads `-2000`. Use the sensor count or presence mask the frame provides.

## Privacy

- **JK device-info frames contain the Bluetooth passcode** (bytes `62..77`). Never publish that frame unredacted, and zero
  those bytes before showing raw frames in a log or a bus monitor.
- MAC addresses and serial numbers identify a battery and its owner. Replace them in captures, as
  [CONTRIBUTING.md](CONTRIBUTING.md) asks.

## Cost on an ESP32-S3

Measured with ESP-IDF 5.5 and NimBLE built as central + observer only (no peripheral role, no bonding, up to 4
connections), alongside Wi-Fi:

- about 180 KB of flash;
- about 25 KB of RAM at boot, and about 65 KB more once the NimBLE host is running;
- a 5 s scan with the Wi-Fi access point up found five nearby devices.

These figures are from one build and no connected battery; expect them to move with configuration.

## Sources

- The brand pages linked in the table above, and the sources each of them cites.
- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/jikong_bms.py` (connection sequence,
  `AT\r\n` handling, frame-type filter, sensor positions), `daly_bms.py` (variant detection, data-relative offsets, MOSFET
  read), `seplos_bms.py` (block table, per-pack polling, temperature positions), and the captured frames in `tests/bms/`.
- Bench measurements with the maintainer's Vega Datalogger firmware (ESP32-S3), 2026-11: scan, retry timing and memory
  figures only; no battery was connected.

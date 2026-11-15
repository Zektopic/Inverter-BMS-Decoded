# Voltronic Power / Axpert & InfiniSolar — RS232 / USB ASCII Protocols (PI30, PI41, PI18, PI17)

> Status: 📄 Vendor doc — transcribed from Voltronic Power RS232 communication protocol manuals ([PI30 HS/MS/MSX](PDFs/PI30_HS_MS_MSX_RS232_Protocol_20140822_after_current_upgrade.pdf), [PI30 Customer](PDFs/PI30_Communication-Protocol-20150924-Customer.pdf), [PI18 InfiniSolar V](PDFs/PI18_InfiniSolar-V-protocol-20170926.pdf)) and cross-checked against [jblance/mpp-solar](https://github.com/jblance/mpp-solar), [syssi/esphome-pipsolar](https://github.com/syssi/esphome-pipsolar), and [ned-kelly/docker-voltronic-homeassistant](https://github.com/ned-kelly/docker-voltronic-homeassistant).

Voltronic Power is the OEM behind a wide range of off-grid and hybrid solar inverters sold under brand names including **Axpert** (KS, MKS, MKS II/III/IV, VM II/III/IV, King, MAX), **MPP Solar** (PIP-HS, PIP-MS, PIP-MSX, PIP-GK, PIP-MK, LV5048, MPI), **InfiniSolar**, **EASUN**, **PowMr**, **Mecer**, **Kodak**, **Effekta**, **Voltacon**, and **SunGoldPower**.

Two ASCII protocol families are used across these models:
1. **[PI30](#pi30-protocol-axpert--pip-series) (and [PI41](#pi41-protocol-split-phase-lv5048) split-phase)**: space-separated ASCII responses starting with `(`, used by Axpert / PIP off-grid and hybrid models.
2. **[PI18 and PI17](#pi18-and-pi17-protocols-infinisolar--sec-framing) (`SEC` format)**: comma-separated ASCII responses with a `^Pnnn` / `^Snnn` / `^Dnnn` length-prefixed header, used by InfiniSolar and high-capacity hybrid models.

---

## PI30 protocol (Axpert / PIP series)

### Summary

| | |
|---|---|
| Products | Voltronic Axpert KS / MKS / VM / King / MAX; MPP Solar PIP-HS / MS / MSX / MK / GK (1–11 kVA, 12 V / 24 V / 48 V) |
| Physical layer | RS232 on RJ45 jack; USB-B or Micro-USB (internal USB-HID `0665:5161` or USB-serial bridge connected to the same 2400 bps UART) |
| Settings | **2400 baud, 8N1** (8 data bits, no parity, 1 stop bit; some newer Bluetooth/Wi-Fi dongles or firmwares also support 9600 8N1) |
| Addressing | Point-to-point on RS232/USB; parallel units (`0`–`9`) are queried via `QPGSn` through any unit in the parallel group |
| Roles | Host (PC / ESP32 / Raspberry Pi) is master and polls; typical polling interval 2–5 s (at 2400 bps, a 110-byte `QPIGS` reply takes ~460 ms on the wire) |
| Byte order / format | Printable ASCII fields separated by spaces (`0x20`); 2-byte big-endian **CRC16-XMODEM** (with reserved-byte escaping) before `\r` (`0x0D`) |

#### RJ45 RS232 pinout

The inverter's RJ45 communication port carries true RS232 voltage levels (±12 V), **not** 3.3 V / 5 V TTL. Use a MAX3232 transceiver between the inverter and a microcontroller.

| RJ45 Pin | T-568B wire color | Signal (inverter side) | DB9 female (PC DTE) | Notes |
|---|---|---|---|---|
| 1 | White / Orange | **TXD** (inverter → host) | Pin 2 (RXD) | Output from inverter |
| 2 | Orange | **RXD** (host → inverter) | Pin 3 (TXD) | Input to inverter |
| 4 *(or 7)* | Blue *(or White/Brown)* | **+12 V VCC** | — | Power supply for dongle / isolated transceiver (model-dependent) |
| 8 | Brown | **GND** | Pin 5 (GND) | Signal ground |

---

## Framing

Every PI30 frame ends with a 2-byte binary CRC (`CRC_H`, `CRC_L`) followed by a carriage return (`\r`, `0x0D`):

- **Host → Inverter (request)**: `<COMMAND><CRC_H><CRC_L>\r` (no start character).
- **Inverter → Host (response)**: `(<PAYLOAD><CRC_H><CRC_L>\r` (starts with ASCII `(`, `0x28`, followed by space-separated ASCII fields).
- **Setter acknowledgment**: `(ACK\x39\x20\r` if accepted, or `(NAK\x73\x73\r` if rejected or unrecognised.

```
Request:   51 50 49 47 53   B7 A9   0D
           'Q' 'P' 'I' 'G' 'S'  └───┴── CRC16 (BE)
                                        └── CR ('\r')

Response:  28  32 33 30 2E 30 20 ...   24 8C   0D
           '(' "230.0 ..."             └───┴── CRC16 (BE)
                                               └── CR ('\r')
```

### Checksum algorithm (CRC16-XMODEM with byte escaping)

1. Compute standard **CRC-16/XMODEM** (`poly = 0x1021`, `init = 0x0000`, `refin = false`, `refout = false`, `xorout = 0x0000`) over all bytes before the CRC (for requests: the command characters; for responses: from the leading `(` up to the last payload character).
2. Split into `crc_high = (crc >> 8) & 0xFF` and `crc_low = crc & 0xFF`.
3. **Reserved-byte escaping**: to prevent CRC bytes from colliding with frame delimiters, if `crc_high` or `crc_low` equals **`0x28`** (`'('`), **`0x0D`** (`'\r'`), or **`0x0A`** (`'\n'`), **add `1` to that byte**:

```python
def voltronic_crc(data: bytes) -> tuple[int, int]:
    crc = 0
    for b in data:
        crc ^= (b << 8)
        for _ in range(8):
            crc = ((crc << 1) ^ 0x1021) & 0xFFFF if (crc & 0x8000) else (crc << 1) & 0xFFFF
    crc_high, crc_low = (crc >> 8) & 0xFF, crc & 0xFF
    if crc_high in (0x28, 0x0D, 0x0A):
        crc_high += 1
    if crc_low in (0x28, 0x0D, 0x0A):
        crc_low += 1
    return crc_high, crc_low
```

#### Worked examples (verified in Python)

- **Normal CRC (`QPIGS`)**:
  - ASCII bytes: `51 50 49 47 53` (`"QPIGS"`).
  - Raw CRC-16/XMODEM = `0xB7A9` (`crc_high = 0xB7`, `crc_low = 0xA9`). Neither byte is in `{0x28, 0x0D, 0x0A}`.
  - Full request frame: `51 50 49 47 53 B7 A9 0D`.
- **Escaped CRC (`QGMN`, `QBOOT`, `POP02`, `MCHGC090`)**:
  - `QGMN` (`51 47 4D 4E`): raw XMODEM = `0x4928` → `crc_low` is `0x28` (`'('`), incremented to `0x29` → frame `51 47 4D 4E 49 29 0D`.
  - `QBOOT` (`51 42 4F 4F 54`): raw XMODEM = `0x0A88` → `crc_high` is `0x0A` (`'\n'`), incremented to `0x0B` → frame `51 42 4F 4F 54 0B 88 0D`.
  - `POP02` (`50 4F 50 30 32`): raw XMODEM = `0xE20A` → `crc_low` is `0x0A` (`'\n'`), incremented to `0x0B` → frame `50 4F 50 30 32 E2 0B 0D`.
  - `MCHGC090`: raw XMODEM = `0x0D8F` → `crc_high` is `0x0D` (`'\r'`), incremented to `0x0E` → CRC bytes `0E 8F`.

---

## Requests and responses (PI30)

All request frames below have their CRC precomputed and verified:

| Command | Request hex (with CRC + `\r`) | Direction | Payload / description |
|---|---|---|---|
| `QPI` | `51 50 49 BE AC 0D` | Host → Inv | Protocol ID inquiry → `(PI30\x9A\x0B\r` |
| `QID` | `51 49 44 D6 EA 0D` | Host → Inv | 14-character device serial number → `(XXXXXXXXXXXXXX<CRC>\r` |
| `QSID` | `51 53 49 44 BB 05 0D` | Host → Inv | Extended serial number (2-digit length `NN` + 20 chars) |
| `QVFW` | `51 56 46 57 62 99 0D` | Host → Inv | Main CPU firmware version → `(VERFW:NNNNN.NN<CRC>\r` |
| `QVFW2` | `51 56 46 57 32 C3 F5 0D` | Host → Inv | SCC1 / secondary CPU firmware version → `(VERFW2:NNNNN.NN<CRC>\r` |
| `QMN` | `51 4D 4E BB 64 0D` | Host → Inv | Model name inquiry → e.g. `(MKS2-8000\xB2\x8D\r` |
| `QGMN` | `51 47 4D 4E 49 29 0D` | Host → Inv | General model number → e.g. `(044\xC8\xAE\r` |
| `QPIRI` | `51 50 49 52 49 F8 54 0D` | Host → Inv | Device rated & current configuration settings ([table below](#qpiri--device-rating-and-current-settings-inquiry)) |
| `QPIGS` | `51 50 49 47 53 B7 A9 0D` | Host → Inv | General status / live telemetry ([table below](#qpigs--device-general-status-parameters-inquiry)) |
| `QPIGS2` | `51 50 49 47 53 32 68 2D 0D` | Host → Inv | Secondary MPPT (SCC2 / SCC3) live telemetry ([table below](#qpigs2--secondary-pv--mppt-status-inquiry)) |
| `QMOD` | `51 4D 4F 44 49 C1 0D` | Host → Inv | Operating mode single-char code ([table below](#qmod--device-mode-inquiry)) |
| `QPIWS` | `51 50 49 57 53 B4 DA 0D` | Host → Inv | 32–38 bit ASCII warning/fault bitmap ([table below](#alarms-and-status-bits-qpiws-and-fault-codes)) |
| `QFLAG` | `51 46 4C 41 47 98 74 0D` | Host → Inv | Enabled/disabled feature flags `(ExxxDxxx<CRC>\r` ([table below](#qflag--device-flag-status-inquiry)) |
| `QDI` | `51 44 49 71 1B 0D` | Host → Inv | Factory default setting values ([table below](#qdi--default-setting-value-information)) |
| `QMCHGCR` | `51 4D 43 48 47 43 52 D8 55 0D` | Host → Inv | Selectable max total charging current values (A, space-separated) |
| `QMUCHGCR` | `51 4D 55 43 48 47 43 52 26 34 0D` | Host → Inv | Selectable max utility (AC) charging current values (A) |
| `QOPM` | `51 4F 50 4D A5 C5 0D` | Host → Inv | Output mode (`00` single, `01` parallel, `02`–`04` 3-phase P1–P3) |
| `QPGS0` | `51 50 47 53 30 3F DA 0D` | Host → Inv | Parallel unit `0` status (`QPGSn` for `n = 0..9`, [table below](#qpgsn--parallel-information-inquiry)) |
| `QBEQI` | `51 42 45 51 49 2B F9 0D` | Host → Inv | Battery equalization parameters and status |
| `Q1` | `51 31 1B FC 0D` | Host → Inv | Extended diagnostic/temperature query (undocumented in base PDF; see [`Q1`](#q1-and-qbms--pbms-extended-queries)) |
| `QBMS` | `51 42 4D 53 61 D9 0D` | Host → Inv | Lithium BMS status parameters ([table below](#q1-and-qbms--pbms-extended-queries)) |

---

### `QPIGS` — Device General Status Parameters Inquiry

Request: `QPIGS\xB7\xA9\r`.
Response: `(` followed by space-separated ASCII tokens. Base HS/MS/MSX firmware returns **17 fields** (indices 0–16); MKS / Plus Duo / MAX firmwares append fields 17–20 (21 fields) or 17–23 (24 fields).

| Index | Format | Type | Unit / scale | Field |
|---|---|---|---|---|
| 0 | `BBB.B` | float | 0.1 V | Grid (AC input) voltage |
| 1 | `CC.C` | float | 0.1 Hz | Grid (AC input) frequency |
| 2 | `DDD.D` | float | 0.1 V | AC output voltage |
| 3 | `EE.E` | float | 0.1 Hz | AC output frequency |
| 4 | `FFFF` | u16 | 1 VA | AC output apparent power |
| 5 | `GGGG` | u16 | 1 W | AC output active power |
| 6 | `HHH` | u16 | 1 % | Output load percent (max of W% or VA%) |
| 7 | `III` | u16 | 1 V | DC bus voltage |
| 8 | `JJ.JJ` | float | 0.01 V | Battery voltage |
| 9 | `KKK` | u16 | 1 A | Battery charging current |
| 10 | `OOO` | u16 | 1 % | Battery capacity (estimated from voltage, or from BMS if connected) |
| 11 | `TTTT` | s16 | 1 °C | Inverter heat sink temperature (raw NTC A/D value on some 1–3 kVA models) |
| 12 | `EEEE` or `EE.E` | float | 1 A or 0.1 A | PV1 input current (to battery on HS/MS/MSX; PV1 string current in `0.1 A` on MAX — see [Errata](#errata-and-disagreements)) |
| 13 | `UUU.U` | float | 0.1 V | PV1 input voltage |
| 14 | `WW.WW` | float | 0.01 V | Battery voltage measured by SCC1 (solar charge controller) |
| 15 | `PPPPP` | u32 | 1 A | Battery discharge current |
| 16 | `b7b6b5b4b3b2b1b0` | bitfield (8 ASCII `'0'`/`'1'`) | — | **Device status 1** (left-to-right `b7`..`b0`; see bit table below) |
| 17 | `QQ` | u8 | 10 mV | *(Extended)* Battery voltage offset for fans on (`RSV1`) |
| 18 | `VV` | u8 | — | *(Extended)* EEPROM version (`RSV2`) |
| 19 | `MMMMM` | u32 | 1 W | *(Extended)* PV1 charging power |
| 20 | `b10b9b8` | bitfield (3 ASCII `'0'`/`'1'`) | — | *(Extended)* **Device status 2**: `b10` charging to float mode, `b9` switch on, `b8` dustproof installed (Axpert V/MAX) or reserved |
| 21 | `Y` | u8 | enum | *(Axpert MAX only)* Solar feed-to-grid status (`0` normal, `1` feed to grid) |
| 22 | `ZZ` | u8 | enum | *(Axpert MAX only)* Country regulation (`00` India, `01` Germany, `02` South America) |
| 23 | `AAAA` | u16 | 1 W | *(Axpert MAX only)* Solar feed-to-grid power |

#### `QPIGS` Device status 1 bits (field 16, `b7b6b5b4b3b2b1b0`)

Characters are transmitted MSB-first (`b7` is the first character, `b0` is the 8th character):

| Bit | Char index | Meaning |
|---|---|---|
| `b7` | 0 | `1` = SBU priority version added, `0` = no |
| `b6` | 1 | `1` = configuration status changed, `0` = unchanged |
| `b5` | 2 | `1` = SCC firmware version updated, `0` = unchanged |
| `b4` | 3 | `1` = Load on, `0` = Load off |
| `b3` | 4 | `1` = battery voltage to steady while charging |
| `b2` | 5 | `1` = charging on (any source), `0` = charging off |
| `b1` | 6 | `1` = SCC (solar) charging on, `0` = off |
| `b0` | 7 | `1` = AC (utility) charging on, `0` = off |

*(Valid `b2b1b0` combinations: `000` not charging, `110` SCC charging only, `101` AC charging only, `111` SCC + AC charging).*

---

### `QPIGS2` — Secondary PV / MPPT Status Inquiry

Request: `QPIGS2\x68\x2D\r`. Used on multi-MPPT models. Note that two layout variants exist in vendor documents:

- **Axpert MAX (dual-MPPT 48 V models)**: 3 fields — `(BB.B CCC.C DDDDD<CRC>\r`:
  1. `BB.B` — PV2 input current (`0.1 A`)
  2. `CCC.C` — PV2 input voltage (`0.1 V`)
  3. `DDDDD` — PV2 charging power (`1 W`)
- **MKS Plus Duo (2/3-MPPT models)**: 12 fields — `(N1..N4 N6..N10 N12..N16 N18..N22 b24..b31 N33..N36 N38..N41 N42..N45 N47..N51 N53..N57 N58..N61 N63..N67<CRC>\r`:
  PV2 current (`A`), PV2 voltage (`0.1 V`), SCC2 battery voltage (`0.01 V`), PV2 power (`W`), 8-bit SCC2/3 status (`b24` SCC2 charging, `b25` SCC3 charging), AC charging current (`A`), AC charging power (`W`), PV3 current (`A`), PV3 voltage (`0.1 V`), SCC3 battery voltage (`0.01 V`), PV3 power (`W`), PV total charging power (`W`).

---

### `QPIRI` — Device Rating and Current Settings Inquiry

Request: `QPIRI\xF8\x54\r`.
Response: `(` followed by 25 space-separated fields (up to 28 on Axpert MAX / newer firmwares):

| Index | Format | Type | Unit / scale | Field |
|---|---|---|---|---|
| 0 | `BBB.B` | float | 0.1 V | Grid rating voltage |
| 1 | `CC.C` | float | 0.1 A | Grid rating current |
| 2 | `DDD.D` | float | 0.1 V | AC output rating voltage |
| 3 | `EE.E` | float | 0.1 Hz | AC output rating frequency |
| 4 | `FF.F` | float | 0.1 A | AC output rating current |
| 5 | `HHHH` | u16 | 1 VA | AC output rating apparent power |
| 6 | `IIII` | u16 | 1 W | AC output rating active power |
| 7 | `JJ.J` | float | 0.1 V | Battery rating voltage (e.g. `12.0`, `24.0`, `48.0`) |
| 8 | `KK.K` | float | 0.1 V | Battery re-charge voltage (back-to-grid voltage in SBU/Solar first mode) |
| 9 | `JJ.J` | float | 0.1 V | Battery under-voltage (low-DC cut-off voltage) |
| 10 | `KK.K` | float | 0.1 V | Battery bulk (C.V.) charging voltage |
| 11 | `LL.L` | float | 0.1 V | Battery float charging voltage |
| 12 | `O` | u8 | enum | **Battery type**: `0` AGM, `1` Flooded, `2` User, `3` [Pylontech](../PYLONTECH/README.md), `4` Shinheung, `5` WECO, `6` Soltaro, `7` BAK/TBD, `8` LIB, `9` LIC (3rd-party lithium) |
| 13 | `PP` (or `PPP`) | u16 | 1 A | Current max AC (utility) charging current |
| 14 | `QQ0` | u16 | 1 A | Current max total charging current (solar + utility) |
| 15 | `O` | u8 | enum | AC input voltage range: `0` Appliance (90–280 V), `1` UPS (170–280 V) |
| 16 | `P` | u8 | enum | **Output source priority**: `0` Utility first (`USB`), `1` Solar first (`SUB`), `2` SBU first (`SBU`) |
| 17 | `Q` | u8 | enum | **Charger source priority**: `0` Utility first, `1` Solar first, `2` Solar + Utility, `3` Only solar |
| 18 | `R` | u8 | count | Max parallel units supported (`0`–`9`, or `-` on non-parallel models) |
| 19 | `SS` | string | enum | **Machine type**: `00` Grid-tie, `01` Off-grid, `10` Hybrid (`11` Off-grid 2 trackers, `20` Off-grid 3 trackers) |
| 20 | `T` | u8 | enum | Topology: `0` transformerless, `1` transformer |
| 21 | `U` | u8 | enum | **Output mode**: `0` single machine, `1` parallel, `2` Phase 1 of 3P, `3` Phase 2 of 3P, `4` Phase 3 of 3P (`5` Phase 1 of 2P, `6` Phase 2 of 2P 120°, `7` Phase 2 of 2P 180°) |
| 22 | `VV.V` | float | 0.1 V | Battery re-discharge voltage (`00.0` means wait until float/full) |
| 23 | `W` | u8 | enum | PV OK condition for parallel: `0` one unit with PV is enough, `1` all units must have PV |
| 24 | `X` | u8 | enum | PV power balance: `0` max PV input current = max charge current, `1` max PV input power = max charge power + load power |
| 25 | `YYY` | u16 | 1 min | *(Extended)* Max charging time at C.V. stage (`000` = auto) |
| 26 | `Z` | u8 | enum | *(Extended / MAX)* Operation logic: `0` Automatic, `1` Online mode, `2` ECO mode |
| 27 | `CCC` | u16 | 1 A | *(Axpert MAX)* Max battery discharging current (`000` = disabled, `030`–`150` A) |

---

### `QMOD` — Device Mode Inquiry

Request: `QMOD\x49\xC1\r`. Response: `(M<CRC>\r` where `M` is a single ASCII character:

| Code | Hex | Mode | Description |
|---|---|---|---|
| `P` | `0x50` | Power On | Initial power-on self-test mode |
| `S` | `0x53` | Standby | Standby mode (AC output off) |
| `L` | `0x4C` | Line | Grid / utility bypass mode |
| `B` | `0x42` | Battery | Inverter operating from battery / PV |
| `F` | `0x46` | Fault | Fault mode |
| `H` | `0x48` | Power Saving | Power-saving / hybrid eco mode |
| `D` | `0x44` | Shutdown | Shutdown mode (Axpert MAX) |
| `Y` | `0x59` | Bypass | Bypass mode (seen in some firmware variants) |

---

### `QFLAG` — Device Flag Status Inquiry

Request: `QFLAG\x98\x74\r`.
Response: `(ExxxDxxx<CRC>\r`, where letters after `E` are **enabled** flags and letters after `D` are **disabled** flags (case-insensitive across docs, usually lowercase in responses e.g. `(EakxyDbjuvz\x2F\x29\r`). The same single-letter codes are used with `PE<x>` (enable) and `PD<x>` (disable):

| Flag char | Setting |
|---|---|
| `a` | Buzzer alarm (enable = open buzzer, disable = silence buzzer) |
| `b` | Overload bypass function (transfer to line on overload in battery mode) |
| `d` | Solar feed to grid *(Axpert MAX reserved feature)* |
| `j` | Power-saving mode |
| `k` | LCD display escape to default page after 1-minute timeout |
| `l` | Data log pop-up function *(MKS Plus Duo)* |
| `u` | Overload auto-restart |
| `v` | Over-temperature auto-restart |
| `x` | LCD backlight on |
| `y` | Beep alarm when primary source is interrupted |
| `z` | Fault code record |

---

### `QDI` — Default Setting Value Information

Request: `QDI\x71\x1B\r`.
Response: `(` followed by 25 space-separated fields (up to 29 on MKS Plus Duo / MAX):

| Index | Format | Unit / scale | Field | Typical 48 V default |
|---|---|---|---|---|
| 0 | `BBB.B` | 0.1 V | Default AC output voltage | `230.0` (`120.0` LV) |
| 1 | `CC.C` | 0.1 Hz | Default AC output frequency | `50.0` (`60.0` LV) |
| 2 | `00DD` | 1 A | Default max AC charging current *(note: swapped with index 7 on MKS Plus Duo)* | `0030` |
| 3 | `EE.E` | 0.1 V | Default battery under-voltage cut-off | `42.0` (`44.0` MAX) |
| 4 | `FF.F` | 0.1 V | Default charging float voltage | `54.0` |
| 5 | `GG.G` | 0.1 V | Default charging bulk (C.V.) voltage | `56.4` |
| 6 | `HH.H` | 0.1 V | Default battery re-charge voltage | `46.0` |
| 7 | `II` | 1 A | Default max total charging current | `60` |
| 8 | `J` | enum | Default AC input voltage range (`0` Appliance) | `0` |
| 9 | `K` | enum | Default output source priority (`0` Utility first) | `0` |
| 10 | `L` | enum | Default charger source priority (`0` Utility first, `2` Solar+Utility) | `0` or `2` |
| 11 | `M` | enum | Default battery type (`0` AGM) | `0` |
| 12 | `N` | bool | Default silence buzzer (`0` buzzer enabled) | `0` |
| 13 | `O` | bool | Default power saving (`0` disabled) | `0` |
| 14 | `P` | bool | Default overload restart (`0` disabled) | `0` |
| 15 | `Q` | bool | Default over-temperature restart (`0` disabled) | `0` |
| 16 | `R` | bool | Default LCD backlight (`1` enabled) | `1` |
| 17 | `S` | bool | Default primary source interrupt alarm (`1` enabled) | `1` |
| 18 | `T` | bool | Default fault code record (`0` disabled) | `0` |
| 19 | `U` | bool | Default overload bypass (`0` disabled) | `0` |
| 20 | `V` | bool | Default LCD escape to default page after 1 min (`1` enabled) | `1` |
| 21 | `W` | enum | Default output mode (`0` single output) | `0` |
| 22 | `YY.Y` | 0.1 V | Default battery re-discharge voltage | `54.0` |
| 23 | `X` | bool | Default PV OK condition for parallel | `0` |
| 24 | `Z` | bool | Default PV power balance | `1` or `0` |
| 25+ | `aaa` … | — | *(Optional)* Max CV time (`min`), max discharge current (`A`, MAX) or charging stage / max solar current (Plus Duo) | `000` |

---

### `QPGSn` — Parallel Information Inquiry

Request: `QPGSn<CRC>\r` where `n` is the parallel machine index `'0'`–`'9'` (e.g. `QPGS0\x3F\xDA\r`, `QPGS1\x2F\xFB\r`).
Response: `(` followed by 27 space-separated fields (29 on Axpert MAX dual-MPPT):

| Index | Format | Type | Unit / scale | Field |
|---|---|---|---|---|
| 0 | `A` | u8 | bool | Parallel unit exists (`0` no, `1` yes) |
| 1 | `BBBBBBBBBBBBBB` | string | 14 ASCII | Unit serial number |
| 2 | `C` | char | enum | Work mode (same letter codes as `QMOD`) |
| 3 | `DD` | u8 | code | Fault code (`00` = no fault; see [Fault codes](#fault-codes-qpgsn-and-lcd-display)) |
| 4 | `EEE.E` | float | 0.1 V | Grid voltage |
| 5 | `FF.FF` | float | 0.01 Hz | Grid frequency |
| 6 | `GGG.G` | float | 0.1 V | AC output voltage |
| 7 | `HH.HH` | float | 0.01 Hz | AC output frequency |
| 8 | `IIII` | u16 | 1 VA | Unit AC output apparent power |
| 9 | `JJJJ` | u16 | 1 W | Unit AC output active power |
| 10 | `KKK` | u16 | 1 % | Unit load percentage |
| 11 | `LL.L` | float | 0.1 V | Battery voltage |
| 12 | `MMM` | u16 | 1 A | Unit battery charging current |
| 13 | `NNN` | u16 | 1 % | Battery capacity |
| 14 | `OOO.O` | float | 0.1 V | PV1 input voltage |
| 15 | `PPP` | u16 | 1 A | **Total** battery charging current (all parallel units) |
| 16 | `QQQQQ` | u32 | 1 VA | **Total** AC output apparent power (all parallel units) |
| 17 | `RRRRR` | u32 | 1 W | **Total** AC output active power (all parallel units) |
| 18 | `SSS` | u16 | 1 % | **Total** AC output load percentage |
| 19 | `b7b6b5b4b3b2b1b0` | bitfield | — | **Parallel inverter status**: `b7` SCC OK (`1`), `b6` AC charging (`1`), `b5` SCC charging (`1`), `b4b3` battery status (`00` normal, `01` under, `10` open), `b2` line loss (`1` loss, `0` OK), `b1` load on (`1`), `b0` config changed (`1`) |
| 20 | `T` | u8 | enum | Output mode (`0` single, `1` parallel, `2`–`4` 3-phase P1–P3, `5`–`7` split-phase) |
| 21 | `U` | u8 | enum | Charger source priority (`0` Utility first, `1` Solar first, `2` Solar+Utility, `3` Solar only) |
| 22 | `VVV` | u16 | 1 A | Max charger current setting |
| 23 | `WWW` | u16 | 1 A | Max charger current selectable range |
| 24 | `ZZ` (or `ZZZ`) | u16 | 1 A | Max AC charger current setting |
| 25 | `XX` | u16 | 1 A | PV1 input current |
| 26 | `YYY` | u16 | 1 A | Battery discharge current |
| 27 | `OOO.O` | float | 0.1 V | *(Axpert MAX only)* PV2 input voltage |
| 28 | `XX` | u16 | 1 A | *(Axpert MAX only)* PV2 input current |

---

### `Q1` and `QBMS` / `PBMS` (Extended queries)

#### `Q1` — Extended temperature & charger stage telemetry

Undocumented in the base HS/MS/MSX manual, decoded by the AEVA community from firmware disassembly and supported in `jblance/mpp-solar`:
- Request: `Q1\x1B\xFC\r`
- Response (17 space-separated fields after `(`):
  `0` absorb time remaining (`s`), `1` float time remaining (`s`), `2` SCC communicating (`00`/`01`), `3` AllowSccOnFlag (`00`/`01`), `4` charge average current (`A`), `5` SCC PWM temperature (`°C`), `6` inverter heatsink temperature (`°C`), `7` battery DC-DC heatsink temperature (`°C`), `8` transformer temperature (`°C`), `9` parallel role (`00` new, `01` slave, `02` master), `10` fan lock (`00` normal, `01` locked), `11` unused (`000`), `12` fan PWM speed (`%`, `0000`–`0100`), `13` SCC charge power (`W`), `14` parallel warning bitmap, `15` sync frequency (`Hz`), `16` charger stage (`10` no charging, `11` bulk, `12` absorb, `13` float).

#### `QBMS` / `PBMS` — Lithium BMS integration commands (Axpert MAX / King / MKS III+)

When an external controller emulates a BMS over serial or queries the inverter's BMS view:
- **`PBMSa bbb c d e fff ggg hhh iiii jjjj<CRC>\r`** (Remote box → Inverter) or **`QBMS\x61\xD9\r`** response in `mpp-solar`:

| Index | Format | Unit / scale | Field |
|---|---|---|---|
| 0 | `a` | bool | Battery connect status (`0` connected, `1` disconnected) |
| 1 | `bbb` | 1 % | Battery SOC percentage (`000`–`100`) |
| 2 | `c` | bool | Force AC charge flag (`0` normal, `1` force charge) |
| 3 | `d` | bool | Battery stop discharge flag (`0` enable discharge, `1` disable discharge) |
| 4 | `e` | bool | Battery stop charge flag (`0` enable charge, `1` disable charge) |
| 5 | `fff` | 0.1 V | BMS bulk (C.V.) charging voltage (e.g. `532` = 53.2 V) |
| 6 | `ggg` | 0.1 V | BMS float charging voltage (e.g. `532` = 53.2 V) |
| 7 | `hhh` | 0.1 V | BMS cut-off voltage (e.g. `450` = 45.0 V) |
| 8 | `iiii` | 1 A | BMS max charging current |
| 9 | `jjjj` | 1 A | BMS max discharging current |

---

## Alarms and status bits (`QPIWS` and fault codes)

### `QPIWS` — Device Warning Status Inquiry

Request: `QPIWS\xB4\xDA\r`.
Response: `(a0a1...a31<CRC>\r` (32 ASCII `'0'`/`'1'` characters on HS/MS/MSX and LV5048; 36 characters `a0..a35` on Axpert MAX; 38 characters `b1..b38` on 3-MPPT MKS Plus Duo). `a0` is the first character after `(`.

> **Note:** Bits marked *(with `a1`)* are **warnings** when `a1 = 0` and **faults** when `a1 = 1`.

| Bit index (0-based) | 1-based (`b1..b38`) | Severity | Meaning (HS / MS / MSX / LV5048) | Variant notes (MAX / MKS Plus Duo) |
|---|---|---|---|---|
| `a0` | `b1` | Warning | Reserved | `PV loss` warning on Axpert MAX |
| `a1` | `b2` | Fault | **Inverter fault** (master fault indicator) | |
| `a2` | `b3` | Fault | Bus over-voltage | |
| `a3` | `b4` | Fault | Bus under-voltage | |
| `a4` | `b5` | Fault | Bus soft-start fail | |
| `a5` | `b6` | Warning | `LINE_FAIL` (grid/utility loss) | |
| `a6` | `b7` | Warning / Fault | `OPVShort` (AC output short-circuit) | |
| `a7` | `b8` | Fault | Inverter voltage too low | |
| `a8` | `b9` | Fault | Inverter voltage too high | |
| `a9` | `b10` | *(with `a1`)* | Over-temperature | |
| `a10` | `b11` | *(with `a1`)* | Fan locked | |
| `a11` | `b12` | *(with `a1`)* | Battery voltage high | |
| `a12` | `b13` | Warning | Battery low alarm | |
| `a13` | `b14` | Fault / Res | Reserved (or `Overcharge` on LV5048 / MKS) | |
| `a14` | `b15` | Warning | Battery under-voltage shutdown | |
| `a15` | `b16` | Warning | Reserved (or `Battery derating` on LV5048 / MAX) | |
| `a16` | `b17` | *(with `a1`)* | Overload | |
| `a17` | `b18` | Warning | EEPROM fault | |
| `a18` | `b19` | Fault | Inverter over-current | |
| `a19` | `b20` | Fault | Inverter soft-start fail | |
| `a20` | `b21` | Fault | Self-test fail | |
| `a21` | `b22` | Fault | Output DC voltage over | |
| `a22` | `b23` | Fault | Battery connection open (`Bat Open`) | |
| `a23` | `b24` | Fault | Current sensor fail | |
| `a24` | `b25` | Fault | Battery short | Reserved on MAX |
| `a25` | `b26` | Warning | Power limit warning | Reserved on MAX |
| `a26` | `b27` | Warning | PV voltage high (PV1 on multi-MPPT) | Reserved on MAX |
| `a27` | `b28` | Warning | MPPT overload fault (MPPT1) | Reserved on MAX |
| `a28` | `b29` | Warning | MPPT overload warning (MPPT1) | Reserved on MAX |
| `a29` | `b30` | Warning | Battery too low to charge (MPPT1) | Reserved on MAX |
| `a30` | `b31` | Warning | Reserved (`Slave` warning on LV5048) | MKS 3-MPPT: `b31..b34` = PV2 high / MPPT2 overload fault / MPPT2 warn / Bat low2 |
| `a31` | `b32` | Warning | Reserved | MAX: `a31` Battery weak, `a35` Battery equalization. MKS 3-MPPT: `b35..b38` = PV3 warnings |

### Fault codes (`QPGSn` and LCD display)

| Code | Meaning | Code | Meaning |
|---|---|---|---|
| `01` | Fan is locked | `55` | Over DC voltage on AC output |
| `02` | Over-temperature | `56` | Battery connection is open |
| `03` | Battery voltage is too high | `57` | Current sensor failed |
| `04` | Battery voltage is too low | `58` | Output voltage is too low |
| `05` | Output short-circuited or over-temperature | `60` | Inverter negative power (power feedback protection) |
| `06` | Output voltage is too high | `71` | Parallel firmware version inconsistent |
| `07` | Overload time out | `72` | Parallel current sharing / output circuit fault |
| `08` | Bus voltage is too high | `80` | CAN communication fault |
| `09` | Bus soft-start failed | `81` | Parallel host line lost |
| `10` | PV over-current *(MAX)* | `82` | Parallel synchronization signal lost |
| `11` | Main relay failed *(or PV over-voltage on MAX)* | `83` | Parallel battery voltage detected different |
| `12` | DC-DC over-current *(MAX)* | `84` | Parallel AC input voltage/frequency detected different |
| `13` | Battery discharge over-current *(MAX)* | `85` | Parallel AC input/output current unbalanced |
| `51` | Inverter over-current | `86` | Parallel AC output mode setting different |
| `52` | Bus soft-start failed / bus voltage too low | | |
| `53` | Inverter soft-start failed | | |
| `54` | Self-test failed | | |

---

## Control commands (dangerous)

> [!CAUTION]
> **Write / setter commands modify inverter output voltage/frequency, battery charging voltages, parallel phase assignment, or shut down charging/discharging.**
> - Changing battery voltages (`PCVV`, `PBFT`, `PSDV`) or battery type (`PBT`) to values incompatible with your pack can cause overcharge or deep-discharge damage.
> - Changing parallel output mode (`POPM`) on a live multi-inverter or 3-phase system can cause phase-to-phase short circuits.
> - Many Voltronic firmwares write setting changes directly to **EEPROM**, which has a limited write endurance (~100,000 cycles). Avoid changing `MCHGC` / `MUCHGC` / `POP` in fast automation loops.

All setter commands end with `<CRC_H><CRC_L>\r` and return `(ACK\x39\x20\r` on success or `(NAK\x73\x73\r` on failure:

| Command | Arguments / format | Effect |
|---|---|---|
| `PE<x>` / `PD<x>` | `x` = flag char (`a`,`b`,`j`,`k`,`u`,`v`,`x`,`y`,`z`) | Enable (`PE`) or disable (`PD`) feature flag (see [`QFLAG`](#qflag--device-flag-status-inquiry)) |
| `PF` | none | **Reset all control parameters to factory defaults** |
| `F<nn>` | `50` or `60` | Set AC output rating frequency (`50` Hz or `60` Hz) |
| `V<nnn>` | `220`/`230`/`240` (HV) or `110`/`120`/`127` (LV) | Set AC output rating voltage *(MKS / MAX)* |
| `POP<nn>` | `00` Utility first, `01` Solar first, `02` SBU | Set output source priority |
| `PCP<nn>` | `00` Utility first, `01` Solar first, `02` Solar+Utility, `03` Solar only | Set charger source priority |
| `PPCP<mnn>` | `m` = unit `0`–`9`, `nn` = `00`–`03` | Set parallel unit `m` charger source priority |
| `PGR<nn>` | `00` Appliance, `01` UPS | Set AC grid input voltage working range |
| `PBT<nn>` | `00` AGM, `01` Flooded, `02` User (`03`–`09` lithium on MAX) | Set battery type |
| `PSDV<nn.n>` | e.g. `40.0`–`48.0` (48 V unit) | Set battery under-voltage cut-off (`0.1 V`) |
| `PCVV<nn.n>` | e.g. `48.0`–`58.4` (48 V unit) | Set battery C.V. (bulk/absorption) charging voltage (`0.1 V`) |
| `PBFT<nn.n>` | e.g. `48.0`–`58.4` (48 V unit) | Set battery float charging voltage (`0.1 V`) |
| `PBCV<nn.n>` | e.g. `44.0`–`51.0` (48 V unit) | Set battery re-charge voltage (SBU → grid threshold) |
| `PBDV<nn.n>` | `00.0` (full) or `48.0`–`58.0` (48 V unit) | Set battery re-discharge voltage (grid → battery threshold) |
| `MCHGC<nnn>` or `MCHGC<mnn>` | `nnn` = current in A, or `m` (unit) + `nn` (A) | Set max total charging current (check `QMCHGCR` for valid steps) |
| `MNCHGC<mnnn>` | `m` = unit `0`–`9`, `nnn` = current in A | Set max total charging current (> 100 A capable; required on MAX/LV5048) |
| `MUCHGC<nnn>` or `MUCHGC<mnn>` | `m` = unit `0`–`9`, `nn` = current in A | Set max utility (AC) charging current (check `QMUCHGCR`) |
| `POPM<mn>` or `POPM<nn>` | `m` = unit (on HS/MS), `n`/`nn` = mode `00`–`07` | **Set single / parallel / 3-phase output mode** |
| `PPVOKC<n>` | `0` one unit, `1` all units | Set parallel PV OK condition |
| `PSPB<n>` | `0` max charge current, `1` charge + load | Set solar power balance |
| `PBATCD<abc>` | `a` disch full, `b` disch standby, `c` charge (`0`/`1` each) | *(Axpert MAX)* Direct battery charge/discharge override (`000` clears override) |
| `PBATMAXDISC<nnn>` | `000` (disable) or `030`–`150` (A) | *(Axpert MAX)* Set max battery discharge current |
| `BTA<±0n>` | `-01` to `+09` (steps of `0.05 V`) | *(Undocumented)* Calibrate inverter battery voltage ADC reading |
| `RTEY` / `RTDL` | none | *(Axpert MAX)* Erase all stored PV/load energy counters (`RTEY`) or data logs (`RTDL`) |

---

## PI41 protocol (Split-phase LV5048)

Used on North American split-phase (120 V / 240 V, L1 + L2) units such as the **MPP Solar LV5048** (`PI41_LV5048.5KW.protocol-20190222.for.customer.pdf`). Framing, baud rate (2400 8N1), and CRC16-XMODEM are identical to PI30, with three key differences:

1. **`QPIGS` reports L1 metrics and uses the `QPGSn` status bitmap**:
   - Fields 0–6 of `QPIGS` are **L1** AC input/output voltage, frequency, VA, W, and load %.
   - Field 16 (`b7b6b5b4b3b2b1b0`) uses the **parallel status bit definitions** instead of the standard PI30 `QPIGS` bits:
     `b7` SCC1 OK, `b6` L1 AC charging, `b5` L1 SCC charging, `b4b3` battery status (`00` normal, `01` under, `10` open), `b2` L1 line loss, `b1` load on, `b0` config status.
2. **`QPIGS2` reports L2 + PV2 metrics** (11 fields after `(`):
   `(AAA.A BB.B CCC.C DD.D EEEE FFFF GGG HHHH III.I JJ.JJ b7b6b5b4b3b2b1b0<CRC>\r`:
   - `0` L2 AC input voltage (`0.1 V`), `1` L2 AC input frequency (`0.1 Hz`), `2` L2 AC output voltage (`0.1 V`), `3` L2 AC output frequency (`0.1 Hz`), `4` L2 apparent power (`VA`), `5` L2 active power (`W`), `6` L2 load (`%`), `7` PV2 input current for battery (`1 A`), `8` PV2 input voltage (`0.1 V`), `9` L2 battery voltage (`0.01 V`), `10` L2 status bits (`b7` SCC2 OK, `b6` L2 AC charging, `b5` SCC2 charging, `b4b3` reserved, `b2` L2 line loss, `b1` L2 load on, `b0` reserved).
3. **`QP2GSn` queries parallel L2 status** (17 fields after `(`):
   `(A BBBBBBBBBBBBBB C DD EEE.E FF.FF GGG.G HH.HH IIII JJJJ KKK LL.L MMM NNN OOO.O PP b7b6b5b4b3b2b1b0<CRC>\r`.
4. **Output mode codes (`QPIRI` field 21 / `QOPM` / `POPM`)**:
   `1` parallel output (0°), `2` Phase 1 of 3P, `3` Phase 2 of 3P, `4` Phase 3 of 3P, `5` parallel output (120°), `6` parallel output (180°).

---

## PI18 and PI17 protocols (InfiniSolar / `SEC` framing)

Used on **Voltronic InfiniSolar** (V series, 5kW, 10kW/15kW 3P, WP LV6K/TWIN), **MPP Solar MPI / LV5048 Hybrid V2**, **EASUN SV IV**, and **SunGoldPower 6048** models.

### Framing (`^P` / `^S` / `^D`)

Instead of `(`, SEC-family protocols use a 5-character header `^Tnnn` followed by comma-separated fields:

```
^   T   n n n   <DATA>   [<CRC_H><CRC_L>]   \r
│   │   └───┴── 3-digit ASCII decimal length of everything after "^Tnnn" (including CRC and '\r')
│   └────────── Type: 'P' = PC query, 'S' = PC setter, 'D' = Device data reply
└────────────── Start byte '^' (0x5E)
```

- **Field separator**: comma `,` (`0x2C`). Reserved/empty fields appear as consecutive commas `,,`.
- **Setter acknowledgment**: `^1\x0B\xC2\r` (accepted / ACK) or `^0\x1B\xE3\r` (refused / NAK).
- **Crucial difference between PI18 and PI17**:
  - **PI18**: Both requests (`^Pnnn...<CRC>\r`, `^Snnn...<CRC>\r`) and responses (`^Dnnn...<CRC>\r`) include the 2-byte CRC16-XMODEM before `\r`, so `nnn = len(DATA) + 3`.
    - Example (`PI` query): `DATA = "PI"` (2 bytes) → `nnn = 2 + 3 = 005` → `^P005PI` + CRC `0x71 0x8B` + `\r` (`5E 50 30 30 35 50 49 71 8B 0D`).
    - Response: `^D00518\x3B\x03\r` (`DATA = "18"`, `nnn = 2 + 3 = 005`).
  - **PI17**: Host requests (`^Pnnn...\r`, `^Snnn...\r`) do **not** include a CRC before `\r`, so `nnn = len(DATA) + 1` (e.g. `^P003PI\r`, `^P003GS\r`, `^P005PIRI\r`), whereas device responses (`^Dnnn...<CRC>\r`) **do** include the 2-byte CRC16-XMODEM (`nnn = len(DATA) + 3`, e.g. `^D00517\xCA\xEC\r`).
    *(Exception in PI17: energy history queries `EY`, `EM`, `ED`, `EH` append a 3-digit decimal 8-bit sum `sum(ascii_bytes) & 0xFF` before `\r`, e.g. `^P010EY2023nnn\r`).*

### Core PI18 commands (`PI18_InfiniSolar-V-protocol-20170926.pdf`)

| Command | Request frame (hex / ASCII) | Response prefix | Description |
|---|---|---|---|
| `PI` | `^P005PI\x71\x8B\r` | `^D00518\x3B\x03\r` | Protocol ID (`18`) |
| `ID` | `^P005ID\x51\x3E\r` | `^D025LLXXXX...<CRC>\r` | Serial number (`LL` = 2-digit valid length + 20 chars) |
| `VFW` | `^P006VFW\x68\x0B\r` | `^D020aaaaa,bbbbb,ccccc<CRC>\r` | Main, Slave 1, Slave 2 CPU versions |
| `PIRI` | `^P007PIRI\xEE\x38\r` | `^D085...` (or `^D088...`) | Rated information (25 comma-separated fields; voltages/currents in integer `0.1 V` / `0.1 A`) |
| `GS` | `^P005GS\x58\x14\r` | `^D106...<CRC>\r` | General status (28 comma-separated fields; table below) |
| `MOD` | `^P006MOD\xDD\xBE\r` | `^D005XX<CRC>\r` | Working mode (`00` Power on, `01` Standby, `02` Bypass, `03` Battery, `04` Fault, `05` Hybrid/Line) |
| `FWS` | `^P005FWS\x5E\x9F\r` | `^D034AA,B,C...Q<CRC>\r` | Fault code `AA` + 16 comma-separated warning flags (`0`/`1`) |
| `FLAG` | `^P007FLAG\x8E\x18\r` | `^D020A,B,C,D,E,F,G,H,I<CRC>\r` | 9 comma-separated `0`/`1` enable flags |
| `ET` | `^P005ET\x6E\x91\r` | `^D011NNNNNNNN<CRC>\r` | Total generated energy (`Wh` or `kWh`) |
| `PGSn` | `^P007PGS0<CRC>\r` | `^D113...<CRC>\r` | Parallel unit `n` general status (29 comma-separated fields) |

#### PI18 `^P005GS` response fields (`^D106...`, 28 comma-separated fields)

Unlike PI30 (which includes decimal points), PI18 sends fixed-point values as **integers without decimal points** (e.g. `2232` = 223.2 V, `499` = 49.9 Hz):

| Index | Format | Unit / scale | Field | Index | Format | Unit / scale | Field |
|---|---|---|---|---|---|---|---|
| 0 | `AAAA` | 0.1 V | Grid voltage | 14 | `OOO` | 1 °C | MPPT1 charger temperature |
| 1 | `BBB` | 0.1 Hz | Grid frequency | 15 | `PPP` | 1 °C | MPPT2 charger temperature |
| 2 | `CCCC` | 0.1 V | AC output voltage | 16 | `QQQQ` | 1 W | PV1 input power |
| 3 | `DDD` | 0.1 Hz | AC output frequency | 17 | `RRRR` | 1 W | PV2 input power |
| 4 | `EEEE` | 1 VA | AC output apparent power | 18 | `SSSS` | 0.1 V | PV1 input voltage |
| 5 | `FFFF` | 1 W | AC output active power | 19 | `TTTT` | 0.1 V | PV2 input voltage |
| 6 | `GGG` | 1 % | Output load percent | 20 | `U` | bool | Configuration state (`0` unchanged, `1` changed) |
| 7 | `HHH` | 0.1 V | Battery voltage | 21 | `V` | enum | MPPT1 status (`0` abnormal, `1` normal/idle, `2` charging) |
| 8 | `III` | 0.1 V | Battery voltage from SCC1 | 22 | `W` | enum | MPPT2 status (`0` abnormal, `1` normal/idle, `2` charging) |
| 9 | `JJJ` | 0.1 V | Battery voltage from SCC2 | 23 | `X` | bool | Load connection (`0` disconnected, `1` connected) |
| 10 | `KKK` | 1 A | Battery discharge current | 24 | `Y` | enum | Battery power direction (`0` idle, `1` charge, `2` discharge) |
| 11 | `LLL` | 1 A | Battery charging current | 25 | `Z` | enum | DC/AC power direction (`0` idle, `1` AC→DC, `2` DC→AC) |
| 12 | `MMM` | 1 % | Battery capacity | 26 | `a` | enum | Line power direction (`0` idle, `1` input, `2` output) |
| 13 | `NNN` | 1 °C | Inverter heatsink temperature | 27 | `b` | id | Local parallel ID (`0` to `parallel_num - 1`) |

*(In **PI17** (`PI17_InfiniSolar-5KW-protocol-20160309.pdf`), `^P003GS\r` returns 25 comma-separated fields covering 3-phase R/S/T voltages and currents plus signed battery current `±GGGGG` in `0.1 A`, while active/apparent power and flow directions are queried separately via `^P003PS\r`).*

---

## Captured frames

Test frames below are taken from the [`jblance/mpp-solar`](https://github.com/jblance/mpp-solar) protocol test suite (`mppsolar/protocols/pi30.py`, `pi18.py`, `pi41.py`); all CRCs were independently verified in Python against `voltronic_crc()`:

```text
# PI30: Protocol ID inquiry
QPI\xBE\xAC\r  ->  (PI30\x9A\x0B\r

# PI30: Firmware version inquiry (main CPU v72.70)
QVFW\x62\x99\r  ->  (VERFW:00072.70\x53\xA7\r

# PI30: Device flag status (Enabled: a,k,x,y; Disabled: b,j,u,v,z)
QFLAG\x98\x74\r  ->  (EakxyDbjuvz\x2F\x29\r

# PI30: General status (QPIGS) — 0 V grid, 230.0 V / 49.9 Hz out, 161 VA / 119 W (3% load),
# 460 V bus, 57.50 V bat, 12 A charge, 100% SOC, 69 °C, 14 A / 103.8 V PV1 (856 W), status 00110110
QPIGS\xB7\xA9\r  ->  (000.0 00.0 230.0 49.9 0161 0119 003 460 57.50 012 100 0069 0014 103.8 57.45 00000 00110110 00 00 00856 010\x24\x8C\r

# PI30: Rated information (QPIRI) — 230 V / 21.7 A grid, 230 V / 50 Hz / 5000 VA / 4000 W out,
# 48 V nominal (46.0 V recharge, 42.0 V cut-off, 56.4 V bulk, 54.0 V float), AGM (0), 10 A AC / 10 A total
QPIRI\xF8\x54\r  ->  (230.0 21.7 230.0 50.0 21.7 5000 4000 48.0 46.0 42.0 56.4 54.0 0 10 010 1 0 0 6 01 0 0 54.0 0 1\x6F\x7E\r

# PI30: Warning status (QPIWS) — a5 (LINE_FAIL) and a16 (Overload) active
QPIWS\xB4\xDA\r  ->  (00000100000000001000000000000000\x56\xA6\r

# PI30: Parallel unit 0 status (QPGS0) — Battery mode ('B'), fault 00, 230.6 V / 50.00 Hz, 51.4 V bat
QPGS0\x3F\xDA\r  ->  (1 92931701100510 B 00 000.0 00.00 230.6 50.00 0275 0141 005 51.4 001 100 083.3 002 00574 00312 003 10100110 1 2 060 120 10 04 000\xCC\x23\r

# PI18: Protocol ID inquiry and response
^P005PI\x71\x8B\r  ->  ^D00518\x3B\x03\r

# PI17: Protocol ID inquiry (no request CRC) and response (with CRC)
^P003PI\r  ->  ^D00517\xCA\xEC\r
```

---

## Errata and disagreements

1. **Null-byte (`0x00`) escaping in `jblance/mpp-solar` vs. vendor `CRC.c`**:
   - Voltronic's official `cal_crc_half()` C reference implementation (and `ned-kelly/docker-voltronic-homeassistant`) only increments CRC bytes equal to `0x28` (`'('`), `0x0D` (`'\r'`), or `0x0A` (`'\n'`).
   - `jblance/mpp-solar` (`protocol_helpers.py:crcPI`) additionally increments `0x00` to `0x01` (likely a workaround for null-terminated C strings in early wrappers). Check whether your target firmware escapes `0x00` if a frame with a `0x00` CRC byte is rejected.
2. **`QPIGS` field 12 (`PV Input current`) scaling and meaning**:
   - In the 2014 HS/MS/MSX and 2015 Customer PDFs, field 12 is formatted as `EEEE` (4-digit integer, unit `1 A`) and described as *"PV Input current for battery"* (charging current at battery voltage, not PV panel current — see `ned-kelly/docker-voltronic-homeassistant/sources/inverter-cli/main.cpp` line 230).
   - In the 2021 Axpert MAX PDF (`PI30MAX`), field 12 changes format to `EE.E` (`0.1 A` resolution) and represents PV1 panel string current. Parsers should split on spaces and parse field 12 as a float rather than using fixed byte offsets.
3. **`QPIGS` vs. `QPGSn` / PI41 status bit ordering (field 16)**:
   - Standard PI30 `QPIGS` field 16 puts Load On at `b4` and Charging bits at `b2..b0`.
   - `QPGSn` (parallel query) and PI41 (LV5048 `QPIGS` and `QPGSn`) completely redefine `b7..b0` (`b7` SCC OK, `b6` AC charging, `b5` SCC charging, `b4b3` battery state, `b2` line loss, `b1` load on, `b0` config changed).
4. **`QDI` fields 2 and 7 swapped on MKS Plus Duo**:
   - In `PI30_Communication-Protocol-20150924-Customer.pdf` (pages 11–12), `QDI` field 2 (`N12..N15`) is *Max AC charging current* on KS/MKS series but *Max total charging current* on MKS Plus Duo, while field 7 (`N35N36`) is swapped accordingly.
5. **Typo in vendor PDFs for `PPCP` command**:
   - Both the 2014 HS/MS/MSX PDF (section 3.18) and 2015 Customer PDF (section 3.15) title the command `PPCP<MNN><cr>`, but write `Computer: PCP<MNN><CRC><cr>` in the example line. The actual parallel command implemented in firmware and `mpp-solar` is `PPCP<MNN>`.
6. **Invalid CRC in `jblance/mpp-solar` `QMOD` test vector**:
   - `mppsolar/protocols/pi30.py` lists `b"(S\x64\x39\r"` as the test response for `QMOD`. The correct CRC for `(S` is `0xE5 0xD9` (`b"(S\xE5\xD9\r"`).
7. **PI18 `PIRI` response length header discrepancy**:
   - `PI18_InfiniSolar-V-protocol-20170926.pdf` specifies `^D085...` for the `^P007PIRI` response, whereas real captures in `jblance/mpp-solar` (`pi18.py`) return `^D088...` with an extra trailing field. Always parse by splitting on `,` rather than hardcoding the `^Dnnn` length.

---

## Sources

- [PI30_HS_MS_MSX_RS232_Protocol_20140822_after_current_upgrade.pdf](PDFs/PI30_HS_MS_MSX_RS232_Protocol_20140822_after_current_upgrade.pdf) — Voltronic Power RS232 communication protocol for HS/MS/MSX series, 2014-08-22 ([mirror in `jblance/mpp-solar`](https://github.com/jblance/mpp-solar/blob/master/docs/protocols/PI30_HS_MS_MSX_RS232_Protocol_20140822_after_current_upgrade.pdf)).
- [PI30_Communication-Protocol-20150924-Customer.pdf](PDFs/PI30_Communication-Protocol-20150924-Customer.pdf) — Voltronic Power *Inverter Communication Protocol* (KS / MKS / Plus Duo, includes `QPIGS2` and `QP2GSn`), 2015-09-24 ([mirror in `jblance/mpp-solar`](https://github.com/jblance/mpp-solar/blob/master/docs/protocols/PI30_Communication-Protocol-20150924-Customer.pdf)).
- [PI18_InfiniSolar-V-protocol-20170926.pdf](PDFs/PI18_InfiniSolar-V-protocol-20170926.pdf) — Voltronic Power InfiniSolar V series (`^P`/`^S`/`^D` SEC format) protocol specification, 2017-09-26 ([mirror in `jblance/mpp-solar`](https://github.com/jblance/mpp-solar/blob/master/docs/protocols/PI18_InfiniSolar-V-protocol-20170926.pdf)).
- Additional protocol manuals in [`jblance/mpp-solar/docs/protocols`](https://github.com/jblance/mpp-solar/tree/master/docs/protocols) and [`ardupic/voltronic-inverter-communication-protocols`](https://github.com/ardupic/voltronic-inverter-communication-protocols): `PI30MAX.Communication.Protocol20210217.pdf` (Axpert MAX, `QBMS`/`PBMS`, `PBATCD`), `PI17_InfiniSolar-5KW-protocol-20160309.pdf` (PI17), and `PI41_LV5048.5KW.protocol-20190222.for.customer.pdf` (PI41 split-phase).
- [`jblance/mpp-solar`](https://github.com/jblance/mpp-solar) — `mppsolar/protocols/pi30.py`, `pi30max.py`, `pi41.py`, `pi18.py`, `pi17.py`, `protocol_helpers.py`, and `docs/protocols/PI30_Q1 definition.md` (field definitions, `Q1` decode, and test frames).
- [`syssi/esphome-pipsolar`](https://github.com/syssi/esphome-pipsolar) — `README.md` (RJ45 pinout, MAX3232 wiring, and supported model list).
- [`ned-kelly/docker-voltronic-homeassistant`](https://github.com/ned-kelly/docker-voltronic-homeassistant) — `sources/inverter-cli/inverter.cpp` and `main.cpp` (`cal_crc_half` reference implementation and `QPIGS`/`QPIRI` parsing notes).

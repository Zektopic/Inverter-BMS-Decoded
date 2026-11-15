# LiTime / Redodo / Power Queen — Smart Bluetooth LiFePO4 BMS BLE Protocol

> Status: ✅ Verified — field offsets, scaling, and 8-bit sum checksum verified against captured 105-byte BLE notification frames in [`patman15/aiobmsble` (`tests/bms/test_redodo_bms.py`)](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_redodo_bms.py) and cross-checked with [`calledit/LiTime_BMS_bluetooth`](https://github.com/calledit/LiTime_BMS_bluetooth), [`mirosieber/Litime_BMS_ESP32`](https://github.com/mirosieber/Litime_BMS_ESP32), [`rubenmuehlhans/litime-ble-hacs`](https://github.com/rubenmuehlhans/litime-ble-hacs), and [`gearboxlabs/gbl-litime-ble`](https://github.com/gearboxlabs/gbl-litime-ble/blob/main/docs/protocol.md).

## Summary

| | |
|---|---|
| Products | **LiTime** (formerly **Ampere Time**, including **Starry Sea**), **Redodo**, **Power Queen**, and **ECO-WORTHY** (selected 12 V / 24 V smart models) 12 V / 24 V / 51.2 V smart Bluetooth LiFePO4 batteries |
| Physical layer | Bluetooth Low Energy (BLE 4.0+ GATT, typically using a Beken `BK-BLE-1.0` BLE module) |
| Settings | Primary service `0xFFE0` (`0000ffe0-0000-1000-8000-00805f9b34fb`); Notify RX `0xFFE1`; Write TX `0xFFE2` |
| Addressing | Point-to-point BLE GATT connection (one active central connection at a time); payload header byte 3 fixed to `0x01` |
| Roles | Client (phone app, ESP32, Home Assistant) is GATT central and polls every 1–30 s; BMS responds with a single notification on `0xFFE1` |
| Byte order | Little-endian (`LE`) for all multi-byte integer fields |

### BLE discovery and GATT profile

Batteries using this protocol advertise service UUID `0xFFE0` (`0000ffe0-0000-1000-8000-00805f9b34fb`) and manufacturer data under company ID `0x585A` (`22618`, containing the 6-byte MAC address, often with OUI `C8:47:80`).

| Brand | Observed BLE `local_name` prefixes | Example advertised name |
|---|---|---|
| **LiTime** (Ampere Time) | `L-12*`, `L-24*`, `L-51*`, `LT-12*`, `LT-24*`, `LT-51*` | `LT-24100B-A00473`, `L-12100BNNA70-A09683`, `LT-51230BGC-A00229` |
| **Starry Sea** (by LiTime) | `S-*`, `SS-*` | `S-12100BNNA70-A00101`, `SS-12100BGH-A00102` |
| **Redodo** | `R-12*`, `R-24*`, `RO-12*`, `RO-24*` | `R-24100BNN160-A00643`, `RO-24100B-A00162` |
| **Power Queen** | `P-12*`, `P-24*`, `PQ-12*`, `PQ-24*` | `P-24050BNNA70-A01152`, `PQ-24100B-A00292` |
| **ECO-WORTHY** | `T-12*`, `T-24*` | `T-12100BNNH19-A00130` |

> **Note:** Some older `LT-12*` / `L-12V*` batteries use the E&J Technology ASCII/binary protocol rather than this `00 00 … 55 AA` protocol. In [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/redodo_bms.py), Redodo/LiTime/Power Queen devices are distinguished by advertising manufacturer ID `0x585A` alongside service `0xFFE0`.

| GATT Attribute | 16-bit UUID | Full 128-bit UUID | Properties | Direction / Role |
|---|---|---|---|---|
| Primary Service | `0xFFE0` | `0000ffe0-0000-1000-8000-00805f9b34fb` | — | BMS telemetry and control service |
| RX Characteristic | `0xFFE1` | `0000ffe1-0000-1000-8000-00805f9b34fb` | Notify, Read, Write | **BMS → Client**: subscribe to notifications for responses |
| TX Characteristic | `0xFFE2` | `0000ffe2-0000-1000-8000-00805f9b34fb` | Write, Write Without Response | **Client → BMS**: write 8-byte command frames |
| Control / Aux | `0xFFE3` | `0000ffe3-0000-1000-8000-00805f9b34fb` | Notify, Write | Present on some firmware; unused for standard polling |

---

## Framing

Both requests (written to `0xFFE2`) and responses (notified on `0xFFE1`) use the same binary envelope:

| Offset | Size | Type | Value / Meaning |
|---|---|---|---|
| `0` | 2 | `u8[2]` | Start of frame (SOF): `00 00` |
| `2` | 1 | `u8` | Payload length `L` (number of bytes between byte `2` and the final checksum byte; total frame length = `L + 4`) |
| `3` | 1 | `u8` | Protocol / address byte: `0x01` |
| `4` | 1 | `u8` | Command code (`CMD` in requests, e.g. `0x13`; `CMD \| 0x80` in status responses, e.g. `0x93`) |
| `5` | 2 | `u8[2]` | Marker bytes: `55 AA` |
| `7 … L+2` | `L − 4` | `u8[…]` | Command parameters (0 bytes for standard 8-byte requests, where `L = 0x04`) or response data (`97` bytes for status response, where `L = 0x65 = 101`) |
| `L + 3` | 1 | `u8` | Checksum: 8-bit unsigned sum of all preceding bytes (`sum(frame[:-1]) & 0xFF`) |

### Reassembly and MTU notes

- **Requests** are 8 bytes (`L = 0x04`) and fit in a single default BLE ATT write (`Write With Response` or `Write Without Response` on `0xFFE2`).
- **Status responses** (`CMD = 0x93`) have `L = 0x65` (`101` decimal) and a total length of `101 + 4 = 105` bytes (`0x69`). With BLE 4.2+ data length extension / negotiated MTU ≥ 108, the entire 105-byte frame arrives in a single notification on `0xFFE1`. If a BLE stack fragments the notification across multiple ATT chunks, buffer starting from `00 00 65 01 93 55 AA` until `frame[2] + 4` (`105`) bytes have been received, then verify the trailing checksum byte.

### Checksum algorithm and worked examples

The checksum is an 8-bit modulo-256 sum (`crc_sum`) over every byte of the frame except the final checksum byte itself:

$$\text{checksum} = \left(\sum_{i=0}^{N-2} \text{frame}[i]\right) \bmod 256$$

For any 8-byte command frame `00 00 04 01 <CMD> 55 AA <CHK>`, the constant bytes sum to `0x00 + 0x00 + 0x04 + 0x01 + 0x55 + 0xAA = 0x104`, which reduces modulo 256 to:

$$\text{CHK} = (\text{0x04} + \text{CMD}) \ \& \ \text{0xFF}$$

#### Worked example 1 — Status query request (`0x13`)

```
Frame (without checksum): 00 00 04 01 13 55 AA
Sum: 0x00 + 0x00 + 0x04 + 0x01 + 0x13 + 0x55 + 0xAA = 0x0117
Checksum: 0x0117 & 0xFF = 0x17
Complete 8-byte request:  00 00 04 01 13 55 AA 17
```

#### Worked example 2 — Captured 105-byte status response (`0x93`)

Verified in Python against the captured Redodo/LiTime frame in [`tests/bms/test_redodo_bms.py`](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_redodo_bms.py):

```python
frame = bytes.fromhex(
    "000065019355aa0046660000bc670000f50cf70cfc0cfb0cf80cf20cfa0cf50c"
    "0000000000000000000000000000000065faffff17001600feff00000000e91a"
    "0429000000000000000000000000000000000000000000000000410064000000"
    "030000005f010000a2"
)
assert len(frame) == frame[2] + 4 == 105
assert sum(frame[:-1]) == 0x13A2
assert (sum(frame[:-1]) & 0xFF) == frame[-1] == 0xA2
```

---

## Requests and responses / frame ids

Commands reverse-engineered from the official LiTime mobile app ([`calledit/LiTime_BMS_bluetooth`](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html)) and implemented in [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/redodo_bms.py) and [`rubenmuehlhans/litime-ble-hacs`](https://github.com/rubenmuehlhans/litime-ble-hacs):

| Command | Direction | Request frame (hex) | Payload / Notes |
|---|---|---|---|
| `0x01` | Client → BMS | `00 00 04 01 01 55 AA 05` | Product registration (app handshake) |
| `0x02` | Client → BMS | `00 00 04 01 02 55 AA 06` | Disconnect registration |
| `0x0A` | Client → BMS | *(control — see below)* | Turn on charging MOSFET |
| `0x0B` | Client → BMS | *(control — see below)* | Turn off charging MOSFET |
| `0x0C` | Client → BMS | *(control — see below)* | Turn on discharging MOSFET |
| `0x0D` | Client → BMS | *(control — see below)* | Turn off discharging MOSFET |
| `0x10` | Client → BMS | `00 00 04 01 10 55 AA 14` | Get serial number (ASCII string in bytes `8 … len-2`) |
| `0x13` | Client → BMS | `00 00 04 01 13 55 AA 17` | **Query battery status** (primary polling command) |
| `0x93` | BMS → Client | `00 00 65 01 93 55 AA … <chk>` | **Battery status response** (105 bytes total, `L = 0x65`) |
| `0x16` | Client → BMS | `00 00 04 01 16 55 AA 1A` | Get firmware / hardware version and build date |
| `0x40` | Client → BMS | *(control — see below)* | Set SOH and SOC *(unverified payload)* |
| `0x41` | Client → BMS | `00 00 04 01 41 55 AA 45` | Get SOH and SOC |
| `0x43` | Client → BMS | `00 00 04 01 43 55 AA 47` | Get nominal capacity |
| `0x60` | Client → BMS | *(control — see below)* | Shutdown BMS |
| `0x64` | Client → BMS | *(control — see below)* | Sleep BMS |

### Battery status response (`0x93`, 105 bytes)

All multi-byte fields are **little-endian (`LE`)**. Byte offsets below are 0-based from the start of the notification frame (`00 00 65 01 93 55 AA …`).

| Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `0` | 2 | `u8[2]` | — | Start of frame (`00 00`) |
| `2` | 1 | `u8` | bytes | Payload length (`0x65` = `101` bytes; total frame = `105` bytes) |
| `3` | 1 | `u8` | — | Protocol / address byte (`0x01`) |
| `4` | 1 | `u8` | — | Response command code (`0x93` = `0x13 \| 0x80`) |
| `5` | 2 | `u8[2]` | — | Marker (`55 AA`) |
| `7` | 1 | `u8` | — | Reserved / status prefix (`0x00`) |
| `8` | 4 | `u32 LE` | 0.001 V (1 mV) | Measured total pack terminal voltage (`messured_total_voltage`) |
| `12` | 4 | `u32 LE` | 0.001 V (1 mV) | Sum of cell voltages (`cells_added_together_voltage`; [`aiobmsble`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/redodo_bms.py) reads `u16 LE` at offset `12`, which holds the full value up to 65.535 V) |
| `16` | 32 | `16 × u16 LE` | 0.001 V (1 mV) | Cell voltages 1–16 (`0x0000` = cell slot unused; 4 cells on 12 V packs, 8 cells on 24 V packs, 16 cells on 51.2 V packs) |
| `48` | 4 | `s32 LE` | 0.001 A (1 mA) | Pack current (**positive = charging**, **negative = discharging**) |
| `52` | 2 | `s16 LE` | 1 °C | Cell / battery temperature 1 (`cell_temp`) |
| `54` | 2 | `s16 LE` | 1 °C | MOSFET temperature (`mosfet_temp`; treated as temperature sensor 2 in `aiobmsble`) |
| `56` | 6 | `3 × s16 LE` | 1 °C | Additional temperature sensors 3–5 (offsets `56`, `58`, `60`; `0x0000` when unused) |
| `62` | 2 | `u16 LE` | 0.01 Ah | Remaining capacity (`remaining_amph` / `cycle_charge`) |
| `64` | 4 | `u32 LE` | 0.01 Ah | Full charge / design capacity (`full_charge_capacity_amph`; upper 2 bytes at `66–67` are `00 00` on standard capacities) |
| `68` | 4 | `u32 LE` | bitmask | Heat / discharge-switch status (`heat`; see status bits below) |
| `72` | 4 | `u32 LE` | bitmask | Balance memory active (`balance_memory_active`: cells flagged for balancing) |
| `76` | 4 | `u32 LE` | bitmask | Protection state flags (`protection_state`; lower 32 bits of `problem_code` in `aiobmsble`) |
| `80` | 4 | `u32 LE` | bitmask | Failure state flags (`failure_state`; upper 32 bits of `problem_code` in `aiobmsble`) |
| `84` | 4 | `u32 LE` | bitmask | Active cell balancing bitmask (`is_balancing` / `balancer`: bit $n = 1$ when cell $n+1$ is balancing) |
| `88` | 2 | `u16 LE` | enum / bits | Battery charging state (`battery_state`: `0x0000` discharging/idle, `0x0001` charging, `0x0002` unknown, `0x0004` charge disabled) |
| `90` | 2 | `u16 LE` | 1 % | State of Charge (`SOC` / `battery_level`, 0–100 %) |
| `92` | 4 | `u32 LE` | 1 % | State of Health (`SOH` / `battery_health`) |
| `96` | 4 | `u32 LE` | 1 cycle | Discharge cycle count (`discharges_count` / `cycles`) |
| `100` | 4 | `u32 LE` | 1 Ah (or 0.001 Ah — see Errata) | Cumulative discharged capacity (`discharges_amph_count`) |
| `104` | 1 | `u8` | — | Checksum (`sum(frame[0:104]) & 0xFF`) |

---

## Alarms and status bits

In [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/redodo_bms.py), bytes `76–83` are read together as an 8-byte (`u64 LE`) `problem_code` covering both protection flags (`76–79`) and failure flags (`80–83`). Any non-zero bit indicates an active protection or hardware fault.

### Protection state (`u32 LE` at offset `76`)

| Byte | Bit (in `u32 LE`) | Hex mask (`u32`) | Meaning |
|---|---|---|---|
| `76` | Bit 2 | `0x00000004` | Overcharge protection (Cell / Pack OVP) |
| `76` | Bit 5 | `0x00000020` | Over-discharge protection (Cell / Pack UVP) |
| `76` | Bit 6 | `0x00000040` | Charging over-current protection (Charge OCP) |
| `76` | Bit 7 | `0x00000080` | Discharging over-current protection (Discharge OCP) |
| `77` | Bit 8 | `0x00000100` | High-temperature protection 1 (Charge/Discharge OTP) |
| `77` | Bit 9 | `0x00000200` | High-temperature protection 2 |
| `77` | Bit 10 | `0x00000400` | Low-temperature protection 1 (Charge/Discharge UTP) |
| `77` | Bit 11 | `0x00000800` | Low-temperature protection 2 |
| `77` | Bit 14 | `0x00004000` | Short-circuit protection |

### Failure state (`u32 LE` at offset `80`)

Decoded in [`calledit/LiTime_BMS_bluetooth`](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html) from the lower bytes of `frame[80:84]`:
- **Byte `80` / `81`**: Cell error flags.
- **Byte `82` / `83`**: BMS hardware / MOSFET error flags (tested up to bit 63 in `aiobmsble` `test_redodo_bms.py`, where `frame[83] = 0x80` maps to `problem_code = 1 << 63`).

### Heat and switch state (`u32 LE` at offset `68`) and battery state (`u16 LE` at offset `88`)

| Offset | Value / Mask (`LE`) | Meaning |
|---|---|---|
| `68` (`u32`) | `0x00000001` | Heater active (observed on heated LiTime models; `aiobmsble` treats any non-zero `u32` at offset `68` as `heater = True`) |
| `68` (`u32`) | `0x00000002` | Heater error (`heater_error`) |
| `68` (`u32`) | `0x00000080` | Discharge disabled via app command (`turn_off_discharge`) |
| `88` (`u16`) | `0x0000` | Discharging or idle |
| `88` (`u16`) | `0x0001` | Charging active |
| `88` (`u16`) | `0x0002` | Unknown state |
| `88` (`u16`) | `0x0004` | Charging disabled via app command (`turn_off_charging`) |

---

## Control commands (dangerous)

> **Warning:** The commands below switch the high-current charge/discharge MOSFETs, alter calibrated SOC/SOH, or put the BMS into sleep/shutdown. Opening the discharge MOSFET under load or while an alternator/inverter is connected can cause voltage spikes or equipment damage, and shutting down the BMS may require applying an external charger voltage to wake the pack. Use read-only polling (`0x13`) for monitoring.

| Command | Function | Effect |
|---|---|---|
| `0x0A` | Turn on charging MOSFET | Re-enables the charge FET (`00 00 04 01 0A 55 AA 0E`) |
| `0x0B` | Turn off charging MOSFET | Disables charging via the charge FET (`00 00 04 01 0B 55 AA 0F`) |
| `0x0C` | Turn on discharging MOSFET | Re-enables the discharge FET (`00 00 04 01 0C 55 AA 10`) |
| `0x0D` | Turn off discharging MOSFET | Cuts off battery output via the discharge FET (`00 00 04 01 0D 55 AA 11`) |
| `0x40` | Set SOH and SOC | Overwrites stored State of Health / State of Charge calibration |
| `0x60` | Shutdown command (`0x60`) | Powers down the BMS |
| `0x64` | Sleep command (`0x64`) | Puts the BMS into low-power sleep mode |

---

## Captured frames

### 1. 24 V 100 Ah (8S) Redodo / LiTime battery discharging at 1.435 A

Source: [`patman15/aiobmsble` — `tests/bms/test_redodo_bms.py`](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_redodo_bms.py) (`MockRedodoBleakClient._RESP`).

```
Request (TX -> 0xFFE2):
00 00 04 01 13 55 AA 17

Response (RX <- 0xFFE1, 105 bytes):
00 00 65 01 93 55 AA 00 46 66 00 00 BC 67 00 00
F5 0C F7 0C FC 0C FB 0C F8 0C F2 0C FA 0C F5 0C
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
65 FA FF FF 17 00 16 00 FE FF 00 00 00 00 E9 1A
04 29 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 41 00 64 00 00 00 03 00
00 00 5F 01 00 00 A2
```

Decoded fields from this capture:
- **Header (`0..7`)**: `00 00` SOF, length `0x65` (101 bytes), `01 93` (status reply), `55 AA 00`
- **Measured terminal voltage (`8..11`)**: `46 66 00 00` → `26182` mV = **26.182 V**
- **Cell sum voltage (`12..15`)**: `BC 67 00 00` → `26556` mV = **26.556 V**
- **Cell voltages 1–8 (`16..31`)**: `0x0CF5` (**3.317 V**), `0x0CF7` (**3.319 V**), `0x0CFC` (**3.324 V**), `0x0CFB` (**3.323 V**), `0x0CF8` (**3.320 V**), `0x0CF2` (**3.314 V**), `0x0CFA` (**3.322 V**), `0x0CF5` (**3.317 V**); cells 9–16 (`32..47`) are `00 00` (8S pack, $\Delta V = 0.010\text{ V}$)
- **Current (`48..51`)**: `65 FA FF FF` (`s32 LE`) → `-1435` mA = **−1.435 A** (discharging, **−38.108 W**)
- **Temperatures (`52..61`)**: sensor 1 = `17 00` (**23 °C**), sensor 2 (MOSFET) = `16 00` (**22 °C**), sensor 3 = `FE FF` (**−2 °C**), sensors 4–5 = `00 00`
- **Remaining capacity (`62..63`)**: `E9 1A` → `6889` × 0.01 Ah = **68.89 Ah**
- **Full charge capacity (`64..67`)**: `04 29 00 00` → `10500` × 0.01 Ah = **105.00 Ah** (nominal 100 Ah pack)
- **Heat (`68..71`), Balance memory (`72..75`), Protection (`76..79`), Failure (`80..83`), Balancing (`84..87`), State (`88..89`)**: all `0` (no alarms, idle/discharging)
- **SOC (`90..91`)**: `41 00` → **65 %**
- **SOH (`92..95`)**: `64 00 00 00` → **100 %**
- **Cycles (`96..99`)**: `03 00 00 00` → **3 cycles**
- **Cumulative discharge (`100..103`)**: `5F 01 00 00` → `351` (Ah)
- **Checksum (`104`)**: `0xA2` (`sum(frame[0:104]) = 0x13A2`, `0x13A2 & 0xFF = 0xA2`)

---

## Errata and disagreements

- **Copy-paste checksum bug for `0x0C` / `0x0D` in `calledit/LiTime_BMS_bluetooth`:** In [`index.html` lines 128–129](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html#L128-L129), the JavaScript `commands` dictionary defines `turn_on_discharge` (`0x0C`) with checksum `0x0E` and `turn_off_discharge` (`0x0D`) with checksum `0x0F` (copied from `0x0A` / `0x0B`). The decompiled app table in the comment block immediately below ([lines 140–141](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html#L140-L141)) and [`rubenmuehlhans/litime-ble-hacs`](https://github.com/rubenmuehlhans/litime-ble-hacs) give the correct checksums `0x10` (`0x04 + 0x0C`) and `0x11` (`0x04 + 0x0D`).
- **Two's complement current bug in `calledit/LiTime_BMS_bluetooth`:** [`index.html` lines 74–75](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html#L74-L75) computes negative current using bitwise NOT (`~parseInt(...)`), which is off by 1 mA (`-1434 mA` instead of `-1435 mA`). Standard signed 32-bit two's complement (`s32 LE`) at offset `48` is used by `aiobmsble`, `Litime_BMS_ESP32`, and `litime-ble-hacs`.
- **Pack voltage offset (`8` vs `12`):** Offset `8` (`u32 LE`) is the BMS's measured terminal voltage (`26.182 V` in the capture), while offset `12` (`u32 LE`) is the sum of the individual cell voltages (`26.556 V`). `calledit/LiTime_BMS_bluetooth` and `Litime_BMS_ESP32` report both; `aiobmsble` and `litime-ble-hacs` use offset `12` as the primary pack voltage.
- **Cumulative discharge scaling at offset `100` (`u32 LE`):** [`calledit/LiTime_BMS_bluetooth`](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html#L69) displays the raw integer at `100..103` directly as `discharges_amph_count` (in the 105 Ah test capture with 3 cycles, raw value `0x0000015F = 351`, which matches ~3.3 cycles × 105 Ah = `351 Ah`, or `1 Ah` per LSB, or `0.1 Ah` on some models). [`Litime_BMS_ESP32`](https://github.com/mirosieber/Litime_BMS_ESP32/blob/main/src/BMSClient.cpp#L86) and [`litime-ble-hacs`](https://github.com/rubenmuehlhans/litime-ble-hacs) divide by `1000` (treating it as mAh), while [`gearboxlabs/gbl-litime-ble`](https://github.com/gearboxlabs/gbl-litime-ble/blob/main/docs/protocol.md) observes `262` on a 230 Ah battery with 1 cycle, `733` on a 100 Ah battery with 7 cycles, and `962` on a 100 Ah battery with 9 cycles — confirming that the raw integer is **cumulative Ah discharged** (`1 Ah` per LSB, matching `cycles × capacity_Ah`), not mAh.
- **Truncated example hex dump in `gearboxlabs/gbl-litime-ble`:** The example packet in [`gbl-litime-ble/docs/protocol.md`](https://github.com/gearboxlabs/gbl-litime-ble/blob/main/docs/protocol.md) is 103 bytes instead of 105 bytes (missing 2 zero bytes in the idle current field at `48..51`), which shifts all offsets after byte 48 down by 2 in that raw dump (`SOH` `0x0069 = 105` misidentified as "packet length field" at `92–93`).

---

## Sources

- [`patman15/aiobmsble` — `aiobmsble/bms/redodo_bms.py`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/redodo_bms.py) — GATT UUIDs (`0xFFE0`, `0xFFE1`, `0xFFE2`), BLE advertisement matchers (`0x585A`, name prefixes for LiTime, Starry Sea, Redodo, Power Queen, ECO-WORTHY), status query `00 00 04 01 13 55 AA 17`, frame validation, and `_FIELDS` offset table.
- [`patman15/aiobmsble` — `tests/bms/test_redodo_bms.py`](https://github.com/patman15/aiobmsble/blob/main/tests/bms/test_redodo_bms.py) and [`aiobmsble/test_data/redodo_bms.json`](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/test_data/redodo_bms.json) — captured 105-byte response frame, protection bit tests, temperature sensor detection tests, and real BLE advertisement captures.
- [`patman15/BMS_BLE-HA`](https://github.com/patman15/BMS_BLE-HA) — Home Assistant BLE BMS integration using `aiobmsble`.
- [`calledit/LiTime_BMS_bluetooth` — `index.html`](https://github.com/calledit/LiTime_BMS_bluetooth/blob/main/index.html) — original Web Bluetooth reverse-engineering of the LiTime mobile app, including the command table (`0x01`–`0x64`), protection bit masks, heat/state codes, and version/serial parsers.
- [`mirosieber/Litime_BMS_ESP32` — `src/BMSClient.cpp`](https://github.com/mirosieber/Litime_BMS_ESP32/blob/main/src/BMSClient.cpp) — ESP32 Arduino BLE client implementation confirming compatibility across LiTime, Redodo, and Power Queen batteries.
- [`rubenmuehlhans/litime-ble-hacs` — `README.md`](https://github.com/rubenmuehlhans/litime-ble-hacs/blob/main/README.md) — Home Assistant custom integration protocol summary and command/protection tables.
- [`gearboxlabs/gbl-litime-ble` — `docs/protocol.md`](https://github.com/gearboxlabs/gbl-litime-ble/blob/main/docs/protocol.md) — Device Information Service strings (`BEKEN SAS` / `BK-BLE-1.0`), `0xFFE3` characteristic observation, and multi-battery empirical values for SOC, full capacity, cycles, and offset `100`.

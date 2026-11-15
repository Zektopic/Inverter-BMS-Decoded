# NEEY / Heltec / Enerkey — Smart Active Balancer (BLE Protocol)

> Status: ✅ Verified — GATT UUIDs, `AA 55 11` / `55 AA 11` framing, IEEE-754 float offsets, and sum-8 checksums verified against [`patman15/aiobmsble` (`neey_bms.py` & `tests/bms/test_neey_bms.py`)](https://github.com/patman15/aiobmsble/blob/main/aiobmsble/bms/neey_bms.py) and [`syssi/esphome-neey-bms`](https://github.com/syssi/esphome-neey-bms).

## Summary

| | |
|---|---|
| Products | NEEY 4A / 10A / 15A active balancers, Enerkey (`EK-*`), GW (`GW-*`), Heltec smart active balancers |
| GATT Service UUID | `0000ffe0-0000-1000-8000-00805f9b34fb` (advertisement filter: `0000fee7-…`) |
| Characteristic (Notify + Write) | `0000ffe1-0000-1000-8000-00805f9b34fb` |
| Byte order | **Little-endian** (including IEEE-754 32-bit floats `<f`) |

---

## Framing & Checksum

### Host → Balancer command (20 bytes)

```
AA 55 11 01  <cmd>  00 14 00  <10 zero/data bytes>  <sum8>  FF
```

- **Checksum**: 8-bit sum of bytes `0..17` (`sum(frame[:18]) & 0xFF`), followed by `0xFF`.
- **Worked example (verified in Python):**
  ```
  AA 55 11 01 01 00 14 00 00 00 00 00 00 00 00 00 00 00 26 FF   # cmd 0x01: request device info (sum = 0x126 -> 0x26)
  ```

### Balancer → Host notification

```
55 AA 11 01  <type>  00  <len_u16_LE>  <payload...>  <sum8>  FF
```

- `type = 0x01`: Device info (`[8:24]` model ASCII, `[24:32]` HW version, `[32:40]` SW version; models starting with `EK-B` use `v2` offsets).
- `type = 0x02`: Live cell & balancer telemetry (reassemble chunks until `len == frame[6:8]`).

---

## `0x02` Live Telemetry Frame

All floats are 4-byte little-endian IEEE-754 (`<f`).

| Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `8..103` | 24 × 4 | `<f` | 1 V | Cell voltages 1–24 |
| `105..200` | 24 × 4 | `<f` | 1 Ω | Cell internal / wire resistances 1–24 |
| `201` | 4 | `<f` | 1 V | Total pack voltage |
| `209` | 4 | `<f` | 1 V | Max cell voltage delta |
| `216` | 1 | u8 | enum | Status / problem code (`0x05` = balancing active; `1, 3, 7, 8, 9, 10, 11` = error) |
| `218` (`217` on v1) | 4 | `<f` | 1 A | Balance current |
| `222` | 4 | `<f` | 1 A | Pack current (if equipped) |
| `242` | 4 | `<f` | 1 Ah | Design capacity (`v2`) |
| `246` | 4 | `<f` | 1 Ah | Remaining capacity (`v2`) |
| `250` | 4 | `<f` | 1 % | State of Charge (`v2`) |

---

## Control Commands (Dangerous)

Balancer configuration commands (enabling/disabling balancing, changing cell count or balance current) use `AA 55 11 01` write frames. Misconfiguring cell count or chemistry thresholds can over-discharge cells. See [`syssi/esphome-neey-bms`](https://github.com/syssi/esphome-neey-bms) before sending write commands.

## Sources

- [`patman15/aiobmsble`](https://github.com/patman15/aiobmsble) — `aiobmsble/bms/neey_bms.py` and `tests/bms/test_neey_bms.py`
- [`syssi/esphome-neey-bms`](https://github.com/syssi/esphome-neey-bms) — ESPHome BLE implementation

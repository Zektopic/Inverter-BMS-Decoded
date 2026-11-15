# SHINEGLE motor controller → instrument cluster CAN

> Status: 📄 Vendor doc. Transcribed from [`SHINEGLE-CAN与仪表通讯.pdf`](SHINEGLE-CAN与仪表通讯.pdf) ("CAN communication with the
> instrument"). **This is not a BMS protocol**: it is the message a Shineglai (盛朗谷) electric-vehicle motor controller sends to the
> dashboard. It is kept here because it shares the CAN bus with the battery in small EVs.

## Summary

| | |
|---|---|
| Scope | pure-electric vehicles: Shineglai motor controller ↔ instrument (dashboard) |
| Physical layer | CAN 2.0B |
| Bitrate | 250 kbit/s |
| Identifiers | 29-bit extended, SAE J1939 layout (priority, R, DP, PDU format, PDU specific, source address) |
| Byte order | little-endian (low byte first) |

## Controller (MCU) frame

| | |
|---|---|
| ID | `0x10088A5A` |
| Length | 8 bytes |
| Period | 20 ms |

| Byte | Field | Scaling |
|---|---|---|
| 1–2 | battery voltage (low byte first) | 0.1 V |
| 3–4 | motor speed (low byte first) | 1 rpm |
| 5 | reserved | |
| 6 | motor temperature | °C = raw − 50 |
| 7 | gear, bits 0–1 | `00` reverse, `01` forward, `10`/`11` neutral; bits 2–7 reserved |
| 8 | fault code | 1–255, see table |

## Fault codes (byte 8)

From the document's fault table (English names as printed):

| Code | Meaning |
|---|---|
| 1 | no known faults |
| 2 | low battery voltage |
| 3 | phase U current shunt fault |
| 4 | phase V current shunt fault |
| 5 | system fault |
| 6 / 7 / 8 | phase U / V / W fault |
| 9 | motor (temperature) sensor fault |
| 10 | low precharge voltage |
| 11 | hardware failsafe |
| 12 | controller over-temperature |
| 13 | contactor or brake fault |
| 14 | over-voltage |
| 15 | battery voltage too low |
| 16 | precharge voltage too slow |
| 17 / 18 | current sensor 1 / 2 fault |
| 19 | high voltage fault in brake |
| 20 / 21 | encoder fault 1 / 2 |

The document lists further codes after 21; see the PDF.

## Sources

- `SHINEGLE-CAN与仪表通讯.pdf` (4 pages, Chinese with English fault names), stored in this folder.

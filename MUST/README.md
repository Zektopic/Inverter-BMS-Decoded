# MUST battery — BMS CAN bus protocol V2.0.2

> Status: 📄 Vendor doc. Summary of [`PDFs/MUST BMS CAN bus protocol V2.0.2 20210602 (1).pdf`](PDFs/) (16 pages, Chinese/English,
> 2021-06-02). Field-level details are in the PDF; not yet checked against captures.

## Summary

| | |
|---|---|
| Products | MUST lithium battery modules (master battery talks to the inverter) |
| Physical layer | CAN, standard frames for the data set; extended ids for control/upgrade |
| Bitrate | **100 kbit/s** as stated in the document (note: most Pylontech-style batteries use 500 kbit/s; check your unit) |
| Period | BMS sends its data set every 250 ms; the inverter sends `0x305` once per second |
| Byte order | little-endian |

## Standard data set (Pylontech-compatible ids)

| Id | Name | Content (scaling per the document) |
|---|---|---|
| `0x351` | operation limits | charge voltage limit 0.1 V, charge current limit 0.1 A, discharge current limit 0.1 A, discharge voltage limit 0.1 V |
| `0x355` | SOC and SOH | % |
| `0x356` | analog quantity | voltage 0.01 V, current 0.1 A, temperature 0.1 °C |
| `0x359` | protection and alarm | protection / alarm bit fields (see PDF §2.1.4) |
| `0x35C` | BMS request | charge / discharge enable and force-charge flags |
| `0x35E` | brand | manufacturer name, ASCII |
| `0x305` | inverter → BMS | heartbeat, `00 00 00 00 00 00 00 00` every second |

Legacy extended ids `0x30100`, `0x30200`, `0x30400`, `0x30500`, `0x30700` exist for compatibility with early MUST inverters; the
document says other integrators can ignore them.

## Extended system and module information

| Id | Content |
|---|---|
| `0x35A` | extended system information |
| `0x372` | module status |
| `0x373` | max / min analog quantities |
| `0x374`–`0x377` | addresses of the modules holding the values in `0x373` |
| `0x379` | total capacity |
| `0x350`, `0x35F`, `0x378`, `0x380`… | further system / module frames (§2.4–2.5) |

## Control (dangerous, inverter → battery)

The document asks inverters that do not need these to neither send nor answer them.

| Id | Effect |
|---|---|
| `0x4800` | turn off the whole battery group (only when there is no wake-up signal, no external DC voltage and no paralleled pile) |
| `0x1001` | heartbeat: once received, the inverter must send it at least twice every 5 minutes or the batteries turn off |
| `0x0020`, `0x0060` | switch the protocol the BMS sends |
| `0x4000000`…, `0x5000000`… | firmware upgrade sequence (size, data, CRC, restart, status) |

## Sources

- MUST, *BMS CAN Bus Protocol Version 2.0.2*, 2021-06-02 — stored in [`PDFs/`](PDFs/).
- Compare with the Pylontech low-voltage CAN protocol in [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol).

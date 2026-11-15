# TDT battery — RS485 ("Force_L" / Pylon dialect)

> Status: 🔬 Team decoded. The workbook in this folder decodes request/response pairs captured from a TDT battery "according to
> the PDF" the vendor supplied (PDF not included). The battery's own basic-information reply names `Force_L` and `Pylon`, i.e.
> it implements Pylontech's high-voltage *Force-L* RS485 command set.

## Files

| File | Content |
|---|---|
| [`Communication Decoding - TDT - According to the PDF.xlsx`](.) | sheets: *All Codes*, *2. ChargeDischarge Status Decode*, *3. Warning Data Decoded*, *4. Full Data Decoded*, *5. Basic Data Decoded*, *Miscellaneous Codes* |
| [`Logs/TDT- BMS Software.log`](Logs/) | serial capture (2024-05-05), COM port at 9600 8N1, of the vendor PC software talking to the battery |

## Framing

YD/T 1363 ASCII, the same framing as [Seplos V2](../SEPLOS/README.md#frame-format) and [Pylontech RS485](../PYLONTECH/README.md#rs485-protocol-v33):
`~` VER ADR CID1 CID2 LENGTH INFO CHKSUM `CR`. In the captures VER = `0x20`, ADR = `0x12`, CID1 = `0x46`.

## Commands (CID2)

| CID2 | Request (ADR 0x12) | Content of the reply |
|---|---|---|
| `0x60` | `~201246600000FDAB` | basic information: device name (`Force_L`), manufacturer (`Pylon`), versions, serial strings |
| `0x61` | `~201246610000FDAA` | battery system analog data (voltages, currents, temperatures, SOC…) |
| `0x62` | `~201246620000FDA9` | battery system status / warning data |
| `0x63` | `~201246630000FDA8` | charge / discharge management: charge voltage upper limit, discharge voltage lower limit, max charge current, max discharge current, status flags |
| `0x64` | `~201246640000FDA7` | **shut down** — do not send to a live system |

Example from the workbook (CID2 `0x63`): reply `~201246008008DCD35DC009C407E4B0F985` decodes to suggested charge voltage limit
56.531 V, discharge voltage lower limit 24.00 V, max charge current 25.0 A, max discharge current 20.2 A.
Field-by-field tables are in the workbook sheets.

## The capture log

The log shows the PC software polling with frames whose first byte appears as `0xBC` instead of `0x7E` (probably a framing or
baud mismatch in the capture tool), e.g. `BC 30 31 30 31 34 36 34 32 …` = `?0101 46 42 …` — ADR `0x01`, CID2 `0x91`, `0xF2`,
`0xC1`, `0xC2`, `0x42`, `0x44`. Treat it as raw evidence; it has not been decoded yet.

## Sources

- Team workbook and log in this folder.
- Pylontech low-voltage RS485 protocol for the shared framing: [PYLONTECH](../PYLONTECH/README.md#rs485-protocol-v33).

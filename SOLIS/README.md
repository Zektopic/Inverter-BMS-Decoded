# Ginlong Solis — Hybrid Inverter Modbus RTU Protocol

> Status: 🌐 Community — transcribed from [`StephanJoubert/home_assistant_solarman` (`solis_hybrid.yaml`)](https://github.com/StephanJoubert/home_assistant_solarman/blob/main/custom_components/solarman/inverter_definitions/solis_hybrid.yaml), referencing Solis *RS485 MODBUS Communication Protocol* (`RHI-(3-6)K-48ES-5G` / S5 / S6).

## Summary

| | |
|---|---|
| Products | Solis RHI-(3–6)K-48ES-5G, S5-EH1P, S6-EH1P / EH3P hybrid inverters |
| Physical layer | RS485 (COM port / stick logger) |
| Settings | `9600` 8N1 |
| Addressing | Slave ID `1` (default; configurable `1–247`) |
| Roles | EMS / logger is Modbus master; inverter is Modbus slave |
| Byte order | **Big-endian** for registers (`u16`, `s16`, `u32`, `s32`); Modbus CRC-16 is little-endian |

## Framing

Standard **Modbus RTU**:

- **Input registers (`FC 0x04`)**: `33000+` (live telemetry: PV, grid, load, meter, battery, faults).
- **Holding registers (`FC 0x03` / `0x06` / `0x10`)**: `43000+` (charge/discharge timers, backup SOC, overdischarge SOC).
- Worked request example (verified in Python):
  ```
  01 04 81 6F 00 0A 68 2C   # Read 10 input registers from 33135 (0x816F): battery status, SOC, SOH, BMS V/I
  ```

## Key Input Registers (`FC 0x04`)

| Register | Size | Type | Unit / scale | Field |
|---|---|---|---|---|
| `33000` | 1 | u16 | — | Product model |
| `33001`–`33003` | 3 | u16 | — | DSP / LCD / Protocol software version |
| `33035` | 1 | u16 | 0.1 kWh | Daily PV generation |
| `33049` / `33050` | 2 | u16 | 0.1 V / 0.1 A | PV1 voltage / current |
| `33051` / `33052` | 2 | u16 | 0.1 V / 0.1 A | PV2 voltage / current |
| `33057`–`33058` | 2 | u32 | 1 W | Inverter DC power (word order: `33058` high, `33057` low *(check endianness)*) |
| `33073` / `33076` | 2 | u16 | 0.1 V / 0.1 A | Inverter AC voltage / current |
| `33079`–`33080` | 2 | s32 | 1 W | Inverter active AC power |
| `33093` | 1 | s16 | 0.1 °C | Inverter temperature |
| `33094` | 1 | u16 | 0.01 Hz | Grid frequency |
| `33095` | 1 | u16 | enum | Inverter status (`0x0` Waiting, `0x1` Open Loop, `0x2` Soft Start, `0x3` Generating, `0x1004+` faults) |
| `33133` | 1 | u16 | 0.1 V | Battery voltage |
| `33134` | 1 | s16 | 0.1 A | Battery current |
| `33135` | 1 | u16 | enum | Battery status |
| `33139` / `33140` | 2 | u16 | 1 % | Battery SOC / SOH |
| `33141` / `33142` | 2 | u16 / s16 | 0.01 V / 0.1 A | BMS-reported battery voltage / current |
| `33143` / `33144` | 2 | u16 | 0.1 A | BMS charge / discharge current limit |
| `33147` / `33148` | 2 | u16 | 1 W | House load power / Backup load power |
| `33149`–`33150` | 2 | s32 | 1 W | Battery power |
| `33206` / `33207` | 2 | u16 | 0.1 A | Battery charge / discharge current limit |

## Key Holding Registers (`FC 0x03` / `0x06`, Write with Caution)

| Register | Unit / scale | Field |
|---|---|---|
| `43011` | 1 % | Overdischarge SOC |
| `43024` | 1 % | Backup mode reserve SOC |
| `43141` / `43142` | 0.1 A | Timed charge / discharge current |
| `43143`–`43150` | H / M | Timed charge & discharge start/end hour and minute |

## Battery CAN Interface

On the BMS CAN port (`500 kbit/s`), Solis hybrid inverters accept Pylontech low-voltage CAN (`0x351`, `0x355`, `0x356`, `0x359`, `0x35C`, `0x35E`); see [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol).

## Sources

- [`StephanJoubert/home_assistant_solarman`](https://github.com/StephanJoubert/home_assistant_solarman) — `inverter_definitions/solis_hybrid.yaml`
- [`fboundy/ha_solis_modbus`](https://github.com/fboundy/ha_solis_modbus) & [`wills106/homeassistant-solax-modbus`](https://github.com/wills106/homeassistant-solax-modbus)

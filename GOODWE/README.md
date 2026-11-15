# GoodWe — Hybrid Inverters (ET / EH / ES / EM / BT) Modbus & UDP Protocol

> Status: 🌐 Community — transcribed from [`marcelblijleven/goodwe`](https://github.com/marcelblijleven/goodwe) (`goodwe/et.py`) and [`dalathegreat/Battery-Emulator`](https://github.com/dalathegreat/Battery-Emulator).

## Summary

| | |
|---|---|
| Products | GoodWe ET, EH, BT, BH (high-voltage) and ES, EM, SBP (low-voltage) hybrid inverters |
| Physical layer | RS485 (Modbus RTU) or Wi-Fi / LAN dongle (UDP port `8899`, Modbus-over-UDP or `AA 55` protocol) |
| Settings | RS485: `9600` 8N1; UDP port `8899` |
| Addressing | Default Modbus slave address **`0xF7` (247)** |
| Byte order | **Big-endian** for registers; Modbus CRC-16 is little-endian |

## Framing

GoodWe ET/EH/BT/ES inverters speak **Modbus RTU** over RS485 and encapsulated Modbus RTU (`F7 03 <reg_hi> <reg_lo> <count_hi> <count_lo> <crc_lo> <crc_hi>`) over UDP port `8899`.

Worked request example (verified in Python):

```
F7 03 89 1C 00 14 BA C9   # Read 20 registers starting at 35100 (0x891C) from slave 0xF7
```

## Key ET / EH Registers (`FC 0x03`)

### Running Data (`35100+` & `37000+`)

| Register | Type | Unit / scale | Field |
|---|---|---|---|
| `35103` / `35104` | u16 | 0.1 V / 0.1 A | PV1 voltage / current |
| `35107` / `35108` | u16 | 0.1 V / 0.1 A | PV2 voltage / current |
| `35121` / `35122` | u16 / s16 | 0.1 V / 0.1 A | On-grid L1 voltage / current (`35126/35127` L2, `35131/35132` L3) |
| `35145` / `35146` | u16 / s16 | 0.1 V / 0.1 A | Backup L1 voltage / current (`35151/35152` L2, `35157/35158` L3) |
| `35174` / `35175` / `35176` | s16 | 0.1 °C | Inverter air / module / radiator temperature |
| `35178` / `35179` | u16 | 0.1 V | DC bus / NBus voltage |
| `35180` | u16 | 0.1 V | Battery 1 voltage (`35262` Battery 2 voltage) |
| `37003` | s16 | 0.1 °C | Battery 1 temperature (`39001` Battery 2 temperature) |
| `37020` / `37021` | s16 | 0.1 °C | Battery max / min cell temperature |

### Battery Settings & BMS Data (`45350+` & `47900+`, Write with Caution)

| Register | Unit / scale | Field |
|---|---|---|
| `45352` / `45353` | 0.1 V / 0.1 A | Battery charge voltage / charge current limit |
| `47906` / `47907` / `47910` | 0.1 V / 0.1 A / 0.1 °C | BMS 1 voltage / current / temperature |
| `47924` / `47925` / `47928` | 0.1 V / 0.1 A / 0.1 °C | BMS 2 voltage / current / temperature |

## Sources

- [`marcelblijleven/goodwe`](https://github.com/marcelblijleven/goodwe) — `goodwe/et.py`
- [`dalathegreat/Battery-Emulator`](https://github.com/dalathegreat/Battery-Emulator) — `Software/src/inverter/`

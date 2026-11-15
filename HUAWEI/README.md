# Huawei SUN2000 & LUNA2000 — Modbus RTU / TCP Interface

> Status: 📄 Vendor doc — transcribed from Huawei *Solar Inverter Modbus Interface Definitions* ([Issue 05, 2023-02-16](PDFs/solar-inverter-modbus-interface-definitions-v5.pdf)) and cross-checked with [wlcrs/huawei-solar-lib](https://github.com/wlcrs/huawei-solar-lib).

## Summary

| | |
|---|---|
| Products | SUN2000-L0 / L1 / M0 / M1 / M2 / M3 inverters, LUNA2000 battery storage, DTSU666-H power meter |
| Physical layer | RS485 (Modbus RTU) or Ethernet / WLAN via Smart Dongle (Modbus TCP) |
| Settings | RS485: `9600` 8N1 (default); Modbus TCP: port `502` or `6607` (inverter AP / Smart Dongle) |
| Addressing | Slave ID `1`–`247` (single inverter default `1` on RS485, or `0` / `1` depending on Smart Dongle vs direct AP) |
| Roles | Monitoring system / EMS is Modbus master; inverter is Modbus slave |
| Byte order | **Big-endian** for all multi-byte registers (`U16`, `I16`, `U32`, `I32`, `STR`); Modbus RTU CRC-16 is little-endian |

## Framing

Standard **Modbus RTU** (with CRC-16/Modbus, polynomial `0xA001` reflected, init `0xFFFF`, appended `CRC_Lo CRC_Hi`) or **Modbus TCP** (6-byte MBAP header + PDU, no CRC).

- Supported function codes: `0x03` (read holding registers), `0x06` (write single register), `0x10` (write multiple registers), `0x2B` (read device identifiers), `0x41` (Huawei file upload).
- **Important**: Huawei uses *Gain* as a divisor: `physical_value = raw_register / Gain` (e.g., `Gain = 10` means `0.1` per LSB; `Gain = 1000` means `0.001 kW = 1 W` per LSB).

Worked Modbus RTU request examples (verified in Python):

```
01 03 7D 50 00 02 DC 76   # Read Active Power (reg 32080 = 0x7D50, 2 regs I32) from slave 0x01
01 03 93 80 00 01 A8 A6   # Read combined Energy Storage SOC (reg 37760 = 0x9380, 1 reg U16) from slave 0x01
```

## Key registers (`FC 0x03`)

### Device identification (`30000+`)

| Address | Count | Type | Gain | Unit | Field |
|---|---|---|---|---|---|
| `30000` | 15 | STR | — | — | Model name |
| `30015` | 10 | STR | — | — | Serial number (SN) |
| `30025` | 10 | STR | — | — | Part number (PN) |
| `30070` | 1 | U16 | 1 | — | Model ID |
| `30071` | 1 | U16 | 1 | — | Number of PV strings |

### Inverter running data (`32000+`)

| Address | Count | Type | Gain | Unit | Field |
|---|---|---|---|---|---|
| `32000` | 1 | Bitfield16 | 1 | — | State 1 (bit 0 standby, bit 1 grid-connected, bit 2 normal, bit 3 derating power rationing, bit 4 derating internal) |
| `32064` | 2 | I32 | 1000 | kW (`1 W`) | PV input power |
| `32080` | 2 | I32 | 1000 | kW (`1 W`) | Active AC power |
| `32082` | 2 | I32 | 1000 | kVar | Reactive power |
| `32084` | 1 | I16 | 1000 | — | Power factor (`0.001`) |
| `32085` | 1 | U16 | 100 | Hz (`0.01 Hz`) | Grid frequency |
| `32086` | 1 | U16 | 100 | % (`0.01 %`) | Inverter efficiency |

### Energy storage / LUNA2000 (`37000+` & `37700+`)

| Address | Count | Type | Gain | Unit | Field |
|---|---|---|---|---|---|
| `37000` | 1 | U16 | 1 | — | Battery unit 1 running status (`0` offline, `1` standby, `2` running, `3` fault, `4` sleep) |
| `37004` | 1 | U16 | 10 | % (`0.1 %`) | Battery unit 1 SOC |
| `37700` | 10 | STR | — | — | Battery unit 2 SN |
| `37738` | 1 | U16 | 10 | % (`0.1 %`) | Battery unit 2 SOC |
| `37741` | 1 | U16 | 1 | — | Battery unit 2 running status (`0`–`4`, same as `37000`) |
| `37760` | 1 | U16 | 10 | % (`0.1 %`) | Combined energy storage SOC (`0.0–100.0 %`) |
| `37762` | 1 | U16 | 1 | — | Combined energy storage running status |

## Control / settings registers (write with caution)

| Address | Count | Type | Access | Field |
|---|---|---|---|---|
| `47000` | 1 | U16 | RW | Storage product mode (`0` None, `1` LG-RESU, `2` HUAWEI-LUNA2000) |
| `47028` | 41 | MULTIDATA | RW | Time-of-use electricity price periods |
| `47086` | 1 | U16 | RW | Storage working mode (`0` Adaptive, `1` Fixed charge/discharge, `2` Maximise self-consumption, `3` TOU LG, `4` Fully fed to grid, `5` TOU Luna) |

> **Warning:** Writing `47000+` registers alters battery chemistry/mode and grid-export behaviour. Do not write them without verifying the exact firmware version in the vendor PDF.

## Sources

- Huawei Technologies, *Solar Inverter Modbus Interface Definitions*, Issue 05 (2023-02-16): [PDFs/solar-inverter-modbus-interface-definitions-v5.pdf](PDFs/solar-inverter-modbus-interface-definitions-v5.pdf) (also [Solar_Inverter_Modbus_Interface_Definitions.pdf](PDFs/Solar_Inverter_Modbus_Interface_Definitions.pdf), [SUN2000L_V200R001_MODBUS_Interface_Definitions.pdf](PDFs/SUN2000L_V200R001_MODBUS_Interface_Definitions.pdf))
- [wlcrs/huawei-solar-lib](https://github.com/wlcrs/huawei-solar-lib) — Python Modbus client and register definitions

# Growatt — Inverter Modbus RTU & ShineWiFi (`grott`) Protocol

> Status: 🌐 Community — transcribed from [`johanmeijer/grott`](https://github.com/johanmeijer/grott) (`grottconf.py`) and [`wills106/homeassistant-solax-modbus`](https://github.com/wills106/homeassistant-solax-modbus).

## Summary

| | |
|---|---|
| Products | Growatt SPF (Off-Grid), SPH / SPA (Hybrid), MIN / MOD / MID (Grid-tie) |
| Physical layer | RS485 (Modbus RTU) or USB/RS232 ShineWiFi logger (TCP port `5279` to ShineServer) |
| Settings | `9600` 8N1 (Modbus RTU) |
| Addressing | Slave ID `1` (default) |
| Byte order | **Big-endian** for registers (`u16`, `u32`, `s32`); Modbus CRC-16 is little-endian |

## Framing

Standard **Modbus RTU** (`FC 0x04` read input registers, `FC 0x03` read holding registers, `FC 0x06`/`0x10` write holding registers).

Worked request example (verified in Python):

```
01 04 00 00 00 2D 30 17   # Read 45 input registers starting at 0 (0x0000) from slave 0x01
```

- **ShineWiFi / `grott` TCP encapsulation**: Growatt ShineWiFi sticks forward the raw Modbus input/holding register blocks over TCP, XOR-masked with `"Growatt"` on newer firmware (`v1.20` / `v1.24`).

## Key Input Registers (`FC 0x04`, v1.20 / v1.24 & SPF)

| Register | Type | Unit / scale | Field |
|---|---|---|---|
| `0` | u16 | enum | Inverter run status (`0` Waiting, `1` Normal, `3` Fault) |
| `1`–`2` | u32 | 0.1 W | PV total input power (`pvpowerin`) |
| `3` / `4` | u16 | 0.1 V / 0.1 A | PV1 voltage / PV1 current |
| `5`–`6` | u32 | 0.1 W | PV1 power |
| `7` / `8` | u16 | 0.1 V / 0.1 A | PV2 voltage / PV2 current |
| `37` | u16 | 0.01 Hz | Grid frequency *(register varies between SPF and SPH/MIN)* |
| `1014`–`1015` *(SPH/SPA)* | s32 | 0.1 W | Battery charge/discharge power *(unverified across all FW)* |
| `1014` *(SPF)* / `1014`+ | u16 | 1 % | Battery SOC (varies by protocol version `v1.20` vs `SPF`; check `grottconf.py` for exact model) |

## Battery BMS Interface

Growatt inverters support Pylontech-style RS485 (`9600` 8N1) and CAN (`500 kbit/s`, `0x351`–`0x35E`, plus Growatt-specific `0x311`–`0x322` variants on HV/WIT models). See [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol).

## Sources

- [`johanmeijer/grott`](https://github.com/johanmeijer/grott) — `grottconf.py` (protocol layouts `T060120`, `T06NNNN`, etc.)
- [`syssi/esphome-growatt`](https://github.com/syssi/esphome-growatt) & [`wills106/homeassistant-solax-modbus`](https://github.com/wills106/homeassistant-solax-modbus)

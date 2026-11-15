# SMA — Sunny Island (LV CAN) & Sunny Boy Storage / Tripower (HV CAN)

> Status: 🌐 Community — cross-checked with [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol), [`dalathegreat/Battery-Emulator`](https://github.com/dalathegreat/Battery-Emulator), and [`Sleeper85/esphome-yambms`](https://github.com/Sleeper85/esphome-yambms).

SMA uses **500 kbit/s, 11-bit CAN 2.0A** (little-endian) for battery-to-inverter communication, closely related to the Pylontech/Victron CAN layout.

## 1. SMA Sunny Island Low-Voltage CAN (48 V)

| CAN ID | Direction | Content |
|---|---|---|
| `0x305` | Inverter → BMS | Keep-alive / inverter status |
| `0x351` | BMS → Inverter | Charge voltage limit (`0.1 V`, u16), max charge current (`0.1 A`, s16), max discharge current (`0.1 A`, s16), discharge voltage limit (`0.1 V`, u16) |
| `0x355` | BMS → Inverter | SOC (`%`, u16), SOH (`%`, u16) |
| `0x356` | BMS → Inverter | Battery voltage (`0.01 V`, s16), current (`0.1 A`, s16), temperature (`0.1 °C`, s16) |
| `0x35A` | BMS → Inverter | Alarms (bytes 0–3) & Warnings (bytes 4–7), 2 bits per fault (general, high/low voltage, high/low temp, over-current, contactor, BMS internal, cell imbalance) |
| `0x35B` | BMS → Inverter | Events / extended status |
| `0x35E` | BMS → Inverter | Manufacturer name (8 ASCII, e.g. `PYLON   ` or `SMA     `) |
| `0x35F` | BMS → Inverter | Battery type (`u16`), BMS firmware version (`u16`), nominal capacity (`Ah`, `u16`), Sunny Island ID (`u16`) |

## 2. SMA High-Voltage CAN (Sunny Boy Storage / Sunny Tripower Smart Energy)

High-voltage SMA inverters (SBS 2.5–6.0, STP SE) use BYD-style / SMA-specific HV CAN frames (`500 kbit/s`) with `0x358`, `0x3D8`, `0x4D8`, `0x518`, `0x558`, `0x598`, `0x5D8` (see `dalathegreat/Battery-Emulator` `Software/src/inverter/` for exact byte mappings per firmware).

## Sources

- [`dalathegreat/Battery-Emulator`](https://github.com/dalathegreat/Battery-Emulator) — `Software/src/inverter/`
- [`Sleeper85/esphome-yambms`](https://github.com/Sleeper85/esphome-yambms) — SMA CAN generator
- [PYLONTECH](../PYLONTECH/README.md#low-voltage-can-protocol) — shared `0x351`–`0x35E` base frames

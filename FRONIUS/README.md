# Fronius GEN24 Plus (inverter side)

> Status: 📄 Vendor doc. Summary of Fronius solution sheets and interface documentation, with links. Fronius does not publish the
> battery-side Modbus register map.

This page covers how a Fronius hybrid inverter talks **to its battery** and how you can read the inverter **from the network**.

## Battery link

| | |
|---|---|
| Inverters | Fronius Symo GEN24 Plus, Primo GEN24 Plus |
| Compatible batteries | BYD Battery-Box Premium **HVS / HVM**, LG RESU FLEX, Fronius Reserva (check the current list, it differs by inverter and region) |
| Physical layer | RS485 over a shielded 4-conductor cable, at least CAT5 |
| Protocol | **Modbus RTU** (the inverter is the master) |
| Termination | resistors at both ends of the chain; on BYD it is set with a DIP switch on the battery |
| Parallel | up to three HVS / HVM towers in parallel (capacity adds up, charge/discharge power stays that of one battery) |

Fronius's compatibility pages list, for example, BYD HVS 5.1 / 7.7 / 10.2 / 12.8 and HVM 11.0 / 13.8 / 16.6 / 19.3 / 22.1 for the
Symo GEN24 Plus. Lists differ between documents and dates; use the current one for your country:
[compatible batteries](https://fronius.com/en-gb/uk/photovoltaics/products/all-products/storage-units/compatible-batteries),
[BYD HVS/HVM technical data](https://fronius.com/en/solar-energy/installers-partners/technical-data/all-products/storage-units/compatible-batteries/byd-battery-box-premium-hvshvm).

The Modbus RTU registers between GEN24 and BYD are not public. If you capture this link (the inverter polls continuously), please
contribute the frames; see [CONTRIBUTING](../CONTRIBUTING.md).

## Reading a GEN24 from the network

### Solar API (HTTP + JSON)

Read-only JSON over HTTP on the inverter's IP address. On GEN24 it must be enabled in the inverter's web interface;
older Fronius devices have it on by default.

| Endpoint | Content |
|---|---|
| `/solar_api/v1/GetPowerFlowRealtimeData.fcgi` | site power flow: PV, grid, load, battery power, SOC |
| `/solar_api/v1/GetStorageRealtimeData.cgi` | battery details (depends on battery model) |
| `/solar_api/v1/GetInverterRealtimeData.cgi` | inverter AC/DC values |
| `/solar_api/v1/GetMeterRealtimeData.cgi` | smart-meter values |

Reference: Fronius *Solar API V1* operating instructions (document 42,0410,2012).

### Modbus TCP (SunSpec)

| | |
|---|---|
| Enable | in the inverter's web interface (Modbus TCP) |
| Port | 502 |
| Data model | SunSpec; choose *float* or *int + scale factor* in the settings (this shifts every register address, so match your reader to it) |
| Models | standard SunSpec models, e.g. 1 common, inverter models (10x / 11x), nameplate/settings, **124 basic storage control** (SOC, charge/discharge rate limits, grid charging), 160 multiple MPPT, meter models |

Writing model 124 controls the battery. Keep it read-only unless you intend to control charging.

References: [Fronius Modbus TCP interface](https://www.fronius.com/en-gb/uk/solar-energy/installers-partners/technical-data/all-products/system-monitoring/open-interfaces/modbus-tcp),
GEN24 *Modbus TCP & RTU* manual, [Home Assistant Fronius integration](https://www.home-assistant.io/integrations/fronius/).

## Sources

- [Fronius compatible batteries](https://fronius.com/en-gb/uk/photovoltaics/products/all-products/storage-units/compatible-batteries)
- [Fronius battery storage solution with BYD (solution sheet)](https://fronius.com/~/downloads/Solar%20Energy/Solution%20Sheets/SE_SOLS_Batterystorage_byd_EN.pdf)
- [Fronius help centre: BYD solutions](https://www.fronius.com/en/help-center/solar-energy/products/solar-energy-storage/solutions/byd)
- [Fronius Modbus TCP](https://www.fronius.com/en-gb/uk/solar-energy/installers-partners/technical-data/all-products/system-monitoring/open-interfaces/modbus-tcp)
- [Home Assistant: Fronius](https://www.home-assistant.io/integrations/fronius/)

# BYD Battery-Box Premium (HVS / HVM / LVS)

> Status: 🌐 Community. The CAN layout below is the BYD battery emulation in
> [dalathegreat/Battery-Emulator `Software/src/inverter/BYD-CAN.cpp`](https://github.com/dalathegreat/Battery-Emulator), which
> inverters accept as a real Battery-Box. Not yet checked against captures from a real BYD battery.

BYD Battery-Box Premium systems are high-voltage (HVS, HVM) or low-voltage (LVS, LVL) stacks with a battery control unit (BCU).
The BCU talks to the inverter over **CAN** for most inverter brands, and over **RS485 Modbus RTU** for Fronius GEN24 Plus
(see [FRONIUS](../FRONIUS/README.md)).

## HVS / HVM inverter CAN

| | |
|---|---|
| Bus | CAN 2.0A, 11-bit ids, 500 kbit/s |
| Byte order | **big-endian** |
| Start-up | the battery starts broadcasting only after a BYD-compatible inverter has sent its `0x151` identification |

| Id | Dir | Period | Content |
|---|---|---|---|
| `0x110` | battery | 2 s | max charge voltage (0.1 V), min discharge voltage (0.1 V), max discharge current (0.1 A), max charge current (0.1 A) |
| `0x150` | battery | 10 s | SOC (0.01 %), SOH (0.01 %), remaining capacity (0.1 Ah), full-charge capacity (0.1 Ah) |
| `0x1D0` | battery | 10 s | voltage (0.1 V), current (0.1 A, signed), average temperature (0.1 °C) |
| `0x210` | battery | 10 s | max temperature (0.1 °C), min temperature (0.1 °C) |
| `0x190` | battery | 60 s | alarms (layout not public) |
| `0x250` | battery | start-up | bytes 4–5 total capacity (0.1 kWh); other bytes not documented here |
| `0x3D0` | battery | start-up | model name in 4 parts, byte 0 = part number: `Battery` `-Box Pr` `emium H` `VS` |
| `0x151` | inverter | | identification; bit 0 of byte 0 set = "please identify yourself", otherwise bytes 1–7 = inverter brand |
| `0x091` | inverter | | inverter voltage (0.1 V), current, temperature (0.1 °C) |

Example (built from the layout above):

```
0x150  1F 40 27 10 01 2C 01 F4   -> SOC 80.00 %, SOH 100.00 %, 30.0 Ah remaining, 50.0 Ah full
0x1D0  0E 74 FF 38 00 FA 00 00   -> 370.0 V, -20.0 A, 25.0 °C
```

## LVS (low voltage)

The Battery-Box LVS uses its own 500 kbit/s CAN protocol for Victron / Venus OS integration, documented by
[dfch/BydCanProtocol](https://github.com/dfch/BydCanProtocol) (frames padded to 8 bytes). Many hybrid inverters also accept LVS
units via the Pylontech-style protocol; check the inverter's compatibility list.

## Local monitoring

BYD's *Be Connect* app reads the BCU over the local network. The Python library [bydhvs](https://pypi.org/project/bydhvs/)
reads HVS / HVM / LVS systems the same way.

## Sources

- [dalathegreat/Battery-Emulator `BYD-CAN.cpp`](https://github.com/dalathegreat/Battery-Emulator) — CAN ids, scaling, periods, handshake
- [dfch/BydCanProtocol](https://github.com/dfch/BydCanProtocol) — Battery-Box LVS CAN
- [bydhvs on PyPI](https://pypi.org/project/bydhvs/) — local monitoring of HVS/HVM/LVS
- Fronius / BYD solution sheet: [Fronius Battery Storage Solution BYD](https://fronius.com/~/downloads/Solar%20Energy/Solution%20Sheets/SE_SOLS_Batterystorage_byd_EN.pdf)

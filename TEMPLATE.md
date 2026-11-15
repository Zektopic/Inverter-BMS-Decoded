# <Brand> <Product family> — <Protocol name and version>

> Status: ✅ Verified / 🔬 Team decoded / 📄 Vendor doc / 🌐 Community — one line on what was checked and how.

## Summary

| | |
|---|---|
| Products | models / firmware versions known to use this protocol |
| Physical layer | RS485 / CAN / UART / BLE, connector and pinout |
| Settings | baud or bitrate, data bits, parity, stop bits; CAN id length |
| Addressing | how a device is addressed (DIP switches, defaults, ranges) |
| Roles | who is master, polling interval, broadcast period |
| Byte order | little / big endian |

## Framing

Byte layout of one frame, checksum algorithm, and a worked example:

```
<example frame>
```

## Requests and responses / frame ids

| Command or id | Direction | Payload | Notes |
|---|---|---|---|

### <Frame or command name>

| Offset | Size | Type | Unit / scale | Field |
|---|---|---|---|---|

## Alarms and status bits

| Byte | Bit | Meaning |
|---|---|---|

## Control commands (dangerous)

List them, what they do, and a warning. Do not include ready-to-send frames unless needed to explain the format.

## Captured frames

```
<request>  -> <response>   # what the device was doing
```

## Errata and disagreements

- Where the vendor document is wrong, or sources disagree, and what the captures show.

## Sources

- [Vendor document title](link) — version, date
- [Implementation](link) — file and what was taken from it

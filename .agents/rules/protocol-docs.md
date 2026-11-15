# Protocol Documentation Rules

When creating or editing any `<BRAND>/README.md`, `<BRAND>/BLE.md`, or `<BRAND>/<PROTOCOL>.md`:

1. **Follow `TEMPLATE.md`**: Include Summary table, Framing + Checksum/CRC with a Python-verified worked example, Requests/Responses or Register tables (`Offset/Register | Size | Type | Unit/scale | Field`), Alarms/Status bits, Control commands (dangerous) warning, Captured frames (if available), Errata/Disagreements, and Sources.
2. **Cross-link shared protocols**: Many BMSs and inverters share Pylontech LV CAN (`../PYLONTECH/README.md#low-voltage-can-protocol`), YD/T 1363 ASCII (`../SEPLOS/README.md#frame-format`), or PACE (`../PACE/README.md`). Link rather than duplicating.
3. **BLE pages must include**:
   - Advertisement name patterns / manufacturer IDs
   - 128-bit or normalized GATT Service UUID, RX (notify/read) Characteristic UUID, and TX (write) Characteristic UUID
   - Packet reassembly / chunking rules (and any `AT\r\n` or `AT+STAT\r\n` bridge noise to strip)
   - Upstream test-frame citations (e.g., `patman15/aiobmsble/tests/bms/test_*.py` or `syssi/esphome-*/tests/`).
4. **Conventions**:
   - `LE` / `BE` for endianness; `u8`, `u16`, `s16`, `u32`, `s32` for types.
   - Scaling as LSB value (e.g., `0.01 V`, `0.1 A`, `0.1 K: °C = (raw - 2731) / 10`).
   - Hex bytes as uppercase space-separated (`AA 55 90 EB`) or `0x1A2B` for IDs/registers.
   - UTF-8 encoding; verify markdown tables have matching column counts on every row.

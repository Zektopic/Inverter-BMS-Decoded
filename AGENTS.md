# Agent Guidelines — Inverter-BMS-Decoded

This repository is a community reference for battery management system (BMS) and hybrid inverter communication protocols (RS485, RS232, UART, CAN, and Bluetooth Low Energy). Every number, register, byte offset, scaling factor, and checksum in this repository must be **traceable to a real source or hardware capture**.

## Repository Structure

- `README.md` — Root protocol index table (`| Brand | Product / family | Link | Settings | Framing | Status | Page |`) and status legend.
- `TEMPLATE.md` — Required structure for every new brand or protocol page.
- `CONTRIBUTING.md` — Style, privacy (redact serial numbers / MACs / BLE passcodes), and safety rules.
- `RELATED_PROJECTS.md` — Curated index of upstream open-source decoders and test-frame repositories (`patman15/aiobmsble`, `syssi/esphome-*`, `dbus-serialbattery`, `Battery-Emulator`, etc.).
- `<BRAND>/` — Uppercase brand folder (`README.md`, optional `<PROTOCOL>.md` or `BLE.md`, `PDFs/` for freely downloadable vendor docs, `captures/` for real device dumps).

## Non-Negotiable Rules for Agents

1. **Never invent protocol details.** Every register address, CAN ID, GATT UUID, byte offset, scaling factor, and frame must come from a vendor document, an open-source implementation you actually read, or a real device capture. Mark uncertain items `*(unverified)*`.
2. **Verify every checksum / CRC in Python.** Before writing a worked example (`CRC-16/Modbus`, `CRC-16/XMODEM`, `YD/T 1363` `LCHKSUM`/`CHKSUM`, `sum8`, etc.), compute it with Python and confirm it matches the source frame.
3. **Use honest status badges** (see `README.md`):
   - `✅ Verified` — ONLY when checked against real captured frames (either in `<BRAND>/captures/` or cited from an upstream test suite such as `patman15/aiobmsble/tests/` or `dbus-serialbattery/tests/`).
   - `🔬 Team decoded` — Decoded by contributors from their own captures.
   - `📄 Vendor doc` — Transcribed from a manufacturer's PDF/spreadsheet.
   - `🌐 Community` — Derived from open-source code or forum posts.
4. **Document dangerous control commands clearly.** FET switching, SOC/calibration writes, shutdowns, and inverter grid/charge settings must carry an explicit warning and should not be presented as copy-paste commands for live batteries.
5. **Privacy & copyright.** Redact serial numbers, MAC addresses, and Bluetooth passcodes in captures. Only commit vendor PDFs under `<BRAND>/PDFs/` if they are freely and publicly downloadable without login, paywall, CAPTCHA, or NDA markings; keep original filenames and cite the source URL.
6. **Keep `README.md` in sync.** Whenever you add or split a protocol page, add or update its row in the root `README.md` protocol index.

## How to Enhance This Repo

See `.agents/skills/add-bms-protocol/SKILL.md` for the step-by-step runbook on researching new brands, decoding BLE protocols, verifying frames, and expanding existing pages.

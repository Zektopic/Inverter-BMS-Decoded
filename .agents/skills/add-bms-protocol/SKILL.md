---
name: add-bms-protocol
description: >-
  Step-by-step workflow for researching, verifying, and adding or enhancing BMS
  and inverter protocol documentation (RS485, UART, CAN, BLE) in
  Inverter-BMS-Decoded. Use when asked to add a new brand, decode a BLE/serial
  protocol, upgrade a page to Verified, or extract vendor PDFs.
---

# Adding or Enhancing Protocol Documentation

Follow this runbook whenever expanding `Inverter-BMS-Decoded`.

## 1. Check Existing Coverage & Upstream Sources

1. Check `README.md` to see if the brand already exists (e.g., as a sub-protocol or under `RELATED_PROJECTS.md`).
2. Read `TEMPLATE.md`, `CONTRIBUTING.md`, and a reference page such as `PYLONTECH/README.md` or `JK/BLE.md`.
3. Look for primary sources in this order:
   - **Vendor PDFs / spreadsheets** on manufacturer sites or public GitHub `docs/` folders (e.g., `syssi/esphome-*/docs/`).
   - **BLE decoders & captured test frames**: `patman15/aiobmsble` (`aiobmsble/bms/*.py` and `tests/bms/test_*.py`), `syssi/esphome-*`, `fl4p/batmon-ha`.
   - **RS485 / UART / CAN drivers**: `mr-manuel/venus-os_dbus-serialbattery` (`dbus-serialbattery/bms/` and `tests/`), `dalathegreat/Battery-Emulator` (`Software/src/inverter/`), `jblance/mpp-solar`, `kellerza/sunsynk`, `wlcrs/huawei-solar-lib`, `marcelblijleven/goodwe`, `StephanJoubert/home_assistant_solarman`.

## 2. Fetch & Extract Without Polluting the Repo

- Store temporary downloads, extracted PDF text, and scratch scripts in `$env:TEMP` (Windows) or `/tmp` (Linux/macOS), never inside the repo.
- To extract text from PDFs in Python, use `pypdf` (install to a temp target if needed).
- Only commit vendor PDFs under `<BRAND>/PDFs/` if they are freely downloadable without login, paywall, CAPTCHA, or NDA markings. Keep original filenames and cite the source URL in `## Sources`.

## 3. Verify Checksums, CRCs, and Frame Decodes in Python

Before writing any example frame or checksum formula:
1. Write a short Python snippet to compute the checksum/CRC (`sum8`, `CRC-16/Modbus`, `CRC-16/XMODEM`, `YD/T 1363` `LCHKSUM`/`CHKSUM`, etc.) over the exact bytes.
2. If an upstream test suite has a captured frame (e.g. `test_daly_bms.py`, `test_jikong_bms.py`), decode at least one frame in Python and confirm the decoded voltage/current/SOC matches the test assertion.
3. If a vendor PDF example has a wrong checksum (like Pylontech V3.3's `E1A2` vs `E27A`), document it under `## Errata and disagreements`.

## 4. Write the Page & Update the Index

1. Create `<BRAND>/README.md` (or `<BRAND>/BLE.md` / `<BRAND>/<VERSION>.md` for distinct protocols) following `TEMPLATE.md`.
2. Focus on the most useful 80% of registers/fields (live telemetry, cells, temps, alarms, key settings) and link the full vendor doc or upstream definition file for the rest.
3. Add or update the brand's row(s) in `README.md` (`| Brand | Product / family | Link | Settings | Framing | Status | Page |`).
4. Check `git status` and `git diff` to ensure no scratch files leaked into the repo and all relative links resolve.

## Priority Areas for Future Enhancement

- **Upgrading `🌐 Community` / `📄 Vendor doc` pages to `✅ Verified`**: Add real hardware frame captures to `<BRAND>/captures/` (with serials/MACs redacted) for `DALY`, `BYD`, `SOLIS`, `GOODWE`, `GROWATT`, `SMA`, `SOK`, `OGT`.
- **Missing brands to add**: `SOLAX` (SolaX X1/X3 Hybrid Modbus & CAN), `SOFAR` (Sofar HYD Modbus & CAN), `FOXESS` (FoxESS H1/H3 Modbus & BMS CAN), `SUNGROW` (SH5K/SH10RT Modbus), `FELICITY` (Felicity Solar BMS RS485), `REC` (REC BMS CAN/RS485), `ORION` (Orion BMS 2 CAN), `BATRIUM` (Batrium WatchMon CAN/UDP).

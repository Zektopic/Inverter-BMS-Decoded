# Contributing

Thank you for helping. The value of this repository is that every number can be trusted, so please follow these rules.

## What we want

- **Captured frames** from real devices (hex dumps or logic-analyser exports) with the device model, firmware version and what
  the device was doing (charging at 10 A, SOC 54 %, …). These turn a page from *Vendor doc* into *Verified*.
- **Corrections** with evidence: a capture, a vendor document, or a reference implementation that disagrees.
- **New brands and models**, using [TEMPLATE.md](TEMPLATE.md).

## How

1. Fork, create a branch (`docs/<brand>-<topic>`), and add or edit the page in the brand's folder (`BRAND/README.md`; use extra
   files such as `BRAND/V3.md` for distinct protocol versions).
2. Put captures in `BRAND/captures/` as text (`.txt`, `.csv`, `.log`) with a short header describing the setup.
3. Link every fact to a source in the page's **Sources** section. Prefer linking a vendor PDF over committing it; if you commit a
   document, keep its original name and say where it came from.
4. Add or update the row in the root [README](README.md) index, with an honest status.
5. Open a pull request describing what changed and how it was checked.

## Style

- English; keep the vendor's original field names in brackets when they help (`SOCStateOfcharge`).
- Tables for field layouts: offset/index, size, type, unit/scale, meaning.
- Hex as `0x1A2B`; frames as space-separated bytes or the exact ASCII string.
- State byte order and checksum algorithm explicitly, with a worked example.
- Mark uncertain fields with *(unverified)* rather than leaving them out.

## Privacy and safety

- Remove serial numbers, MAC addresses, passwords and Bluetooth passcodes from captures (replace with `X`), unless they are needed
  to explain a field; then use an obviously fake value.
- Document control commands (FET switching, parameter writes, calibration, shutdown) clearly as such, with a warning.

## Reporting a problem

Use the issue templates: *Protocol correction* for errors, *New protocol request* for missing devices.

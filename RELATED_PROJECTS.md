# Related Open-Source Projects & Protocol Repositories

This page lists open-source GitHub repositories that decode, document, and include real captured test frames or vendor PDFs for battery management systems (UART, RS485, CAN, and **BLE**) and hybrid inverters. When adding a new brand or checking a field offset, start here.

---

## 1. Bluetooth Low Energy (BLE) Smart BMS & Balancers

| Repository | Languages / Platform | Protocols & Brands Covered | Where the Protocol Details Live |
|---|---|---|---|
| **[patman15/BMS_BLE-HA](https://github.com/patman15/BMS_BLE-HA)** | Python (Home Assistant) | **15+ BLE BMS protocols**: ABC / SOK (`abc_bms.py`), CBT Power / Creabest (`cbtpwr_bms.py`), Daly BLE (`daly_bms.py`), D-Powercore (`dpwrcore_bms.py`), ECO-WORTHY (`ecoworthy_bms.py`), E&J Technology (`ej_bms.py`), JBD / Xiaoxiang (`jbd_bms.py`), JK / Jikong (`jikong_bms.py`), NEEY Balancer (`neey_bms.py`), Offgridtec / Liontron / Topband (`ogt_bms.py`), Pro BMS (`pro_bms.py`), Redodo / LiTime / Power Queen (`redodo_bms.py`), RoyPow (`roypow_bms.py`), Seplos V2 & V3 BLE (`seplos_v2_bms.py`, `seplos_bms.py`), TDT BLE (`tdt_bms.py`) | Decoders in [`custom_components/bms_ble/plugins/`](https://github.com/patman15/BMS_BLE-HA/tree/main/custom_components/bms_ble/plugins); real captured BLE notification frames in [`tests/`](https://github.com/patman15/BMS_BLE-HA/tree/main/tests) |
| **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)** | C++ (ESPHome) | **JK (Jikong)**: UART (`4E 57`), BLE (`55 AA EB 90` for JK04, JK02_24S, JK02_32S), and JK-PB RS485 Modbus | [`components/`](https://github.com/syssi/esphome-jk-bms/tree/main/components), vendor PDFs in [`docs/`](https://github.com/syssi/esphome-jk-bms/tree/main/docs) |
| **[syssi/esphome-jbd-bms](https://github.com/syssi/esphome-jbd-bms)** | C++ (ESPHome) | **JBD / Jiabaida / Xiaoxiang / Overkill Solar** UART & BLE (`DD A5 … 77`) | [`components/jbd_bms`](https://github.com/syssi/esphome-jbd-bms/tree/main/components/jbd_bms), [`components/jbd_bms_ble`](https://github.com/syssi/esphome-jbd-bms/tree/main/components/jbd_bms_ble), [`docs/`](https://github.com/syssi/esphome-jbd-bms/tree/main/docs) |
| **[syssi/esphome-ant-bms](https://github.com/syssi/esphome-ant-bms)** | C++ (ESPHome) | **ANT BMS** Legacy (`AA 55 AA FF`, 140 bytes) and New 2021+ (`7E A1 …` CRC16) over UART & BLE | [`components/`](https://github.com/syssi/esphome-ant-bms/tree/main/components), [`docs/`](https://github.com/syssi/esphome-ant-bms/tree/main/docs) |
| **[syssi/esphome-neey-bms](https://github.com/syssi/esphome-neey-bms)** | C++ (ESPHome) | **NEEY / Heltec / Enerkey** 4A Smart Active Balancer BLE (`55 AA 11` / `AA 55 11`) | [`components/neey_bms_ble`](https://github.com/syssi/esphome-neey-bms/tree/main/components/neey_bms_ble) |
| **[syssi/esphome-seplos-bms](https://github.com/syssi/esphome-seplos-bms)** | C++ (ESPHome) | **Seplos V2** YD/T 1363.3 ASCII (`~20…`) and BLE; also hosts the Pylontech RS485 V3.3 PDF | [`components/`](https://github.com/syssi/esphome-seplos-bms/tree/main/components), [`docs/`](https://github.com/syssi/esphome-seplos-bms/tree/main/docs) |
| **[syssi/esphome-daly-bms](https://github.com/syssi/esphome-daly-bms)** | C++ (ESPHome) | **Daly Smart BMS** UART/RS485 (`A5`, 13-byte frames) and Sinowealth | [`components/`](https://github.com/syssi/esphome-daly-bms/tree/main/components) |
| **[syssi/esphome-pace-bms](https://github.com/syssi/esphome-pace-bms)** & **[nkinnan/esphome-pace-bms](https://github.com/nkinnan/esphome-pace-bms)** | C++ (ESPHome) | **PACE BMS** RS232/RS485 (`~25…` YD/T 1363.3 variant) used in many server-rack batteries | `components/pace_bms` |
| **[fl4p/batmon-ha](https://github.com/fl4p/batmon-ha)** | Python | Multi-BMS Bluetooth monitor supporting JBD, JK, Daly, ANT, Victron SmartShunt BLE, Supervolt, and SOK | [`bmslib/models/`](https://github.com/fl4p/batmon-ha/tree/master/bmslib/models) |
| **[cyrils/renogy-bt](https://github.com/cyrils/renogy-bt)** | Python | **Renogy** BT-1 / BT-2 BLE module & Modbus RTU decoder for Smart Lithium batteries, Rover MPPTs, and inverters | [`renogybt/`](https://github.com/cyrils/renogy-bt/tree/main/renogybt) |

---

## 2. Multi-Brand BMS Drivers & CAN Translators (RS485 / UART / CAN)

| Repository | Languages / Platform | Protocols & Brands Covered | Where the Protocol Details Live |
|---|---|---|---|
| **[mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery)** | Python (Victron Venus OS) | **20+ BMS protocols**: ANT, Daly, Daly Sinowealth, ECS, EG4 Lifepower, EG4-LL, Felicity, HLPdata, JBD/LLT, JK (UART, BLE, PB Modbus), Lifepower, MNB, Pace, Pylontech, Renogy, Revov, Seplos V2/V3, Sinowealth, TianPower/TDT | Drivers in [`dbus-serialbattery/bms/`](https://github.com/mr-manuel/venus-os_dbus-serialbattery/tree/master/etc/dbus-serialbattery/bms); captured test frames in [`tests/`](https://github.com/mr-manuel/venus-os_dbus-serialbattery/tree/master/tests) |
| **[dalathegreat/Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator)** | C++ (ESP32) | **Inverter CAN & RS485 protocols**: BYD CAN (`BYD-CAN.cpp`), BYD Modbus (`BYD-MODBUS.cpp`), Pylontech CAN (`PYLON-CAN.cpp`), Victron CAN, SMA Sunny Island LV CAN & SMA Tripower HV CAN (`SMA-CAN.cpp`, `SMA-TRIPOWER-CAN.cpp`), GoodWe CAN, SolaX CAN, Sofar CAN, Kostal RS485, FoxESS CAN, Growatt CAN/Modbus, Sungrow, Schneider | [`Software/src/inverter/`](https://github.com/dalathegreat/Battery-Emulator/tree/main/Software/src/inverter) and [`Software/src/battery/`](https://github.com/dalathegreat/Battery-Emulator/tree/main/Software/src/battery) |
| **[Sleeper85/esphome-yambms](https://github.com/Sleeper85/esphome-yambms)** | YAML / C++ (ESPHome) | Multi-BMS aggregator translating JK, JBD, Seplos, PACE, and Daly BMSs into **Pylontech CAN**, **GoodWe CAN**, **SMA CAN**, **Victron CAN**, and **Luxpower CAN** | [`packages/`](https://github.com/Sleeper85/esphome-yambms/tree/main/packages) |
| **[Uksa007/esphome-jk-bms-can](https://github.com/Uksa007/esphome-jk-bms-can)** | YAML / C++ (ESPHome) | JK BMS to Pylontech / SMA / Victron CAN bus bridge | Repository root & `docs/` |
| **[ArminJo/JK-BMSToPylontechCAN](https://github.com/ArminJo/JK-BMSToPylontechCAN)** | C++ (Arduino) | JK-B UART to Pylontech CAN bridge (included as a submodule under [`JK/`](JK/)) | `Pylontech_CAN.h`, `JK-BMS.hpp` |
| **[maland16/daly-bms-uart](https://github.com/maland16/daly-bms-uart)** | C++ (Arduino) | Daly Smart BMS `A5` UART protocol library and vendor PDF | `src/`, `docs/` |
| **[phinix-org/Multiple-JK-BMS-by-Modbus-RS485](https://github.com/phinix-org/Multiple-JK-BMS-by-Modbus-RS485)** | Python / Docs | JK-PB Inverter BMS RS485 Modbus V1.0 / V1.1 official vendor specifications | [`docs/`](https://github.com/phinix-org/Multiple-JK-BMS-by-Modbus-RS485/tree/main/docs) |

---

## 3. Inverter RS485 / Modbus / ASCII / TCP Protocol Repositories

| Brand Family | Repositories | Protocols & Details Covered |
|---|---|---|
| **Voltronic / Axpert / MPP Solar / EASUN / PowMr / InfiniSolar** | • **[jblance/mpp-solar](https://github.com/jblance/mpp-solar)**<br>• **[ned-kelly/docker-voltronic-homeassistant](https://github.com/ned-kelly/docker-voltronic-homeassistant)**<br>• **[syssi/esphome-pipsolar](https://github.com/syssi/esphome-pipsolar)** | Full command definitions (`mppsolar/protocols/`) for **PI16, PI17, PI18, PI30, PI30MAX, PI41** (`QPI`, `QPIRI`, `QPIGS`, `QMOD`, `QPIWS`, CRC16-XMODEM), plus Daly/JK/JBD BMS helpers |
| **Deye / Sunsynk / Sol-Ark** | • **[kellerza/sunsynk](https://github.com/kellerza/sunsynk)**<br>• **[StephanJoubert/home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman)** | Single-phase (`definitions/single_phase.py`) and three-phase HV/LV (`definitions/three_phase_hv.py`) Modbus register maps; YAML definitions for Deye, Sofar, Solis, Chint, KStar, and Invt |
| **Growatt (SPF / SPH / SPA / MIN / MOD)** | • **[johanmeijer/grott](https://github.com/johanmeijer/grott)**<br>• **[syssi/esphome-growatt](https://github.com/syssi/esphome-growatt)**<br>• **[OpenInverterGateway/OpenInverterGateway](https://github.com/OpenInverterGateway/OpenInverterGateway)** | Growatt Modbus RTU & ShineWiFi record layouts (protocol versions v1.05, v1.20, v1.24, and SPF Off-Grid) |
| **Luxpower / EG4 (18kPV, 6000XP, 12000XP, LXP-LB, SNA)** | • **[celsworth/lxp-bridge](https://github.com/celsworth/lxp-bridge)**<br>• **[guybw/LuxPython_DEV](https://github.com/guybw/LuxPython_DEV)** | Luxpower `A1 1A` TCP dongle framing and full Modbus input/holding register maps |
| **Huawei SUN2000 & LUNA2000** | • **[wlcrs/huawei-solar-lib](https://github.com/wlcrs/huawei-solar-lib)** | Complete SUN2000 Modbus RTU/TCP register definitions (`src/huawei_solar/registers.py`) for inverters, LUNA2000 storage, and DTSU666-H meters |
| **GoodWe (ET / EH / ES / EM / BT)** | • **[marcelblijleven/goodwe](https://github.com/marcelblijleven/goodwe)** | GoodWe `AA 55` UDP protocol and Modbus RTU/TCP register maps (`et.py`, `es.py`, `dt.py`) |
| **Ginlong Solis / SolaX / Sofar / FoxESS** | • **[fboundy/ha_solis_modbus](https://github.com/fboundy/ha_solis_modbus)**<br>• **[wills106/homeassistant-solax-modbus](https://github.com/wills106/homeassistant-solax-modbus)**<br>• **[hultenvp/solis-sensor](https://github.com/hultenvp/solis-sensor)** | Solis Hybrid (`33000+` input, `43000+` holding) and multi-brand Modbus register definitions covering Solis, SolaX, Sofar, Growatt, and FoxESS |
| **Fronius & BYD Battery-Box** | • **[Efflon/byd_battery_box](https://github.com/Efflon/byd_battery_box)**<br>• **[redpomodoro/byd_bat](https://github.com/redpomodoro/byd_bat)** | BYD Battery-Box Premium HVS/HVM BMU/BMS Modbus TCP & CAN module telemetry |

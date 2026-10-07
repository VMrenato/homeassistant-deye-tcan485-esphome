# Deye Hybrid Inverter + LilyGO T-CAN485 — ESPHome Modbus RTU

Monitor a **Deye SUN-SG05LP1-EU** hybrid inverter locally from Home Assistant using a **LilyGO T-CAN485** board talking Modbus RTU over RS485 — no cloud, no stick logger, ~5 second updates. This config was hardened over several days of testing against a real **Deye SUN-6K-SG05LP1-EU-AM2-P** and documents every pitfall found along the way (especially the T-CAN485 enable pins, which the official LilyGO docs get wrong for the current board revision).

## Why

Deye's newer WiBLE plug-and-play loggers **block local access entirely** (port 8899 is closed), pushing you into the Solarman cloud. The wired RS485 path on the inverter itself is always available, logger-free, and fully local: PV, battery, grid, load, energy totals and temperatures straight into Home Assistant over your LAN, with fast data freshness and no third-party servers involved.

## Features

- **Three-tier polling** ~5 s for fast-changing values (PV/battery/grid/load power, SOC, grid-connected), ~30 s for slow-changing ones (voltages, currents, temperatures, daily/total energy), ~120 s for the read-only settings mirror — 35 live-data Modbus sensors + 6 template sensors + 34 read-only settings entities
- **Battery health** — cycle count (BMS) and battery status (charge/discharge/idle state)
- **Load voltage monitoring** — house voltage from register 157
- **Read-only settings mirror** — a third controller (120 s) shows the inverter's current configuration in Home Assistant: device info / firmware (regs 0–19), battery shutdown/restart/low SOC, grid-charge current and enable, and the full Time-of-Use program (6 slots: start time, power, SOC, charge flags)
- **Database-friendly publishing** — power sensors poll every 5 s but only publish on a ≥ 25 W change or once a minute
- **Optional, hardware-verified write controls** via [`esphome/deye-inversor-write.yaml`](esphome/deye-inversor-write.yaml): Time-of-Use slot 5 target SOC and slot 5 grid-charge flag — range-clamped, function 0x10, opt-in only, shipped as a separate file
- **Grid-connected binary sensor** (register 194, relay state) — detect grid loss / ATS transfer to backup and trigger automations
- **WS2812 activity LED** on the board: blue flash = TX (request), green flash = RX (response) — instant visual confirmation the bus is alive
- **Home Assistant auto-discovery** via the native ESPHome API — every entity appears with proper device/state classes, ready for the Energy dashboard
- **No flow control pin needed** — the T-CAN485's MAX13487E transceiver is AutoDirection

## Tested hardware

| Component | Details |
|---|---|
| Inverter | **Deye SUN-6K-SG05LP1-EU-AM2-P** (single-phase hybrid, 6 kW, 48 V battery, 2 MPPT) |
| Board | **LilyGO T-CAN485** (ESP32 WROOM-32, MAX13487E RS485 transceiver, WS2812 RGB LED, CAN bus) — 2024 1.1 revision |
| Cable | Standard Ethernet patch cable, cut — only 3 wires used |

Should also work with the whole **SUN-3.6/5/6/7/8/10K-SG05LP1-EU** family (same register map). See [docs/hardware.md](docs/hardware.md) for details.

<!-- PHOTO: inverter wiring compartment with the RS485/MODBUS port highlighted -->
<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/1caf2e5f-b1a7-4151-a5d9-4cbf86ff63a3" />


## Wiring

Use the inverter's **RS485 / MODBUS port** inside the wiring compartment — **not** the BMS port and **not** the Meter/CT port. A cut Ethernet patch cable (T568B) works perfectly:

| RJ45 pin (T568B) | Wire color | Signal | Inverter terminal |
|---|---|---|---|
| 1 | White-orange | RS485 **B** | B |
| 2 | Orange | RS485 **A** | A |
| 3 | White-green | **GND** | GND (optional, recommended) |

The inverter is a Modbus RTU slave at **address 1, 9600 8N1**. Full details in [docs/hardware.md](docs/hardware.md).

<!-- DIAGRAM: T-CAN485 A/B/GND -> inverter RS485 port terminals -->

## Quick start

1. **Hardware** — flash nothing yet. Wire A → A, B → B, GND → GND between the T-CAN485 screw terminals and the inverter's RS485/MODBUS port (table above). Power off the inverter's AC and DC before opening the wiring compartment.
2. **Wiring check** — with everything powered, measure DC bias between A and B with a multimeter; you should see a small measurable voltage once the enable pins are driven (see the callout below).
3. **ESPHome** — copy [`esphome/deye-inversor.yaml`](esphome/deye-inversor.yaml) into your ESPHome dashboard (or `esphome run`), keeping the `!secret wifi_ssid` / `!secret wifi_password` references in your `secrets.yaml`. Flash over USB the first time.
4. **Adopt in Home Assistant** — the device is discovered automatically via the ESPHome integration; all `Deye *` entities appear within a minute. Watch the board LED: blue/green flicker every 5 s means the bus is working.

> [!IMPORTANT]
> **Enable pins — the #1 gotcha.** The official LilyGO README says RS485 EN is **GPIO9**. That is **wrong for the 2026 board revision** and leaves the bus completely dead. What actually works, validated on hardware:
>
> | GPIO | Function | State |
> |---|---|---|
> | **GPIO19** | RS485 EN | HIGH (active high) |
> | **GPIO17** | RS485 SE | HIGH (active high) |
> | **GPIO16** | 5V booster enable | HIGH (active high) |
>
> The YAML implements them as `gpio` switches with `restore_mode: ALWAYS_ON`, so they are driven automatically at boot. No `flow_control_pin` is needed — the MAX13487E handles direction itself.

## Register map

The config polls 33 live-data Modbus registers (holding registers, slave 1): PV power/voltage/production, battery SOC/power/voltage/current/temperature/charge/discharge/cycle-count, grid power/CT/voltage/frequency/import/export, load power/voltage/consumption, inverter power and DC/AC temperatures — plus a grid-connected binary sensor on register 194, and a read-only mirror of the configuration registers (0–20, 43, 217–219, 230, 232, 248–279).

Deye 32-bit energy totals are **low-word-first**: the YAML reads the lo/hi words separately and combines them in template sensors, e.g. total production = `(lo + hi × 65536) × 0.1` kWh.

Full table with addresses, types and scaling: **[docs/registers.md](docs/registers.md)**.

## Troubleshooting

The short version of a multi-day debugging journey:

- **Total silence on the bus** → enable pins wrong or not driven (the GPIO9-vs-GPIO19 trap above). Verify A-B bias with a multimeter.
- **Values landing on the wrong sensors (shifted by one)** → command queue overlap: too many commands per cycle for a slow slave. Fixed by relying on block reads (the controller batches consecutive registers) and an interval that fits the cycle: 19 commands ≈ 4–7 s at 9600 baud, so 5 s polling works.
- **Truncated frames / mid-frame byte loss** → cheap generic 5 V MAX485 modules. Avoid them; use the T-CAN485 (or a 3.3 V-native MAX3485).
- **Boot loop / rollback after OTA** → don't touch or reset the device for ~90 s after an OTA flash (`safe_mode` marks the boot successful after 60 s).
- **Grid frequency 0 / grid voltage off by 10×** → grid voltage is register **150** at ×0.1 (2377 = 237.7 V); don't use register 152.
- **Registers don't match the document** → you're using Deye's three-phase address list; the SG05LP1 needs the single-phase V1.19 map.
- **A write does nothing** → don't build raw frames in a `write_lambda`; use a range-limited `number` with function 0x10 (`use_write_multiple: true`).

Upgrading from ESPHome < 2026.9.0 and slow sensors stop updating / config fails to compile → per-sensor skip_updates was removed in 2026.9.0. This config now uses a second modbus_controller (deye_slow, same address, 30 s interval, same command_throttle) for the slow-changing sensors instead. If you forked this repo before that change, pull the latest esphome/deye-inversor.yaml.

## Writeable registers (opt-in, ⚠️ use at your own risk)

A separate file [`esphome/deye-inversor-write.yaml`](esphome/deye-inversor-write.yaml) adds write controls. They are **NOT included** in the main config — add them explicitly:

```yaml
packages:
  deye_read: !include esphome/deye-inversor.yaml
  deye_write: !include esphome/deye-inversor-write.yaml
```

| Control | Register | Notes |
|---------|----------|-------|
| Deye TOU Slot 5 SOC | 272 | 20–90 %, clamped |
| Deye TOU Slot 5 Grid Charge | 278 bit0 | Template switch; writes only 4 ↔ 5 and refuses if the register holds anything else |

Both were **verified on a SUN-6K-SG05LP1-EU-AM2-P** — Modbus read-back and the inverter LCD. Copy the pattern for other slots (SOC = 267 + *N*, flags = 273 + *N*).

How the writes are made safe:

1. **Read first** — the main config mirrors every Time-of-Use register read-only, so you see the current values before you change anything.
2. **Function 0x10** (`use_write_multiple: true`) — what Deye expects for settings.
3. **Range-limited `number` entities** — out-of-range values are rejected by ESPHome before they reach the bus.
4. **No raw `write_lambda` frames** — they are sent as custom frames, not register writes.
5. **No on/off control** — registers 43 and 20 read values that don't match the documentation, so they are mirrored read-only only.

> [!WARNING]
> Earlier versions of the write file used registers 130/131/135/138 (export limit, charge/discharge current, grid charge). Those addresses are **not in the official V1.19 map** for this family and were removed. The documented addresses (210, 211, 232, 243, 245, …) are listed in [docs/registers.md](docs/registers.md#writeable-registers) but have not been tested on hardware.

Full details: **[docs/troubleshooting.md](docs/troubleshooting.md)**.

## Contributing

Issues and PRs welcome — especially confirmations or register-map notes for other Deye/Sunsynk models (SUN-*K-SG05LP1-EU family, SG04LP3 three-phase, Sunsynk rebadges). If you validate this on a different model or board revision, please report back.

## Credits

- [StephanJoubert/home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman) — `deye_hybrid.yaml` register definitions
- [slipx06/Sunsynk-Home-Assistant-Dash](https://github.com/slipx06/Sunsynk-Home-Assistant-Dash) — ESPHome-1P config this register map was cross-checked against
- **"Saentist"** — T-CAN485 enable-pins config that cracked the GPIO19/17/16 puzzle
- [Xinyuan-LilyGO/T-CAN485](https://github.com/Xinyuan-LilyGO/T-CAN485) — board documentation
- [ESPHome](https://esphome.io/) — the `modbus_controller` platform doing all the heavy lifting

## License

MIT — see [LICENSE](LICENSE).

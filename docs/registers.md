# Register map

All registers are **holding registers** (function code 0x03) on Modbus RTU slave **address 1**, 9600 8N1. Addresses below are decimal, exactly as used in [`esphome/deye-inversor.yaml`](../esphome/deye-inversor.yaml).

Sources: the register definitions were built from StephanJoubert's [home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman) `deye_hybrid.yaml` profile and cross-checked against slipx06's [Sunsynk-Home-Assistant-Dash](https://github.com/slipx06/Sunsynk-Home-Assistant-Dash) ESPHome-1P config, then validated against a live SUN-6K-SG05LP1-EU-AM2-P.

`skip_updates` in the tables is the ESPHome throttle (publish only every Nth polled value) used to reduce HA database churn on slow-changing values.

## Solar

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye PV1 Power | 186 | U_WORD | ×1 | W | — |
| Deye PV2 Power | 187 | U_WORD | ×1 | W | — |
| Deye PV1 Voltage | 109 | U_WORD | ×0.1 | V | 2 |
| Deye PV2 Voltage | 111 | U_WORD | ×0.1 | V | 2 |
| Deye Daily Production | 108 | U_WORD | ×0.1 | kWh | 5 |
| *(total_production_lo, internal)* | 96 | U_WORD | low word | — | 5 |
| *(total_production_hi, internal)* | 97 | U_WORD | high word | — | 5 |
| **Deye Total Production** (template) | 96–97 | — | `(lo + hi × 65536) × 0.1` | kWh | — |
| **Deye PV Total Power** (template) | — | — | `PV1 + PV2` | W | — |

## Battery

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Battery SOC | 184 | U_WORD | ×1 | % | — |
| Deye Battery Power | 190 | S_WORD | ×1 (signed; discharge/charge by sign) | W | — |
| Deye Battery Voltage | 183 | U_WORD | ×0.01 | V | 2 |
| Deye Battery Current | 191 | S_WORD | ×0.01 (signed) | A | 2 |
| Deye Battery Temperature | 182 | U_WORD | ×0.1 − 100 | °C | 5 |
| Deye Daily Battery Charge | 70 | U_WORD | ×0.1 | kWh | 5 |
| Deye Daily Battery Discharge | 71 | U_WORD | ×0.1 | kWh | 5 |
| *(total_battery_charge_lo, internal)* | 72 | U_WORD | low word | — | 5 |
| *(total_battery_charge_hi, internal)* | 73 | U_WORD | high word | — | 5 |
| **Deye Total Battery Charge** (template) | 72–73 | — | `(lo + hi × 65536) × 0.1` | kWh | — |
| *(total_battery_discharge_lo, internal)* | 74 | U_WORD | low word | — | 5 |
| *(total_battery_discharge_hi, internal)* | 75 | U_WORD | high word | — | 5 |
| **Deye Total Battery Discharge** (template) | 74–75 | — | `(lo + hi × 65536) × 0.1` | kWh | — |

## Grid

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Grid Power | 169 | S_WORD | ×1 (signed) | W | — |
| Deye Grid CT Power | 172 | S_WORD | ×1 (signed, external CT/meter) | W | — |
| Deye Grid Voltage L1 | 150 | U_WORD | ×0.1 (2377 = 237.7 V) | V | 2 |
| Deye Grid Frequency | 79 | U_WORD | ×0.01 (5002 = 50.02 Hz) | Hz | 5 |
| Deye Daily Energy Bought | 76 | U_WORD | ×0.1 | kWh | 5 |
| Deye Daily Energy Sold | 77 | U_WORD | ×0.1 | kWh | 5 |
| Deye Total Grid Import | 78 | U_WORD | ×0.1 | kWh | 5 |
| Deye Total Grid Export | 81 | U_WORD | ×0.1 | kWh | 5 |

> **Note:** grid voltage is register **150** at ×0.1. Register 152 is *not* the right source — using it gives a 0 Hz frequency / wrongly scaled voltage. See [troubleshooting](troubleshooting.md).

## Load

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Load Power | 178 | U_WORD | ×1 | W | — |
| Deye Daily Load Consumption | 84 | U_WORD | ×0.1 | kWh | 5 |
| *(total_load_consumption_lo, internal)* | 85 | U_WORD | low word | — | 5 |
| *(total_load_consumption_hi, internal)* | 86 | U_WORD | high word | — | 5 |
| **Deye Total Load Consumption** (template) | 85–86 | — | `(lo + hi × 65536) × 0.1` | kWh | — |

## Inverter

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Inverter Power | 175 | S_WORD | ×1 (signed) | W | — |
| Deye DC Temperature | 90 | U_WORD | ×0.1 − 100 | °C | 5 |
| Deye AC Temperature | 91 | U_WORD | ×0.1 − 100 | °C | 5 |

## Binary sensor

| Entity | Address | Notes |
|---|---|---|
| Deye Grid Connected | 194 | 1 = grid present, 0 = grid lost (ATS/backup detection for automations) |

## 32-bit energy totals: low-word-first

Deye stores cumulative energy counters as 32-bit values in two consecutive 16-bit registers, **low word first**. The YAML reads each word as a separate internal `U_WORD` sensor and combines them in a template sensor:

```yaml
lambda: "return (id(total_production_lo).state + id(total_production_hi).state * 65536.0) * 0.1;"
```

This applies to registers 96–97 (total production), 72–73 (total battery charge), 74–75 (total battery discharge) and 85–86 (total load consumption). Total grid import (78) and export (81) fit in a single word on this model and are read directly.

## Polling profile

- `update_interval: 5s` — one full read cycle every 5 s
- `command_throttle: 50ms` — gap between consecutive Modbus commands
- `offline_skip_updates: 3` — declare the inverter offline after 3 failed cycles
- Consecutive registers are automatically batched into block reads by `modbus_controller`; a full cycle is ~19 commands and takes roughly 4–7 s at 9600 baud, which is why 5 s polling is the sweet spot.

# Register map

All registers are **holding registers** (function code 0x03) on Modbus RTU slave **address 1**, 9600 8N1. Addresses below are decimal, exactly as used in [`esphome/deye-inversor.yaml`](../esphome/deye-inversor.yaml).

> **Source:** Official Deye document *Modbus RTU Protocol — Energy Storage / String / Micro-inverter, Ningbo Deye, V1.19 (20220223)* — received directly from Deye technical support. Register definitions, units, scaling factors and R/W flags are taken directly from this document.

`skip_updates` in the tables is the historical publish throttle. Since ESPHome 2026.9.0 it is implemented differently: values marked with a `skip_updates` number are polled by the slow controller (`deye_slow`, 30 s); the others by the fast one (`deye`, 5 s). Instantaneous power sensors additionally use a `throttle: 60s` / `delta: 25` publish filter (see [Polling profile](#polling-profile)).

## Solar

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|--------|---------|------|-----------------|------|-------------|
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
|--------|---------|------|-----------------|------|-------------|
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
|--------|---------|------|-----------------|------|-------------|
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
|--------|---------|------|-----------------|------|-------------|
| Deye Load Power | 178 | U_WORD | ×1 | W | — |
| Deye Load Voltage L1 | 157 | U_WORD | ×0.1 | V | 2 |
| Deye Daily Load Consumption | 84 | U_WORD | ×0.1 | kWh | 5 |
| *(total_load_consumption_lo, internal)* | 85 | U_WORD | low word | — | 5 |
| *(total_load_consumption_hi, internal)* | 86 | U_WORD | high word | — | 5 |
| **Deye Total Load Consumption** (template) | 85–86 | — | `(lo + hi × 65536) × 0.1` | kWh | — |

## Inverter

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|--------|---------|------|-----------------|------|-------------|
| Deye Inverter Power | 175 | S_WORD | ×1 (signed) | W | — |
| Deye DC Temperature | 90 | U_WORD | ×0.1 − 100 | °C | 5 |
| Deye AC Temperature | 91 | U_WORD | ×0.1 − 100 | °C | 5 |

## Battery health (read-only)

| Sensor | Address | Type | Scale / filter | Unit | Notes |
|--------|---------|------|-----------------|------|-------|
| Deye Battery Status | 185 | U_WORD | — | — | 0=idle; 1=charging; 2=discharging (inferred from integration state) |
| Deye Battery Cycle Count | 611 | U_WORD | ×1 | cycles | Battery cycle count (Pack 1 BMS) |

## Binary sensor

| Entity | Address | Description |
|--------|---------|-------------|
| Deye Grid Relay State | 194 | Grid relay contactor state: `1` = closed (grid present); `2` = open (grid lost / ATS in backup mode). Useful for automations detecting off-grid transition. |

## 32-bit energy totals: low-word-first

Deye stores cumulative energy counters as 32-bit values in two consecutive 16-bit registers, **low word first**. The YAML reads each word as a separate internal `U_WORD` sensor and combines them in a template sensor:

```yaml
lambda: "return (id(total_production_lo).state + id(total_production_hi).state * 65536.0) * 0.1;"
```

This applies to registers 96–97 (total production), 72–73 (total battery charge), 74–75 (total battery discharge) and 85–86 (total load consumption). Total grid import (78) and export (81) fit in a single word on this model and are read directly.

## Settings mirror (read-only, `deye_cfg`)

A third controller polls the configuration registers every **120 s**, read-only. All entities are `diagnostic`. Use them to see the inverter's current settings before writing anything.

### Device info (registers 0–19)

Read as one 20-register block into the text sensor **Deye Cfg Info raw 0-19** (hex), plus **Deye Cfg Device type** (0).

| Address | Content (V1.19) | Observed on SUN-6K-SG05LP1-EU-AM2-P |
|---------|-----------------|--------------------------------------|
| 0 | Device type — documented as `0x0300` = single-phase LV hybrid | Reads **`0x0003`** (bytes swapped vs. the document) |
| 1 | Modbus address | `1` |
| 2 | Protocol version | `0x0201` — the V1.19 map still matches |
| 3–7 | Serial number, ASCII | 10 characters packed **2 per register** in regs 3–7 (the document lists one byte per register, 3–12) |
| 11, 13 | — | Main firmware version, shown on the LCD as `XXXX-YYYY` (reg 11 = first half, reg 13 = second half, in hex) |
| 14 | — | HMI firmware version (hex) |
| 16–17 | Rated power, 0.1 W, low word first | `0xEA60` = 60 000 → 6 000 W |
| 18 | MPPTs and phases | `0x0201` = 2 MPPT, 1 phase |

### Control registers (read-only here)

| Sensor | Address | Meaning (V1.19) | Note |
|--------|---------|-----------------|------|
| Deye Cfg Remote lock | 20 | `0` unlocked, `2` locked | ⚠️ Reads **255** on the tested unit while running — meaning unverified |
| Deye Cfg Inverter enabled | 43 | `1` on, `0` off | ⚠️ Reads **0** on the tested unit while it is running — meaning unverified. Do **not** build an on/off control on it |

### Battery thresholds and grid charge

| Sensor | Address | Unit | Meaning |
|--------|---------|------|---------|
| Deye Cfg Battery shutdown SOC | 217 | % | Battery cut-off |
| Deye Cfg Battery restart SOC | 218 | % | Recovery point after a cut-off |
| Deye Cfg Battery low SOC | 219 | % | Low-battery warning |
| Deye Cfg Grid charge current | 230 | A | Grid → battery charge current |
| Deye Cfg Grid charge enabled | 232 | — | Global grid-charge enable |

### Time-of-Use program (registers 248–279)

| Sensor | Address | Meaning |
|--------|---------|---------|
| Deye Cfg Use timer | 248 | Bit0 = TOU enabled; bits 1–7 = Mon…Sun. `255` = enabled every day |
| Deye Cfg Prog*N* time | 249 + *N* (250–255) | Slot start time, **HHMM** as a decimal number (`100` = 01:00, `1700` = 17:00). Each slot runs until the next one starts |
| Deye Cfg Prog*N* power | 255 + *N* (256–261) | Max battery discharge power in the slot, W |
| Deye Cfg Prog*N* SOC | 267 + *N* (268–273) | Target / floor SOC for the slot, % |
| Deye Cfg Prog*N* flags | 273 + *N* (274–279) | Bit0 = charge from grid, Bit1 = charge from generator, Bit2–4 = mode bits. On the tested unit every slot reads **4** (bit2 only); enabling grid charge for a slot sets it to **5** |

Voltage targets per slot (262–267) are not read: they only matter when the battery is configured by voltage instead of SOC.

## Polling profile

Three controllers share the bus (same slave address, same 50 ms `command_throttle`):

| Controller | Interval | What |
|------------|----------|------|
| `deye` | 5 s | Instantaneous power, SOC, battery status, grid relay |
| `deye_slow` | 30 s | Voltages, currents, temperatures, daily/total energy, cycle count |
| `deye_cfg` | 120 s | Read-only settings mirror (above) |

- `offline_skip_updates: 3` — declare the inverter offline after 3 failed cycles.
- Consecutive registers are automatically batched into block reads by `modbus_controller`; the fast cycle is ~19 commands and takes roughly 4–7 s at 9600 baud, which is why 5 s polling is the sweet spot.
- Power sensors (PV1/PV2/PV total, battery, grid, grid CT, load, inverter) publish on a change of **≥ 25 W** or at least **once every 60 s** (`or: [throttle: 60s, delta: 25]`). Polling stays at 5 s, so changes still arrive immediately, but steady values stop flooding the Home Assistant database.
- Template sensors that combine filtered sensors read `.raw_state` (e.g. PV total = `pv1.raw_state + pv2.raw_state`); `.state` would add two already-throttled values and lag behind.

## Writeable registers

> **Opt-in only.** Not part of the main read-only config. See [`esphome/deye-inversor-write.yaml`](../esphome/deye-inversor-write.yaml).

### Shipped — verified on hardware

| Control | Address | Type | Range | Description |
|---------|---------|------|-------|-------------|
| Deye TOU Slot 5 SOC | 272 | U_WORD, function 0x10 | 20–90 % (clamped) | Target SOC for Time-of-Use slot 5 |
| Deye TOU Slot 5 Grid Charge | 278 bit0 | template switch → U_WORD, function 0x10 | writes only `4` ↔ `5` | Allow grid charging in slot 5. Refuses to act if the register currently holds anything other than 4 or 5 |

Both were confirmed by Modbus read-back and on the inverter LCD. The same pattern works for any slot *N*: SOC = 267 + *N*, flags = 273 + *N*.

### Documented in V1.19 but not shipped

The earlier version of the write file used registers 130/131/135/138 from community sources. **Those addresses are not in the official V1.19 map for this inverter family** and have been removed. The documented addresses are:

| Setting | Address | Unit / range (V1.19) |
|---------|---------|----------------------|
| Max charge current | 210 | 1 A, 0–185 |
| Max discharge current | 211 | 1 A, 0–185 |
| Grid charge enable (global) | 232 | — |
| Energy management mode | 243 | 0 = battery first, 1 = load first |
| Limit control | 244 | 0 = selling, 1 = built-in CT, 2 = external meter |
| Export power limit | 245 | 1 W |
| Solar sell | 247 | 0 = off, 1 = on |
| Switch on/off | 43 | See the warning above — reads don't match the document |

None of these have been written on hardware. If you add one, follow the rules in the write file: read first, function 0x10, range-limited `number`, never a raw `write_lambda`.

> ⚠️ **Always read the current register value before writing**, and note it so you can restore it.

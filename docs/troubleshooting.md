# Troubleshooting

Distilled from several days of bring-up against a live Deye SUN-6K-SG05LP1-EU-AM2-P. Symptoms first, then cause and fix.

## 1. Total silence on the bus (no responses at all)

**Symptom:** ESPHome logs show TX requests but zero RX; every sensor stays `unavailable`/`unknown`. The board LED flashes blue (TX) but never green (RX).

**Cause:** the T-CAN485 enable pins are wrong or not driven. The official LilyGO README says RS485 EN = **GPIO9** — that is **wrong for the 2026 board revision**. Without the enables, the MAX13487E never drives the bus.

**Fix:**

- Drive **GPIO19 (RS485 EN), GPIO17 (RS485 SE) and GPIO16 (5V booster) HIGH** — all active HIGH. The YAML does this with `gpio` switches and `restore_mode: ALWAYS_ON`.
- Verify electrically: with a multimeter on DC between A and B you should measure a small bias voltage once the enables are on. No bias = transceiver not enabled.
- Double-check A/B are not swapped and you are on the **RS485/MODBUS port** — not BMS, not Meter/CT.

## 2. Values landing on the wrong sensors (shifted by one)

**Symptom:** responses arrive, but values appear under the wrong entities — e.g. battery SOC showing what looks like PV power, everything offset by one register.

**Cause:** command queue overlap. Too many Modbus commands per cycle for a slow slave: the controller's queue and the inverter's responses drift out of sync, and a response gets matched to the next queued command.

**Fix:**

- Let the controller use **block reads** — `modbus_controller` automatically batches consecutive registers into a single request, drastically cutting the command count.
- Pick an interval that fits the cycle: the full poll is ~19 commands taking **≈ 4–7 s at 9600 baud**, so `update_interval: 5s` works; anything faster reintroduces the overlap.
- Keep `command_throttle: 50ms` so the slave gets breathing room between commands.

## 3. Truncated frames / mid-frame byte loss

**Symptom:** intermittent CRC errors, responses that stop mid-frame, or sporadic single-byte loss — bus mostly works but corrupts regularly.

**Cause:** cheap generic **MAX485 modules (5 V)**. They are not reliably compatible with the ESP32's 3.3 V logic and tend to mangle frames.

**Fix:** don't use them. Use the **T-CAN485** (integrated MAX13487E, AutoDirection) or a **MAX3485** (3.3 V native) if building from discrete parts.

## 4. Boot loop / rollback right after an OTA flash

**Symptom:** device reboots into the previous firmware after an OTA update.

**Cause:** ESPHome `safe_mode` only marks a boot as successful after **60 s** of uptime. Power-cycling or resetting within that window triggers a rollback.

**Fix:** after an OTA flash, **don't touch or reset the device for ~90 s**. Let it sit, confirm it stays up, then walk away.

## 5. Grid frequency reads 0 / grid voltage at the wrong scale

**Symptom:** grid frequency stuck at 0 Hz and/or grid voltage off by a factor of 10 (or nonsense like 23 V).

**Cause:** wrong register / wrong scale. Grid voltage lives at register **150** with a **×0.1** scale — raw 2377 means 237.7 V. Register 152 is *not* the correct source. Grid frequency is register **79** at ×0.01 (5002 = 50.02 Hz).

**Fix:** use 150 (×0.1) for grid voltage and 79 (×0.01) for frequency, as in the YAML — don't use 152.

## Quick diagnostic checklist

1. Are `RS485 EN`, `RS485 SE`, `Booster 5V EN` switches all ON?
2. Multimeter: measurable DC bias between A and B?
3. LED: blue (TX) *and* green (RX) flicker every 5 s?
4. Correct inverter port (RS485/MODBUS, not BMS/Meter)? A on pin 2 (orange), B on pin 1 (white-orange)?
5. ESPHome logs: `uart debug` in the YAML prints both directions — compare TX requests against RX frames.

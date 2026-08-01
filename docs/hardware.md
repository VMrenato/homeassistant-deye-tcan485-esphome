# Hardware

## Inverter

**Deye SUN-6K-SG05LP1-EU-AM2-P** — single-phase hybrid, 6 kW, 48 V battery, 2 MPPT trackers.

This config should work unchanged with the whole single-phase low-voltage family, which shares the same register map:

- SUN-3.6K-SG05LP1-EU
- SUN-5K-SG05LP1-EU
- SUN-6K-SG05LP1-EU (tested)
- SUN-7K/8K/10K-SG05LP1-EU

The inverter exposes a **Modbus RTU slave at address 1, 9600 baud, 8N1** on its RS485 port.

## Board

**LilyGO T-CAN485** (2026 revision):

- ESP32 classic (WROOM-32)
- Integrated **MAX13487E** RS485 transceiver with **AutoDirection** — no DE/RE flow control required from software
- WS2812 RGB LED (GPIO4) — used here as a bus activity indicator
- CAN bus transceiver (unused in this project)
- Screw terminals for RS485 A/B and power

### Enable pins (critical)

The RS485 section of the board is gated by three GPIOs that **must be driven HIGH** or the bus is electrically dead. The official LilyGO README documents RS485 EN as GPIO9 — **this is wrong for the 2026 revision**. The validated mapping:

| GPIO | Function | Required state |
|---|---|---|
| GPIO19 | RS485 EN | HIGH (active high) |
| GPIO17 | RS485 SE | HIGH (active high) |
| GPIO16 | 5V booster enable | HIGH (active high) |

The ESPHome config drives them via `gpio` switches with `restore_mode: ALWAYS_ON`, so they come up automatically at every boot and can also be toggled from Home Assistant for debugging.

### UART

| Signal | GPIO |
|---|---|
| TX | GPIO22 |
| RX | GPIO21 |

9600 baud, 8 data bits, no parity, 1 stop bit. **No `flow_control_pin`** — the MAX13487E handles direction automatically.

## Wiring

Use the inverter's port labeled **RS485** or **MODBUS** inside the wiring compartment.

- **NOT the BMS port** (that's CAN/RS485 for the battery BMS)
- **NOT the Meter/CT port**

A standard Ethernet patch cable, cut, is all you need — only 3 wires are used:

| RJ45 pin (T568B) | Wire color | Signal | Connects to |
|---|---|---|---|
| 1 | White-orange | RS485 B | Inverter terminal B |
| 2 | Orange | RS485 A | Inverter terminal A |
| 3 | White-green | GND | Inverter GND (optional but recommended) |

> **Safety:** the wiring compartment contains mains-voltage terminals. Disconnect AC (grid + backup) and DC (PV + battery) and wait for the inverter to fully discharge before opening it.

### Verifying the connection

With the inverter on and the board flashed:

1. The three enable switches (`RS485 EN`, `RS485 SE`, `Booster 5V EN`) must be ON.
2. Measure DC voltage between A and B with a multimeter — a small bias (typically a few hundred mV to a couple of volts, fluctuating) should be measurable. Zero volts means the transceiver is not enabled.
3. The board LED flickers **blue on TX, green on RX** every 5 s polling cycle.

## Notes on alternatives

Cheap generic **MAX485 modules (5 V)** caused truncated frames and mid-frame byte loss in testing — avoid them. If you don't use a T-CAN485, use a **MAX3485** (3.3 V native) or another AutoDirection 3.3 V transceiver.

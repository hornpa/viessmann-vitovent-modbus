# Viessmann Vitovent 300-W (H-series) — Modbus RTU Integration Guide

**Reverse-engineered Modbus register map for the Viessmann Vitovent 300-W type H32S/H32E heat-recovery ventilation unit — because the Viessmann OEM firmware does *not* speak the documented Brink register set.**

> Verified on: **Vitovent 300-W, Typ H32E C325** (enthalpy variant, order no. Z026527), August 2026, controlled via a Loxone Modbus Extension and a USR-TCP232-410S Modbus-TCP gateway. Read the [disclaimer](#disclaimer) before touching your unit.

---

## TL;DR

- The Vitovent 300-W H-series is an **OEM-rebranded Brink Flair** (H32E C325 ≙ Flair 325 enthalpy) built around Brink's **UWA2-B control board**.
- It has a native **Modbus RTU (RS485)** port on connector **X15** — the same port used by the LB1 wall control and the Vitoconnect V.
- **But:** the Viessmann firmware does **not** implement Brink's published external Modbus register list (Brink doc *614882 "Modbus UWA2-B/UWA2-E"*). All Brink input registers (4000-range) and remote-control registers (8000-range semantics) return exceptions or behave differently.
- A full 0–65535 register scan revealed a **custom register layout**. The essentials:
  - **Write holding register `2002` = target ventilation stage** (`1`…`4` = the four configured flow presets, `5` = intensive boost). The unit follows within ~10 s. Live-verified.
  - Live values (stage, actual airflows, fan RPM, temperatures, humidities, operating hours) live in the **1000/2000 ranges**.
  - Value **`32768` (0x8000) means "not available"**.
- Bus parameters: **19200 baud, 8 data bits, even parity, 1 stop bit, slave address 70** (Brink's default would be 20).
- **Signal GND is mandatory.** X15 pin 3 must be connected to your master's GND, or the bus stays silent.

---

## Repository contents

| File | What it is |
|---|---|
| `README.md` | This document — background, wiring, and the full register map. |
| [`loxone-template-MB_Vitovent_300-W.xml`](loxone-template-MB_Vitovent_300-W.xml) | Ready-to-import Loxone Config Modbus device template (stage actuator + all mapped sensors). |
| [`registers-raw.jsonl`](registers-raw.jsonl) | Raw scan data: every existing register (FC03/FC04, addresses 0–65535) with the value read at scan time. One JSON object per line. |

## Table of contents

1. [Background: a Brink Flair in disguise](#1-background-a-brink-flair-in-disguise)
2. [Physical connection](#2-physical-connection)
3. [What does NOT work](#3-what-does-not-work-brink-standard-vs-viessmann-firmware)
4. [The actual register map](#4-the-actual-register-map)
5. [Quick start (pymodbus)](#5-quick-start-pymodbus)
6. [Reproducing the scan](#6-reproducing-the-scan)
7. [Findings from sniffing the LB1 traffic](#7-findings-from-sniffing-the-lb1--unit-traffic)
8. [Open questions](#8-open-questions--contributions-welcome)

---

## 1. Background: a Brink Flair in disguise

Viessmann's central HRV units of the current generation, **Vitovent 300-W type H32S / H32E** (225/300/325/400 m³/h variants), are OEM versions of the **Brink Flair** series (Brink Climate Systems, NL). Inside sits Brink's **UWA2-B controller PCB** — connector names (X12…X19), DIP switches, everything matches the Brink service documentation.

For the genuine Brink Flair there is excellent third-party Modbus support (e.g. [fonske/Brink-flair-modbus](https://github.com/fonske/Brink-flair-modbus) for ESPHome), based on Brink's official document *614882 — Modbus UWA2-B/UWA2-E installation regulations*, which defines:

- input registers 4000–4544 (live data, FC04),
- holding registers 6000–7992 (settings, FC03/06),
- remote-control registers 8000–8011 (stage/flow override, FC06).

**None of this fully applies to the Viessmann firmware.** Viessmann uses the Modbus port for its own ecosystem (LB1 wall control, Vitocal heat-pump link, Vitoconnect V gateway) and ships a firmware whose external register map differs substantially. That is why every "connect your Vitovent like a Brink" attempt fails with `Illegal data address` exceptions — and why this document exists.

## 2. Physical connection

### X15 connector (on the UWA2-B board, accessible in the unit's connection panel)

The red 3-pole plug-in terminal. Counted from the side of the black X16 (24 V) connector:

| Pin | Function |
|---|---|
| 1 | **Modbus − (B)** |
| 2 | **Modbus + (A)** |
| 3 | **Signal GND — connect it!** |

Sources: Viessmann community ([X15 pinout thread](https://community.viessmann.de/t5/Lueftung/Vitovent-300-W-H32S-Pinbelegung-Stecker-X15-Modbus/td-p/178031)) + own verification.

Practical notes:

- **GND is not optional.** The LB1 wall control gets its ground reference through the X16 24 V loom; a third-party master connected with only A/B will typically get *no response at all* because the transceiver's common-mode reference floats. Run a third wire from X15 pin 3 to your RS485 master's GND.
- **X12** is the plug-in jumper for the 120 Ω bus termination on the unit side — leave it in place.
- Swapping A/B is harmless (no response → swap once). RS485 "A/B" naming differs between vendors anyway.
- **Only one Modbus master.** LB1, Vitocal link, Vitoconnect V and your building automation are mutually exclusive on X15. Unplug the LB1 before connecting your master (keep it — replugging it restores normal operation, so it is your fallback).
- With no master present the unit keeps ventilating autonomously at its last stage; safety functions (frost protection, bypass automation) stay internal. Removing the LB1 does not stop the unit.

### Serial parameters

| Parameter | Value |
|---|---|
| Baud rate | **19200** |
| Data bits | 8 |
| Parity | **Even** |
| Stop bits | 1 |
| Slave address | **70** (0x46) — Viessmann OEM default; genuine Brink units default to 20 |

There is no accessible on-device menu to change these (the Brink "menu 14" exists only on units with a display; the LB1's service menu exposes no communication parameters — verified against the LB1 service manual 5791 603).

## 3. What does NOT work (Brink standard vs. Viessmann firmware)

Observed behaviour of the Viessmann firmware (H32E C325, 2026) when addressed with Brink's documented registers:

| Brink register range | Result on Viessmann firmware |
|---|---|
| 4000–4544 input registers (FC04) | **Exception 02** (illegal data address) — the whole range is absent |
| 7990–7992 (Modbus interface config) | **Exception 02** — absent |
| 8000–8011 remote control | Registers 8000–8005 *exist* but with different semantics; writing Brink values (e.g. `8000=1`, `8001=0..3`) returns **Exception 03** (illegal data value). Reads return status/counter words (e.g. `8000 = 0x8000`). |
| 6000–6003 flow presets | ✅ readable and matching the configured presets — the only range that happens to line up with Brink |

Interesting detail: the firmware answers **FC03 and FC04 identically** for every existing register.

## 4. The actual register map

Method: full address-space scan (0–65535, FC03 + FC04, single-register reads), followed by a non-destructive writability probe (writing each register's current value back via FC06) and live actuation tests. 245 registers exist; raw scan data is included in [`registers-raw.jsonl`](registers-raw.jsonl).

### 4.1 Control — live-verified ✅

| Register | Access | Function | Values |
|---|---|---|---|
| **2002** | read/**write** (FC06) | **Target ventilation stage** | `1`…`4` = the four flow presets (factory/commissioned values, see 6000–6003; on the test unit 50/165/235/325 m³/h). `5` = intensive boost (runs at max flow, presumably with the configured intensive-run timer). Writes take effect within ~10 s; the airflow ramps to the preset of the selected stage. |
| **6006** | read/**write** (FC06) | **Bypass mode override** | `0` = automatic (default). `1` = force flap to position 524 — by all indications **open** (supply-air humidity/temperature immediately shift towards outdoor air). `2` = force flap to position 0 — **closed** (identical to the automatic state on a hot day). The flap moves within ~30 s. Watch the feedback registers below. Note this is the *opposite* value order of Brink's 6100 register. |

Bypass feedback (read-only): **1128** = flap step position (`0` = closed, `524` = open on the test unit), **1045** = flap status word (low byte `4` = closed, `5` = open; high byte 0x64 when idle).

> Persistence of 2002/6006 across power loss is untested — send them cyclically (e.g. every 60 s) from your master if you rely on a specific state, and prefer leaving 6006 at `0` (automatic) so the unit's internal bypass logic keeps working.

### 4.2 Live values (read-only)

| Register | Function | Unit / scaling |
|---|---|---|
| 1009 | Ventilation stage, actual | 1–5 |
| **1010** | Supply air flow, actual | m³/h |
| 1011 | Supply fan speed | rpm |
| **1013** | Extract air flow, actual | m³/h |
| 1014 | Extract fan speed | rpm |
| 1020 | Temperature sensor 1 | 0.1 °C (signed) |
| 1023 | Temperature sensor 2 | 0.1 °C (signed) |
| 1025 | Temperature sensor 3 | 0.1 °C (signed) |
| 1031 | Humidity sensor 1 | % r.H. |
| 1032 | Humidity sensor 2 | % r.H. |
| 2000 | Operating hours | h |
| 2001 | Presumably filter-related counter (value 356 on test unit) — unconfirmed | |
| 2003 | Status word (value 1) — unconfirmed | |

The exact mapping of the three temperature sensors (outdoor / supply / extract / exhaust) is best confirmed on your own unit by watching which one tracks the outdoor temperature over a day. On the test unit (summer): 25.6 / 27.1 / 28.2 °C.

**`32768` (0x8000) = "value not available"** (sensor not fitted / function inactive) — the classic Viessmann N/A marker. Whole sub-ranges (1101–1116, 2010–2014, 2030–2035, …) read as 32768 on a unit without the optional CO₂/humidity accessories.

### 4.3 Settings (read; write with care)

| Register | Function | Unit / scaling |
|---|---|---|
| 6000–6003 | Flow presets stage 1–4 | m³/h (commissioned airflow values — think twice before writing these) |
| 6004 | Bypass threshold, extract-air side | 0.1 °C (matches LB1 parameter C108) |
| 6005 | Bypass hysteresis | 0.1 K |
| 6031 | Bypass threshold, outdoor side | 0.1 °C |
| 6014–6021 | Paired 400/1200 values — presumably preheater/frost parameters, unmapped | |
| others in 6000–6127 | writable settings block, largely unmapped | |

### 4.4 Existing but unresolved — do not write

- **8000–8005**: status/counter words (`8000=0x8000`, `8001` slowly counting, `8004/8005=0x2000`). Semantics unknown.
- **8888**: exists, reads 0. Suspiciously magic. Leave it alone.
- **2020** (=6), **1000–1007**, **1040–1047**, **1117–1142**, **2100–2106**, **6501–6509**: various diagnostics/internals, unmapped. PRs welcome.

### 4.5 Full register inventory

FC03 ≡ FC04, slave 70. Existing ranges:

```
1000-1007, 1009-1017, 1020-1033, 1040-1047, 1101-1134, 1138-1142,
2000-2003, 2010-2014, 2020, 2030-2035, 2100-2106,
6000-6127, 6501-6509, 8000-8005, 8888
```

Raw values from the scan: [`registers-raw.jsonl`](registers-raw.jsonl) (one JSON object per line: function code, address, value at scan time).

## 5. Quick start (pymodbus)

```python
from pymodbus.client import ModbusTcpClient   # or ModbusSerialClient for direct RS485

# via a Modbus-TCP<->RTU gateway (e.g. USR-TCP232-410S, PUSR, Elfin EW11, ...)
c = ModbusTcpClient("192.168.x.x", port=502, timeout=4)
c.connect()

SLAVE = 70

# read actual stage + airflows
r = c.read_holding_registers(1009, count=6, device_id=SLAVE).registers
print(f"stage={r[0]}  supply={r[1]} m3/h  rpm={r[2]}  extract={r[4]} m3/h  rpm={r[5]}")

# set stage 3
c.write_register(2002, 3, device_id=SLAVE)
```

Integration hints:

- **Loxone Modbus Extension**: works out of the box (19200/8/E/1, device address 70). A **ready-made Loxone device template** is included in this repository: [`loxone-template-MB_Vitovent_300-W.xml`](loxone-template-MB_Vitovent_300-W.xml). Drop it into your Loxone Config Modbus template folder (`Documents\Loxone\Templates\Modbus`) or import it via the device-template function, create a device from it on your Modbus Extension, and you get the stage actuator (2002, FC06, with a 60 s cyclic resend as power-loss re-arm) plus all mapped sensors (÷10 corrections for temperatures already configured). Wire a 1–4 slider/logic onto the "Lüftungsstufe Soll" actuator and you are done.
- **Home Assistant** (`modbus:` integration) / **ESPHome** (`modbus_controller`): straightforward with the table above; note that the existing Brink Flair ESPHome projects will *not* work unchanged — replace their register definitions with this map.
- Demand-based control: the unit ships without CO₂/humidity sensors unless ordered — feed your room sensors into your automation and drive register 2002. That is the whole point.

## 6. Reproducing the scan

1. Disconnect the LB1 from X15, connect an RS485 master (A/B/GND!).
2. Scan FC03 and FC04, addresses 0–65535, single-register reads, and log everything that does not return exception 02 (~40 min at 19200 baud).
3. Probe writability by writing each existing register's own current value back (a no-op that reveals FC06 acceptance).
4. Identify actuators by correlation: change something observable (stage via 2002) and watch which registers follow.
5. For ground truth, sniff the LB1: put a Modbus-TCP gateway into *transparent* (raw socket) mode so it only listens, attach the LB1 **in parallel** on the same A/B/GND bus (RS485 is multidrop; the passive gateway causes no collisions), operate the panel, and decode the captured RTU frames (address, FC, register, value, CRC). Every menu change shows up as an FC06 write.

## 7. Findings from sniffing the LB1 ↔ unit traffic

To resolve the remaining unknowns we put the gateway into transparent mode (passive listener on the RS485 bus), reattached the LB1 wall control in parallel, and captured its complete Modbus traffic while operating every menu item. Key results:

**Stage control confirmed on the wire.** Selecting "Intensiv" on the LB1 produces exactly `FC06 write 2002 = 4`; the party/boost function is nothing more than a stage write either. There are no hidden operating-mode commands.

**LB1 configuration parameters that DO map to unit registers:**

| LB1 parameter | Unit register | Notes |
|---|---|---|
| C1A0 bypass mode | 6006 | see §4.1 |
| C108 bypass extract-air threshold | 6004 | 0.1 °C |
| C109–C10C stage flow presets | 6000–6003 | m³/h |
| C1A1 central heating + HRV enable | **6007** | confirmed by sniffing |
| C101 preheater enable | 6008 | value-match hypothesis |
| C105 humidity sensor enable | **6011** | confirmed by sniffing |
| C106 CO₂ sensor enable | **6013** | confirmed by sniffing |

**LB1 parameters that do NOT exist in the unit:** the timer/schedule parameters (7D83 continuous mode, 7D84 eco duration, 7D85 intensive duration, 7781/868x DST settings) generate **no bus traffic at all** — the LB1 stores them locally and only ever sends stage commands. If you replace the LB1 with your own Modbus master, you re-implement scheduling there (which is the point).

**The filter counter lives in the LB1, not in the unit.** The LB1's 365-day filter countdown is panel-local. Its "reset filter warning" menu action writes `2003 = 2` to the unit (an acknowledge/notify; the LB1 also writes `2003 = 1` when it connects, and the register reads `0` with no panel attached) — but the unit-side register 2001 does not change. Consequently there is **no unit-side filter countdown to reset via Modbus**: implement the 365-day countdown in your building automation (and optionally write `2003 = 2` on your reset action for good measure). Register **2001** (read-only, counting; 356 on the test unit) most likely tracks calendar days since commissioning — verify on your unit. **6032** (=365) is presumably the interval setting the LB1 reads.

**LB1 session registers:** on connect the LB1 initialises 8002/8003/8004 (and 8888=0) with timestamp/handshake-looking values and mirrors its N/A slots (2010–2014, 2030–2035 = 32768). These are panel-session registers — irrelevant and best left alone for third-party control.

## 8. Open questions / contributions welcome

- Temperature sensor assignment (1020/1023/1025) per air path.
- Exact meaning of 2001 (days since commissioning?), 2020 (=6, matches the LB1's intensive-duration hours — coincidence?), 6009/6012, the 8000–8005 words and 8888.
- Behaviour with Vitocal/Vitoconnect attached, and on other H-series variants (H32S, C400) and firmware revisions.

Open an issue/PR if you can confirm or extend the map.

## Disclaimer

This is an **unofficial, community reverse-engineering effort**. It is not affiliated with, endorsed by, or supported by Viessmann or Brink Climate Systems. Register semantics were derived from black-box observation of **one** unit (H32E C325, 2026 firmware) and may differ on other variants or firmware versions.

You interact with your ventilation unit **at your own risk**. Writing to undocumented registers can misconfigure or damage the device and void your warranty. Keep the LB1 control unit as a fallback, never run two Modbus masters on the bus, and leave commissioned airflow presets (6000–6003) alone unless you know what you are doing.

## Contributing

Confirmations and extensions are very welcome — especially from other H-series variants (H32S, C400), other firmware revisions, and setups with a Vitocal heat pump or Vitoconnect V attached. Open an issue with your unit type, firmware version and observations, or a PR extending the register map. The most valuable contributions right now are the temperature-sensor assignment and anything that pins down the meaning of registers 2001, 2020 and the 8000-range.

## Acknowledgements

- The [fonske/Brink-flair-modbus](https://github.com/fonske/Brink-flair-modbus) project and Brink's *614882* document, which provided the starting hypotheses (even though the Viessmann firmware turned out to differ).
- The Viessmann community [X15 pinout thread](https://community.viessmann.de/t5/Lueftung/Vitovent-300-W-H32S-Pinbelegung-Stecker-X15-Modbus/td-p/178031).

## License

Documentation and data in this repository: **CC BY 4.0**. Code snippets and the Loxone template: **MIT**.

---

*Reverse-engineered and documented in August 2026. Unit: Vitovent 300-W H32E C325. Method: full FC03/FC04 register scan (0–65535) + non-destructive writability probe + live actuation tests + passive RTU sniffing of the LB1 wall control. Contributions welcome.*

# OBD-II Adapter

<p align="center">
  <img src="../images/charge.png" width="240" alt="The Charge screen">
</p>

A Bluetooth OBD-II adapter allows Numen to read the vehicle directly rather than through Bluelink. It is available only while the iPhone is near the vehicle, but its readings are live. CarPlay navigation, live charging data and trip energy depend on it.

All decoding in this document was developed and verified on an IONIQ 6 AWD. Other E-GMP vehicles share most of the battery data; body module PIDs may differ.

## Hardware Requirements

The adapter must be a **Bluetooth Low Energy** ELM327-compatible device. Classic Bluetooth (SPP) adapters cannot communicate with iOS.

Numen scans for the following services:

| Service | Typical Hardware |
|---|---|
| `FFF0`, `FFE0`, `FFE5`, `18F0` | Generic BLE ELM327 adapters |
| `FEE0` (characteristic `FEE1`) | WiCAN |
| `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` | Nordic UART |

The adapter should enter sleep when the vehicle is off. The OBD-II port remains powered, and an adapter that does not sleep will drain the 12 V battery.

## Initialisation

On connection, the adapter is configured with:

```
ATZ      reset
ATE0     echo off
ATL0     linefeeds off
ATS0     spaces off
ATH0     headers off; each reply line is the ISO-TP frame only
ATSP6    ISO 15765-4 CAN, 11-bit, 500 kbps
```

Each UDS request sets the request header. For modules that reply outside the `7E8`–`7EF` range, the receive address and flow control are also set:

```
ATSH<header>        e.g. ATSH770
ATCRA<reply>        e.g. ATCRA778
ATFCSH<header>
ATFCSD300000        flow control: clear to send, no block limit, no separation time
ATFCSM1             use the flow control frame above
```

## Polling

| Condition | Interval |
|---|---|
| Connected, vehicle off | 60 s |
| Ignition on | 20 s |
| Charge session screen open | 6 s |
| Battery preconditioning starting or stopping | 3 s, for up to 2 minutes |

Readings are written to the shared App Group for use by CarPlay, widgets and Live Activities.

## PIDs

| Request | Header → Reply | Module | Data |
|---|---|---|---|
| `220101` | `7E4` → `7EC` | BMS | Pack voltage and current, charging state, plug flags, lifetime energy counters, raw SOC |
| `220105` | `7E4` → `7EC` | BMS | Display SOC, remaining energy, state of health |
| `220100` | `7B3` → `7BB` | HVAC | Cabin temperature |
| `22BC03` | `770` → `778` | Body (IGMP) | Doors, trunk, hood, ignition, seat belts, rear door locks |
| `22BC04` | `770` → `778` | Body (IGMP) | Front door locks |

Byte offsets below are counted from the first byte after the `62 xx xx` response header, so `d[0]` is the first data byte.

### Battery

| Value | Location | Scaling |
|---|---|---|
| Display SOC | `220105` d[31] | × 0.5, floored to a whole percent |
| Raw SOC | `220101` d[4] | × 0.5. Reads 1.5–2 points above the instrument cluster; used only as a fallback. |
| Pack current | `220101` K, L | Signed, ÷ 10 A. Negative while charging. |
| Pack voltage | `220101` M, N | ÷ 10 V |
| Remaining energy | `220105` d[28..29] | × 2 Wh |
| State of health | `220105` d[25..26] | ÷ 10 % |

Display SOC is reported in half-percent steps, while the vehicle displays the floored value. Numen floors it to match the vehicle.

Charge power is calculated as pack voltage × pack current, which is valid for both AC and DC charging.

### Charging Detection

Pack current is negative whenever energy flows into the battery, including during regenerative braking; currents of −150 A have been observed while driving. The following signals are therefore insufficient on their own:

| Signal | Limitation |
|---|---|
| Plug flags (`220101` byte 9) | Reported false throughout a 107 kW DC session |
| Rising SOC | SOC is floored to whole percent, so consecutive readings at a 6 s interval are often identical |
| Pack current | Indistinguishable from regenerative braking |

The reliable signal is **`220101` d[5..8]**: the charge limits the BMS publishes once a charging session is negotiated. These bytes read `00 00 00 00` at all other times, including during regenerative braking. Across a logged DC session they distinguished charging from not charging in 14 of 14 readings, including three regenerative braking events.

The session logic is:

1. A session starts when current flows into the pack and is corroborated by a signal regenerative braking cannot produce: the charge limit bytes, then a plug flag, then an increase in remaining energy since the previous reading.
2. The session is held while energy continues to flow in, rather than re-evaluated on each reading.
3. The session ends when the charge limit bytes clear, or after two consecutive readings with no inbound current.

Remaining energy is used for the increase check instead of SOC, because it changes on nearly every reading (for example, 23,246 → 23,468 → 23,628 Wh over three readings at 107 kW).

### Regenerative Braking per Trip

Remaining energy and SOC are both net of regenerative braking. The BMS also maintains two lifetime counters: energy charged into the pack and energy discharged from it. While driving, only regenerative braking increases the charged counter, so its increase over a trip is that trip's recovered energy.

The counters are read from `220101` d[42..45] (charged) and d[46..49] (discharged), ÷ 10 kWh. This mapping originates from earlier Hyundai BMS documentation and is guarded by two checks:

- A reading is accepted only if both values are between 10 and 300,000 kWh and within a 0.5–2.0 ratio of each other.
- If the documented offset fails, the payload is scanned for a valid pair and the offset found is logged.

**Not yet verified.** The offsets have not been confirmed against a known value on an E-GMP vehicle.

### Cabin Temperature

`220100`: `(d[5] ÷ 2) − 40` °C. Bluelink does not report cabin temperature; its climate status includes only the set point.

### Doors and Locks

From `22BC03` bytes 4 and 5, and `22BC04` byte 4. On left-hand-drive vehicles, the driver side is the left side.

| Bit | Meaning |
|---|---|
| `22BC03` b4 `0x80` | Trunk open |
| `22BC03` b4 `0x20` | Driver door open |
| `22BC03` b4 `0x10` | Passenger door open |
| `22BC03` b4 `0x04` | Driver-side rear door open |
| `22BC03` b4 `0x01` | Passenger-side rear door open |
| `22BC03` b4 `0x08` | Driver-side rear door **unlocked** |
| `22BC03` b4 `0x02` | Passenger-side rear door **unlocked** |
| `22BC03` b5 `0x01` | Hood open |
| `22BC03` b5 `0x02` | Driver seat belt fastened |
| `22BC03` b5 `0x04` | Passenger seat belt fastened |
| `22BC03` b5 `0x60` | Ignition on (either bit) |
| `22BC04` b4 `0x08` | Driver door **unlocked** |
| `22BC04` b4 `0x04` | Passenger door **unlocked** |

Lock bits are set when the door is **unlocked**, the opposite of the adjacent open bits.

- One door reported unlocked is sufficient to report the vehicle unlocked. Reporting it locked requires all four doors. If `22BC04` does not respond, Bluelink's lock state is used.
- The body module stops responding when the vehicle sleeps. A missing response is treated as unknown, not as off.

The body module also guards walk-away locking. Before an automatic lock, Numen checks for an open door or the ignition being on. If either is found, it posts a **Waiting to Auto-Lock** notification and re-checks every 3 seconds for up to one minute; tapping the notification locks immediately. If the adapter cannot be reached, the lock proceeds, since blocking on a missing reading would disable walk-away locking whenever the adapter is out of range.

## Diagnostic Tools

The **OBD Console** (Settings › Developer) sends raw commands and probes every header listed above. It is the recommended first step when testing a new adapter or vehicle.

Debug builds include a **charge simulator** that supplies synthetic readings for testing the charging screens.

# OBD-II Adapter

A Bluetooth OBD-II adapter lets Numen read the car directly instead of asking Bluelink. It only works when your phone is near the car, but the numbers are live. CarPlay navigation, live charging and trip energy all depend on it.

Everything here was decoded and tested on an IONIQ 6 AWD. Other E-GMP cars share most of the battery side. The body module PIDs may differ.

## Hardware

Any **Bluetooth Low Energy** ELM327-compatible adapter. Classic Bluetooth (SPP) adapters can't talk to an iPhone.

The app scans for these services:

| Service | Usually Found On |
|---|---|
| `FFF0`, `FFE0`, `FFE5`, `18F0` | Generic BLE ELM327 clones |
| `FEE0` (characteristic `FEE1`) | WiCAN |
| `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` | Nordic UART |

Pick an adapter that sleeps when the car is off. The OBD-II port stays powered, and an adapter that never sleeps drains the 12 V battery.

## Connection Setup

After connecting, the app sends:

```
ATZ      reset
ATE0     echo off
ATL0     linefeeds off
ATS0     spaces off
ATH0     headers off, so each reply line is just the ISO-TP frame
ATSP6    ISO 15765-4 CAN, 11-bit, 500 kbps
```

For each UDS request it sets the header and, for modules that reply outside the usual `7E8`–`7EF` window, the receive address and flow control:

```
ATSH<header>        e.g. ATSH770
ATCRA<reply>        e.g. ATCRA778
ATFCSH<header>
ATFCSD300000        flow control: clear to send, no block limit, no delay
ATFCSM1             use the flow control frame above
```

## Polling

| When | Every |
|---|---|
| Connected, car off | 60 s |
| Ignition on | 20 s |
| Charge session screen open | 6 s |
| Battery preconditioning starting or stopping | 3 s, for up to 2 minutes |

Readings go into the shared App Group, so CarPlay, widgets and Live Activities use them too.

## PIDs

| Request | Header → Reply | Module | What It Gives |
|---|---|---|---|
| `220101` | `7E4` → `7EC` | BMS | Pack voltage and current, charging state, plug flags, lifetime energy counters, raw SOC |
| `220105` | `7E4` → `7EC` | BMS | Display SOC, remaining energy, state of health |
| `220100` | `7B3` → `7BB` | HVAC | Cabin temperature |
| `22BC03` | `770` → `778` | Body (IGMP) | Doors, trunk, hood, ignition, seat belts, rear locks |
| `22BC04` | `770` → `778` | Body (IGMP) | Front locks |

Byte numbers below count from the first byte after the `62 xx xx` echo, so `d[0]` is the first data byte. Most battery offsets come from the [OBDb](https://github.com/OBDb) IONIQ 6 signal set. The body module layout comes from [OVMS](https://github.com/openvehicles/Open-Vehicle-Monitoring-System-3).

### Battery (`220101`, `220105`)

| Value | Where | Scale |
|---|---|---|
| Display SOC | `220105` d[31] | × 0.5, floored to a whole percent |
| Raw SOC | `220101` d[4] | × 0.5. Reads 1.5–2 points above the dash, so it's only a fallback. |
| Pack current | `220101` K, L | signed, ÷ 10 A. Negative while charging. |
| Pack voltage | `220101` M, N | ÷ 10 V |
| Remaining energy | `220105` d[28..29] | × 2 Wh |
| State of health | `220105` d[25..26] | ÷ 10 % |

**Why display SOC is floored.** The PID reports in half-percent steps (45.5%), but the dash shows the floored value. Rounding would put Numen 1% above the car.

**Charge power** is pack voltage × pack current. That works on a Level 2 AC charger as well as on DC fast. An earlier version read a DC-only value and said "not charging" on AC.

### Is It Charging?

This was harder than it looks. Pack current goes negative whenever energy flows into the battery, and that includes regen. I've seen −150 A while driving.

What doesn't work on its own:

- **Plug flags** (`220101` byte 9, bits for AC plug, CCS plug and HV charging). They read false for an entire 107 kW DC session.
- **SOC going up.** SOC is floored to whole percent, so at a 6 s poll two reads often land on the same number. It flickered.
- **Current alone.** Regen looks the same.

What does work: **`220101` d[5..8]**. These are the charge limits the BMS publishes once a charging session has been negotiated. They read `00 00 00 00` at every other time, including during regen. Over one logged DC session they split charging from not charging 14 out of 14 times, including three regen events.

So the app:

1. Starts a session when current is flowing in and something regen can't fake agrees: the limit bytes first, then a plug flag, then remaining energy going up since the last read.
2. Holds the session while energy keeps flowing in, rather than re-deciding every read. Re-deciding is what made the CarPlay panel flicker.
3. Ends it the moment the limit bytes clear (the charger disconnected), or after two reads in a row with no inbound current.

Remaining energy is used instead of SOC for the "going up" check. It moves on nearly every read (23,246 → 23,468 → 23,628 Wh over three reads at 107 kW).

### Regen per Trip

Remaining energy and SOC are both net of regen, so they can't show it. The BMS keeps two lifetime counters: energy charged into the pack and energy taken out. While driving, the only thing that adds to "charged" is regen, so the rise across a drive is that drive's regen.

The community Torque mapping puts them at `220101` d[42..45] (charged) and d[46..49] (discharged), ÷ 10 kWh. That mapping was documented for older Hyundai BMSs. Two checks keep it safe on E-GMP:

- A reading only counts if it looks like a pair of lifetime counters: between 10 and 300,000 kWh, with the two within a 0.5–2.0 ratio of each other.
- If the documented offset fails, the app scans the payload for a pair that passes, and logs where it found it.

**Unconfirmed.** The offsets haven't been checked against a known value on an E-GMP car.

### Cabin Temperature (`220100`)

`(d[5] ÷ 2) − 40` °C. Bluelink never reports this. Its climate status only has the set point.

### Doors and Locks

From `22BC03` byte 4 and 5, and `22BC04` byte 4. Driver is left on this car.

| Bit | Meaning |
|---|---|
| `22BC03` b4 `0x80` | Trunk open |
| `22BC03` b4 `0x20` | Driver door open |
| `22BC03` b4 `0x10` | Passenger door open |
| `22BC03` b4 `0x04` | Driver rear door open |
| `22BC03` b4 `0x01` | Passenger rear door open |
| `22BC03` b4 `0x08` | Driver rear **unlocked** |
| `22BC03` b4 `0x02` | Passenger rear **unlocked** |
| `22BC03` b5 `0x01` | Hood open |
| `22BC03` b5 `0x02` | Driver belt fastened |
| `22BC03` b5 `0x04` | Passenger belt fastened |
| `22BC03` b5 `0x60` | Ignition on (either bit) |
| `22BC04` b4 `0x08` | Driver front **unlocked** |
| `22BC04` b4 `0x04` | Passenger front **unlocked** |

The lock bits are set when the door is **unlocked**, the opposite of the open bits right next to them.

Two rules keep this honest:

- One door known unlocked is enough to say "unlocked". Saying "locked" needs all four. If `22BC04` doesn't answer, the app falls back to Bluelink.
- When the car sleeps the body module stops answering. No answer means "don't know", never "off".

The body module also guards walk-away locking. Before an automatic lock, the app checks for an open door or the ignition on. If something's open, it sends a **Waiting to Auto-Lock** notification and checks again every 3 seconds for up to a minute (reaching into the back seat and then shutting the door is the usual case). Tapping the notification locks straight away. If the adapter can't be reached, the lock goes ahead. Blocking on a missing reading would stop walk-away locking every time the adapter is out of range, which is most of the time.

## Debug Tools

The **OBD Console** (Settings › Developer) sends raw commands and runs a probe that checks every header above answers. It's the first thing to try with a new adapter or a different car.

The **charge simulator** (debug builds only) feeds fake readings so the charging screens can be tested without plugging in.

## Credits

The first version of the BMS reads was ported from KiaMaps' IONIQ 6 implementation.

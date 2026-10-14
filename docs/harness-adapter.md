# Harness Adapter

<p align="center">
  <img src="../images/card-hardware.png" width="720" alt="The OBD-II adapter and the harness adapter">
</p>

The harness adapter is a WiCAN on the car's multimedia CAN bus (M-CAN), wired in behind the head unit with the [Ioniq5_CAN](https://github.com/tylerharvey/Ioniq5_CAN) harness kit. It runs the Numen firmware: the `feature/myIONIQ` branch of [my fork](https://github.com/JacobPantuso/wicant-i-precondition), which builds on [wicant-i-precondition](https://github.com/L1Z3/wicant-i-precondition) and [WiCAN](https://github.com/meatpiHQ/wican-fw).

It does three things the OBD-II port can't:

- Start and stop **battery preconditioning**.
- Put CarPlay directions on the **instrument cluster and HUD**.
- **Sniff** the multimedia bus, for decoding.

The app talks to it over the same Bluetooth connection an OBD app uses, through the WiCAN's ELM327 emulation. The firmware adds vendor `AT` commands for each feature. A stock adapter answers `?` to them, which is how the app tells the difference without guessing.

## Why Two Adapters

The battery answers on the OBD-II port. The cluster and HUD take their navigation frames from the head unit's bus, which only the harness can reach. So reading the car needs one, and writing to the cluster needs the other. The app holds both Bluetooth connections at once.

## Battery Preconditioning (`ATXPC`)

| Command | Does | Reply |
|---|---|---|
| `ATXPC` | Status | `XPC:<state>,<car>,<secs>,<tmin>,<tmax>,<flags>` |
| `ATXPC1` | Start (no-op if running) | `OK` |
| `ATXPC0` | Stop (no-op if idle) | `OK` |
| `ATXPC2` | Toggle, same as the harness button | `OK` |

Status fields:

- `state`: 0 idle, 1 requested, 2 starting, 3 active, 4 managed by the car (repeating mode), 5 stopping
- `car`: what the car's own status frame says. 0 unknown, 1 idle, 2 starting, 3 started
- `secs`: countdown to the next retry, 0 when nothing's pending
- `tmin`, `tmax`: lowest and highest pack temperature in °C, valid when flag bit 3 is set
- `flags`: bit 0 car in READY, bit 1 status frame seen, bit 2 button held, bit 3 pack temperature valid

`OK` only means the request was queued. The car takes tens of seconds to acknowledge a start, so the app polls `ATXPC` every 3 seconds while a start or stop is in flight.

### Timing

The firmware retries a start every 10 seconds, 4 times. If the car hasn't accepted by then, it's refused. So a refusal takes about 50 seconds to show up. A start that's accepted but never reaches "started" times out after 70 seconds.

### Who Decides

The firmware has no temperature thresholds. The car does. It turns a start down when the pack is already warm, or when SOC is too low. The firmware just reports what happened:

- Retries run out and the status never leaves idle: **the car refused**. The app only says "already at optimal temperature" when SOC is well above the floor, since the two can't be told apart otherwise.
- Active, then idle on its own: **the car stopped it**. The pack temperature at that moment is the car's target, measured.

### Auto-Precondition on a DC-Fast Route

When a CarPlay route ends at a DC fast charger, the app decides whether and when to start preconditioning. It predicts the pack temperature on arrival and skips the start when it wouldn't help:

- The charger is slow enough that its own limit matters more than pack temperature.
- Arrival SOC is high enough that the charge would taper anyway.
- You're too close to arrive with any real heat delivered.
- You turned it down recently.

It sends the start once per charger and forgets the charger when you arrive or cancel.

### Don't Reset During a Start

The precondition poll is a bare `ATXPC`, never a full refresh. A refresh sends `ATZ` and `ATSP6`, which make the WiCAN restart its CAN controller, right when the firmware is placing its start burst on the bus. At the normal 60 s cadence that restart is harmless.

## Cluster and HUD Directions (`ATXNV`, `ATXNT`)

The head unit keeps sending "no guidance" navigation frames while CarPlay is navigating. The firmware rewrites those on the way to the cluster, and sends the multi-frame text and lane messages the head unit leaves silent under CarPlay.

**The firmware knows no frame layouts.** The app sends the bytes for each frame plus a mask of which bytes to replace. Decoding more of the cluster is an app update, not a reflash. The layouts are in [Cluster and HUD](cluster-and-hud.md).

| Command | Does | Reply |
|---|---|---|
| `ATXNV` | Status | `XNV:<active>,<updates>,<injected>` |
| `ATXNV1<frames>` | Set the frames to rewrite. 1–5 × 22 hex digits, each `<id:4><mask:2><d0..d7:16>` | `OK` |
| `ATXNV0` | Stop, and forget every multi-frame message | `OK` |
| `ATXNT` | Status | `XNT:<messages>,<frames>` |
| `ATXNT<id:4><hex>` | Set the ISO-TP message for that CAN ID and service, up to 96 bytes | `OK` |

### Limits

- `ATXNV1` can only rewrite `4CC`, `63E`, `640`, `641`, `4E8` and `4EA`. The hook sees every frame headed for the cluster side, powertrain included, so a bad payload can't be allowed to touch anything else.
- `ATXNT` can only originate `F0` (next street), `F2` (HUD maneuvers) and `F4` (destination) on `6E7` and `6DF`, and the `F1` lane graphic on `680` and `681`. Frames go out 5 ms apart with `AA` padding and no flow control, the same way the head unit sends them. Each message is only re-sent when it changes.
- Injection stops **4 seconds** after the last `ATXNV1`, and immediately when Bluetooth drops, so a lost connection can't leave a frozen arrow. When it stops, every message the firmware drew is replaced with the blank form.
- Preconditioning wins. Its countdown owns `4E8` and `4CC` while a start or stop is in flight.

### Flicker

On the single-bus WiCAN, the firmware can't stop the head unit's frame. It sends a corrected copy straight after it, so an arrow can flicker now and then. A dual-bus WiCAN in bridge mode would replace the frame outright with the same code.

## CAN Sniffer (`ATXSN`)

Receive only. Nothing is transmitted.

| Command | Does |
|---|---|
| `ATXSN` | Status: `XSN:<on>,<npins>,<rx>,<sent>,<dropped>,<suppressed>,<ids>` |
| `ATXSN1` | Start, sending each ID only when its payload changes |
| `ATXSN1 63E,4CC,…` | Same, plus every frame of the listed IDs |
| `ATXSN1P 63E,4CC,…` | Only the listed IDs |
| `ATXSN0` | Stop |

Frames arrive as lines starting with `$`, between normal replies:

```
$0000 0001A2F3 63E 8 0322FF0000000000    line, timestamp (µs, low 32 bits), ID, length, data
$0001 ! 1532 211 0 1321 94               once a second: rx, sent, dropped, suppressed, IDs
```

A whole bus doesn't fit through Bluetooth. On a drive about 30% of lines were dropped even in change-only mode, and one lost frame ruins a multi-frame message. **Use `ATXSN1P` with only the IDs you need** when capturing ISO-TP messages. The sniffer never sees frames the firmware itself injects. Up to 16 IDs can be pinned.

The **CAN Sniffer** in Settings › Developer records a session to a candump `-L` file (`(epoch) can0 63E#0322FF…`), so SavvyCAN, cantools and python-can open it directly. It writes your GPS track into the file as comments, so turns can be found afterwards without touching the phone while driving. A passenger can add marks too.

## Bluetooth Notes

- Keep each Bluetooth write to **65 bytes or fewer**. The app splits long commands into 60-byte writes. The firmware joins them until it sees `\r`.
- Commands can be up to 254 characters.

## Pairing

The adapter pairs with the phone. If a pairing handshake fails, ESP-IDF deletes the bond, and the stock code started one on every connect. The fork no longer forces encryption on connect, and the app shows **Pairing Lost** with a retry when the bond is gone. To fix it, forget the adapter in the app and in iOS Settings › Bluetooth, then pair again.

## Building the Firmware

Build and flash steps are in the [fork's README](https://github.com/JacobPantuso/wicant-i-precondition). The precondition, cluster and sniffer logic have host tests that run on a Mac without ESP-IDF:

```
make -C test/host
```

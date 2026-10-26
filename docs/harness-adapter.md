# Harness Adapter

The harness adapter is a WiCAN connected to the vehicle's multimedia CAN bus (M-CAN) behind the head unit, using the [Ioniq5_CAN](https://github.com/tylerharvey/Ioniq5_CAN) harness kit. It runs the Numen firmware, published as the [`firmware/Numen`](https://github.com/JacobPantuso/wicant-i-precondition/tree/firmware/Numen) branch of this fork, which builds on [wicant-i-precondition](https://github.com/L1Z3/wicant-i-precondition) and [WiCAN](https://github.com/meatpiHQ/wican-fw).

The harness adapter provides three functions unavailable through the OBD-II port:

- Starting and stopping **battery preconditioning**
- Displaying CarPlay directions on the **instrument cluster and HUD**
- **Capturing** multimedia bus traffic for analysis

Numen communicates with it over Bluetooth through the WiCAN's ELM327 emulation. The firmware adds vendor `AT` commands for each function. A standard adapter responds `?` to these commands, which is how Numen identifies supported firmware.

## Why Two Adapters

Battery data is available on the OBD-II port. The cluster and HUD receive navigation data from the head unit's bus, which only the harness can reach. Reading the vehicle therefore requires the OBD-II adapter, and writing to the cluster requires the harness adapter. Numen maintains both Bluetooth connections simultaneously.

## Battery Preconditioning

| Command | Function | Response |
|---|---|---|
| `ATXPC` | Status | `XPC:<state>,<car>,<secs>,<tmin>,<tmax>,<flags>` |
| `ATXPC1` | Start (no effect if running) | `OK` |
| `ATXPC0` | Stop (no effect if idle) | `OK` |
| `ATXPC2` | Toggle, equivalent to the harness button | `OK` |

| Field | Values |
|---|---|
| `state` | 0 idle, 1 requested, 2 starting, 3 active, 4 managed by the vehicle (repeating mode), 5 stopping |
| `car` | The vehicle's reported status: 0 unknown, 1 idle, 2 starting, 3 started |
| `secs` | Seconds until the next retry; 0 when none is pending |
| `tmin`, `tmax` | Lowest and highest pack temperature in °C; valid when flag bit 3 is set |
| `flags` | Bit 0 vehicle in READY, bit 1 status frame received, bit 2 button held, bit 3 pack temperature valid |

`OK` confirms only that the request was queued. The vehicle takes tens of seconds to acknowledge a start, so Numen polls `ATXPC` every 3 seconds while a start or stop is in progress.

### Timing

The firmware retries a start every 10 seconds, up to four times. A start that is not accepted in that time is treated as refused, so a refusal is reported after approximately 50 seconds. A start that is accepted but does not reach the started state times out after 70 seconds.

### Vehicle Decisions

The firmware applies no temperature thresholds; the vehicle does. It declines a start when the pack is already at temperature or when SOC is too low. The firmware reports the outcome:

| Outcome | Meaning |
|---|---|
| Retries exhausted, status remains idle | The vehicle refused the request. Numen reports the pack as already at temperature only when SOC is well above the minimum, as the two causes cannot otherwise be distinguished. |
| Active, then idle without a stop command | The vehicle ended preconditioning. The pack temperature at that moment is the vehicle's target temperature. |

### Automatic Preconditioning

When a CarPlay route ends at a DC fast charger, Numen predicts the pack temperature on arrival and starts preconditioning only when it is useful. It does not start when:

- The charger's maximum power is low enough that pack temperature would not limit it.
- Arrival SOC is high enough that charging would taper regardless.
- The remaining drive is too short to deliver meaningful heat.
- The driver recently declined.

A start is sent once per charger, and the charger is cleared on arrival or when the route is cancelled.

### Polling During a Start

Preconditioning status is polled with `ATXPC` alone, never with a full adapter refresh. A refresh sends `ATZ` and `ATSP6`, which restart the WiCAN's CAN controller while the firmware is transmitting its start sequence. At the standard 60-second interval this restart has no effect.

## Cluster and HUD Directions

While CarPlay is navigating, the head unit continues to send "no guidance" navigation frames. The firmware rewrites these on their way to the cluster and originates the multi-frame text and lane messages the head unit does not send under CarPlay.

The firmware contains no frame layouts. Numen sends the bytes for each frame together with a mask of the bytes to replace, so additional decoding requires only an app update. The layouts are documented in [Cluster and HUD](cluster-and-hud.md).

| Command | Function | Response |
|---|---|---|
| `ATXNV` | Status | `XNV:<active>,<updates>,<injected>` |
| `ATXNV1<frames>` | Set the frames to rewrite: 1–5 entries of 22 hex digits, each `<id:4><mask:2><d0..d7:16>` | `OK` |
| `ATXNV0` | Stop and clear all multi-frame messages | `OK` |
| `ATXNT` | Status | `XNT:<messages>,<frames>` |
| `ATXNT<id:4><hex>` | Set the ISO-TP message for a CAN ID and service, up to 96 bytes | `OK` |

### Safety Limits

- `ATXNV1` can rewrite only `4CC`, `63E`, `640`, `641`, `4E8` and `4EA`. The firmware hook sees every frame bound for the cluster side, including powertrain frames, so no other identifier may be modified.
- `ATXNT` can originate only `F0` (next street), `F2` (HUD maneuvers) and `F4` (destination) on `6E7` and `6DF`, and the `F1` lane graphic on `680` and `681`. Frames are sent 5 ms apart with `AA` padding and no flow control, matching the head unit. Each message is retransmitted only when it changes.
- Injection stops 4 seconds after the last `ATXNV1` and immediately on Bluetooth disconnection, so a lost connection cannot leave a stale arrow on the cluster. When injection stops, each message the firmware originated is replaced with its blank form.
- Battery preconditioning takes priority: its countdown controls `4E8` and `4CC` while a start or stop is in progress.

### Known Limitation

On the single-bus WiCAN, the firmware cannot suppress the head unit's frame and instead transmits a corrected frame immediately after it. The cluster may briefly show the head unit's frame. A dual-bus WiCAN in bridge mode replaces the frame outright using the same code.

## CAN Capture

The capture function is receive-only and transmits nothing.

| Command | Function |
|---|---|
| `ATXSN` | Status: `XSN:<on>,<npins>,<rx>,<sent>,<dropped>,<suppressed>,<ids>` |
| `ATXSN1` | Start, sending each identifier only when its payload changes |
| `ATXSN1 63E,4CC,…` | As above, plus every frame of the listed identifiers |
| `ATXSN1P 63E,4CC,…` | Only the listed identifiers |
| `ATXSN0` | Stop |

Frames are delivered as lines prefixed with `$`, interleaved with standard responses:

```
$0000 0001A2F3 63E 8 0322FF0000000000    line, timestamp (µs, low 32 bits), ID, length, data
$0001 ! 1532 211 0 1321 94               once per second: rx, sent, dropped, suppressed, IDs
```

The full bus exceeds Bluetooth throughput; approximately 30% of lines were dropped on a drive even in change-only mode, and a single lost frame corrupts a multi-frame message. Use `ATXSN1P` with only the required identifiers when capturing ISO-TP messages. Up to 16 identifiers can be pinned. Frames injected by the firmware are not captured.

The **CAN Sniffer** in Settings › Developer records sessions in candump `-L` format (`(epoch) can0 63E#0322FF…`), readable by SavvyCAN, cantools and python-can. The GPS track is written into the file as comments so events can be located after a drive, and markers can be added during recording.

## Bluetooth

- Each Bluetooth write must be 65 bytes or fewer. Numen splits long commands into 60-byte writes, and the firmware reassembles them up to the terminating `\r`.
- Commands may be up to 254 characters.

## Pairing

The adapter is paired with the iPhone. ESP-IDF deletes the bond when a pairing handshake fails, and the original firmware initiated a handshake on every connection. The Numen firmware no longer forces encryption on connection, and the app reports **Pairing Lost** when the bond is missing. To recover, remove the adapter in the app and in iOS **Settings › Bluetooth**, then pair it again.

## Installing and Updating

Install the [latest release](https://github.com/JacobPantuso/wicant-i-precondition/releases/latest) once from the WiCAN's web page; no build is needed. After that, **Update Adapter** in Settings › OBD Adapter updates it from the app: Numen sends `ATXOTA`, joins the WiCAN's one-boot access point, uploads the build bundled with the app, and checks `ATXVER` once the adapter is back. In-app updates need the Numen firmware already on the WiCAN: stock firmware, or a build from before in-app updates, has no `ATXOTA`, so the first install always goes through the web page.

## Building the Firmware

Build and flashing instructions are in the [firmware README](https://github.com/JacobPantuso/wicant-i-precondition/tree/firmware/Numen). The preconditioning, cluster and capture logic include host tests that run on macOS without ESP-IDF:

```
make -C test/host
```

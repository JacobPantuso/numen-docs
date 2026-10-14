# Cluster and HUD

How the IONIQ 6 instrument cluster and head-up display draw turn-by-turn directions, and how Numen sends its CarPlay route to them through the [harness adapter](harness-adapter.md).

All of this was decoded on my IONIQ 6 from CAN captures of the built-in navigation and from parked sweeps, sending one byte value at a time and watching what changed. The first attempt copied IONIQ 5 layouts from public logs, and the IONIQ 6 ignored them. Trust the tables below over IONIQ 5 sources for this car.

Bytes are written `d0`–`d7`. "LE" means a little-endian 16-bit value.

## How It Fits Together

- The **cluster** reads a handful of periodic 8-byte frames (`4CC`, `63E`, `640`, `4E8`) for the arrow, distances and the approach bar, plus an ISO-TP text message on `6E7` for the street name.
- The **HUD** ignores the cluster's arrow. It draws its own maneuver list from an `F2` message on `6DF`, its turn distance from `63E`, and its lane graphic from an `F1` message on `680` and `681`. Without an `F2` on `6DF`, the HUD draws nothing.
- Under CarPlay the head unit sends "no guidance" on the periodic frames and stays silent on the ISO-TP ones. The firmware rewrites the first and originates the second.

## Periodic Frames

### `4CC`: Arrow and Mode

```
<mode> <roads d1> <roads d2> <arrow> <distance LE> <unit> <tenths<<4>
```

**Mode (`d0`)**

| Value | Meaning |
|---|---|
| `0D` | Turn. Shows the arrow in `d3`. The built-in nav switches to this at 400 m. |
| `11` | Motorway junction graphic, the exit ramp peeling off. From about 1.2 km out. |
| `20` | Roundabout. A ring entered from the bottom, driven counter-clockwise. |
| `09` | Calculating |
| `0A` | Recalculating |
| `08` | No guidance |
| `02` | Off route. `d3` is the heading. |
| other | Far out, `d0` carries the HUD maneuver code (same table as `F2`), with the distance in `d4`–`d7`. |

**Arrow (`d3`)** is the clockwise angle of the turn in 7.5° steps.

| Value | Arrow |
|---|---|
| `00` | Straight |
| `03` | Bend right |
| `06` | Sharp bend right |
| `0C` | Right |
| `12` | Sharp right |
| `18` | U-turn (draws nothing in turn mode) |
| `1E` | Sharp left |
| `24` | Left |
| `2A` | Sharp bend left |
| `2D` | Bend left |

In roundabout mode `d3` picks the exit road using the same angles.

**Side roads (`d1`, `d2`)** draw the junction's other roads behind the arrow. One bit per 22.5° slot, clockwise from straight ahead, with the road you came in on (180°) skipped. `d1` bits 0–7 are 0° to 157.5°. `d2` bits 0–6 are 202.5° to 337.5°.

**Distance.** In metres, `d4`–`d5` is the distance and `d6`–`d7` are zero. In kilometres, `d6` is `01` and `d7` holds the tenths digit in its top nibble. `02 00 01 30` draws "2.3 km".

### `63E`: Turn Distance

```
<tenths<<4 | unit> 00 <to this turn LE> <to the turn after LE> 00 00
```

- The HUD's turn distance comes only from here.
- `d0` is the unit: `00` metres, `01` km. In metres the HUD draws nothing at 1,000 m or more.
- `d4`–`d5` is the distance **from the car** to the turn after this one, not the gap between them. It counts down alongside `d2`–`d3`.

### `640`: Approach Bar

The head unit's idle bytes `FF FF FF 00 FF FF FF` with `d7` as the bar, in percent, in steps of 10.

- For a turn it runs from 100 down to 10 over the last ~400 m.
- For a motorway junction it starts at 90 at 1.2 km, drops 10 per 100 m, and holds at 10 for the last ~200 m.

At motorway lane splits, `d0`–`d1` carry the metres to a lane guidance point while the HUD shows lanes.

### `4E8`: Destination

```
<tenths<<4 | unit> <ETA hour> <ETA minute> <arrow − 1, or 3F> <distance LE> <5 | flag> 00
```

`641` never changes. `4EA` `d6` goes from `00` to `10` when the built-in nav starts guiding, and isn't needed.

## ISO-TP Messages

`AA` padding, no flow control, consecutive frames about 5 ms apart. Text is UTF-16LE. A blank message is the service byte followed by `00 00`.

| Message | IDs | Content |
|---|---|---|
| `F0` | `6E7` cluster, `6DF` HUD | Next street |
| `F4 00` | `6E7`, `6DF` | Destination |
| `F2` | `6DF` | HUD maneuver list |
| `F1` | `680`, `681` | HUD lane graphic, 81 bytes |

### `F2`: HUD Maneuvers

```
F2 <count> <code 1> <code 2> <code 3> <street> 00 00 00 00
```

The first code is the big arrow, the second is the small "then" arrow. Unused codes are `00`.

| Code | Arrow |
|---|---|
| `41` | Straight |
| `42` | Slight right |
| `43` | Right |
| `44` | Sharp right |
| `45` | Sharp left |
| `46` | Left |
| `47` | Slight left |
| `48` | U-turn left |
| `49` | U-turn right |
| `4A` | Blank |
| `70` | Destination (always last) |
| `92` | Keep or fork |
| `A4` | Motorway exit |
| `60`–`6B` | Roundabout. `60` + exit angle ÷ 30°, clockwise. `60` straight, `63` right, `66` full circle, `69` left. |

### `F1`: HUD Lanes

| Byte | Meaning |
|---|---|
| `[1]` | Lane count |
| `[3 + i]` | Lane `i` colours: `overlay << 2 \| underlay`. 0 none, 1 white, 2 blue, 3 grey. |
| `[19 + 3i]` | Lane `i` underlay arrow |
| `[20 + 3i]` | Lane `i` overlay arrow |

Single arrows: `00` off, `01` straight, `02` slight right, `03` right, `04` sharp right, `05` U-turn left, `06` sharp left, `07` left, `08` slight left.

Lanes with two directions use joined shapes, underlay then overlay:

| Pair | Shape |
|---|---|
| `10` / `13` | Straight (hugging the left) and slight right |
| `10` / `12` | Straight and right |
| `09` / `0C` | Straight (hugging the right) and left |
| `09` / `0D` | Straight and slight left |
| `09` / `0B` | Straight and sharp left |
| `09` / `0A` | Straight and U-turn |
| `1F` / `20` | Left and right |

The built-in nav draws the lanes you should use in white and the rest in grey. The cluster only draws two lanes.

## How Numen Fills Them

`ClusterNavFeed` in the app works out what to show for each route step once per route, from Mapbox's step maneuvers, intersection bearings and lane data. It then sends `ATXNV1` with the periodic frames on every progress update and `ATXNT` only when text or lanes change.

- Distances use 10 m steps under 1 km and 100 m steps above. The built-in nav switches to 100 m steps at 300 m, but in traffic that held one HUD reading for a minute.
- Side roads within 16° of the exit arrow are dropped so they don't draw on top of it.
- Roundabouts send mode `20`, the real exit angle, the roads you pass, and the `60`+ HUD code.

**Unconfirmed.** Roundabouts and side roads were decoded parked and haven't been tested on a drive yet.

## Finding More

The **Cluster Codes** explorer in Settings › Developer sends any byte value to any of these frames while parked, with ready-made scenarios (roundabout, T-junction, lanes) and a sweep mode. Try new behaviour there before writing code, and capture the built-in nav with the [sniffer](harness-adapter.md#can-sniffer-atxsn) in pinned-only mode.

Things still unknown:

- `4CC` `d0` `43` appears just after some turns.
- HUD code `72`.
- `F1` pair `16` / `17`, possibly straight and slight-right exit.
- Whether `63E` unit values above `01` mean miles, yards and feet.

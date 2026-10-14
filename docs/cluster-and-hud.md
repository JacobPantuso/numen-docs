# Cluster and HUD

This document describes how the IONIQ 6 instrument cluster and head-up display render turn-by-turn navigation, and how Numen supplies CarPlay route guidance to them through the [harness adapter](harness-adapter.md).

The layouts were decoded on an IONIQ 6 from CAN captures of the built-in navigation and from stationary tests in which individual byte values were varied. Layouts published for the IONIQ 5 do not apply: the IONIQ 6 ignores them.

Bytes are numbered `d0`–`d7`. "LE" denotes a little-endian 16-bit value.

## Overview

| Display | Source |
|---|---|
| Cluster | Periodic 8-byte frames `4CC`, `63E`, `640` and `4E8` for the arrow, distances and approach bar, and an ISO-TP text message on `6E7` for the street name |
| HUD | Its own maneuver list (`F2` on `6DF`), turn distance from `63E`, and lane graphic (`F1` on `680` and `681`). The HUD ignores the cluster arrow and displays nothing without an `F2` message on `6DF`. |

Under CarPlay, the head unit sends "no guidance" in the periodic frames and does not send the ISO-TP messages. The firmware rewrites the former and originates the latter.

## Periodic Frames

### `4CC`: Maneuver

```
<mode> <side roads d1> <side roads d2> <arrow> <distance LE> <unit> <tenths<<4>
```

| Mode (`d0`) | Meaning |
|---|---|
| `0D` | Turn; displays the arrow in `d3`. The built-in navigation uses this mode from 400 m. |
| `11` | Motorway junction graphic, used from approximately 1.2 km |
| `20` | Roundabout: a ring entered from the bottom, travelled counter-clockwise |
| `09` | Calculating |
| `0A` | Recalculating |
| `08` | No guidance |
| `02` | Off route; `d3` is the heading |
| Other | At longer distances, `d0` carries the HUD maneuver code (see `F2`) with the distance in `d4`–`d7` |

The arrow (`d3`) is the clockwise turn angle in 7.5° steps. In roundabout mode it selects the exit.

| Value | Arrow |
|---|---|
| `00` | Straight |
| `03` | Bend right |
| `06` | Sharp bend right |
| `0C` | Right |
| `12` | Sharp right |
| `18` | U-turn (not drawn in turn mode) |
| `1E` | Sharp left |
| `24` | Left |
| `2A` | Sharp bend left |
| `2D` | Bend left |

Side roads (`d1`, `d2`) draw the junction's other roads behind the arrow, one bit per 22.5° sector clockwise from straight ahead, excluding the entry road at 180°. `d1` bits 0–7 cover 0° to 157.5°; `d2` bits 0–6 cover 202.5° to 337.5°.

Distance in metres uses `d4`–`d5` with `d6`–`d7` set to zero. Distance in kilometres sets `d6` to `01` and places the tenths digit in the upper nibble of `d7`; for example, `02 00 01 30` displays 2.3 km.

### `63E`: Turn Distance

```
<tenths<<4 | unit> 00 <distance to this turn LE> <distance to the following turn LE> 00 00
```

- The HUD takes its turn distance from this frame only.
- `d0` is the unit: `00` metres, `01` kilometres. In metres, the HUD displays nothing at 1,000 m or more.
- `d4`–`d5` is the distance from the vehicle to the following turn, not the distance between turns, and counts down with `d2`–`d3`.

### `640`: Approach Bar

The head unit's idle bytes `FF FF FF 00 FF FF FF`, with `d7` as the approach bar in percent, in steps of 10.

- Turns: decreases from 100 to 10 over the final ~400 m.
- Motorway junctions: starts at 90 at 1.2 km, decreases by 10 per 100 m, and holds at 10 for the final ~200 m.

At motorway lane splits, `d0`–`d1` carry the distance in metres to the lane guidance point while the HUD shows lanes.

### `4E8`: Destination

```
<tenths<<4 | unit> <ETA hour> <ETA minute> <arrow − 1, or 3F> <distance LE> <5 | flag> 00
```

`641` does not change. `4EA` `d6` changes from `00` to `10` when built-in guidance starts and is not required.

## ISO-TP Messages

Messages use `AA` padding, no flow control, and approximately 5 ms between consecutive frames. Text is UTF-16LE. A blank message is the service byte followed by `00 00`.

| Message | CAN IDs | Content |
|---|---|---|
| `F0` | `6E7` (cluster), `6DF` (HUD) | Next street |
| `F4 00` | `6E7`, `6DF` | Destination |
| `F2` | `6DF` | HUD maneuver list |
| `F1` | `680`, `681` | HUD lane graphic, 81 bytes |

### `F2`: HUD Maneuvers

```
F2 <count> <code 1> <code 2> <code 3> <street> 00 00 00 00
```

The first code is the primary arrow and the second is the following maneuver. Unused codes are `00`.

| Code | Maneuver |
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
| `60`–`6B` | Roundabout: `60` + exit angle ÷ 30°, clockwise (`60` straight, `63` right, `66` full circle, `69` left) |

### `F1`: HUD Lanes

| Byte | Content |
|---|---|
| `[1]` | Lane count |
| `[3 + i]` | Lane `i` colours: `overlay << 2 \| underlay`, where 0 is none, 1 white, 2 blue, 3 grey |
| `[19 + 3i]` | Lane `i` underlay arrow |
| `[20 + 3i]` | Lane `i` overlay arrow |

Single arrows: `00` off, `01` straight, `02` slight right, `03` right, `04` sharp right, `05` U-turn left, `06` sharp left, `07` left, `08` slight left.

Lanes permitting two directions use combined shapes (underlay / overlay):

| Pair | Shape |
|---|---|
| `10` / `13` | Straight (left-aligned) and slight right |
| `10` / `12` | Straight and right |
| `09` / `0C` | Straight (right-aligned) and left |
| `09` / `0D` | Straight and slight left |
| `09` / `0B` | Straight and sharp left |
| `09` / `0A` | Straight and U-turn |
| `1F` / `20` | Left and right |

The built-in navigation draws recommended lanes in white and others in grey. The cluster displays at most two lanes.

## Implementation

`ClusterNavFeed` computes the display state for each route step once per route, using Mapbox maneuvers, intersection bearings and lane data. It sends `ATXNV1` with the periodic frames on each progress update, and `ATXNT` only when text or lanes change.

- Distances use 10 m steps below 1 km and 100 m steps above.
- Side roads within 16° of the exit are omitted so they do not overlap the arrow.
- Roundabouts use mode `20`, the actual exit angle, the roads passed, and the corresponding `60`-series HUD code.

**Not yet verified.** Roundabouts and side roads were decoded while stationary and have not been tested on a drive.

## Further Decoding

The **Cluster Codes** explorer in Settings › Developer sends arbitrary values to any of these frames while stationary, with predefined scenarios (roundabout, T-junction, lanes) and a sweep mode. New behaviour should be tested there and compared against captures of the built-in navigation made with the [CAN capture](harness-adapter.md#can-capture) function in pinned-only mode.

Unresolved values:

| Value | Observation |
|---|---|
| `4CC` `d0` = `43` | Appears briefly after some turns |
| HUD code `72` | Observed; meaning unknown |
| `F1` pair `16` / `17` | Possibly straight with a slight-right exit |
| `63E` units above `01` | Possibly miles, yards and feet |

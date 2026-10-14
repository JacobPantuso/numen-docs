# Routing and Charging Stops

Numen runs as a full navigation app in CarPlay, and the Trip Planner on the phone plans long drives with charging stops. Both use the same services.

| Service | Used For |
|---|---|
| Mapbox Directions (`driving-traffic`) | Routes, turn-by-turn steps, lanes, traffic, speed limits, signals |
| Mapbox Directions Refresh | Updated traffic on the current route without re-routing |
| Mapbox Map Matching | Snapping a route Apple chose onto Mapbox, to get lanes for it |
| Mapbox Terrain | Elevation, for the [energy model](energy-model.md) |
| Apple MapKit (`MKDirections`) | A second opinion on every route, traffic colouring, ETAs, and the fallback when Mapbox is unavailable |
| Open Charge Map | Chargers and their power |
| WeatherKit | Temperature and wind along the route |

There's no Mapbox SDK. The Navigation SDK wants to own the map and the turn-by-turn UI, which would fight CarPlay's own navigation session. Every Mapbox call is a plain `GET`.

## Why Mapbox

Apple's routes have no lane data. An `MKRoute` step only has instructions, a notice, a distance and a line. Mapbox returns lanes for each intersection, which feed CarPlay's lane guidance and the [cluster and HUD](cluster-and-hud.md). It also flags traffic signals and stop signs along the route, so they can be drawn without a separate map lookup.

The Directions request asks for:

```
geometries=geojson
overview=full
steps=true
banner_instructions=true
annotations=congestion,closure,maxspeed,duration
enable_refresh=true
alternatives=true
exclude=toll           (when Avoid Tolls is on)
bearings=<heading>,45  (the origin only, so the first turn suits the way you're facing)
```

## Why Apple Too

Mapbox misses long-term closures. Its `closures` field is live traffic: crashes, short-lived incidents. A street closed for weeks of construction has to make it into the road map itself, and that happens on Mapbox's own schedule. Apple takes construction feeds from cities directly.

I found this on a street in Toronto that had been closed for weeks. Mapbox drove straight down it. Apple went around.

So every route asks both at once:

1. If Apple's shortest route is **more than 20% longer** than Mapbox's, Apple is avoiding something Mapbox can't see. Apple's road choice wins.
2. That route is then sent through Mapbox Map Matching (up to 600 points), so it still gets lanes, signals and speed limits. Apple's ETA and road names are kept, since that's what you were shown.
3. If matching fails, the Apple route is used as is. You lose lanes, not the right road.
4. If Mapbox is down or there's no token, Apple's routes are used for everything.

## Traffic

- Mapbox's per-segment congestion colours the route line.
- The Refresh API re-reads traffic for the current route every 3 minutes without changing the road.
- Map-matched routes can't be refreshed, so they fall back to Apple. The route is split into legs of about 4.5 km, and each leg's live ETA is compared with its 3 AM ETA. The ratio is the congestion.

## Charging Stops

### Finding Chargers

Chargers come from Open Charge Map, not Apple Maps. Apple's points of interest have almost no chargers outside cities.

The search runs when the chargers already known leave a **gap** along the route longer than the car can drive on a full charge. An earlier version searched when it knew fewer than 20 fast chargers. Toronto alone has more than that downtown, so a Toronto to Winnipeg trip planned one stop and said it wouldn't make it. With the gap test the same trip knows about 143 fast chargers instead of 37, the biggest gap drops from 2,030 km to 219 km, and it plans 8 stops.

Tesla Superchargers that are open to all only count when you've said you have a NACS adapter.

### Sizing Each Stop

Each stop charges to what the **next leg** needs, plus your minimum Arrival SOC, plus a 5% buffer. It never goes past 80%.

DC charging slows down a lot above about 60%. Filling to 80% at every stop buys range you then carry unused to your destination. The cost is a lower Arrival SOC at the end, which is fine as long as it's above the minimum you set (10% by default).

Planning is two passes. The first places the stops, which only needs what's ahead. The second sizes them, which needs to know where the *next* stop is.

### Editing

You can add, remove, reorder and swap stops on the phone and in CarPlay. Once you edit, the planner keeps your list and only re-works each stop's charge. Swapping one stop only re-asks for the two legs that touch it. A stop that isn't a charger (coffee, a friend's place) is allowed.

Each leg shows its distance, its climb, and the weather on arrival.

## Favourite Routes

Numen learns which road you usually take to places you drive often, and picks it in the CarPlay route preview with a badge. See [Trips](trips-and-charging.md#favourite-routes).

If Mapbox didn't offer the road you usually take, the app asks again through a point on that road to get it back.

## Getting a Destination In

- Search in the app or in CarPlay, by voice or by typing. Search returns businesses as well as addresses.
- Share a place from any app (Maps, Safari, Messages). The share extension sends it to CarPlay if it's connected, or to the Trip Planner if not.
- **Send to Vehicle** holds a destination until CarPlay next connects.
- When CarPlay connects, Numen may suggest where you're headed. See [Trips](trips-and-charging.md#where-youre-headed).

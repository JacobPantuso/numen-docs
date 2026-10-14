# Routing and Charging Stops

Numen provides full navigation in CarPlay and trip planning with charging stops on iPhone. Both use the same services.

| Service | Purpose |
|---|---|
| Mapbox Directions (`driving-traffic`) | Routes, turn-by-turn steps, lanes, traffic, speed limits and traffic signals |
| Mapbox Directions Refresh | Updated traffic for the active route without re-routing |
| Mapbox Map Matching | Matching an Apple-selected route to Mapbox data to obtain lane guidance |
| Mapbox Terrain | Elevation for the [energy model](energy-model.md) |
| Apple MapKit (`MKDirections`) | Route validation, traffic colouring, ETAs, and fallback routing |
| Open Charge Map | Charger locations and power |
| WeatherKit | Temperature and wind along the route |

Mapbox is accessed through its REST APIs rather than the Navigation SDK, which requires control of the map and guidance interface and would conflict with CarPlay's navigation session.

## Route Selection

### Mapbox

`MKRoute` steps contain only instructions, a notice, a distance and a polyline; they carry no lane data. Mapbox returns lanes for each intersection, which drive CarPlay lane guidance and the [cluster and HUD](cluster-and-hud.md), along with traffic signal and stop sign positions.

Directions requests include:

```
geometries=geojson
overview=full
steps=true
banner_instructions=true
annotations=congestion,closure,maxspeed,duration
enable_refresh=true
alternatives=true
exclude=toll           when Avoid Tolls is enabled
bearings=<heading>,45  origin only, so the first maneuver matches the vehicle's heading
```

### Apple Validation

Mapbox's `closures` annotation reflects live incidents only. Long-term closures, such as multi-week construction, must be incorporated into the road network itself, which Mapbox updates on its own schedule. Apple receives municipal construction data directly. In testing, Mapbox routed through a street that had been closed for several weeks while Apple routed around it.

Each route is therefore requested from both services:

1. If Apple's shortest route is more than 20% longer than Mapbox's, Apple is assumed to be avoiding a restriction Mapbox does not know about, and Apple's route is used.
2. That route is matched to Mapbox (up to 600 points) to obtain lanes, signals and speed limits. Apple's ETA and road names are retained.
3. If matching fails, the Apple route is used without lane data.
4. If Mapbox is unavailable, Apple routes are used throughout.

## Traffic

- Mapbox per-segment congestion colours the route.
- The Directions Refresh API updates traffic for the active route every 3 minutes.
- Map-matched routes cannot be refreshed. For these, the route is divided into segments of approximately 4.5 km, and each segment's live ETA is compared with its 3 AM ETA to estimate congestion.

## Charging Stops

### Charger Discovery

Chargers are sourced from Open Charge Map. Apple Maps lists very few chargers outside urban areas.

The corridor search runs when the largest gap between known chargers along the route exceeds the vehicle's full-charge range. A previous threshold based on the number of known chargers caused long routes from dense urban areas to be under-searched. On a Toronto to Winnipeg route, the gap-based search increased known DC fast chargers from 37 to 143, reduced the largest gap from 2,030 km to 219 km, and produced an 8-stop plan where the previous version reported the trip as not possible.

Tesla Superchargers open to all vehicles are included only when the user has indicated they carry a NACS adapter.

### Charge Targets

Each stop charges to the energy required for the following leg, plus the user's minimum Arrival SOC, plus a 5% buffer, up to a maximum of 80%.

DC charging slows significantly above approximately 60%, so charging every stop to 80% adds time for range that is not needed. The trade-off is a lower Arrival SOC at the destination, which remains above the user's configured minimum (10% by default).

Planning is performed in two passes: the first places stops, and the second sets each stop's target based on the position of the next stop.

### Editing

Stops can be added, removed, reordered and replaced on iPhone and in CarPlay. Once edited, the planner preserves the user's stops and recalculates only charge targets. Replacing a stop re-requests only the two adjacent legs. Non-charging stops are supported.

Each leg displays its distance, total climb and the forecast weather on arrival.

## Favourite Routes

Numen identifies the road usually taken to frequent destinations and selects it in the CarPlay route preview. See [Trips](trips-and-charging.md#favourite-routes). If the favourite road is not among the returned alternatives, the route is re-requested through a point on that road.

## Destination Entry

<p align="center">
  <img src="../images/navigate.png" width="240" alt="A charger selected on iPhone">
</p>

- Search in the app or in CarPlay, by voice or text. Results include businesses and addresses.
- Places shared from other apps are sent to CarPlay when connected, or to the Trip Planner otherwise.
- **Send to Vehicle** holds a destination until CarPlay next connects.
- On connection, CarPlay may suggest a likely destination. See [Trips](trips-and-charging.md#destination-prediction).

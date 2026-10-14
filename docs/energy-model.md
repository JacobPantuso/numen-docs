# Energy Model

All Arrival SOC estimates in Numen are produced by a single model: a physical model of the vehicle evaluated along the actual route, then calibrated against the vehicle's measured trips.

This replaces an earlier estimate based on distance as a fraction of Bluelink's range, which did not account for elevation, speed, traffic or temperature.

**Not yet verified.** The model is new and is being validated against real drives.

## Vehicle Parameters

Defaults for the IONIQ 6 AWD with a driver:

| Parameter | Value |
|---|---|
| Mass | 2,150 kg |
| Drag area (Cd × frontal area) | 0.51 m² (Cd 0.22 with 20" wheels × ~2.32 m²) |
| Rolling resistance coefficient | 0.010 |
| Drivetrain efficiency | 85% |
| Regenerative recovery | 65% |
| Base electrical load | 0.35 kW |
| Usable capacity | Last capacity reported by the BMS, or 77.4 kWh |

These produce approximately 15 kWh/100 km at 100 km/h and 18.5 kWh/100 km at 120 km/h, consistent with published highway consumption.

## Route Evaluation

The route is divided into segments of approximately 250 m. Each segment has its own speed, elevation change, heading and weather, and its energy cost is the sum of:

| Component | Model |
|---|---|
| Rolling resistance | Scaled by tire type (All-Season × 1.00, Winter × 1.08, Summer × 0.95), with an allowance for wet roads |
| Aerodynamic drag | Calculated at airspeed, so headwinds increase and tailwinds reduce consumption. Air density follows temperature (approximately 10% higher at −15 °C than at 20 °C). |
| Elevation | Climbing consumes energy; descending recovers part of it through regenerative braking |
| Stop-and-go | Kinetic energy not recovered at each stop, with stop frequency derived from average speed |
| Cabin and battery | Heating or cooling for the time spent on the segment. The heat pump is effective to approximately −10 °C, below which resistive heating is used. A cold pack incurs additional internal losses. |

Segment speeds come from Mapbox's per-segment durations and include traffic. The user's driving profile adjusts cruising speed only.

## Elevation

Elevation is read from Mapbox Terrain-DEM tiles, in which each pixel encodes a height:

```
height = −10000 + (R × 65536 + G × 256 + B) × 0.1   metres
```

- Zoom level depends on route length, from zoom 12 for short routes (~28 m per pixel at Toronto's latitude) to zoom 10 for long ones (~110 m per pixel).
- Tiles are cached on disk.
- Values are read directly from the PNG data, without drawing, so that colour management cannot alter them.
- The elevation profile is smoothed to suppress pixel-level noise.

## Weather

WeatherKit hourly forecasts are sampled along the route for the expected time of arrival at each point. Forecasts are cached for one hour per ~25 km area.

## Progressive Estimates

An initial estimate is produced immediately from route speeds and current temperature. Elevation and forecasts are then fetched in the background, the estimate is recalculated, and all screens update. iPhone, CarPlay and the CarPlay dashboard all use the same estimator.

## Calibration

Each OBD-measured trip of 3 km or longer is replayed through the model with its recorded speeds, elevation and temperature, and the prediction is compared with the energy reported by the BMS.

Two correction factors are fitted from these pairs:

| Factor | Covers | Range |
|---|---|---|
| Driving | Rolling resistance, drag, elevation and regenerative braking | 0.75–1.35 |
| Cabin and battery | Heating and cooling | 0.5–2.0 |

Fitting them separately allows cold-weather driving to adjust the thermal factor without distorting highway consumption. With few trips the factors are weighted toward 1.0, and full weight is reached after 10 trips.

**Settings › Trip Planning › Range Estimates** shows the calibration status (**Not Confident** under 3 trips, **Learning** under 10, then **Confident**) and the vehicle's consumption relative to the model.

Calibration is derived from trip history, which is included in the [iCloud backup](icloud-backup.md), and is rebuilt after a reinstall.

## In-Drive Correction

During guidance, energy used so far is compared with the prediction, and the remainder of the route is rescaled by the difference. When guidance resumes partway through a route, the starting charge is back-calculated so the correction starts neutral.

## Range Without Bluelink

The OBD-II signal set has no range value. When Bluelink is not in use, range is calculated as the adapter's SOC multiplied by a full-charge range learned from the vehicle's own trips:

| Parameter | Value |
|---|---|
| Initial full-charge range | 475 km (the typical 450–500 km for an IONIQ 6), weighted as 200 km of driving |
| Recency | A trip's weight halves every 21 days; trips older than 90 days are excluded |
| Adapter-measured trips | Full weight, 3 km minimum |
| Trips measured by battery percentage | Half weight, 15 km minimum |
| Per-trip consumption | Trips outside 8–40 kWh/100 km are excluded |
| Result | Limited to 250–650 km per full charge |

| Recent Driving | Full Charge | At 64% |
|---|---|---|
| No trips | 475 km | 304 km |
| 14 kWh/100 km for a month | 526 km | 337 km |
| 22 kWh/100 km for a month | 383 km | 245 km |
| Summer, followed by two weeks of winter | 401 km | 257 km |

When Bluelink is available, its range figure is used instead.

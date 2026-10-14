# Energy Model

Every Arrival SOC in Numen comes from one place: a physics model of the car, run over the actual route, then corrected by how your car has really driven.

It used to be "route km ÷ Bluelink range × battery %". That charged a 50 km climb the same as 50 km of flat road, stop-and-go the same as 120 km/h, and a −15 °C morning the same as July.

**Unconfirmed.** The model is new and still being checked against real drives.

## The Car

Defaults are the IONIQ 6 AWD with a driver:

| | Value |
|---|---|
| Mass | 2,150 kg |
| Drag area (Cd × frontal area) | 0.51 m² (Cd 0.22 on 20" wheels × about 2.32 m²) |
| Rolling resistance | 0.010 |
| Drivetrain efficiency | 85% |
| Regen recovery | 65% |
| Base electrical load | 0.35 kW |
| Usable pack | The last capacity the BMS reported, or 77.4 kWh |

These give about 15 kWh/100 km at 100 km/h and 18.5 at 120 km/h, which matches Hyundai's published highway figures.

## A Route, Piece by Piece

The route is cut into pieces about 250 m long. Each piece gets its own speed, elevation change, heading and weather, and the model adds up what each one costs.

For each piece:

- **Rolling resistance.** Scaled by your tires: All-Season × 1.00, Winter × 1.08, Summer × 0.95. Wet roads add a bit more.
- **Air resistance.** At your speed through the air, not over the road, so a headwind costs and a tailwind helps. Cold air is denser: about 10% more at −15 °C than at 20 °C.
- **Elevation.** Going uphill uses energy. Going downhill gets some back through regen. A long descent can come out ahead.
- **Stop-and-go.** Each stop throws away energy regen can't recover. The slower the average speed, the more stops it assumes.
- **Cabin and battery.** Heating or cooling for the time spent on that piece. The heat pump keeps heating cheap down to about −10 °C. Below that the resistive heater takes over. A cold pack also loses a bit more to internal resistance.

Speeds come from Mapbox's per-segment durations, so traffic is included. Your driving profile scales cruising speed only, since nobody's driving style changes how fast a traffic jam moves.

## Elevation

From Mapbox's Terrain-DEM tiles. Each pixel's colour is a height:

```
height = −10000 + (R × 65536 + G × 256 + B) × 0.1   metres
```

- Zoom depends on the route length: zoom 12 for a commute (about 28 m a pixel in Toronto), down to zoom 10 for a road trip (about 110 m). Either is fine enough to follow a hill.
- Tiles are cached on disk, so a commute's tiles are only downloaded once.
- Bytes are read straight from the PNG, not drawn into an image first. Drawing can apply colour management and nudge the values, and here the values are the data.
- The profile is smoothed so pixel-to-pixel noise doesn't count as climbing.

## Weather

WeatherKit's hourly forecast at points along the route, read for the hour you'll be there. A 300 km drive that leaves in sun can finish in rain and a headwind. Forecasts are cached for an hour per 25 km area, so re-planning a trip doesn't spend more WeatherKit calls.

## Quick First, Better After

Estimates have to show up straight away. So the first answer is built from the route's speeds and today's temperature. Elevation and the forecast are fetched in the background, the estimate is rebuilt, and every screen updates. The phone, CarPlay and the dashboard all ask the same place, so they always agree.

## Learning Your Car

Every trip the OBD-II adapter measured (3 km or longer) is run back through the model with the same speeds, the same elevation and that day's temperature. The prediction is compared with what the BMS says the trip actually used.

Two corrections are fitted from those pairs:

- **Driving**, for rolling, air, elevation and regen. Kept between 0.75 and 1.35.
- **Cabin and battery**, for heating and cooling. Kept between 0.5 and 2.0.

Keeping them apart means a cold week teaches the heating side without throwing off the highway side.

With only a few trips, the corrections are pulled toward no change, so three trips nudge the model instead of rewriting it. It's trusted fully after 10.

**Settings › Trip Planning › Range Estimates** shows how it's going: **Not Confident** under 3 trips, **Learning** under 10, then **Confident**. It also shows how your car compares with the physics, as a single figure ("uses 6% more").

The corrections come from your trip history, which is already [backed up](icloud-backup.md). A reinstall relearns them instead of syncing them.

## While You Drive

During guidance the app compares what the drive has used so far with what was predicted, and re-rates the rest of the route by that difference. If you pick up guidance partway (after the app restarts, say), it works backwards to your starting charge so it doesn't blame the model for road it never saw.

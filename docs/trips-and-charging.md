# Trips and Charge History

Bluelink Canada has no trip history and no charge history. Numen records both on the phone.

## Recording Trips

Trips are detected from the phone's speed, not from the beacon or the OBD-II adapter. The adapter is optional, and the beacon only says the phone is *near* the car, not that it's moving.

- Significant-location monitoring runs all the time. It's cheap, and it wakes the app even after it's been closed.
- A fix at **18 km/h** or more starts a trip. That's faster than running and slower than any road, so walking never starts one.
- Under **4 km/h** counts as stopped. **3 minutes** stopped ends the trip, long enough to survive a long red light.
- Trips shorter than **400 m** or **90 seconds** are thrown away.
- Points are saved at most every 10 seconds. Long trips are thinned so the file stays small.

Anything that needs the network (street names for the start and end, the temperature from WeatherKit) happens after the trip is saved, and updates it in place. A trip is never lost because a lookup failed.

### Energy and Regen

- With the OBD-II adapter connected, energy used is the change in the BMS's remaining energy. It's exact.
- Without it, energy comes from the change in Bluelink's SOC, which is rough. The trip says which one it used.
- Regen comes from the BMS's lifetime counters. See [OBD-II Adapter](obd-adapter.md#regen-per-trip).

## Charge History

Readings come from two places and are stitched into sessions:

- The **OBD-II adapter** every 60 seconds while connected, and straight away when a charge starts or stops.
- **Bluelink** on each status refresh, but only while the adapter isn't connected. A cached "not charging" from Bluelink would otherwise end a session that's still running.

Rules for stitching:

- A session already under way can go up to **24 hours** between readings and still be the same charge, as long as SOC hasn't dropped. The adapter disconnects the moment you walk away, and overnight the next Bluelink refresh can be hours later.
- If SOC went up by **5% or more** between two idle readings within 18 hours, a charge happened that the app never saw. It's filed as an **estimated** session: the duration says "Estimated" and power says "At least".

Each session keeps a reading per poll, so it can show the charge curve, energy added, time taken and peak power.

## Favourite Routes

Numen learns which road you usually take to places you drive often.

1. Trips are grouped by where they start (within 300 m) and end (within 150 m).
2. Within each group, trips are compared by the road they used, on an 80 m grid. Trips that share at least 55% of their cells are the same road.
3. A road becomes a favourite after **4 drives on at least 2 different days**, and only if it's clearly ahead of the others (1.5× the next one). Recent drives count more. Older ones fade with a 45-day half-life, and trips over 180 days old are ignored.
4. In the CarPlay route preview, a route that follows at least **85%** of the favourite is picked and badged.

Two traps that made this silently never fire:

- **Thinned trips.** A long trip keeps fewer points, so two recordings of the same road can differ 8× in point spacing. Comparing only the recorded points scored two identical drives at 0.27. Filling in points along each segment first scores them at 0.87.
- **Shared ends.** Every route to a destination shares the last few streets, which makes all routes look alike. The first and last 300 m are trimmed before comparing.

The grid uses one fixed latitude per route for its longitude spacing. Working it out per point made the cells drift along a long drive.

## Where You're Headed

When CarPlay connects, Numen may suggest a destination. It looks at trips that started near where the car is now (within 750 m, looser than elsewhere, because a trip's first point is recorded after the car is already moving) and at the time of day and day of the week you usually leave.

- Same weekday counts most. Other weekdays count a bit, so a Monday can learn from Tuesdays.
- Weekdays and weekends don't mix. Saturday isn't a quiet Wednesday.

A suggestion you accept goes straight into the route preview, already knowing your favourite road.

## Trip Insights

On iPhones with Apple Intelligence, the commute card writes a short summary on the device with Apple's Foundation Models: the drive time, a charging note (or how much range you'll have to spare), and the traffic. Nothing is sent to a server for this.

# Architecture

<p align="center">
  <img src="../images/home.png" width="240" alt="The Home screen">
</p>

Numen is a SwiftUI app for iOS 26 and later. It has no server of its own for car data. Everything about the car is fetched, decoded and stored on the phone.

## Targets

| Target | What It Does |
|---|---|
| Numen (iPhone) | The app. Also hosts the CarPlay scenes. |
| Watch App | Shows status and sends commands back to the phone over WatchConnectivity. |
| Widget Extension | Home Screen and Lock Screen widgets, and the Climate and Charge Stop Live Activities. |
| Charging Live Activity | The Live Activity shown while the car charges. |
| Share Extension | Takes an address or place from any app's share sheet and sends it to the Trip Planner or straight to CarPlay. |

## Where the Data Comes From

There are three ways to talk to the car, and they don't agree all the time.

1. **[Bluelink](bluelink.md)** works from anywhere. It's the only way to send remote commands (lock, climate, charge). Its status is often the car's last upload, not the car right now.
2. **The [OBD-II adapter](obd-adapter.md)** reads the car directly over Bluetooth. It only works when your phone is near the car, but the numbers are live.
3. **The [harness adapter](harness-adapter.md)** sits on the car's multimedia CAN bus. It starts battery preconditioning and puts directions on the cluster and HUD.

### Which One Wins

When the OBD-II adapter has a reading, it beats Bluelink for SOC, charge power, door locks and charging state. The rule is "newest measurement wins", not "OBD always wins". Each value carries the time it was measured, and Bluelink's own timestamp is compared against the adapter's.

A few specific rules:

- **Lock state.** The adapter's lock bits only override Bluelink when they're newer. If even one door reads unlocked, the car is shown unlocked. Showing "locked" needs all four doors to answer.
- **Charge history.** While the adapter is connected, Bluelink polls are ignored for charge history. A stale "not charging" from Bluelink would otherwise end a session that's still running.
- **Charge power.** There are three tiers. A measured figure (adapter, or a non-zero Bluelink value) is shown as is. Without one, the app shows the rate SOC has actually been climbing at, with a `~` in front. If there's nothing to measure, it shows nothing rather than "0 kW". See [Bluelink quirks](bluelink.md#quirks).

## State

`VehicleViewModel` is the single source of truth on the phone. It's created once at launch and shared with every screen. Every vehicle command goes through it.

It's created in the app delegate, not in a SwiftUI view. A background refresh launches the app with no window, and a view-owned object would never exist in that launch.

## Sharing Between Processes

The app, widgets, Live Activities, Watch bridge and share extension all read one App Group. The app writes a snapshot of status (SOC, range, locks, climate, charge time, target SOC, the last OBD reading) after every refresh. Widgets read that snapshot. They never call Bluelink themselves.

History stores (trips, charges, climate presets) are JSON files in the App Group container rather than UserDefaults. A charge session carries one sample per poll, which is too big for a defaults blob, and widgets can read a file without loading everything.

The share extension writes a destination to the App Group and then either:

- posts a Darwin notification, if CarPlay is connected, so the running CarPlay scene picks it up, or
- opens `myioniq://destination`, so the phone's Trip Planner picks it up.

"Send to Vehicle" holds the destination in the App Group until CarPlay next connects.

## Commands When the Car Is Busy

Bluelink rejects a command while the car is still handling the last one. It also fails when the phone is offline. In both cases the command goes into a queue that's saved to disk.

- A "busy" reply retries after 15 seconds, then 30.
- When the network comes back, the queue is replayed in order.
- If a background refresh hits a busy car, the next background task is rescheduled for when the busy window ends. An in-memory retry wouldn't survive the app being suspended.

## Background Refresh

`BackgroundRefreshManager` schedules a background app refresh at the interval you pick, inside the active hours you set. Both settings live in the App Group so the widget can show when the next refresh is due.

While charging, refreshes also come from the [wake-up service](wake-up-service.md).

## No-Refresh Places

A status refresh wakes the car over its cellular link. In an underground garage that fails and can leave Bluelink calling the car busy for a while. At home it wakes the car for nothing.

You can mark places where Numen leaves the car alone. While the car is parked at one, automatic refreshes read the status Bluelink already has instead of waking the car. A refresh you ask for still goes through. The car's location is whichever is newer: Bluelink's, or where the phone last saw the adapter connected.

## Deep Links

Widgets use `myioniq://action/<action>` to lock, toggle climate or charge, or open a specific screen.

## Diagnostics

Settings › Developer has an activity log, recent network requests, an OBD console and tools for the cluster and CAN sniffer. To show it, tap the version number in Settings seven times.

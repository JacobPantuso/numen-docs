# Apple Platforms

Everything outside the main app: widgets, Live Activities, Apple Watch, Shortcuts, the share sheet and walk-away locking.

None of these talk to Bluelink themselves. They read the status snapshot the app writes to the shared App Group after every refresh, and send commands back through the app. See [Architecture](architecture.md#sharing-between-processes).

## Widgets

**Vehicle Status** comes in small and medium sizes for the Home Screen, plus circular, rectangular and inline for the Lock Screen. It shows SOC, range, lock state, climate and charging.

Buttons on the widget open `myioniq://action/<action>`, which the app handles: lock or unlock, climate on or off, charging on or off, or open a specific screen.

## Live Activities

<p align="center">
  <img src="../images/live-activities.png" width="420" alt="Charging, Climate and Battery Preconditioning Live Activities">
</p>

| Activity | Shows |
|---|---|
| Charging | SOC, charge power, time to your target, and the target |
| Climate | Cabin target, defrost, time left, and whether the car is locked |
| Charge Stop | The charger you're heading to on a CarPlay route: arrival time, distance, power, pack temperature while preconditioning, and time to your target once you're charging |

They show on the Lock Screen, in the Dynamic Island, on Apple Watch and in CarPlay. CarPlay needs nothing extra. iOS shows the activity there on its own.

While charging, the [wake-up service](wake-up-service.md) sends a silent push every 2 to 5 minutes so the app can refresh and keep the Charging activity current. With the OBD-II adapter connected, the app updates it from live readings instead.

## Apple Watch

The Watch app shows the same status as the phone and sends six commands back: lock, unlock, start and stop climate, start and stop charging.

The phone sends its status over WatchConnectivity (`updateApplicationContext`) after every refresh, so the Watch always has the latest even when the phone app isn't open. Commands come back as plain strings (`"lock"`, `"startClimate"`) and run through the same code as a tap on the phone.

## Shortcuts and Siri

| Shortcut | Does |
|---|---|
| Lock Vehicle | Locks, with the walk-away checks below. **Force Lock** skips them. |
| Unlock Vehicle | Unlocks |
| Check Battery Level | SOC and range, from the last status |
| Check Charging Status | Whether the car is charging or plugged in |
| Check Lock Status | Locked or unlocked |
| Check Climate Status | Whether climate is on |
| Check Vehicle Status | SOC, range, what the car is doing, and where it is |
| Find My Car | The car's last known location as a street address |
| Plan EV Trip | Opens the Trip Planner for a destination and checks whether you have the range |

## Walk-Away Locking

Walk-away locking is a Shortcuts automation (for example, when the phone disconnects from CarPlay) that runs **Lock Vehicle**. The shortcut checks a few things before it actually locks:

1. **Cooldown.** An automatic lock within 60 seconds of a successful lock is skipped quietly. A reconnect followed by a disconnect would otherwise fire a second lock.
2. **CarPlay or car audio still connected.** You haven't left.
3. **Beacon**, if you've set one up. A Bluetooth beacon in the car tells the app how far away you are. The lock waits up to about 30 seconds for you to be more than 2.5 m away.
4. **The car itself**, if the OBD-II adapter is in range. An open door or the ignition on holds the lock. See [OBD-II Adapter](obd-adapter.md#doors-and-locks).

Every hold sends a notification. Tapping it locks straight away.

### The Beacon

Any iBeacon works. You enter its UUID, major and minor in Settings, or pick it from a scan.

- Distance readings are noisy, so they're smoothed with a Kalman filter. You count as in the car at 1.5 m or closer, and as out of it at 2.5 m or further. The gap between the two stops the state flipping back and forth.
- Three readings in a row with no beacon in range count as "out of range".
- iOS stops beacon ranging when the app is suspended, so the last reading just sits there. A stale "outside" looks exactly like you walking away. The lock check never reads that saved value. It waits for a reading newer than the request. If none comes, it says so and leaves the decision to the OBD-II check.
- To keep ranging alive in the background, the app runs a low-accuracy location session alongside it. You can turn it off with **Background Ranging** in the auto-lock settings to save battery.

## Share Sheet

Share an address, a place or a map link from any app and pick Numen. The share extension finds the place, shows it on a map to confirm, then:

- if CarPlay is connected, sends it straight to the CarPlay screen, or
- if not, opens the Trip Planner with it.

**Send to Vehicle** holds it until CarPlay next connects.

## CarPlay

Numen runs as a navigation app in CarPlay, with its own map, turn-by-turn directions, lane guidance, speed limits and charging stops. It also has a dashboard view and supports iOS 27's map panels. Routing is covered in [Routing](routing.md), and sending directions to the car's cluster in [Cluster and HUD](cluster-and-hud.md).

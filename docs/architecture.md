# Architecture

<p align="center">
  <img src="../images/home.png" width="240" alt="The Home screen">
</p>

Numen is a SwiftUI application for iOS 27 and later. It has no backend for vehicle data: all vehicle information is retrieved, decoded and stored on the device.

## Targets

| Target | Purpose |
|---|---|
| Numen | The iPhone app. Also hosts the CarPlay scenes. |
| Watch App | Displays vehicle status and relays commands to the iPhone over WatchConnectivity. |
| Widget Extension | Home Screen and Lock Screen widgets, and the Climate and Charge Stop Live Activities. |
| Charging Live Activity | The Live Activity displayed during a charging session. |
| Share Extension | Accepts an address or place from the system share sheet and passes it to the Trip Planner or CarPlay. |

## Data Sources

Numen communicates with the vehicle through three channels:

| Source | Availability | Provides |
|---|---|---|
| [Bluelink](bluelink.md) | Anywhere with a network connection | Remote commands and the vehicle's last uploaded status |
| [OBD-II adapter](obd-adapter.md) | Bluetooth, at the vehicle | Live battery, charging, cabin and body data |
| [Harness adapter](harness-adapter.md) | Bluetooth, at the vehicle | Battery preconditioning, and navigation on the cluster and HUD |

### Source Precedence

Bluelink often reports the vehicle's last upload rather than its current state. When the OBD-II adapter has a reading, values are resolved by measurement time: each value carries the time it was measured, and the newer measurement is used. This applies to SOC, charging state, charge power and door locks.

| Value | Rule |
|---|---|
| Lock state | The adapter's reading is used when it is newer than Bluelink's. A single unlocked door reports the vehicle as unlocked; reporting it locked requires all four doors. |
| Charge history | Bluelink readings are ignored while the adapter is connected, so that a cached "not charging" status cannot end an active session. |
| Charge power | A measured value (adapter, or a non-zero Bluelink value) is shown as reported. Otherwise the observed rate of SOC increase is shown, prefixed with `~`. With neither, no value is shown. See [Bluelink](bluelink.md#known-behaviour). |

## Application State

`VehicleViewModel` is the single source of truth for vehicle state. One instance is created at launch and shared with every screen, and all vehicle commands are issued through it.

The instance is created in the app delegate rather than by a SwiftUI view. Background refresh launches the app without a scene, so an object owned by a view would not exist during those launches.

## Shared Storage

The app, widgets, Live Activities, Watch bridge and share extension share an App Group. After every refresh the app writes a status snapshot (SOC, range, locks, climate, charge time, target SOC and the latest OBD reading). Widgets read this snapshot and never contact Bluelink directly.

Trip, charge and climate preset history is stored as JSON files in the App Group container rather than in UserDefaults. Charge sessions hold one sample per reading, which exceeds what UserDefaults is suited to, and widgets can read a file without loading the full history.

The share extension writes the destination to the App Group, then either posts a Darwin notification to the running CarPlay scene or opens `myioniq://destination` to hand it to the Trip Planner. **Send to Vehicle** holds the destination until CarPlay next connects.

## Command Queue

Bluelink rejects commands while the vehicle is processing a previous one, and commands fail when the device is offline. In both cases the command is added to a persistent queue.

- A busy response is retried after 15 seconds, then after 30 seconds.
- When connectivity returns, queued commands are replayed in order.
- If a background refresh encounters a busy vehicle, the next background task is scheduled for the end of the busy window, since an in-memory retry does not survive suspension.

## Background Refresh

`BackgroundRefreshManager` schedules background app refresh at a user-selected interval within user-selected active hours. Both settings are stored in the App Group so the widget can display the next refresh time. During charging, additional refreshes are triggered by the [wake-up service](wake-up-service.md).

## No-Refresh Places

A status refresh wakes the vehicle over its cellular connection. In underground parking this can fail and leave Bluelink reporting the vehicle as busy; at home it is unnecessary.

Users can designate places where automatic refreshes read Bluelink's cached status instead of waking the vehicle. Manual refreshes are not affected. The vehicle's position is taken from whichever is newer: Bluelink's reported location or the location where the OBD-II adapter was last connected.

## Early Access

During early access, each installation checks in with Numen's server at launch. Installations are admitted while seats are available and queued in order of arrival once they are full; the capacity is raised as the beta grows. The installation identifier is stored in iCloud Keychain, so reinstalling the app or moving to a new iPhone keeps an admitted seat or a place in line. Admitted installations do not depend on the server to open.

## Deep Links

Widgets use `myioniq://action/<action>` to lock or unlock, toggle climate or charging, or open a specific screen.

## Diagnostics

**Settings › Developer** provides an activity log, recent network requests, an OBD console, and tools for the cluster and CAN capture. The menu is enabled by tapping the version number in Settings seven times.

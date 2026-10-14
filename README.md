# Numen Docs

How Numen works under the hood. The website covers what the app does. This repo covers how: where each number comes from, what talks to what, and the parts of the car I had to decode along the way.

Numen is an iPhone, CarPlay and Apple Watch app for Hyundai and Kia E-GMP cars. It's built and tested on an IONIQ 6 AWD.

## Start Here

- [Architecture](docs/architecture.md): the app targets, how data moves between them, and which source wins when two disagree.
- [Privacy](docs/privacy.md): what leaves your phone, where it goes, and what never does.

## Data Sources

- [Bluelink](docs/bluelink.md): remote commands and the car's cloud status, and the quirks I work around.
- [OBD-II Adapter](docs/obd-adapter.md): reading the car directly over Bluetooth. Every PID the app sends, and how each is decoded.
- [Harness Adapter](docs/harness-adapter.md): the WiCAN firmware fork and its commands for battery preconditioning, cluster directions and CAN sniffing.
- [Cluster and HUD](docs/cluster-and-hud.md): the IONIQ 6 navigation frames, byte by byte.

## Features

- [Routing and Charging Stops](docs/routing.md): Mapbox, MapKit, Open Charge Map and how a trip gets its stops.
- [Energy Model](docs/energy-model.md): how Arrival SOC is worked out, and how it learns your car.
- [Trips and Charge History](docs/trips-and-charging.md): recording drives and charges on the phone, learning your routes, and guessing where you're headed.
- [Apple Platforms](docs/apple-platforms.md): widgets, Live Activities, Apple Watch, Shortcuts, the share sheet and walk-away locking.

## Backend

- [Wake-Up Service](docs/wake-up-service.md): the small AWS backend that keeps Live Activities fresh while you charge.
- [Bug Reports](docs/bug-reports.md): how a report from the app becomes a GitHub issue.
- [iCloud Backup](docs/icloud-backup.md): settings, history and sign-in, and how a fresh install gets them back.

## Status

Numen is in TestFlight early access. These docs describe the current build. Anything marked **Unconfirmed** works in testing but hasn't been checked on a real drive yet.

## Credits

- [BetterBlueKit](https://github.com/JacobPantuso/BetterBlueKit) for the Bluelink client
- [OBDb](https://github.com/OBDb) for the IONIQ 6 signal set, and [OVMS](https://github.com/openvehicles/Open-Vehicle-Monitoring-System-3) for the body module layout
- [wicant-i-precondition](https://github.com/L1Z3/wicant-i-precondition), the [Ioniq5_CAN](https://github.com/tylerharvey/Ioniq5_CAN) harness and [WiCAN](https://github.com/meatpiHQ/wican-fw)
- [Mapbox](https://www.mapbox.com) and [Open Charge Map](https://openchargemap.org)

Numen is an independent project and isn't affiliated with or endorsed by Hyundai or Kia.

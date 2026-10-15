<p align="center">
  <img src="images/app-icon.png" width="112" alt="Numen app icon">
</p>

<h1 align="center">Numen Docs</h1>

<p align="center">
  Technical documentation for Numen, a companion app for Hyundai and Kia E-GMP vehicles<br>
  on iPhone, CarPlay and Apple Watch.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/iOS-27%2B-black.svg" alt="iOS 27+">
  <img src="https://img.shields.io/badge/CarPlay-navigation-2f6f3e.svg" alt="CarPlay navigation">
  <img src="https://img.shields.io/badge/TestFlight-early%20access-0D96F6.svg" alt="TestFlight early access">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <a href="https://buymeacoffee.com/jacobpantuso"><img src="https://img.shields.io/badge/Buy%20Me%20a%20Coffee-FFDD00?logo=buymeacoffee&logoColor=black" alt="Buy Me a Coffee"></a>
</p>

<p align="center">
  <img src="images/home.png" width="190" alt="Home">
  <img src="images/charge.png" width="190" alt="Charge">
  <img src="images/navigate.png" width="190" alt="Navigate">
  <img src="images/service.png" width="190" alt="Service">
</p>

These documents describe how Numen works: where its data comes from, how it communicates with the vehicle, and the services it depends on. Product information is available on the website. The protocols and vehicle data described here were developed and verified on an IONIQ 6 AWD.

## Contents

| Document | Description |
|---|---|
| [Architecture](docs/architecture.md) | App targets, data flow between them, and how conflicting data sources are resolved |
| [Bluelink](docs/bluelink.md) | Remote commands, vehicle status, supported regions and known API behaviour |
| [OBD-II Adapter](docs/obd-adapter.md) | Adapter requirements, polling, and the decoding of each PID |
| [Harness Adapter](docs/harness-adapter.md) | The Numen firmware: battery preconditioning, cluster directions and CAN capture |
| [Cluster and HUD](docs/cluster-and-hud.md) | The IONIQ 6 instrument cluster and head-up display navigation frames |
| [Routing and Charging Stops](docs/routing.md) | Route selection, traffic, charger discovery and charge planning |
| [Energy Model](docs/energy-model.md) | Arrival SOC estimation and per-vehicle calibration |
| [Trips and Charge History](docs/trips-and-charging.md) | On-device trip and charge recording, route learning and destination prediction |
| [Apple Platforms](docs/apple-platforms.md) | Widgets, Live Activities, Apple Watch, Shortcuts, the share extension and walk-away locking |
| [Wake-Up Service](docs/wake-up-service.md) | The background push service that keeps charging Live Activities current |
| [Bug Reports](docs/bug-reports.md) | In-app bug reporting and diagnostic redaction |
| [iCloud Backup](docs/icloud-backup.md) | Backup and restore of settings, history and credentials |
| [Privacy](docs/privacy.md) | Data that leaves the device, its destination, and data that does not |

## Status

Numen is in TestFlight early access. These documents describe the current build. Sections marked **Not yet verified** describe behaviour that works in testing but has not yet been confirmed on the road.

## Built On

- [BetterBlueKit](https://github.com/JacobPantuso/BetterBlueKit): the Bluelink client
- [wicant-i-precondition](https://github.com/L1Z3/wicant-i-precondition), the [Ioniq5_CAN](https://github.com/tylerharvey/Ioniq5_CAN) harness and [WiCAN](https://github.com/meatpiHQ/wican-fw): the basis of the Numen firmware

## Support Numen

I love building experiences that make owning an EV a little better, and Numen will always be free because of that. If you are enjoying Numen, a coffee helps keep it going, and I appreciate every bit of support for the time that goes into it.

<p align="center">
  <a href="https://buymeacoffee.com/jacobpantuso"><img src="https://img.shields.io/badge/Buy%20Me%20a%20Coffee-support%20Numen-FFDD00?logo=buymeacoffee&logoColor=black" alt="Buy Me a Coffee"></a>
</p>

<p align="center"><sub>Numen is an independent project and is not affiliated with or endorsed by Hyundai or Kia.</sub></p>

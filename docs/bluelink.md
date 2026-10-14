# Bluelink

Bluelink (Kia Connect on Kias) is the car's cloud service. It's how Numen sends remote commands from anywhere, and it's the fallback for status when the [OBD-II adapter](obd-adapter.md) isn't connected.

Numen talks to it through [BetterBlueKit](https://github.com/JacobPantuso/BetterBlueKit), my fork of an open-source Swift client. The fork adds Kia Canada, location-only fetches, and the Canada fixes I needed for EV status and the surround-view cameras.

## Brands and Regions

You choose a brand and a region when you sign in.

| | Canada | USA | Europe |
|---|:---:|:---:|:---:|
| Hyundai | Tested | Supported | Beta |
| Kia | Untested | Supported | Beta |

Hyundai Canada on an IONIQ 6 is the only combination I've driven with. USA and Europe use the same client code other BetterBlueKit users rely on. Kia Canada is new in my fork and hasn't been tried on a real account yet.

## Signing In

- Your username, password and PIN live in the iOS Keychain, nowhere else. They sync through iCloud Keychain so a reinstall doesn't need you to sign in again. The session token stays on the device.
- Keychain items are readable after first unlock, so a background refresh can sign in while the phone is locked.
- **Canada only:** the Canadian services sit behind Cloudflare. Before signing in, the app loads the login page once in an invisible web view, waits for Cloudflare's cookies (`__cf_bm`, `cf_clearance`), and hands them to the API client. It gives up after 15 seconds.

Every vehicle command asks for Face ID with a passcode fallback, never Face ID alone.

## Commands

Lock, unlock, climate (with temperature, defrost, heated wheel and seats), start and stop charging, charge limits, horn and lights, and the surround-view cameras.

Commands that fail because the car is busy or the phone is offline are queued and retried. See [Architecture](architecture.md#commands-when-the-car-is-busy).

### Climate Suggestions

When you start climate without a preset, the app suggests a cabin temperature from the weather outside. It overshoots on purpose: at −10 °C outside it suggests 24 °C, because the cabin is starting from −10 °C and has about ten minutes to get comfortable. It also suggests defrost and the heated wheel when it's cold enough to need them. Suggestions stay inside the car's own 17–27 °C range.

## Surround View

The car can upload a snapshot from its surround-view cameras. The request looks like a command but isn't one, and treating it like one gets error `6459`.

- `POST /tods/api/rfc/fndmcrsvm` with the same headers the official app uses for its other in-app requests, an empty body, and no PIN.
- The reply is `{"result":{"lastRequestCount":0}}` with no transaction ID to poll.
- The official app finds out the capture is ready by a push notification that third-party apps can't receive. So Numen polls `/rfc/lastmcrsvm` every 10 seconds for up to about 5 minutes until its timestamp is newer than the request, then sends its own "Surround View Ready" notification.

The image is one panorama 4,472 px wide: front, rear, left and right at 960 px each, then the top-down view at 632 px. The app slices it at those widths.

**Unconfirmed.** The request format was captured from the official app. The full round trip hasn't been tested on the car yet.

## Quirks

These are all from Hyundai Canada on an IONIQ 6. Other regions may differ.

- **Status is cached.** A plain status request returns whatever the car last uploaded. Asking for a fresh one wakes the car over cellular, which is slow, uses the car's 12 V battery, and can leave the car busy for a while. That's why [No-Refresh Places](architecture.md#no-refresh-places) exist.
- **Charge power and time remaining read zero.** On an AC charge, `evStatus` says `charging: true` while `batteryFstChrgPower`, `batteryStndChrgPower` and `remainTime2.atc` all come back as `0` for the whole session. The app works out its own rate from how fast SOC climbs instead of showing "0 kW".
- **No trip history.** The USA and Europe APIs have trip details. Canada doesn't. Numen records drives on the phone instead. See [Trips](trips-and-charging.md).
- **Kia models aren't detected.** Bluelink only reports Hyundai model names, so an EV6 or EV9 can't be detected. You pick your model in Settings so the app draws the right car.
- **Lock state goes stale.** Bluelink's lock status is only as fresh as the car's last upload. With the OBD-II adapter connected, the app reads the locks directly. See [OBD-II Adapter](obd-adapter.md#doors-and-locks).

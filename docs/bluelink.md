# Bluelink

Bluelink (Kia Connect for Kia vehicles) is the manufacturer's cloud service. Numen uses it for remote commands and as the source of vehicle status when the [OBD-II adapter](obd-adapter.md) is not connected.

The client is [BetterBlueKit](https://github.com/JacobPantuso/BetterBlueKit), a fork of an open-source Swift Bluelink library. The fork adds Kia Canada, location-only status requests, and fixes to the Canadian EV status and surround-view endpoints.

## Supported Services

The brand and region are selected at sign-in.

| | Canada | USA | Europe |
|---|:---:|:---:|:---:|
| Hyundai | Verified | Supported | Beta |
| Kia | Not yet verified | Supported | Beta |

Hyundai Canada has been verified on an IONIQ 6. The USA and Europe services use the same client code as other BetterBlueKit applications. Kia Canada support is new in the fork and has not yet been tested against a live account.

Bluelink is optional. Users who continue without it run Numen on the OBD-II adapter alone; remote controls are hidden and range is estimated from the vehicle's own trips (see [Energy Model](energy-model.md#range-without-bluelink)).

## Authentication

- The username, password and PIN are stored only in the iOS Keychain. They sync through iCloud Keychain, so reinstalling the app does not require signing in again. The session token is stored on the device only.
- Keychain items are accessible after first unlock, allowing background refresh to authenticate while the device is locked.
- The Canadian services are protected by Cloudflare. Before signing in, the app loads the login page in an invisible web view, waits up to 15 seconds for the `__cf_bm` and `cf_clearance` cookies, and passes them to the API client.

Every vehicle command requires Face ID with passcode fallback.

## Commands

Lock, unlock, climate (temperature, defrost, heated steering wheel and seats), start and stop charging, charge limits, horn and lights, and surround-view capture. Commands that fail because the vehicle is busy or the device is offline are queued and retried; see [Architecture](architecture.md#command-queue).

### Climate Recommendations

When climate is started without a preset, Numen recommends a cabin temperature based on the outside temperature. The recommendation deliberately overshoots: at −10 °C outside it recommends 24 °C, because the cabin starts from the outside temperature and has roughly ten minutes to reach a comfortable level. Defrost and the heated steering wheel are recommended in cold conditions. Recommendations stay within the vehicle's 17–27 °C range.

## Surround View

The vehicle can upload an image from its surround-view cameras. Numen supports it in Canada and the US. No way to request one in Europe is known yet, so the feature is hidden there.

### Canada

The request is not a remote command, and sending it as one returns error `6459`.

| Step | Detail |
|---|---|
| Request | `POST /tods/api/rfc/fndmcrsvm` with the headers used for the official app's in-app requests, an empty body, and no PIN |
| Response | `{"result":{"lastRequestCount":0}}`, with no transaction ID |
| Completion | The official app is notified by a push notification unavailable to third-party apps. Numen polls `/rfc/lastmcrsvm` every 10 seconds for up to five minutes, until the image timestamp is newer than the request, then posts a local notification. |

The image is a single 4,472 px panorama: front, rear, left and right views at 960 px each, followed by the overhead view at 632 px. Numen splits it at these widths.

**Not yet verified.** The request format was captured from the official app; the full round trip has not been confirmed on the vehicle.

### United States

| Brand | Request | Captures |
|---|---|---|
| Hyundai | `POST /ac/v2/svm/findMyCarSVM` | `GET /ac/v2/svm/getSVMDetails`, with the images inline |
| Kia | `POST lbs/svm/req` | `lbs/svm/inquire` lists captures; `lbs/svm/info` returns one capture's image |

After a request, Numen checks the capture list every 15 seconds for up to five minutes. The image is the same 4,472 px panorama as in Canada.

**Not yet verified.** The US flow has not been tested on a vehicle.

## Known Behaviour

Observed on Hyundai Canada with an IONIQ 6. Other regions may differ.

| Behaviour | Effect | Handling |
|---|---|---|
| Status is cached | A standard status request returns the vehicle's last upload. A fresh request wakes the vehicle over cellular, which is slow, draws on the 12 V battery and can leave the vehicle busy. | [No-Refresh Places](architecture.md#no-refresh-places) |
| Charge power and time read zero | During AC charging, `evStatus` reports `charging: true` while `batteryFstChrgPower`, `batteryStndChrgPower` and `remainTime2.atc` remain `0`. | The rate of SOC increase is shown instead of "0 kW". |
| No trip history | The USA and Europe APIs provide trip details; the Canadian API does not. | Trips are recorded on the device; see [Trips](trips-and-charging.md). |
| Kia models not reported | Bluelink reports only Hyundai model names. | The vehicle model is selected in Settings. |
| Lock state is stale | Lock status reflects the vehicle's last upload. | Read directly from the vehicle when the OBD-II adapter is connected; see [OBD-II Adapter](obd-adapter.md#doors-and-locks). |

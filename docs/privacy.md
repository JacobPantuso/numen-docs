# Privacy

Numen has no account of its own, no analytics and no third-party SDKs. Your car's data is fetched, decoded and stored on your phone.

Some features do need other services. This page lists every one, what it gets, and when.

## What Leaves Your Phone

| Service | Gets | When |
|---|---|---|
| Hyundai or Kia (Bluelink) | Your Bluelink sign-in, PIN and commands | Signing in, refreshing status, sending a command |
| Mapbox | Start, stops and destination of a route. Points along a route, for elevation and map matching. | Planning or driving a route |
| Apple Maps | Route endpoints, searches | Routes, ETAs, search |
| Apple WeatherKit | Points along a route, where a trip started and ended | Range estimates, trip temperatures |
| Open Charge Map | The area being searched for chargers | Planning a trip, the CarPlay charger list |
| Apple Foundation Models | Nothing leaves the phone | Trip insights run on the device |
| [Wake-up service](wake-up-service.md) | A Live Activity ID and two push tokens. No car data. | While a charging Live Activity is running. Deleted within 8 hours. |
| [Bug report server](bug-reports.md) | Your report, with redacted diagnostics | Only when you file one |
| Your iCloud | Settings, history and sign-in | [Backup](icloud-backup.md), in your own account |

## What Stays on Your Phone

- Trip history, with routes, speeds and energy
- Charge history
- OBD-II readings, including door, lock and seat belt state
- Learned routes, destination guesses and the energy model's learning
- Beacon readings
- Activity and HTTP logs

These go into your iCloud backup if you have iCloud on. They never go anywhere else.

## Credentials

Your Bluelink username, password and PIN are stored in the iOS Keychain, and nowhere else. They're only ever sent to Hyundai or Kia. Every command asks for Face ID or your passcode first.

## Bug Reports

Before a report leaves the phone, everything that could identify you or your car is removed: VIN, account details, email, PIN, place names, adapter names, coordinates and more. See [Redaction](bug-reports.md#redaction). Reports become GitHub issues, so don't put anything personal in the description.

## The Adapters

The OBD-II and harness adapters talk only to your phone, over Bluetooth. Nothing they read is sent anywhere.

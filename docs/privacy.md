# Privacy

Numen has no user accounts, no analytics and no third-party SDKs. Vehicle data is retrieved, decoded and stored on the device.

Some features rely on external services. This document lists each service, the data it receives, and when.

## Data Sent From the Device

| Service | Data | When |
|---|---|---|
| Hyundai or Kia (Bluelink) | Bluelink credentials, PIN and commands | Signing in, refreshing status, sending commands |
| Mapbox | Route origin, stops and destination; points along a route for elevation and map matching | Planning or navigating a route |
| Apple Maps | Route endpoints and searches | Routing, ETAs and search |
| Apple WeatherKit | Points along a route; trip start and end locations | Range estimates and trip temperatures |
| Open Charge Map | The area being searched | Trip planning and the CarPlay charger list |
| [Wake-up service](wake-up-service.md) | A Live Activity ID and two push tokens | While a charging Live Activity is active; deleted within 8 hours |
| [Bug report server](bug-reports.md) | The report and redacted diagnostics | Only when a report is submitted |
| Early access server | A random installation ID and the app version | At app launch, during early access |
| iCloud | Settings, history and credentials | [Backup](icloud-backup.md), in the user's own account |

Trip insights are generated on the device with Apple's Foundation Models and are not sent anywhere.

## Data Kept on the Device

- Trip history, including routes, speeds and energy
- Charge history
- OBD-II readings, including door, lock and seat belt state
- Learned routes, destination predictions and energy model calibration
- Beacon readings
- Activity and network logs

This data is included in the user's iCloud backup when iCloud is enabled, and is not sent elsewhere.

## Credentials

Bluelink credentials are stored only in the iOS Keychain and are sent only to Hyundai or Kia. Every vehicle command requires Face ID or the device passcode.

## Bug Reports

Identifying information is removed on the device before a report is sent. See [Bug Reports](bug-reports.md#redaction). Reports are published as GitHub issues, so the description should not include personal information.

## Adapters

The OBD-II and harness adapters communicate only with the iPhone over Bluetooth. Their readings are not transmitted elsewhere.

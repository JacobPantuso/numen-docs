# Wake-Up Service

iOS does not allow an app to refresh often enough to keep a charging Live Activity current. The wake-up service sends the device a background push on a schedule, and the app refreshes on receipt.

The service consists of AWS Lambda functions and a DynamoDB table. It receives no vehicle data.

## Operation

1. When a charging Live Activity starts, the app sends `POST /wakeup` with the activity ID, the activity's push token, the device's push token and an interval.
2. A scheduled function runs every minute and sends a background push to each registration that is due.
3. The app wakes, refreshes vehicle status and updates the Live Activity.
4. When the activity ends, the app sends `POST /wakeup/unregister`.

The push is sent to the device token rather than the Live Activity token, as a Live Activity token can update the activity but cannot wake the app.

```json
{ "aps": { "content-available": 1 }, "source": "myioniq-wakeup", "activityId": "…" }
```

## Intervals

| Interval | Push Frequency |
|---|---|
| `charging` | Every 5 minutes |
| `dcfast` | Every 2 minutes |

Wake-ups are paused while the OBD-II adapter is connected, as it already provides live readings.

## Data Retention

Each registration stores the activity ID, the two push tokens, the interval and the registration time. Registrations expire automatically after 8 hours and are deleted immediately if Apple reports the token as invalid (`BadDeviceToken` or `Unregistered`).

The app tracks its own registrations and removes any that remain after it was closed before unregistering.

## Security

Requests are validated against the exact format the app sends, and all routes are rate limited. The service never receives credentials or vehicle data.

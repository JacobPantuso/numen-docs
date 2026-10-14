# Wake-Up Service

iOS doesn't let an app refresh itself often enough to keep a charging Live Activity current. So a small AWS backend sends the phone a silent push on a schedule, and the app refreshes when it gets one.

It's a few small functions on AWS Lambda with a DynamoDB table, and it never sees anything about your car.

## How It Works

1. When a charging Live Activity starts, the app sends `POST /wakeup` with the activity's ID, its push token, the phone's push token and an interval.
2. A Lambda runs every minute. It reads every active registration and sends a silent background push to each one that's due.
3. The app wakes, refreshes status, and updates the Live Activity.
4. When the activity ends, the app sends `POST /wakeup/unregister`.

The push goes to the phone's regular push token, not the Live Activity's. A Live Activity token can update the activity, but it can't wake the app.

The payload:

```json
{ "aps": { "content-available": 1 }, "source": "myioniq-wakeup", "activityId": "…" }
```

## Intervals

| Interval | Push Every |
|---|---|
| `charging` | 5 minutes |
| `dcfast` | 2 minutes |

Wake-ups pause while the OBD-II adapter is connected, since it already has fresher numbers.

## What It Stores

One DynamoDB row per Live Activity: the activity ID, the two push tokens, the interval and when it was registered. Nothing else. Each row expires after **8 hours** on its own.

If Apple says a token is bad (`BadDeviceToken` or `Unregistered`), the row is deleted.

The app keeps its own list of registered activities. If it was closed before it could unregister, it cleans them up next time it's not charging.

## Keeping It Safe

The service only accepts requests shaped exactly like the app's, and it's rate limited. It never receives your credentials or anything about your car.

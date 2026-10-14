# iCloud Backup

Deleting and reinstalling the app used to lose every setting, trip and charge. Now three things are backed up to your own iCloud account, each in the way that suits it.

| What | Where | How |
|---|---|---|
| Settings | iCloud key-value store | Mirrored as they change |
| History | CloudKit, your private database | One snapshot file per store |
| Sign-in | iCloud Keychain | Synced by iOS |

Nothing goes to a server of mine. It's all in your iCloud account.

## Settings

Only an allowlist of settings is synced:

- Anything under `settings.` (units, theme, trip preferences, tires, No-Refresh Places)
- Your car's model and paint
- Trip recording, destination suggestions and route memory switches
- CarPlay shortcuts, pinned places, recent destinations and map orientation
- Cluster directions and auto-precondition switches
- Background refresh interval and active hours
- Bluelink brand and region

Anything not on the list stays on the device, including the paired adapter IDs and widget snapshots. Beacon history and HTTP logs are left out even though they sit under `settings.`. The whole key-value store is capped at 1 MB, and the HTTP log alone filled it on the first sync. For the same reason, any single value over 64 KB is skipped.

### Not Overwriting the Backup

A fresh install has default settings, and pushing those would wipe out the backup it's about to restore. So settings are restored at launch, before anything else starts, and the app doesn't push any changes until one of these happens:

- the restore finished,
- iCloud reported its first sync, or
- 30 seconds passed with no backup found.

## History

Trips, charge sessions and climate presets are each one JSON file. Each is uploaded to CloudKit as a single record (`StoreSnapshot`) with the file attached:

- 20 seconds after a change, so a burst of edits is one upload.
- Skipped if the file's SHA-256 hasn't changed.
- Restored only into an **empty** store. A backup never overwrites history that's already on the phone.

Whole-file snapshots work because the iPhone is the only thing that ever writes history. There's nothing to merge.

Route memory and the energy model's learning aren't backed up. Both are rebuilt from trip history.

## Sign-In

Your Bluelink username, password and PIN are iCloud Keychain items, so they come back on a new install or a new iPhone. The session token stays on the device and is never synced.

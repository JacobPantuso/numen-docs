# iCloud Backup

Numen backs up settings, history and credentials to the user's own iCloud account, so that reinstalling the app or moving to a new iPhone restores them. No data is sent to a Numen server.

| Data | Storage | Method |
|---|---|---|
| Settings | iCloud key-value store | Mirrored on change |
| History | CloudKit private database | One snapshot per store |
| Credentials | iCloud Keychain | Synced by iOS |

## Settings

Only an allowlist of settings is synced:

- Settings under `settings.` (units, theme, trip preferences, tires, No-Refresh Places)
- Vehicle model and paint
- Trip recording, destination suggestion and route memory preferences
- CarPlay shortcuts, pinned places, recent destinations and map orientation
- Cluster directions and automatic preconditioning preferences
- Background refresh interval and active hours
- Bluelink brand and region, and the choice to continue without Bluelink

Device-specific values, such as paired adapter identifiers and widget snapshots, are not synced. Beacon history and HTTP logs are excluded despite their `settings.` prefix, and individual values over 64 KB are skipped, to stay within the key-value store's 1 MB limit.

### Restore Order

Settings are restored at launch before other components start. To prevent a fresh installation's defaults from overwriting the backup, no changes are uploaded until the restore completes, iCloud reports its initial sync, or 30 seconds pass with no backup found.

## History

Trips, charge sessions and climate presets are each stored as a JSON file and backed up as a single CloudKit record (`StoreSnapshot`) with the file attached.

- Uploads occur 20 seconds after a change, combining successive edits.
- Uploads are skipped when the file's SHA-256 is unchanged.
- A backup is restored only into an empty store and never overwrites existing history.

Whole-file snapshots are sufficient because the iPhone is the only writer. Route memory and energy model calibration are not backed up; both are rebuilt from trip history.

## Credentials

The Bluelink username, password and PIN are iCloud Keychain items and are restored with the keychain. The session token is stored on the device only.

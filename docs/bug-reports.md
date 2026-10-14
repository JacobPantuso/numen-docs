# Bug Reports

**Settings › Bug Reports** lists every bug reported from the app, open and recently fixed, and lets you file a new one. Each report becomes a GitHub issue labelled `in-app`.

## How It Works

```
App  ──POST /v1/reports──▶  Bug server  ──GitHub API──▶  Issue (in-app)
App  ◀──GET /v1/issues───  Bug server  ◀──────────────  Issues list
```

The app never holds a GitHub token. A small server sits in between: it holds the token, files the issue, and serves the list back to the app.

## On the Phone

The app builds the whole issue, so every report has the same layout. It has:

- Your title and description
- A summary table: app version, iOS version, iPhone model and car model
- Diagnostics, grouped into App, Device, Vehicle, OBD-II and harness adapters, Features and Permissions. For example: theme and units, Low Power Mode, whether CarPlay is connected, last status time, SOC, range, charging, locks, adapter firmware, and which permissions are granted.
- The last 80 lines of the app's activity log
- Recent HTTP requests, with status codes and timings

Diagnostics are always attached. They're what makes most reports fixable.

### Redaction

Everything is scrubbed on the phone before it's sent. There are two passes:

1. **Known values.** The exact strings this phone holds: your VIN, account ID, email, PIN, Bluelink IDs and keys, adapter names, saved place names, your dealer and your device name. A match is removed anywhere it appears, even where it doesn't look sensitive. Longer values go first, so "123 Main St Toronto" is removed whole before "Toronto".
2. **Patterns.** For anything not known in advance: email addresses, VIN-shaped strings, UUIDs, tokens, coordinates and phone numbers.

## The Server

One Python file using only the standard library, running on a Raspberry Pi behind a Cloudflare tunnel.

| Endpoint | Does |
|---|---|
| `GET /healthz` | Liveness check |
| `GET /v1/issues` | `in-app` issues: every open one, and closed ones from the last 30 days |
| `POST /v1/reports` | Files a report. Needs the app's `X-App-Key` header. |

- It posts the body the app built as is. It doesn't add anything.
- Once GitHub has the issue, the server keeps a copy of the report in SQLite with the issue number. If GitHub is down, the app is told so and nothing is filed.
- New issues are added to the list straight away. GitHub's own list takes a few seconds to catch up.
- **5 reports an hour** per address.
- Titles are capped at 120 characters and text at 5,000.
- The issue list is cached for 5 minutes, so opening the screen doesn't use up GitHub's API limit.

The app key isn't a secret in any real sense. It ships inside the app. It's there to stop casual spam, and the rate limit does the rest.

## Filing by Hand

You can open an issue on GitHub directly too. The issue template asks for the same sections the app fills in.

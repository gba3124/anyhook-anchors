# AnyHook anchors

Daily tamper-evidence anchors for the [AnyHook](https://anyhook.net) event log.

Each file in `anchors/` commits one UTC day of the log to a single SHA-256 value.
Because every day's anchor folds in the previous day's, altering any past event
invalidates every anchor after it — and the commits here are timestamped by
GitHub, not by AnyHook. AnyHook holds no credentials for this repository; a
scheduled workflow pulls the public API and commits what it finds.

That is the whole point: AnyHook can rewrite its own database. It cannot rewrite
this commit history.

## Verifying an event

Everything below is computable from data you already hold — the payload you were
sent — plus the public API. No key and no account are needed.

**1. Leaf.** For each event, with `created_at` as an ISO-8601 UTC string
(`2026-07-28T12:34:56.789Z`):

```
body_hash = sha256(raw_request_body)
leaf      = sha256("{id}|{created_at}|{body_hash}")
```

**2. App root.** Take that app's events for the day ordered by `(created_at, id)`,
and fold. `prev` is the app's most recent earlier root, or `""` if it has none:

```
h = prev
for leaf in leaves:  h = sha256("{h}|{leaf}")
root = h
```

**3. Day anchor.** `GET /api/v1/anchors?day=YYYY-MM-DD` returns that day's
`(app_id, root)` pairs. Sort by `app_id` and fold, seeded with the previous day's
anchor (`""` at genesis):

```
h = sha256("{prev_anchor}|{day}")
for (app_id, root):  h = sha256("{h}|{app_id}|{root}")
anchor = h
```

The result must equal `anchors/{day}.json` in this repository. If it does not,
the log was altered after publication.

## Known limits

- **One-day window.** Days are sealed once, the following day. Tampering inside
  the current, unsealed day is not detectable.
- **Retention bounds individual proof.** Once an event is purged under its plan's
  retention, its leaf can no longer be recomputed. The day's anchor still commits
  the event count and ordering, but proving a specific payload requires that the
  event still exist.
- **Payload commitments start 2026-07-29.** Events written before then have no
  `body_hash`; they are committed by id and timestamp only.

## Endpoints

| | |
|---|---|
| `GET /api/v1/anchors` | the anchor chain, most recent 400 days |
| `GET /api/v1/anchors?day=YYYY-MM-DD` | one day, plus the `(app_id, root)` pairs it folds over |

Both are public and unauthenticated.

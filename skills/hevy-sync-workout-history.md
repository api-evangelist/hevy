---
generated: '2026-08-27'
method: generated
name: Sync a Hevy training history
description: Pull a Hevy account's full workout history once, then keep it current with the since-based change feed instead of re-walking the collection.
api: openapi/hevy-public-api-openapi.json
operations: ['GET /v1/user/info', 'GET /v1/workouts/count', 'GET /v1/workouts', 'GET /v1/workouts/events', 'GET /v1/workouts/{workoutId}']
source: >-
  Grounded in openapi/hevy-public-api-openapi.json, captured 2026-08-27 from
  https://api.hevyapp.com/docs. Every path and method above was verified verbatim in that document.
  NOTE — the published spec declares NO operationId on any operation, so operations are identified by
  method + path. overlays/hevy-public-api-overlay.yaml proposes operationIds; they are OUR proposal,
  not Hevy's, and must not be sent to the API. Auth per authentication/hevy-authentication.yml,
  pagination and error rules per conventions/hevy-conventions.yml and errors/hevy-problem-types.yml.
---

# Sync a Hevy training history

This is the read flow, and it is the one Hevy actually designed for: a backfill followed by a delta loop.

## Auth
- Send the account's key in an `api-key` header. The value is a UUID issued at `https://hevy.com/settings?developer` and it requires an active Hevy Pro subscription.
- Base URL: `https://api.hevyapp.com`.
- One key is the whole account, read and write. There are no scopes and no read-only credential, so an agent doing sync work holds the same authority as one that could overwrite the log. See `authentication/hevy-authentication.yml`.

## Steps

1. **Identify the account** — `GET /v1/user/info`. Returns `id`, `name` and profile `url`. Store the `id` as the key's owner so a rotated or swapped key cannot silently write another person's history into the same cache.
2. **Size the job** — `GET /v1/workouts/count`. This is the only absolute total the API gives; other collections expose `page_count` only. Divide by your page size to know how many requests the backfill will take before you start.
3. **Backfill** — walk `GET /v1/workouts` with `page` starting at 1 and `pageSize=10` (the maximum; the default is 5). Stop when `page` exceeds `page_count`. Each `Workout` arrives complete — `exercises[]` each containing `sets[]` — so no per-workout follow-up is needed.
4. **Record a watermark.** Keep the highest `updated_at` you have seen, as an RFC 3339 timestamp.
5. **Stay current** — `GET /v1/workouts/events?since=<watermark>`. Events come back newest first, paginated the same way, and are one of two shapes: `UpdatedWorkout` (`type` plus the full `Workout`) or `DeletedWorkout` (`type`, `id`, `deleted_at`). Apply updates as upserts and deletions as removals, then advance the watermark. Hevy states the intent plainly in the contract: it exists "to allow clients to keep their local cache of workouts up to date without having to fetch the entire list of workouts."
6. **Fetch one workout when you need it** — `GET /v1/workouts/{workoutId}`.

## Rules an agent must follow

- **The change feed covers workouts and nothing else.** There is no events endpoint for routines, routine folders, exercise templates or body measurements. Those must be re-listed to detect change; budget for it.
- **Page size is capped at 10.** A 2,000-workout history is 200 sequential requests. `GET /v1/exercise_templates` is the one exception at 100 per page.
- **`page` below 1 or `pageSize` above the cap returns 400 "Invalid page size"** with no body explaining which parameter was wrong. Clamp before sending.
- **There is no published rate limit and no rate-limit headers** — see `rate-limits/hevy-rate-limits.yml`. That is not permission to run flat out: the origin is a single Express app behind the Heroku router with no gateway. Pace the backfill and back off on any 5xx.
- **`GET /v1/workouts/events` is the only operation that declares a 500.** It is safe to retry because `since` is a cursor, not an offset — the same call returns the same window.
- **Errors carry no body you can parse.** An invalid key returns HTTP 401 with the bare string `InvalidApiKey` under a `text/html` content type. Calling `.json()` on it throws — Hevy's own web proxy makes exactly that mistake and surfaces `Unexpected token 'I'`. Branch on the status code. See `errors/hevy-problem-types.yml`.
- **A 404 can mean "belongs to someone else."** Cross-account reads are reported as 404, not 403.

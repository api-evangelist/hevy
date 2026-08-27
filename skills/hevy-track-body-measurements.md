---
generated: '2026-08-27'
method: generated
name: Track body measurements in Hevy
description: Read and write a Hevy account's dated body-composition series, using the one endpoint in the API that is naturally replay-safe.
api: openapi/hevy-public-api-openapi.json
operations: ['GET /v1/body_measurements', 'GET /v1/body_measurements/{date}', 'POST /v1/body_measurements', 'PUT /v1/body_measurements/{date}']
source: >-
  Grounded in openapi/hevy-public-api-openapi.json, captured 2026-08-27 from
  https://api.hevyapp.com/docs. Paths, methods, status codes and the BodyMeasurement /
  PutBodyMeasurement schemas verified verbatim; the spec declares no operationIds.
---

# Track body measurements in Hevy

Body measurements are the only Hevy resource keyed by a natural identifier — the date — and that single design choice makes this the safest write surface in the API.

## Auth
- `api-key` header, UUID, Hevy Pro only. Base URL `https://api.hevyapp.com`.

## Steps

1. **List the series** — `GET /v1/body_measurements` with `page` (default 1) and `pageSize` (default 10, max 10). Note this endpoint declares both 400 "Invalid page or pageSize" *and* 404 "Page not found", so a page beyond the end is a 404 rather than an empty list.
2. **Read one date** — `GET /v1/body_measurements/{date}` where `{date}` is `YYYY-MM-DD`. 404 when that date has no entry.
3. **Create an entry** — `POST /v1/body_measurements` with a `BodyMeasurement` body: `date` plus any of `weight_kg`, `lean_mass_kg`, `fat_percent`, `neck_cm`, `shoulder_cm`, `chest_cm`, `left_bicep_cm`, `right_bicep_cm`, `left_forearm_cm`, `right_forearm_cm`, `abdomen`, `waist`, `hips`, `left_thigh`, `right_thigh`, `left_calf`, `right_calf`.
4. **Update an entry** — `PUT /v1/body_measurements/{date}` with `PutBodyMeasurement`. 404 when no entry exists for that date.

## Rules an agent must follow

- **A repeated create is a 409, not a duplicate.** `POST /v1/body_measurements` returns 409 "A measurement for this date already exists" when the date is taken. This is the only replay protection anywhere in the Hevy API — it comes from the natural key, not from an idempotency mechanism — and it means a timed-out create is genuinely safe to retry: you will get either the create or a 409, never two entries.
- **The upsert pattern is: POST, and on 409 switch to PUT** `/v1/body_measurements/{date}`.
- **An entry cannot be removed.** There is no delete. You can overwrite a date's values but you cannot clear the date itself, so writing to the wrong date is permanent.
- **`PUT` replaces the entry.** Read the current values with step 2 first if you intend to change only one field, or you will null out the rest.
- **Watch the unit naming.** `weight_kg`, `lean_mass_kg` and the `*_cm` fields state their unit; `abdomen`, `waist`, `hips`, `left_thigh`, `right_thigh`, `left_calf` and `right_calf` do not. The contract never states their unit — treat them as centimetres to match their siblings, and surface the ambiguity to the user rather than silently converting.
- **This is health data about a person.** Read only what the task needs and do not copy the series into anything the user did not ask for.

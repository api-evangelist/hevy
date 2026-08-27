---
generated: '2026-08-27'
method: generated
name: Log a workout to Hevy
description: Write a completed training session to a Hevy account safely, given an API with no delete, no idempotency key and no test mode.
api: openapi/hevy-public-api-openapi.json
operations: ['GET /v1/workouts', 'POST /v1/workouts', 'GET /v1/workouts/{workoutId}', 'PUT /v1/workouts/{workoutId}', 'GET /v1/exercise_templates', 'GET /v1/exercise_history/{exerciseTemplateId}']
source: >-
  Grounded in openapi/hevy-public-api-openapi.json, captured 2026-08-27 from
  https://api.hevyapp.com/docs. Paths and methods verified verbatim; the spec declares no
  operationIds. Write-safety rules from conventions/hevy-conventions.yml (reversibility block) and
  errors/hevy-problem-types.yml.
---

# Log a workout to Hevy

This is the highest-consequence operation in the Hevy API. A workout you write lands directly in a real person's training log, it counts toward their records and streaks, and **you cannot remove it**.

## Auth
- `api-key` header, UUID, Hevy Pro only. Base URL `https://api.hevyapp.com`.

## Steps

1. **Resolve exercise ids** — `GET /v1/exercise_templates`. Every exercise in the workout needs a real `exercise_template_id`.
2. **Optionally ground the numbers** — `GET /v1/exercise_history/{exerciseTemplateId}` returns the account's set-level history for that exercise (`weight_kg`, `reps`, `rpe`, `set_type`, plus the parent `workout_id`/`workout_title`/`workout_start_time`), with optional `start_date` and `end_date`. This is how you answer "how much did I lift last time" before proposing today's numbers.
3. **Check for a duplicate** — `GET /v1/workouts?page=1&pageSize=10`. Compare `start_time`/`end_time` and `title` against what you are about to write. Do this even on a first attempt; a previous run may have succeeded without you seeing the response.
4. **Create the workout** — `POST /v1/workouts` with `PostWorkoutsRequestBody`: `title`, optional `description`, `start_time` and `end_time` (RFC 3339), and `exercises[]`. Each exercise carries `index`, optional `notes`, `exercise_template_id`, optional `superset_id`, and `sets[]`. Each set carries `index`, `type`, and the metrics its exercise type uses — `weight_kg` and `reps` for weight_reps, `duration_seconds` for duration, `distance_meters` for distance, plus optional `rpe` and `custom_metric`. Returns 201.
5. **Verify** — `GET /v1/workouts/{workoutId}` on the returned id and confirm the set count matches what you sent.
6. **Fix mistakes by overwriting** — `PUT /v1/workouts/{workoutId}` replaces the workout's contents. This is the *only* remedy available; it cannot remove the workout, only change what it says.

## Rules an agent must follow

- **Never retry a `POST /v1/workouts` on an ambiguous failure.** There is no idempotency key, no client-supplied request id and no dedupe window. A timeout followed by a retry produces two identical workouts in the user's log, and the API offers no way to delete either. Re-run step 3 and only re-post if the workout genuinely is not there.
- **There is no sandbox.** No test mode, no test account, no validate-only flag. Do not "try" a call to see what happens.
- **Get confirmation before writing.** The user should approve the exact session — exercises, sets, weights, times — before step 4, because after step 4 the only correction available is an overwrite.
- **Units are fixed by the field name**: `weight_kg` kilograms, `distance_meters` metres, `duration_seconds` seconds. Convert before sending; there is no unit parameter.
- **400 "Invalid request body" tells you nothing else.** No field path, no message. Validate against `PostWorkoutsRequestBody` in the spec before sending.
- **`routine_id` on a Workout is read context, not a write target.** It links a logged session back to the routine it came from.
- **Errors are unparseable.** Branch on status codes; do not attempt to JSON-decode a Hevy error body. See `errors/hevy-problem-types.yml`.

---
generated: '2026-08-27'
method: generated
name: Build a Hevy training program
description: Create a routine folder and populate it with routines built from real exercise template ids — the one flow Hevy itself trusts an agent with, and an irreversible one.
api: openapi/hevy-public-api-openapi.json
operations: ['GET /v1/exercise_templates', 'GET /v1/exercise_templates/{exerciseTemplateId}', 'GET /v1/routine_folders', 'POST /v1/routine_folders', 'GET /v1/routines', 'POST /v1/routines', 'PUT /v1/routines/{routineId}']
source: >-
  Grounded in openapi/hevy-public-api-openapi.json (captured 2026-08-27 from
  https://api.hevyapp.com/docs) and cross-checked against Hevy's own first-party ChatGPT action
  contract at https://github.com/hevyapp/hevy-gpt (saved as
  openapi/hevy-gpt-action-openapi.json). Paths and methods verified verbatim; the public spec
  declares no operationIds. Reversibility and idempotency rules per conventions/hevy-conventions.yml.
---

# Build a Hevy training program

In Hevy, a program is *n* routines inside a folder — that is Hevy's own framing, taken from the instructions it wrote for its ChatGPT Custom GPT. This is the flow the company chose to expose to an agent, so it is the best-supported write path in the API. It is also permanent: nothing created here can be deleted through the API.

## Auth
- `api-key` header, UUID, Hevy Pro only. Base URL `https://api.hevyapp.com`.

## Steps

1. **Resolve real exercise ids first** — `GET /v1/exercise_templates` (`pageSize` up to 100, the highest in the API). Each `ExerciseTemplate` carries `id`, `title`, `type`, `primary_muscle_group`, `secondary_muscle_groups[]`, `equipment_category` and `is_custom`. Ids are short and stable, e.g. `79D0BB3A` = "Bench Press (Barbell)". Use `GET /v1/exercise_templates/{exerciseTemplateId}` to confirm a single one.
2. **Check what already exists** — `GET /v1/routine_folders` and `GET /v1/routines`. Do this every time. Because there is no delete, a duplicate folder or routine created by a retry is permanent.
3. **Create the folder** — `POST /v1/routine_folders` with `PostRoutineFolderRequestBody`. Hevy places every new folder at index 0 and returns the created `RoutineFolder` with a **numeric** `id` (folders are the one entity with a numeric id).
4. **Create each routine** — `POST /v1/routines` with `PostRoutinesRequestBody`, setting `folder_id` to the id from step 3. Each exercise needs a real `exercise_template_id` from step 1, an `index`, and `sets[]` whose fields match the exercise's `type`: `weight_kg` + `reps` for a weight_reps exercise, `duration_seconds` for a duration exercise, `distance_meters` for a distance one. Set `superset_id` only when two exercises are genuinely supersetted; leave it null otherwise.
5. **Correct a routine in place** — `PUT /v1/routines/{routineId}` with `PutRoutinesRequestBody`. This is a full replacement of the routine's contents, not a patch.

## Rules an agent must follow

- **Nothing here can be undone.** There is no `DELETE` verb anywhere in the Hevy API. A folder in particular has no update *and* no delete — a folder created with the wrong name is permanent until a human removes it in the app. Confirm the plan with the user *before* step 3, not after.
- **Creates consume a capped allowance.** `POST /v1/routines` returns 403 "Routine limit exceeded" and `POST /v1/exercise_templates` returns 403 "Exceeds custom exercise limit" when the account hits its plan cap. The caps are not published. Because the objects cannot be deleted via API, a runaway loop can permanently exhaust the allowance.
- **There is no idempotency key and no dry run.** If a `POST` times out, do **not** retry blind — re-list with `GET /v1/routines` or `GET /v1/routine_folders` and create only what is genuinely missing. See `conventions/hevy-conventions.yml`.
- **Invent nothing.** `exercise_template_id` must come from step 1. A made-up id is a 400 "Invalid request body" with no indication of which field failed.
- **Hevy's own agent refuses routine updates.** Its Custom GPT is instructed: "If asked to update an existing routine. Explain you aren't allowed to do that." Step 5 is available in the public API, but treat an update to a routine the user did not create in this session as a destructive act and confirm it.
- **Snapshot before you overwrite.** `PUT` returns no prior state and there is no version history. `GET /v1/routines/{routineId}` first if you want any chance of restoring it.

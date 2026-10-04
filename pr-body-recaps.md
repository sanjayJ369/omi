## What changed and why

Closes #4468 (weekly and monthly recaps). It also covers the ranking half of #3808 (the people you talk to most).

### Summary

Daily Recaps answer "what happened today?" There is nothing that answers "how did my week go?" or "what did this month look like?" without opening seven or thirty daily recaps one by one. This PR adds **weekly (Monday to Sunday) and monthly recaps**.

A period recap is **rolled up deterministically** from the daily recaps that are already stored. It makes **no LLM call**: opening one costs a handful of Firestore reads, returns quickly, never spends the user's chat quota and adds no model cost. Since the inputs are the daily recaps, the period view always agrees with the days it summarizes.

A recap shows:

- **Overview:** conversations, time recorded, tasks and memories for the period. Conversations and time recorded are shown against the previous period ("Previous: 9").
- **Busiest day:** the day with the most recorded time.
- **People you talked to most:** up to five named people, ranked by talk time, with their conversation counts.
- **Highlights:** at most two per day, so one busy day cannot crowd out the rest of the week.
- **Decisions, open questions and open tasks** from the period. Tasks that are already completed are left out.

### User flow

1. Home → Daily Recaps "View All" → the new **Recaps** button (insights icon) in the app bar.
2. The page opens on **This Week**. A chip switches to **This Month**.
3. While loading, the page shows `OmiLoadingState`. A failure shows `OmiErrorState` with **Try Again**, so a failure never appears as an empty recap.
4. If nothing was recorded in the period, the page shows `OmiEmptyState`: "Nothing recorded in this period yet".

### API

`GET /v1/users/recaps/{period}?date=YYYY-MM-DD`

- `period` is `week` or `month`. Any other value returns `422`.
- `date` is any local date inside the wanted period and defaults to today **in the user's timezone**. A malformed date or a date in the future returns `422`.
- Auth is the normal user token (`get_current_user_uid`).

Example response for `GET /v1/users/recaps/week?date=2026-10-01`:

```json
{
  "period": "week",
  "start_date": "2026-09-28",
  "end_date": "2026-10-04",
  "days_recorded": 3,
  "stats": {"total_conversations": 10, "total_duration_minutes": 180, "action_items_created": 3, "memories_created": 6},
  "previous": {"total_conversations": 5, "total_duration_minutes": 100},
  "busiest_day": {"date": "2026-10-01", "summary_id": "…", "total_conversations": 6, "total_duration_minutes": 120},
  "top_people": [{"person_id": "…", "name": "Sam", "conversations": 4, "talk_minutes": 25}],
  "highlights": [{"date": "2026-09-28", "topic": "Vendor quote", "emoji": "💼", "summary": "Agreed to sign this week.", "conversation_ids": ["…"]}],
  "decisions": [{"date": "2026-09-28", "decision": "Move the offsite to March", "conversation_id": "…"}],
  "open_questions": [{"date": "2026-09-28", "question": "Who owns the budget?", "conversation_id": "…"}],
  "open_action_items": [{"date": "2026-09-28", "description": "Send revised numbers", "priority": "high", "source_conversation_id": "…"}]
}
```

Every item carries the `date` it came from and its conversation or summary ID. A later change can link each item back to its day or conversation without an API change.

### How a recap is built (`backend/utils/period_recaps.py`)

Everything in this module is a pure function with no I/O, so the rules below are covered by plain unit tests.

| Part | Rule |
|---|---|
| Week | Monday to Sunday (ISO week) around the anchor date |
| Month | the 1st to the last day of the calendar month, including 29 February in leap years |
| Previous period | the period that contains the day before `start`: the week before, or the calendar month before (January rolls back to December) |
| Days counted | daily recaps whose `date` falls inside the period; one per date (duplicates are ignored); sorted by date |
| Totals | sums of `total_conversations`, `total_duration_minutes`, `action_items_created` and `memories_created`; a missing, negative or non-integer stat counts as 0 |
| Trend | `previous` holds the previous period's conversation and duration totals; it is `null` when the previous period was not requested |
| Busiest day | the most recorded minutes, then the most conversations, then the earliest date; `null` when no day has any activity |
| Highlights | up to 2 per day, in date order, 10 in total; entries with neither a topic nor a summary are skipped |
| Decisions / open questions | in date order, 10 of each; blank or malformed entries are skipped |
| Open tasks | action items not marked `completed: true`, in date order, at most 10 |
| People | ranked by talk seconds, then conversation count, then ID (stable); people with no name are left out; top 5; talk time is rounded up to whole minutes |

The builder is defensive against malformed daily-summary documents: a non-dict `stats` field, list entries that are strings, `None` topics and similar inputs are skipped rather than raising. One bad document from an older summarizer version cannot break the whole recap.

### Where the data comes from (`backend/routers/recaps.py`)

| Read | Bound | Notes |
|---|---|---|
| Daily summaries | **one** query, `limit=62` | covers the period and the previous one together, from the previous period's start to the period's end; 62 is two 31-day months, so a month and its previous month always fit |
| Conversations for people stats | `recap_people_scan`, capped at **500** conversations | the People-stats projection (IDs, visibility flags, lock gate, transcript and speaker-assignment fields only; no titles or structured content), filtered by `discarded == false` and a `created_at` range, under the request-scoped `conversation_scan_budget` |
| People names | one read of the user's People list | only names of people who appear in the period are kept |
| LLM | **none** | |

- **Timezone.** The period is a range of local calendar dates. `period_utc_bounds` turns the first and last local moments into UTC instants in the user's stored timezone, which handles DST transitions inside the period. An unknown or missing timezone falls back to UTC.
- **People reuse existing code.** The ranking comes from `aggregate_people_stats` in `utils/people_stats.py`, the same aggregator behind the People stats in the app. `recap_people_scan` is a new recipe in `database/conversation_scan.py` over the shared bounded reader (`iter_conversations`). It uses the same `PEOPLE_STATS_FIELD_PATHS` projection and batch size, adds date bounds, and lowers the cap to 500.
- **Failure handling.** If the people scan fails for any reason (Firestore error, budget exhausted), the route still serves the recap without people, records the fallback with `record_fallback(component='period_recap', from_mode='with_people', to_mode='without_people', outcome='degraded')`, and logs only the error class. A daily-summaries read failure fails the request, which the app shows as an error with Try Again.
- **Firestore indexes.** `recap_people_scan` uses the same `discarded == false` plus `created_at` range shape that `speaker_browse_scan` already uses, so **no new composite index** is needed. The daily-summaries query filters and orders on the single `date` field, using the existing `start_date`/`end_date` parameters of `database/daily_summaries.get_daily_summaries`. Firestore's automatic single-field index serves it.
- **Firestore query-shape registries.** The new recipe is registered in `tests/support/firestore_conversation_profiles.py`, `firestore_query_driver_registry.py` and `firestore_caller_witnesses.py`. Two witnesses drive the real route and the recipe, so the index guard covers the new caller.
- **Route policy.** The route has a reviewed `route_policy_manifest.yaml` entry. It lives in a new router file (included in `main.py`) because `routers/users.py` is over the line-count threshold.

### Models and generated clients

- `backend/models/period_recap.py`: `PeriodRecapResponse` with typed sub-models for stats, previous totals, busiest day, highlights, decisions, questions, action items and people. Every list defaults to empty and every optional field to `null`, so clients never see a missing key.
- The schemas are adopted into the `users` Dart wire group (`scripts/generate_dart_models.py`). The app decodes the response through the generated `GeneratedPeriodRecapResponse`, with no hand-written JSON parsing.
- Regenerated `docs/api-reference/app-client-openapi.json` plus the generated Dart wire models, TypeScript clients (web app, admin, personas, Windows desktop) and the macOS Swift client. The change is additive: one new route and new schemas. The public and integration contracts are unchanged.

### App changes

- **`lib/backend/http/api/recaps.dart`:**
  - `RecapPeriod` enum and `periodRecapUrl(period, {date, baseUrl})`.
  - `getPeriodRecap(period, {date, send, baseUrl})` returns an `ApiResult` through `executeApi`. A 5xx becomes `ApiFailure(server)` and a non-object body becomes `ApiFailure(decode)`. A failure is never turned into an empty recap.
- **`lib/pages/conversations/period_recap_page.dart` (new):**
  - Built only from shared `omi/ui` components: `OmiFilterChip`, `OmiSettingsGroup` / `OmiSettingsRow`, `OmiLoadingState`, `OmiErrorState`, `OmiEmptyState`, `OmiDuration`, `OmiDateFormat` and `OmiBackButton`.
  - The loader is injectable for tests.
  - Switching periods quickly cannot show a stale result: a response for a period that is no longer selected is dropped.
- **`daily_recaps_page.dart`:** a **Recaps** `OmiIconButton` in the app bar opens the new page. The loader is injectable here too, for the widget test.
- **Strings:** six new keys (`thisWeek`, `busiestDay`, `timeRecorded`, `peopleYouTalkedToMost`, `recapPrevious`, `noRecapForPeriod`) in all 49 locales via `scripts/l10n.py add`. Everything else reuses existing strings.
- **Changelog:** fragment `app/changelog/unreleased/20261004-weekly-monthly-recaps.json`.

### Design decisions

- **Roll-up instead of a new LLM summary.**
  - The daily recaps already contain the LLM-written highlights, decisions and tasks, so a weekly LLM pass would mostly re-summarize summaries.
  - It would also add latency, model cost and quota spend every time the page is opened, and it could disagree with the daily recaps.
  - A deterministic roll-up is free to open, testable, and always consistent with the days. A short AI-written "week in one paragraph" can be added later as an opt-in on top of this payload.
- **Computed on read, not stored.** There is no new collection, no cron job and no backfill. A recap is always current, including the current, unfinished week or month.
- **ISO weeks (Monday start).** This matches the ISO standard and most calendars. A locale-dependent first day of the week could be added later as a query parameter.
- **People from conversations, not from daily recaps.** Daily recaps do not store per-person talk time. The People-stats projection already computes it, respecting locked conversations, data-protection levels and manual speaker reassignments.
- **New router file.** `routers/users.py` is over the line-count ratchet, so the route lives in `routers/recaps.py` under the same `/v1/users/` prefix.

### Known limitations

- A period recap can only include days that have a daily recap. Days without one (for example, before the user enabled daily recaps) count as not recorded. The people section is computed from conversations directly, so it can still show people for those days.
- People stats cover at most 500 conversations per period. That is far above typical use for a week, and for most users for a month. Beyond the cap, the ranking reflects the first 500 conversations in the period.

### Out of scope and possible follow-ups

- Tapping a highlight, decision or task to open its day or conversation. The IDs are already in the payload.
- Browsing past weeks and months. The API already accepts any past `date`; the app shows the current period only.
- An optional AI-written one-paragraph summary of the period, metered like other AI features.
- A weekly recap push notification on Sunday evening, reusing this endpoint.

## Product invariants affected

none

## How it was verified

- **Red first.** Every test file was written before the code:
  - `test_period_recaps.py` failed to import `utils.period_recaps`.
  - The route test failed to import `routers.recaps`.
  - The Dart tests failed to compile (no `recaps.dart` or `period_recap_page.dart`).
- **Backend.** These all pass through `backend/test.sh`:
  - `test_period_recaps.py` (11), `test_period_recaps_route.py` (7) and `test_app_client_schema_inventory.py`.
  - Every test file touching `conversation_scan`, `people_stats` or `generate_dart_models`: 12 files, 273 tests.
  - The Firestore guards: `test_firestore_caller_witnesses.py`, `test_firestore_query_shapes.py`, `test_firestore_outside_query_contract.py` and `test_conversation_scan.py`: 531 tests.
- **Contracts:**
  - `export_openapi.py --check` passes for the public, app-client and integration surfaces, after regenerating the app-client one.
  - `generate_dart_models.py --all --check`, `generate_ts_openapi_types.py --check` and `generate_swift_openapi_types.py --check` pass.
  - The route-policy baseline is current. I ran its `route_policy_inventory.py --enforce-missing-baseline` command directly, because my dev environment blocks the runner's tiktoken warm-up download.
- **App:**
  - `flutter test test/unit/period_recap_api_test.dart test/widgets/period_recap_page_test.dart test/widgets/daily_recaps_page_test.dart`: 13 passed.
  - `flutter analyze` on the touched files: no issues.
  - `check_mobile_ux_contract.py --report`: both pages are clean.
  - `scripts/l10n.py check`: 49 locales.
- **`scripts/pr-preflight --pr-body-file`**: every check passes except `backend-route-policy-baseline`, which needs that blocked download and was verified directly as above.
- **Not verified:** I did not run the app on a device against a deployed backend, or against real users' Firestore data. The people scan reuses the People-stats projection and the bounded reader that the People list already uses in production.

## Tests

- `backend/tests/unit/test_period_recaps.py`:
  - **Periods:** week bounds around any anchor day (Monday anchor); month bounds including a leap February; the previous week and month, including January → December; an unknown period is rejected.
  - **Totals:** period totals and days recorded; days outside the period are never counted; the busiest day.
  - **Content:** highlights capped at two per day; decisions, open questions and open tasks keep their day; completed tasks are dropped.
  - **Caps and malformed data:** lists are capped at 10, and blank, non-dict and malformed entries (including a non-dict `stats`) are skipped.
  - **Trend:** the previous period's totals.
  - **People:** ranked by talk time, people with no name are left out, and talk time is rounded up to minutes.
  - **Edge cases:** an empty period is a valid recap; local period bounds become the correct UTC instants in the user's timezone.
- `backend/tests/routers/test_period_recaps_route.py`:
  - One daily-summaries query covers both periods.
  - The people-scan bounds are in the user's timezone.
  - `date` defaults to today in the user's timezone (the month period).
  - Future and malformed dates, and an unknown period, return 422.
  - A failed people scan still serves the recap and records the fallback.
- `app/test/unit/period_recap_api_test.dart`: URL building, decoding, a 503 becomes `ApiFailure(server)`, and a non-object body becomes `ApiFailure(decode)`.
- `app/test/widgets/period_recap_page_test.dart`:
  - Content, trend, busiest day, people and the period's highlights, decisions, questions and tasks.
  - Switching to the month loads it.
  - An error, then Try Again, reloads.
  - The empty state.
  - The Daily Recaps entry point opens the recap.

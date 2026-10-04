## What changed and why

### Summary

Settings → Import has supported only **Limitless**, with an "Other devices coming soon" placeholder underneath. This PR adds a general **Transcript files** importer. Someone switching to Omi from another recorder or meeting tool can bring their history as `.srt`, `.vtt` or `.txt` transcripts, either one file or a `.zip` of many. Every transcript becomes a completed Omi conversation, so their past conversations are searchable and available to chat from the first day, without re-recording anything.

It is a **light import**, the same model as the existing Limitless importer: parse and store the transcript, with no AI processing. That makes it fast (no LLM latency per file), adds no model cost per imported conversation, and makes it safe to run twice.

### User flow

1. Settings → Import → **Transcript files** ("Select SRT, VTT or TXT transcripts, or a ZIP of them").
2. Pick one `.srt` / `.vtt` / `.txt` file or a `.zip`. The picker only offers those four extensions.
3. The app uploads it with the user's primary language and the device's IANA timezone. The server answers immediately with a `pending` job.
4. The job row in import history shows progress, then a completed or failed state with created and skipped counts, and a push arrives when it finishes. Transcript jobs get their own icon instead of the Limitless logo.
5. The imported conversations appear in the normal conversation list, dated by the recording time read from each file name, or by the time recorded in the ZIP entry.

### API

`POST /v1/import/transcripts?language=en&tz=America/New_York&origin=other`, multipart with field `file`:

```json
{"job_id": "3f0c…", "status": "pending", "source_type": "transcript_files"}
```

- Allowed `file` types: `.zip`, `.srt`, `.vtt`, `.txt`. Anything else returns `400 Upload a .zip, .srt, .vtt or .txt file`, and no job is created.
- `origin`: `plaud` or `other`. An unknown value returns `400`.
- `tz` is used only to read date-times written in file names.
- Progress goes through the existing `GET /v1/import/jobs` and `GET /v1/import/jobs/{job_id}` routes. Cancel and delete use the existing job routes.
- `ImportJobResponse.source_type` is a new **optional, additive** field on list, status, start and cancel, so clients can tell importers apart. Older stored jobs and unknown values serialize as `null`.

### How files are parsed (`backend/utils/imports/transcript_files.py`)

| Shape | Recognised by | Speaker | Times |
|---|---|---|---|
| **SRT** | numbered cues with `HH:MM:SS,mmm --> HH:MM:SS,mmm`; multi-line cue text joined | `Name: text` prefix (rules below) | from the cue |
| **WebVTT** | `WEBVTT` header; `NOTE` / `STYLE` / `REGION` blocks and cue IDs skipped; cue settings ignored; inline tags (`<c>`, timestamps) stripped; HTML entities unescaped | `<v Name>` voice tag, else the `Name:` prefix | from the cue (`MM:SS.mmm` or `HH:MM:SS.mmm`) |
| **Text, speaker headers** | a line like `Jane Doe  0:03` or `[0:03] Jane Doe`, followed by paragraph lines | the header | the header's timestamp |
| **Text, inline timestamps** | `[00:00:02] Jane: text`, used when at least half the lines start with a timestamp | the `Name:` prefix | the line's timestamp |
| **Text, paragraphs** | blank-line separated paragraphs | none | estimated at about 150 words per minute so turns keep their order |

- **Content wins over extension.** A `.txt` that contains SRT timings, or that starts with `WEBVTT`, is parsed as SRT or WebVTT.
- **Encoding.** Files are read as UTF-8 (BOM allowed), falling back to cp1252. A file containing NUL bytes is rejected as binary.
- **Speaker labels are conservative.** A `Name: text` prefix becomes a speaker only when one of these holds:
  - the file is mostly labeled (at least 60% of cues)
  - the label recurs
  - it is a generic `Speaker N`, `Participant N` or `Person N` label

  This keeps a one-off `Note: bring the contract` as text instead of inventing a speaker called "Note".
- **People.** Speakers get `speaker_id`s in order of first appearance. A label matching the account owner's profile name sets `is_user`. A label matching one of the user's **People**, compared case-insensitively, sets `person_id`, so imported conversations are attributed the same way live ones are.
- **Times.** A missing end time closes at the next turn. Turns never go backwards. `finished_at` is the start time plus the last segment's end.
- **Start time** comes from a date-time in the file name, read in `tz`: `2026-09-12 14_30_05 …`, `20260912_143005`, or `…2026-09-12T09-15`. An impossible date (`2026-02-30`) is ignored. Otherwise the ZIP entry's timestamp is used, then the upload time.
- **Title** is the file name with the date and extension removed and `_`/`-` turned into spaces (`2026-09-12 14_30_05 Vendor quote call.srt` becomes "Vendor quote call"). The fallback is "Imported transcript".

### What each conversation contains

| Field | Value |
|---|---|
| `id` | `document_id_from_seed('transcript-file:{uid}:{sha256(file bytes)}')`, deterministic per user and per file |
| `source` | `plaud` when `origin=plaud`, otherwise `unknown` |
| `structured.title` / `overview` | title from the file name; overview is the opening ~500 characters of the transcript |
| `structured.category` / `emoji` | `other` / 💬 (same as Limitless imports) |
| `transcript_segments` | parsed turns with `speaker`, `speaker_id`, `is_user`, `person_id`, `start` and `end` |
| `status` / `imported` / `discarded` | `completed` / `true` / `false` |

Conversations are written through the existing create-if-absent `lifecycle.persist_imported_conversation`, so re-importing the same file is skipped and never overwrites user edits ("first import wins").

### Limits and safety

| Guard | Limit | Behaviour |
|---|---|---|
| Upload size | existing `IMPORT_MAX_PART_SIZE` (100 MB) | rejected by the multipart limiter |
| Transcripts per archive | 1,000 | the job fails with "An import can contain at most 1000 transcript files…" before anything is read |
| Total uncompressed transcript bytes | 200 MB | the job fails before reading, which guards against ZIP bombs |
| Per file | 5 MB, read with a bounded `read(limit + 1)` | that file is skipped and counted; the rest still import |
| Archive members | only `.srt/.vtt/.txt`; `__MACOSX/` and dotfiles skipped | audio, images and documents in the ZIP are ignored |
| Staged filename | reduced to its basename | `../../etc/evil.srt` cannot leave the temp directory |
| Cleanup | the staged upload is deleted in `finally` | also after failures and cancellation |

- Files are parsed in memory from the staged upload. Nothing is sent to third parties.
- Logs record job IDs and error class names only, never transcript text or file names.

### Job lifecycle

This mirrors the Limitless worker:

- `pending` → `processing` → `completed` or `failed`
- progress updates every 10 files
- a user cancel stops the work and is never overwritten by a final status
- a partial failure stays `completed`, with "N file(s) could not be imported"
- a failure with nothing imported is `failed`, with the first reason
- the push notification is best effort and cannot change a committed status

### App changes

- **Import page.** A Transcript files card, built with the page's existing source-card builder, which now also accepts an icon instead of a logo.
- **Shared flow.** The Limitless and transcript flows share one `_startImport(...)` path: picker, upload, refresh and retry, so their behaviour stays identical.
- **Job rows.** `importJobSourceIcon` shows the Limitless logo only for Limitless jobs. Jobs from before `source_type` existed decode as Limitless, which all of them were.
- **API client.** `startTranscriptImport`, `transcriptImportUrl` and `transcriptImportExtensions` live in `lib/backend/http/api/imports.dart`. The job model decodes `source_type` through the regenerated wire DTO.
- **Strings.** Two new keys (`importTranscriptFiles`, `importTranscriptFilesDescription`) in all 49 locales via `scripts/l10n.py add`.
- **Changelog.** Fragment `app/changelog/unreleased/20261004-import-transcript-files.json`.

### Design decisions

- **No AI processing on import.** This matches Limitless and keeps a 1,000-file import from creating 1,000 LLM calls. Users can still reprocess an individual imported conversation with the existing reprocess flow.
- **No new `ConversationSource` value.** `plaud` and `unknown` already exist and have no special processing branches, and a new enum value is something every released client decoder would have to handle. `external_integration` was avoided because reprocessing and memory extraction expect its `external_data` text.
- **Content-hash IDs.** Unlike Limitless lifelogs, transcript exports have no stable recording ID, so the file's bytes are the identity. Re-importing the same export is a no-op. An edited file is treated as a new transcript.
- **Generic formats instead of vendor-specific parsers.** SRT, WebVTT and timestamped text are what most recorders and meeting tools export. Vendor-specific formats (for example DOCX exports) can be added later as more parsers behind the same job.

### Out of scope and possible follow-ups

- Audio-file import is handled separately in #17854. This PR imports transcripts only.
- An opt-in "summarize my imported conversations" pass, metered like other AI features.
- A "Delete imported transcripts" action on the import page, mirroring the existing Limitless bulk delete.
- More vendor-specific formats (DOCX/PDF transcript exports).

## Product invariants affected

none

## How it was verified

- **Red first.** The tests were written before the implementation:
  - `test_transcript_file_import.py` failed to import (no `utils.imports.transcript_files`).
  - The route tests failed on the missing `create_transcript_import_job`.
  - The `source_type` tests failed on the missing response field.
  - The Dart tests failed to compile (no `ImportJobSource`, `transcriptImportUrl`, `transcriptImportExtensions` or `importJobSourceIcon`).
- **Backend.** These files all pass through `backend/test.sh`:
  - `test_transcript_file_import.py` (29)
  - `test_import_transcripts_route.py` (12)
  - `test_imports.py` (15)
  - `test_limitless_import_idempotency.py` (12)
  - `test_import_jobs_malformed.py`, `test_import_job_status_detail_enum.py`, `test_app_client_schema_inventory.py`, `test_multipart_limits.py`
- **Contracts:**
  - `export_openapi.py --check` passes for the public, app-client and integration surfaces, after regenerating `app-client-openapi.json`.
  - `generate_dart_models.py --all --check`, `generate_ts_openapi_types.py --check` and `generate_swift_openapi_types.py --check` pass.
  - `check_app_client_openapi_compatibility.py --base-ref origin/main` passes, and so does `scripts/check_client_compat.py`.
  - The route-policy baseline is current. I ran its `route_policy_inventory.py --enforce-missing-baseline` command directly, because my dev environment blocks the runner's tiktoken warm-up download.
- **App:**
  - `flutter test test/unit/import_transcript_files_test.dart test/unit/import_job_timestamp_test.dart`: 14 passed.
  - `flutter analyze` on the touched files: no issues.
  - `python3 scripts/l10n.py check`: 49 locales.
- **`scripts/pr-preflight --pr-body-file`**: every check passes except `backend-route-policy-baseline`, which needs that blocked download and was verified directly as above.
- **Not verified:**
  - I did not run the app on a device against a deployed backend. The picker → upload → poll flow reuses the Limitless code path end to end.
  - I did not test real exports from specific vendors. The parsers cover the SRT, WebVTT and text shapes above and are tested on representative samples.

## Tests

- `backend/tests/unit/test_transcript_file_import.py`:
  - **Parsing:** SRT (BOM, CRLF, multi-line cues, recurring speakers); WebVTT (header, NOTE, cue IDs, settings, voice tags, inline tags); text with speaker headers (including `H:MM:SS`), inline timestamps and plain paragraphs.
  - **Speaker labels:** a one-off `Note:` in an unlabeled file stays text, while `Speaker 3:` is a speaker.
  - **Rejected files:** binary, empty and unsupported files are not transcripts.
  - **File names:** date and timezone parsing, including an impossible date and no date; title cleanup.
  - **Segments:** speaker numbering, owner (`is_user`) and People (`person_id`) binding; estimated times for untimed text; end times closing at the next turn.
  - **IDs:** deterministic per user and per content.
  - **End to end with a fake store:**
    - A ZIP with three transcripts plus an MP3 and a `__MACOSX` fork creates exactly three completed conversations with the right fields, sends the completion push and deletes the upload.
    - Re-import skips everything.
    - A single-file upload with `origin=plaud` works.
    - No transcripts fails with a clear message.
    - An oversized member is skipped while the rest import.
    - The archive member and size budget rejects before reading.
    - Cancellation stops creating conversations and writes no final status.
    - Jobs carry their own source type.
- `backend/tests/routers/test_import_transcripts_route.py`:
  - Unsupported uploads (`.mp3`, `.docx`, no extension, empty name) and unknown origins are rejected before a job exists.
  - Each supported type is staged byte-for-byte and queued with its language, timezone and origin.
  - Path components in the filename never leave the temp directory.
  - The job list and job status report `source_type` (legacy and unknown values become `null`).
- `app/test/unit/import_transcript_files_test.dart`:
  - `source_type` decoding (legacy and unknown importers).
  - Upload URL encoding and defaults.
  - Accepted extensions.
  - The job-row icon choice.

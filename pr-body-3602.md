## What changed and why

Closes #3602. General (non-meeting) conversation titles now name the identified people in the conversation, and only names that are correct.

Notes v2 already binds speaker clusters to people through hard evidence (voice profile or a tagged person). The general notes path never asked for those names in the title, and the model could not tell which bound name was the account owner. So titles rarely said who the conversation was with, and when they named someone it could be the owner.

- **`SpeakerMap`** (`utils/conversations/transcript_for_llm.py`): the cluster→name map now also records the owner's clusters and the words each cluster spoke. It is still a plain mapping for every existing caller, and the rendered `spk` lines are unchanged, so shared-prefix bytes and cache keys don't move.
- **`ConversationPromptPrefix.title_people` / `owner_names`**: non-owner people who said at least 10% of the words, most-spoken first, plus the owner's name.
  - A bare mapping, where the owner is unknown, names nobody.
  - Rich meeting notes (roster path) are unchanged and keep their own "lead with their name" rules.
- **`utils/llm/conversation_title_people.py`** (new):
  - A static `TITLE` block, inserted before the parser schema, cacheable and on the general path only. It follows the same pattern as `rich_static_instructions`: name the one or two central people, never the account owner, never an unlisted or invented name.
  - A volatile `PEOPLE IN THIS CONVERSATION` block that carries the names.
- **Presentation contract** (`enforce_conversation_note_presentation`): if the title names none of the identified people, it is led by them: `Sarah Chen: Q2 Budget Review`.
  - Language-neutral `Name: Title` form.
  - At most two names, and the second is dropped rather than exceed 70 characters.
  - Never invents a title or a name.
  - Each repair is counted on the existing `omi_conversation_note_presentation_total` (`static_repair` / `title_people_lead`), so the model's miss rate is visible.

The earlier attempt (#8497) was closed only for conflicts with #10796. This PR does not touch `process_conversation.py`: owner identity rides on the speaker map that the existing call sites already pass through.

Line-Count-Exception: backend/utils/llm/conversation_processing.py | 1961 -> 1965 | four call-site lines wire the new title module in (static rules, volatile people block, title_people into the presentation contract); the logic itself lives in conversation_title_people.py

## Product invariants affected

none

## How it was verified

- Red first: `tests/unit/test_conversation_title_people.py` written before the change. Through `backend/test.sh`: **14 failed, 3 passed**. The 3 that passed are the "title that already names the person is kept" cases, which pin existing good behavior.
- Green: the same file, **17 passed**.
- Related suites through `backend/test.sh`, all passing:
  - `test_conversation_notes_v2.py` (26)
  - `test_compact_speaker_transcript.py` (7)
  - `test_llm_feature_prompt_cache_prefix.py` (3): the static notes prefix stays byte-identical across conversations.
  - `test_meeting_notes_rich_context.py` (84)
- Every backend test file that imports the touched modules (140 files) through `backend/test.sh`: **3886 passed, 1 failed**. The one failure, `test_verify_pusher_promotion_evidence.py`, shells out to `helm`, which is not installed in my environment. It is unrelated to this change.
- `make preflight` / `scripts/pr-preflight --pr-body-file`: every selected check passes. `backend-route-policy-baseline` needed one workaround because my dev environment blocks the `openapi_runner.sh` tiktoken warm-up download. I ran the same `route_policy_inventory.py --enforce-missing-baseline` command against `origin/main`'s baseline with the OpenAPI venv directly: `route policy missing-route baseline is current`. `export_openapi.py --check` reports both committed contracts up to date.
- Not verified: live model output. I had no provider credentials, so the prompt wording is checked by tests and the inclusion guarantee rests on the deterministic guard, not on model compliance.

## Tests

`backend/tests/unit/test_conversation_title_people.py`:

- **Speaker map:** records the owner's clusters and words per cluster, and still compares equal to the plain dict.
- **Who gets named:** title people exclude the owner and minor speakers and are ordered by talk share; a bare mapping names nobody; owner-only conversations name nobody.
- **Prompt cache:** the shared prefix `context` bytes are unchanged; title rules are static while names stay volatile.
- **Title guard:**
  - A title with no identified name is led by the name.
  - Titles that already name the person by full name, first name or lowercase mention are kept.
  - The lead is language-neutral and joins at most two people within 70 characters.
  - It never invents a title or a name.
  - Whole-word matching escapes punctuation.
- **Metric:** the repair is counted.
- **Rich path:** rich meeting notes are untouched.

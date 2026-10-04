## What changed and why

Fixes #20500. Ring-custody checkpoint writes are serialized on `PendantRingCustody._saveQueue` and `PendantCustodyStore._ioQueue`. Mismatch invalidation and ring reincarnation persist them fire-and-forget, and nothing let a caller wait for queued writes.

The tests delete their custody directory after a fixed 50 ms `_settle()`. Under CI load a queued write (for example the replay advance ACK → `_persist`) was still pending. Its `.json.tmp` → `rename` then ran after teardown and failed the suite with `PathNotFoundException` "after it has completed", on main and on unrelated PRs.

- **`PendantRingCustody.flush()` / `PendantCustodyStore.flush()`**: complete once every write queued so far has landed, re-checking for writes queued while waiting. It is an additive API with no behavior change for existing callers.
- **The three flaky suites** now wait for queued writes before deleting their storage:
  - `omi_ring_custody_test.dart`: end the custody session with `disconnect()`, which is synchronous, then `flush()`, then delete. Once the session ends, no new write can be queued.
  - `ring_custody_download_test.dart`: flush every custody it creates before removing the `path_provider` mock.
  - `pendant_custody_main_red_test.dart`: flush `PendantRingCustody.shared` before removing the mock and deleting `tempDir`.

## Product invariants affected

none

## How it was verified

All runs used Flutter 3.44.5, the same as CI.

- Red first: `test/unit/pendant_ring_custody_flush_test.dart` written before the fix fails to compile, because `flush()` does not exist on `PendantRingCustody` or `PendantCustodyStore`.
- Green: `TZ=UTC flutter test test/unit/pendant_ring_custody_flush_test.dart` → all 5 tests passed.
- Stress: the new test plus the three affected suites, `pendant_ring_custody_flush_test.dart omi_ring_custody_test.dart ring_custody_download_test.dart pendant_custody_main_red_test.dart`, ran 10 times under `TZ=UTC` and 10 times under `TZ=Pacific/Kiritimati`. **20/20 runs passed** (29 tests each).
- Every app test file that imports `pendant_ring_custody.dart`, `omi_connection.dart` or `local_wal_sync.dart` (25 files): `+344 ~9: All tests passed!` under both `TZ=UTC` and `TZ=Pacific/Kiritimati`.
- `flutter analyze` on the touched files: no issues. `dart format --line-length 120`: no changes.
- `scripts/failure-class explain FC-queued-write-outlives-storage-teardown` validates the new class definition.

## Tests

`app/test/unit/pendant_ring_custody_flush_test.dart` holds the store's directory behind a gate, so a fire-and-forget write is provably pending with no sleeps. It covers:

- `flush()` waits for a pending reincarnation write and for a pending mismatch-invalidation write.
- `flush()` waits for a write queued while it was already waiting.
- `flush()` completes immediately when nothing is queued.
- A teardown that flushes before deleting storage raises no async error.

## Failure class (fixes)

Failure-Class: new

## Failure-class transition narrative (only when needed)

- **Violated contract**: a serialized write queue that callers feed without awaiting must expose a completion covering every queued write, and whoever tears down the storage those writes target must await it first. Otherwise a queued temp-file write and rename runs after its directory is deleted. It then fails after the owning test has passed, or silently recreates storage that was just torn down.
- **Why `new`**: the nearest existing classes are about other owners. `FC-continuation-resumed-after-unmount` is a widget `State` continuation. `FC-run-loop-timer-outlives-owner` is a desktop run-loop timer. `FC-dropped-derived-future-rethrows-handled-error` re-emits an already-handled error. None covers queued storage writes outliving storage teardown.
- **Canonical guard**: `flush()` on the queue owner, awaited before storage teardown. Its pinning test is `app/test/unit/pendant_ring_custody_flush_test.dart`, and the class file is `.github/failure-classes/FC-queued-write-outlives-storage-teardown.json`.
- **Evidence**: CI failures cited in #20500: main at `0c00747` and `4e93805`, and PR #20497 runs 37147694253 and 37150132070.

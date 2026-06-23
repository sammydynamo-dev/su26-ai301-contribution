# Phase III — TDD Test Plan (beets #840)

Tests to write **before** touching `beetsplug/importfeeds.py`, each red first and failing
for the right reason. File: `test/plugins/test_importfeeds.py`, added to
`ImportFeedsTest(PluginTestCase)` (mirrors the existing tests and the
`test/plugins/test_fetchart.py:123` precedent).

## The contract the fix must satisfy
With `formats: link`, when a symlink can't be created the plugin must:
1. **Not abort** the import (no exception escapes `_record_items`).
2. **Log a per-item warning** (not fail silently).
3. **Continue** to the remaining items (per-item skip, not whole-loop abort).
4. **Catch only `FilesystemError`** — not mask unrelated errors.
5. **Still create symlinks** normally when they succeed (no regression).

Each property below maps to one test, so a wrong/over-broad fix fails a specific test.

## Tests (TDD order)

### 1. `test_link_failure_does_not_abort_import` — the driver
- **Arrange:** `formats = "link"`; one album+item; `mock.patch("beetsplug.importfeeds.link", side_effect=FilesystemError("simulated", "link", (b"a", b"b")))`.
- **Act:** `self.importfeeds.album_imported(self.lib, album)`.
- **Assert:** does not raise.
- **Red today:** the exception propagates. **Drives:** `try/except FilesystemError … continue`.
- **Bonus:** using a `FilesystemError` side-effect *also* enforces property #4 — a fix that catches only `OSError` leaves this test red (FilesystemError is not an OSError).

### 2. `test_link_failure_logs_warning`
- **Arrange:** as #1, plus `with mock.patch.object(self.importfeeds, "_log") as log:`.
- **Assert:** `log.warning.assert_called_once()` and the offending path appears in the call args (assert the path is *present*, not the exact wording — avoid brittleness).
- **Drives:** emitting `self._log.warning(...)` instead of a silent `continue`.

### 3. `test_link_failure_continues_to_remaining_items`
- **Arrange:** album with **two** items; `side_effect=[FilesystemError(...), None]` (fail first, succeed second).
- **Assert:** `mock_link.call_count == 2` and no exception. *Stronger variant:* make the second call invoke the real `beets.util.link` and assert its symlink exists on disk (tests real behavior, not just the mock).
- **Drives:** putting the `try/except` **inside** the loop with `continue` (a fix wrapping the whole loop fails this).

### 4. `test_non_filesystem_error_still_propagates` — inversion guard
- **Arrange:** `formats = "link"`; `mock.patch("beetsplug.importfeeds.link", side_effect=ValueError("boom"))`.
- **Assert:** `with pytest.raises(ValueError): self.importfeeds.album_imported(...)`.
- **Drives:** `except FilesystemError` specifically — blocks an over-broad `except Exception` that would hide real bugs.

### 5. `test_link_success_creates_symlink` — happy-path regression guard (real behavior, no mock)
- **Arrange:** `formats = "link"`; real writable `feeds_dir` (from `setUp`); one album+item.
- **Assert:** the expected symlink exists in `feeds_dir` (`os.path.islink(dest)`).
- **Guards:** the fix doesn't break normal linking. Likely green already; keep it green.

### Optional (coverage parity / robustness)
- **6.** Same as #1 but via `self.importfeeds.item_imported(...)` (singleton path) — both listeners call `_record_items`.
- **7.** Fold the "warning names the path" assertion into #2 if not done there.

Plus: the existing 3 tests (`test_multi_format_album_playlist`, `test_playlist_in_subdir`,
`test_playlist_per_session`) must stay green throughout.

## Mechanics (verified this session)
- `from unittest import mock`; `from beets.util import FilesystemError`.
- **Patch target:** `beetsplug.importfeeds.link` (NOT `beets.util.link`) — importfeeds does
  `from beets.util import … link`, so the name must be patched where it's used.
- `FilesystemError(reason, verb, paths, tb=None)` → e.g. `FilesystemError("simulated", "link", (b"src", b"dst"))`.
- **Log assertion:** `mock.patch.object(self.importfeeds, "_log")` then assert on `.warning`
  (the fetchart precedent asserted on `run_with_output` stdout, which doesn't apply —
  importfeeds is listener-based with no CLI command).
- Run: `poe test test/plugins/test_importfeeds.py` (then full `poe test` before PR).

## TDD loop
For each test 1→5: write it → `poe test …` and watch it **fail for the right reason** →
write the *minimal* code to pass → confirm green + the 3 existing tests stay green → next.
Tests 1–4 each drive a distinct property; test 5 is a guard. The whole fix is ~4 lines, so
expect tests 1–4 to mostly go green together once the `try/except FilesystemError: warn; continue`
lands — but writing them first proves each property is actually exercised.

## To confirm when writing (assumptions)
- Exact `paths` / `relative_to` setup so the happy-path `dest` is predictable — mirror the
  existing m3u tests' `Album`/`Item` construction; `dest = feedsdir/basename(path)` and the
  `if not os.path.exists(dest)` guard must pass in the temp `feeds_dir`.
- Log-capture mechanism: prefer `mock.patch.object(plugin, "_log")`; fall back to pytest
  `caplog` if needed (beets' `StrFormatLogger` can make `assertLogs` finicky).

## Risk / trade-offs
- Tests 1–4 mock `link` — accepted beets convention for simulating FS failures (can't fail a
  real symlink deterministically cross-platform in-process). Test 5 balances with real behavior.
- Test 3's `call_count` assertion edges toward "testing the mock"; the real-link survivor
  variant is preferred where practical.
- Confidence: **high** on tests 1/4/5 and the patch target; **medium** on the exact log-capture
  call and the multi-item `side_effect` ergonomics — both trivially resolved at write time.

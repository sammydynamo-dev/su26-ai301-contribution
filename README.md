# Contribution #1: ImportFeeds — catch symlink failures on Windows (beets #840)

- **Contribution Number:** 1
- **Student:** Temitope S. Olugbemi
- **Issue:** <https://github.com/beetbox/beets/issues/840>
- **Status:** Phase III complete — fix + 5 regression tests (TDD) and changelog on `fix-issue-840`; full suite green (2408 passed) and all pre-submission checks pass. Next: open the PR (Phase IV)

---

## Why I Chose This Issue

A real, well-scoped Python bug labeled `bug` + `good first issue`: a focused change to one plugin file (`beetsplug/importfeeds.py`) plus a test, with the maintainer's intended fix already stated ("catch and skip"). It lets me practice reading a mature codebase, Python error handling, mock-based testing, and the full PR workflow.

---

## Understanding the Issue

### Problem Description

With `importfeeds` set to `formats: link`, the plugin creates a symlink per imported track. That call is **not wrapped in `try/except`**, so when it fails — Windows privilege, a read-only dir, or a filesystem without symlink support — the exception aborts the whole `beet import` run, even though the tracks were already imported.

### Expected Behavior

A failed symlink logs a per-item warning and the import continues ("catch and skip", per the maintainer).

### Current Behavior

`beet import` exits `1` with `Error: Permission denied during link of paths …`; the track is already in the library/DB, but the command reports failure.

### Affected Components

- `beetsplug/importfeeds.py` → `_record_items()`: unguarded `link(path, dest)` at **line 131** (`link` imported line 27).
- `beets/util/__init__.py` → `link()` (line 552) wraps `OSError` / `NotImplementedError` into `beets.util.FilesystemError` (lines 571–572).
- `FilesystemError` (line 129) subclasses `Exception`, **not** `OSError` — so catching only `OSError` would miss it.

---

## Reproduction Process

### Environment Setup

Set up per `CONTRIBUTING.rst` on macOS / Python 3.12.6: `pipx install poetry poethepoet`, `poetry install` (in-project `.venv`), `pre-commit install`. Verified with `poe check-format` ("299 files already formatted"), `poe test test/plugins/test_importfeeds.py` (3 passed), and `beet version` (beets 2.11.0). Only snag: `pipx`'s launcher dir wasn't on `PATH`, so I run it as `pipx ensurepath`.

Working branch: <https://github.com/sammydynamo-dev/beets/tree/fix-issue-840>

![dev environment set up with poetry, poethepoet and pre-commit](docs/screenshots/01-env-setup.png)
![toolchain verified via poe](docs/screenshots/02-tests-pass.png)

### Steps to Reproduce

Reported on Windows, but it reproduces on any OS by making the symlink fail — here, a read-only feeds dir:

```yaml
# config.yaml
directory: _repro/lib
library:   _repro/lib.db
import: { copy: yes, write: no }
plugins: importfeeds
importfeeds: { formats: link, dir: _repro/feeds }
```

1. `chmod 0500 _repro/feeds` — make the feeds dir non-writable
2. `poetry run beet -c config.yaml import -A -q _repro/music`
3. **Expected:** a warning is logged and the import completes (exit `0`)
4. **Actual:** `Error: Permission denied during link of paths …`, exit `1` — yet the track is already imported

### Reproduction Evidence

![reproducing beets #840 on macOS](docs/screenshots/03-repro-crash.png)

The import exits `1` on the unhandled link error, yet `beet ls` shows the track was imported and the file copied into the library while the feeds dir stays empty (the symlink was never created). Key findings: the crash is OS-agnostic (a missing `try/except`, not "Windows symlinks"), and the escaping exception is `beets.util.FilesystemError` — **not** `OSError` — which settles the maintainer scoping question.

---

## Solution Approach

### Analysis

`_record_items()` calls `link()` unguarded (`importfeeds.py:131`); `util.link()` raises `beets.util.FilesystemError` (`util/__init__.py:571`), which propagates out of the import listener and aborts the run. Root cause = missing error handling, not the Windows symptom.

### Proposed Solution

Wrap the `link()` call in `try/except FilesystemError`, log a per-item `self._log.warning(...)`, and `continue` — mirroring patterns already in the codebase (see Match).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A single failed symlink raises `FilesystemError` out of `_record_items` and aborts the whole `beet import`, even though the tracks were already imported. Desired behavior is warn-and-continue. Confirmed by the repro above.

**Match:** beets already does warn-and-continue on `FilesystemError` — `beetsplug/playlist.py:139` and `beetsplug/fetchart.py:1504` (the #6193 / PR #6662 fix). Test template: `test/plugins/test_fetchart.py:123`.

**Plan:**

1. Guard `link()` in `_record_items` (`importfeeds.py:131`) with `try/except FilesystemError` → `self._log.warning(...)` + `continue`.
2. Add regression test `test_link_failure_is_warned_not_fatal`, mocking `beetsplug.importfeeds.link` to raise `FilesystemError`.
3. Add a `docs/changelog.rst` entry under *Unreleased → Bug fixes*, ending `` :bug:`840` ``.
4. Run `poe format / lint / check-format / test / check-types`.

**Implement:** Done on `fix-issue-840` — commits `9d1394ac1` (fix + tests) and `a027e2563` (changelog).

**Review:** Self-check against `CONTRIBUTING.rst` (catch `FilesystemError` not `OSError`; f-strings + logging shim; pytest `PluginTestCase`; ≤ 80 cols). Full checklist: [`docs/beets-contribution-checklist.md`](docs/beets-contribution-checklist.md). Per beets' AI policy, I open and drive the PR personally.

**Evaluate:** The new test passes, the existing 3 importfeeds tests stay green, and the repro now exits `0` with a warning (symlink skipped, track still imported).

---

## Testing Strategy

### Unit Tests

Five new tests in `test/plugins/test_importfeeds.py` (written TDD — failing first), all passing alongside the existing three:

- [x] `test_link_failure_does_not_abort_import` — `link` raises `FilesystemError`; `album_imported` doesn't raise.
- [x] `test_link_failure_logs_warning` — a per-item warning is logged.
- [x] `test_link_failure_continues_to_remaining_items` — remaining items still attempted after one fails.
- [x] `test_non_filesystem_error_is_not_swallowed` — a `ValueError` still propagates (only `FilesystemError` is caught).
- [x] `test_link_creates_symlink` — happy path still creates the symlink.

### Integration Tests

- [x] Read-only feeds dir: `beet import` now exits `0`, warns, and imports the track (symlink skipped).
- [x] Writable feeds dir: the symlink is still created (covered by `test_link_creates_symlink`).

### Manual Testing

Re-ran the read-only-dir repro on the fixed code: **exit `0`** (was `1`), logged `importfeeds: could not create symlink for …: Permission denied …`, and the track imported.

---

## Implementation Notes

### Week 2 Progress — reproduction & environment

Reproduced #840 on macOS, confirmed the root cause (`importfeeds.py:131`), set up the official poetry / poe / pre-commit toolchain, and verified the suite. Next: failing test → fix (Phase III).

### Week 3 Progress (Jun 17–23)

**What I built:**
- Wrapped the `link()` call in `_record_items` (`beetsplug/importfeeds.py`) in `try/except beets.util.FilesystemError` — logs a per-item warning and continues, so one failed symlink no longer aborts the whole import.
- Added 5 regression tests in `test/plugins/test_importfeeds.py` (written TDD, failing first):
  - `test_link_failure_does_not_abort_import` — a failed link doesn't raise
  - `test_link_failure_logs_warning` — the failure is logged
  - `test_link_failure_continues_to_remaining_items` — later items still attempted
  - `test_non_filesystem_error_is_not_swallowed` — only `FilesystemError` is caught
  - `test_link_creates_symlink` — happy path still creates the symlink
- Added a `docs/changelog.rst` entry (`:bug:`840``).
- All existing tests still pass — full suite: **2408 passed, 163 skipped, 0 failed**; `poe lint` / `check-format` / `mypy` clean.

**Challenges faced:**
- The mid-phase rebase bumped beets 2.11 → 2.12, so I re-verified the fix-site line numbers and re-ran `poetry install` before writing code.
- `ruff`'s PT011 rule rejected a bare `pytest.raises(ValueError)`; fixed by asserting on the message with `match=`.
- The `link` branch never creates the feeds dir, so the happy-path test had to `mkdir` it first.
- Key design call: `link()` raises `FilesystemError` (not `OSError`), so the mock target is `beetsplug.importfeeds.link` and the `except` catches `FilesystemError`. Found the pattern in `beetsplug/playlist.py` line 139 and `beetsplug/fetchart.py` line 1504.

**Commits this week:**
- 9d1394ac1: fix(importfeeds): keep import going when a symlink can't be created
- a027e2563: docs(changelog): add entry for importfeeds symlink fix (#840)

### Code Changes

- **Branch:** <https://github.com/sammydynamo-dev/beets/tree/fix-issue-840>
- **Files modified:** `beetsplug/importfeeds.py` (`try/except` around `link()`), `test/plugins/test_importfeeds.py` (5 tests), `docs/changelog.rst` (entry).
- **Approach decisions:** mirrors `playlist.py:139` / `fetchart.py:1504` (warn-and-continue on `FilesystemError`); fix and changelog kept as separate Conventional-Commits commits.

---

## Pull Request

**PR Link:** <https://github.com/beetbox/beets/pull/6765> — *Fix importfeeds to continue on symlink creation failures*

**PR Description:** `importfeeds` with `formats: link` no longer aborts the whole `beet import` when a symlink can't be created — it catches `beets.util.FilesystemError`, logs a per-item warning, and continues. Includes 5 regression tests and a changelog entry (`Fixes #840`). Full text: [`docs/pr-description-840.md`](docs/pr-description-840.md).

**Maintainer Feedback:**

- 2026-06-23: PR opened; awaiting first review (`reviewDecision: REVIEW_REQUIRED`) — no feedback yet.
- Next steps: add a comment tagging `@snejus` (merged the near-identical fetchart fix #6662 and is the most recent committer on `importfeeds.py`); if there's no response within ~5–7 business days, post a polite follow-up.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- Issue: [beets #840](https://github.com/beetbox/beets/issues/840)
- `CONTRIBUTING.rst` + `docs/changelog.rst` (entry style `` :bug:`NNNN` ``)
- Precedents: `beetsplug/playlist.py:139`, `beetsplug/fetchart.py:1504` (PR #6662), `test/plugins/test_fetchart.py:123`
- Detailed working notes: [`docs/reproduction-plan-840.md`](docs/reproduction-plan-840.md)

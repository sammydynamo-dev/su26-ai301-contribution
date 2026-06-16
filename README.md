# Contribution #1: ImportFeeds — catch symlink failures on Windows (beets #840)

**Contribution Number:** 1
**Student:** Temitope S. Olugbemi
**Issue:** <https://github.com/beetbox/beets/issues/840>
**Status:** Phase II complete (reproduced + dev environment ready)

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

Set up per `CONTRIBUTING.rst` on macOS / Python 3.12.6: `pipx install poetry poethepoet`, `poetry install` (in-project `.venv`), `pre-commit install`. Verified with `poe check-format` ("299 files already formatted"), `poe test test/plugins/test_importfeeds.py` (3 passed), and `beet version` (beets 2.11.0). Only snag: `pipx`'s launcher dir wasn't on `PATH`, so I run it as `sudo pipx ensurepath --global`.

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

**Implement:** Pending — Phase III, on branch `fix-issue-840` in my fork; commits linked here as I work.

**Review:** Self-check against `CONTRIBUTING.rst` (catch `FilesystemError` not `OSError`; f-strings + logging shim; pytest `PluginTestCase`; ≤ 80 cols). Full checklist: [`docs/beets-contribution-checklist.md`](docs/beets-contribution-checklist.md). Per beets' AI policy, I open and drive the PR personally.

**Evaluate:** The new test passes, the existing 3 importfeeds tests stay green, and the repro now exits `0` with a warning (symlink skipped, track still imported).

---

## Testing Strategy

### Unit Tests

- [ ] `test_link_failure_is_warned_not_fatal` (new): with `formats: link` and `beetsplug.importfeeds.link` patched to raise `FilesystemError`, `album_imported` completes without raising and logs a warning.
- [ ] Existing `test_multi_format_album_playlist`, `test_playlist_in_subdir`, `test_playlist_per_session` still pass.

### Integration Tests

- [ ] Read-only feeds dir: `beet import` now exits `0`, warns, and imports the track (symlink skipped).
- [ ] Writable feeds dir: the symlink is still created (happy path unchanged).

### Manual Testing

Re-run the read-only-dir repro after the fix → expect exit `0` + a warning instead of `Error:`.

---

## Implementation Notes

### Week 2 Progress — reproduction & environment

Reproduced #840 on macOS, confirmed the root cause (`importfeeds.py:131`), set up the official poetry / poe / pre-commit toolchain, and verified the suite. Next: failing test → fix (Phase III).

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**

- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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

# Reproduction & Root-Cause Plan — beets #840 (ImportFeeds symlink crash)

**Issue:** https://github.com/beetbox/beets/issues/840 — "ImportFeeds: fix/disable/catch symlink usage on windows"
**Labels:** `bug`, `good first issue`, `importfeeds`, `windows`
**Student:** Temitope S. Olugbemi (CodePath AI301)
**Dev environment:** macOS (darwin), Python 3.12.6
**Method:** Phase 1–2 of systematic debugging (root cause **before** any fix), grounded in current `master` source (not the obsolete 2014 Py2.7/beets-1.3.6 traceback in the issue).

---

## 1. Problem Analysis

### The symptom (what the reporter saw)
On Windows, running `beet import` with the `importfeeds` plugin configured for `formats: link` produces a traceback and **aborts the whole import run** — even though the audio files were already imported. The 2014 traceback travels `ui/commands.py → importer.py → util/pipeline.py → importfeeds`.

### The root cause (OS-agnostic — this is the real bug)
The defect is **not** "Windows can't do symlinks." It is a **missing exception handler around a fallible filesystem call**, executed inside an import-pipeline event listener.

Exact location (current `master`):

- **`beetsplug/importfeeds.py` → `ImportFeedsPlugin._record_items(self, lib, basename, items)`**

  ```python
  if "link" in formats:
      for path in paths:
          dest = os.path.join(feedsdir, os.path.basename(path))
          if not os.path.exists(syspath(dest)):
              link(path, dest)          # <-- NO try/except. Any failure propagates.
  ```
  `_record_items` is invoked by the `album_imported` / `item_imported` listeners the plugin registers, so it runs inside the import pipeline.

- **`beets/util/__init__.py` → `link(path, dest, replace=False)`** (the `link` imported above is `beets.util.link`):

  ```python
  def link(path, dest, replace=False):
      if os.path.exists(syspath(dest)) and not replace:
          raise FilesystemError("file exists", "rename", (path, dest))
      try:
          os.symlink(syspath(path), syspath(dest))
      except NotImplementedError:                       # old Windows / no symlink support
          raise FilesystemError("OS does not support symbolic links.link", (path, dest), ...)
      except OSError as exc:                             # privilege denied, EACCES, etc.
          raise FilesystemError(exc, "link", (path, dest), ...)
  ```

So `util.link()` **wraps** the underlying `OSError`/`NotImplementedError` into **`beets.util.FilesystemError`**.

### Critical detail — which exception to catch
`FilesystemError(HumanReadableError)` and `HumanReadableError(Exception)`. **`FilesystemError` is NOT a subclass of `OSError`.**

> Therefore catching `OSError` in `importfeeds` would **not** catch the crash. The correct catch is **`FilesystemError`** (import it from `beets.util`).

This resolves the scoping question already posted on the issue. Note the fetchart template (PR #6662) caught `OSError` only because `Album.set_art()` raises `OSError` **directly** — a different layer than `importfeeds`, which goes through the wrapping `util.link()`.

### Why Windows is just the common trigger
`util.link()` raises `FilesystemError` whenever the symlink can't be created, for **any** reason on **any** OS:
- Windows without Developer Mode / admin → `OSError` WinError 1314 ("a required privilege is not held").
- Old Windows / no symlink support → `NotImplementedError`.
- **Any POSIX system** (macOS/Linux): destination directory not writable → `PermissionError`/`EACCES`; target filesystem doesn't support symlinks (FAT/exFAT) → `OSError`.

### Critical success factors
1. Trigger the crash on demand **on macOS** (no Windows box needed).
2. Reproduction is deterministic and someone else can replay it.
3. Name exact file(s)/function(s) — done above.
4. Fix targets the root cause (graceful handling of a fallible op), not the symptom (Windows).

---

## 2. Reproduction Strategy (two complementary levels)

### Level A — Automated / deterministic (this BECOMES the regression test)
Force `link()` to raise by monkeypatching it, then drive the plugin and assert today's broken behavior (exception escapes).

**Patch target matters:** `importfeeds.py` does `from beets.util import ... link ...`, so the name bound *inside the module* is `beetsplug.importfeeds.link`. Patch **that**, not `beets.util.link`:

```python
from unittest import mock
from beets.util import FilesystemError

# in test/plugins/test_importfeeds.py, mirroring the existing ImportFeedsTest(PluginTestCase)
def test_link_failure_currently_aborts(self):          # RED test (documents the bug)
    self.config["importfeeds"]["formats"] = "link"
    album = Album(album="album/name", id=1)
    item = Item(title="song", album_id=1, path=os.path.join("path", "to", "item.mp3"))
    self.lib.add(album); self.lib.add(item)
    with mock.patch(
        "beetsplug.importfeeds.link",
        side_effect=FilesystemError("simulated symlink failure", "link", ("a", "b")),
    ):
        # TODAY: this raises FilesystemError (bug). AFTER the fix: it must NOT raise,
        # should log a warning, and the import should complete.
        self.importfeeds.album_imported(self.lib, album)
```

- Portable: runs in CI on macOS/Linux/Windows.
- Existing harness to copy: `test/plugins/test_importfeeds.py` already uses `PluginTestCase`, `ImportFeedsPlugin`, and `album_imported(...)`.

### Level B — Manual / integration repro on macOS (proves it end-to-end)
**Verified mechanism on this machine:** `os.symlink()` into a `chmod 0500` (read-only) directory raises `PermissionError` (errno 13, EACCES); and the destination does not pre-exist, so the plugin's `if not os.path.exists(dest)` guard passes and *does* reach `link()` → `FilesystemError`.

Steps:
```bash
# 1. Sample library + a real audio file
mkdir -p ~/beets-repro/music ~/beets-repro/feeds
cp some_song.mp3 ~/beets-repro/music/        # any importable audio file

# 2. Minimal config (~/beets-repro/config.yaml)
cat > ~/beets-repro/config.yaml <<'YAML'
directory: ~/beets-repro/lib
library: ~/beets-repro/lib.db
import: { copy: yes }
plugins: importfeeds
importfeeds:
  formats: link
  dir: ~/beets-repro/feeds
YAML

# 3. Make the feeds dir non-writable so the symlink call fails
chmod 0500 ~/beets-repro/feeds

# 4. Run the import and watch it crash AFTER importing
BEETSDIR=~/beets-repro beet import -q ~/beets-repro/music
#   -> FilesystemError traceback; the run aborts even though the file was imported

# cleanup
chmod 0700 ~/beets-repro/feeds
```
Expected (bug present): unhandled `FilesystemError` traceback, import aborts.
Expected (after fix): a logged warning like "could not create symlink for …", import completes.

### Level C — True Windows repro (optional, for fidelity)
On Windows without Developer Mode/admin, `formats: link` reproduces the original WinError-1314 path. Use a Windows VM or a GitHub Actions `windows-latest` runner. **Not required** — Levels A+B already give an on-demand, portable repro — but it confirms the reporter's exact environment.

---

## 3. Environment Setup Plan + Documented Challenges

```bash
git clone https://github.com/beetbox/beets
cd beets
python3 -m venv .venv && source .venv/bin/activate
python3 -m pip install -e '.[test]'        # editable install + test deps
python3 -m pytest test/plugins/test_importfeeds.py -q   # sanity-check the harness
```

**Challenges to record (mentors want to see this):**
1. **beets was neither installed nor cloned** on the dev machine (`beet` and `pip show beets` both absent) — must clone + create a venv from scratch.
2. **`pip` not on PATH** — use `python3 -m pip` inside the venv (don't rely on a bare `pip`).
3. **OS mismatch:** the bug is reported on Windows; dev is macOS. Justification for repro validity: macOS exercises the *same* code path (`_record_items → util.link → os.symlink`) and raises the *same* wrapped `FilesystemError`; we just trigger the underlying `OSError` with a read-only dir instead of a Windows privilege error.
4. **Version drift:** the issue's traceback is beets 1.3.6 / Python 2.7 (2014). The file layout and code have changed (e.g., tests now live under `test/plugins/`). We reproduce the *current* equivalent, not the verbatim 2014 trace.
5. **Patch-target gotcha:** because `importfeeds` does `from beets.util import link`, the mock target is `beetsplug.importfeeds.link`, not `beets.util.link`.

---

## 4. Solution Plan (root cause, not symptom) — preview for the fix phase

1. In `_record_items`, wrap the per-item `link()` call in `try/except FilesystemError`, log `self._log.warning(...)` with the offending path, and `continue` the loop so the rest of the import proceeds. (Pattern mirrors fetchart PR #6662, but with `FilesystemError` instead of `OSError` — see §1.)
2. Add the regression test from §2 Level A, flipped to assert the import **completes** and a warning is logged.
3. Add a `docs/changelog.rst` entry (required by `CONTRIBUTING.rst`).
4. Self-review against `CONTRIBUTING.rst`; open PR referencing #840.

**Why this is root-cause, not a patch:** it makes the plugin resilient to *any* symlink failure on *any* OS, rather than special-casing Windows.

---

## 5. Confidence & Open Questions
- **High confidence:** file/function/exception identification and the macOS trigger (read-only dir) — both verified against current source and an empirical `os.symlink` test on this machine.
- **Confirm with maintainer:** warn-and-skip vs. warn-once-per-run; exact log wording; whether to also guard the `m3u`/file-write paths similarly (out of scope for #840, but worth noting).

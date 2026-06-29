# PR #6765 — review-request comment

**Who to tag:** `@snejus` (Šarūnas Nejus) — merged the near-identical fetchart
fix (#6662) and is the most recent committer on `beetsplug/importfeeds.py`.
`CODEOWNERS` assigns the repo to the `@beetbox/maintainers` team with no specific
owner for importfeeds, so an individual tag is better than pinging the team.
Alternate: `@semohr`. Posted by Temitope — per beets' AI-assistance policy, a
human handles all PR communication.

---

Hi @snejus 👋 — this is my first contribution to beets.

This PR (#6765) fixes #840: when `importfeeds` is set to `formats: link` and a
symlink can't be created (e.g. on Windows, or into a read-only directory), the
failure is now logged as a per-item warning and the import continues, instead of
the unhandled `FilesystemError` aborting the whole `beet import` run. It follows
the same warn-and-skip pattern as the fetchart fix (#6662) and `playlist.py`.

It includes five regression tests and a changelog entry, and the full test suite
passes locally. I tagged you since you reviewed the very similar fetchart change —
no rush at all, but I'd appreciate a review whenever you have time. Thanks!

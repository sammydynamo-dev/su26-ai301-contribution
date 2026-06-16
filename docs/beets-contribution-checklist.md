# beets Contribution Compliance Checklist

Binding rules distilled from the upstream beets governance docs, verified against
the local clone (commit `60047df`). Sources: `CONTRIBUTING.rst`, `README.rst`,
`SECURITY.md`, `CODE_OF_CONDUCT.rst`, `.github/pull_request_template.md`. This is
the standing reference I follow for every contribution (currently issue #840).

## ⚠️ AI-assistance policy — read first

beets allows AI-assisted contributions, **but all communication must be handled
by a real person who understands the change.** *"We are not opposed to
AI-generated contributions, but communication should be handled by a real
person … AI as a tool, not a contributor."* (`CONTRIBUTING.rst`)

- AI may help draft code/tests/docs **locally**.
- **Temitope** personally writes and posts every issue comment, PR description,
  and review reply, and must be able to explain and defend the fix.
- No autonomous agent commenting/opening PRs on GitHub. No AI-authorship label is
  required, but human comprehension and accountability are.

## Code (`CONTRIBUTING.rst`)

- [ ] f-strings (not `%` / `str.format`) in general code.
- [ ] User-facing output via the beets logging shim with `{}` placeholders
      (e.g. `self._log.warning("… {}", x)`) — **never `print()`**.
- [ ] `except A as B:` style; DB access only via `Library` methods / a
      `Transaction` (never `lib.conn.*`).
- [ ] ruff-clean, **line length 80** (`pyproject.toml [tool.ruff]`; the `88` in
      `setup.cfg` is legacy flake8, not authoritative).
- [ ] Minimal change; match surrounding code.

## Tests

- [ ] A bugfix MUST include a regression test proving the fix.
- [ ] Written pytest-style; for plugin tests follow the existing
      `PluginTestCase` pattern (precedent: `test/plugins/test_fetchart.py:123`).
- [ ] Mock external HTTP with `requests-mock` (not needed for importfeeds — it
      is local-only). No personal data in fixtures (CoC).

## Documentation

- [ ] Required only when user-facing behavior/flags change. A pure bugfix usually
      needs none; if importfeeds' documented behavior changes, update
      `docs/plugins/importfeeds.rst`.

## Changelog (required for code changes — CI-enforced)

- [ ] Add a bullet at the **bottom of the appropriate list under "Unreleased"**
      in `docs/changelog.rst`, ending with `` :bug:`840` ``.
- [ ] A CI bot auto-comments if a `.py` file changes without a changelog edit —
      include both in the **same** PR.

## Local validation (poetry + poe + pre-commit)

- [ ] Official dev setup is **poetry** (`poetry install`) + **pre-commit**
      (`pre-commit install`) — not a bare venv.
- [ ] Before pushing: `poe format`, `poe lint`, `poe check-format`, `poe test`,
      `poe check-types` (all clean).

## Pull request (`.github/pull_request_template.md`)

- [ ] Description starts with `Fixes #840.`
- [ ] Keep the three To-Do boxes. For this bugfix: **Changelog** ✓, **Tests** ✓,
      **Documentation** → strike through `~Documentation~` and still check it
      (unless docs change).
- [ ] Remove the template's helper sentences. Work fork → feature branch → PR
      against `beetbox/beets`.
- [ ] After pushing new commits to an open PR, comment / re-request review
      (GitHub does not notify maintainers of pushes).

## Communication etiquette (`README.rst` + `CODE_OF_CONDUCT.rst`)

- #840 already exists — do **not** open duplicate issues. Concrete bug work
  stays on the issue + PR thread; open-ended / design questions go to **GitHub
  Discussions**.
- Use 👍 reactions to signal support, not "me too" comments.
- Be professional and empathetic; give and **gracefully accept** review feedback;
  own and apologize for mistakes. Maintainers may reject contributions on conduct
  grounds. The CoC (Contributor Covenant v2.1) applies to all GitHub activity.

## Security (`SECURITY.md`)

- Only the latest release is supported — target `master`; changelog under the
  unreleased section.
- If a fix ever exposes a real vulnerability (e.g. path traversal / arbitrary
  file write via crafted feed/playlist paths), **do not describe it publicly** —
  email the Zulip security address in `SECURITY.md` and let maintainers
  coordinate disclosure.
- **#840 gut-check:** warn-and-continue around a symlink of an already-trusted
  imported path — **no security dimension**; the standard public workflow
  applies.

---
name: hackingtool-readme-ci-review
description: Checks whether this repo's CI actually verifies that README.md matches generate_readme.py's output, rather than assuming a green build means the README is in sync. Neither .github/workflows/lint_python.yml nor test_install.yml runs generate_readme.py or diffs its output against the committed README.md — a PR that edits hackingtool.py's tool_definitions/all_tools (or README_template.md) without regenerating README.md can merge with a silently stale README. Use this whenever a diff touches generate_readme.py, README.md, README_template.md, hackingtool.py's tool_definitions/all_tools lists, or any .github/workflows/*.yml file — and whenever anyone claims "CI keeps the README in sync" for this repo, since that claim needs verifying, not assuming.
---

# README/generate_readme.py drift is not CI-enforced

`generate_readme.py` builds README.md's table of contents and per-category tool listing from
`hackingtool.py`'s `all_tools` (via `README_template.md`'s `{{toc}}`/`{{tools}}` placeholders — see
`hackingtool-menu-review`'s SKILL.md, section 3, for how that generation works and the hand-edit
risk it already documents). What that section does not cover, and what this one is for: **nothing
in CI actually runs the generator and checks the result matches what's committed.**

Confirmed by reading both workflow files as of this pass:
- `.github/workflows/lint_python.yml` runs ruff/black/codespell/mypy/pytest — never
  `generate_readme.py`, never touches `README.md`.
- `.github/workflows/test_install.yml` installs the package and smoke-tests the `hackingtool`
  entrypoint via piped menu input (`99`, `1`→`99`→`99`) — it never regenerates or diffs the README
  either.
- There is no pre-commit config and no other workflow file in `.github/workflows/`.

So a PR that adds/removes/reorders a category in `tool_definitions`/`all_tools`, edits
`README_template.md`, or hand-edits `README.md`'s generated sections without re-running
`generate_readme.py`, will merge cleanly — both jobs go green regardless. The README's tool count,
table of contents, and per-category links can drift from the actual menu indefinitely with no
signal, since nothing red ever appears.

## What to check on a relevant diff

- If `hackingtool.py`'s `tool_definitions`/`all_tools`, or `README_template.md`, changed: run
  `python3 generate_readme.py` locally against the branch and diff the result against the
  committed `README.md`. Flag the PR if they disagree — this is currently the only way that gap
  gets caught, since no CI job does it.
- If `README.md`'s generated portions changed but neither `all_tools` nor `README_template.md`
  did: per the existing skill's section 3, this is very likely a hand-edit that the next
  regeneration will silently discard — flag it there too.
- Do not treat a green `lint_python`/`test_install` run as evidence the README is current; those
  jobs are silent on this regardless of drift.
- If someone proposes closing this gap for real, the fix belongs in CI (e.g. a step in
  `lint_python.yml` that runs `generate_readme.py` and `git diff --exit-code README.md`), not in
  this review file — this skill exists to catch drift manually until that step exists.

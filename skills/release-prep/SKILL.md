---
name: release-prep
description: Preps a Princeton-CDH-style Python release branch (git flow) and opens a PR for CI — version bump, changelog/deploy-notes drafts, Django/JS checks, dependency checks, tests/coverage. Stops before merge/tag/`git flow release finish`. Use whenever the user wants to cut, start, or prep a release, mentions "git flow release start", or references the software_release.md checklist.
license: Apache-2.0
metadata:
  author: princeton-cdh
  version: "1.0.0"
---

# Release Prep

Automates the "release prep" half of Princeton-CDH's [software release checklist](https://github.com/Princeton-CDH/.github/blob/main/.github/ISSUE_TEMPLATE/software_release.md) for Python repos using git flow. Ends with a draft PR (to verify CI) and a tracking-issue comment. **Never** merge, tag, force-push, rewrite shared history, or run `git flow release finish` — the human does those, plus the "after release" checklist.

Step markers: 🤖 automated · 📝 drafted for review · 🚩 flagged for human judgment. Steps marked _Django_ or _JS_ run only for that stack.

## Reporting rules

- Steps that don't apply to the repo's stack are **omitted** from the summary (no "n/a").
- Steps that should have run but couldn't (missing tool, failed command, expected file not found after a brief search) go under **Skipped or failed** with the reason. Never skip silently or imply success.
- If a 🤖 step fails, stop and report rather than pushing a broken branch.

## Setup

1. **Tools:** `git` + `git-flow`, `gh` (authenticated), `uv` if used. Run `gh auth status` first; if `gh` is unusable, warn that changelog issue/PR sources, PR creation, CI monitoring, and the issue comment depend on it, and ask whether to continue local-only.
2. **Ask:** repo path (or offer to clone) and PR base branch (default `main`). Don't ask for the version.
3. **Clean tree:** if `git status --porcelain` is dirty, stop and ask — don't stash or discard.
4. **Resume check:** `git branch --list 'release/*'` and `git ls-remote --heads origin 'release/*'`. If a release branch exists, check it out (skip step 3), diff against develop/main, and report what's done before changing anything. If a PR is already open, skip to step 16.
5. **Config:** read `.release-prep.yml` at repo root if present; set fields override auto-detection. If a field contradicts the repo (e.g. `django: false` with `manage.py`), flag it rather than override.

   ```yaml
   django: false
   js_frontend: false
   dependency_manager: uv # uv | pip
   version_file: src/mypackage/__init__.py
   release_candidate_suffix: "" # literal, e.g. "rc1"; omit for final versions
   test_command: "uv run pytest --cov"
   changelog_file: CHANGELOG.md
   deploy_notes_file: docs/deploynotes.rst
   settings_sample_file: mypackage/settings/local_settings.py.sample
   coverage_threshold: 95
   pr_base_branch: main
   ```

   Unset fields are auto-detected:
   - **Django:** `manage.py` at root, or `django` in `requirements*.txt`/`pyproject.toml`/`setup.cfg`.
   - **JS frontend:** `package.json` with real `dependencies`/`scripts`.
   - **Dependency manager:** `uv.lock` or `[tool.uv]` in `pyproject.toml` → uv; else pip.

   If no config exists, offer to write one after a successful run (separate commit, only if the user agrees).

## Workflow

**1. Determine version** 🤖. Get the last tag (`git describe --tags --abbrev=0` on `main`) and gather signal from `git log <last-tag>..develop --oneline --no-merges` plus closed issues/PRs and labels (step 6). Breaking → major; features → minor; only fixes/chores → patch. **Ask** if the signal is mixed/unclear, or the repo uses major.minor versioning and signal is patch-only. Otherwise state version + reason in one line.

**2. Sync** 🤖

```bash
git fetch --all
git checkout develop && git pull
git checkout main && git pull
```

**3. Start release branch** 🤖: `git flow release start <version>` (fallback, noted: `git checkout -b release/<version> develop`).

**4. Migrations** 🚩 _Django_: `git diff main...HEAD --stat -- '**/migrations/*.py'`. List changed files; don't judge them.

**5. Bump version** 🤖. Use `version_file` if set; otherwise update **all** of: `__version__` in `<package>/__init__.py` or `src/<package>/__init__.py`, `version` in `pyproject.toml`, `setup.cfg`/`setup.py`. Strip `-pre`/`-dev`/`.dev0`; append `release_candidate_suffix` if configured. Show the diff.

**6. Changelog** 📝

```bash
git log <last-tag>..HEAD --oneline --no-merges
gh issue list --state closed --milestone "<version>" --json number,title,labels   # if milestones used, else:
gh issue list --state closed --search "closed:>=<last-tag-date>" --json number,title,labels
gh pr list --state merged --search "merged:>=<last-tag-date>" --json number,title,labels
```

Dedupe via `#123` references in commits; group Features/Fixes/Other by label or commit prefix. Insert at the top of `changelog_file` (or `CHANGELOG.rst`/`.md`/`docs/changelog.rst`) with `<!-- DRAFT — generated from git log + closed issues/PRs -->`. Match existing style; keep it short and user-facing. In the summary, list every closed issue/PR found as ✅ included / ⚠️ excluded + reason.

**7. Open work** 🚩

```bash
gh issue list --state open --milestone "<version>" --json number,title,labels   # if milestones used
gh pr list --author "app/dependabot" --state open --json number,title
```

List open milestone issues, excluding the tracking issue and post-release tasks (e.g. "Deploy …"); list uncertain ones separately as "possibly post-release". List dependabot PRs for the user to consider; don't merge.

**8. Deploy notes** 📝 _Django_. In `deploy_notes_file` (or `docs/deploynotes.rst` / `docs/deploy_notes.md`), draft bullets: new settings → "needs configuring"; new migrations → "run migrations"; new/changed management commands → possible one-off step.

**9. Settings sample** 🚩 _Django_. Diff the sample against real settings for missing keys. If there's no sample (or only a test one), omit.

**10. Python deps** 🚩: `grep -E 'git\+|@ git|-e git' requirements*.txt pyproject.toml`. Flag internal deps pinned to a branch/commit.

**11. JS deps** 🚩🤖 _JS_: `grep -E '"file:|"git\+|workspace:' package.json` — flag unreleased internal deps; then `npm audit fix` (not `--force`).

**12. Tests + coverage** 🤖 (info only). Use `test_command`, else `pytest --cov --cov-report=term-missing` or `python manage.py test`, prefixed with `uv run` in uv projects. Report pass/fail and coverage %, noting plainly if below `coverage_threshold` (default 95). Don't gate the PR or write tests.

**13. Docs/schema** 🚩 _Django_. If models changed and schema tooling exists (e.g. `django-extensions graph_models`), regenerate and commit; otherwise flag.

**14. Lock deps** 🤖. pip: `pip freeze > requirements.lock` and stage. uv: `uv lock --check`; if it fails, `uv lock` and stage.

**15. Commit, push, draft PR** 🤖.

- **Tracking issue:** `gh issue list --search "in:title Release <version>" --state open` (or a `release-checklist` label). If not exactly one match, don't guess; omit and note it.
- **Commits:** one or a few logical, reviewable commits, each with an `Assisted-by: <agent>:<model>` trailer. If a pre-commit hook only reformatted, re-stage and retry once; otherwise stop and report. Never `--no-verify`.
- **PR body:** the final summary (minus CI status), wrapped as:

  ```
  :robot: This PR was prepared by an AI agent (release-prep skill) as a draft to verify CI. Review the drafted and flagged sections below before finishing the release.

  Associated issue: Closes #<tracking-issue>   [omit if none]

  <final summary>

  ---
  :robot: Generated by the `release-prep` skill v<metadata.version> — https://github.com/Princeton-CDH/skills/tree/main/skills/release-prep. Something look wrong? Flag it there.
  ```

```bash
git push -u origin release/<version>
gh pr create --draft --base <base-branch> --head release/<version> --title "Release <version>" --body "<body>"
```

Only claim a PR exists if `gh pr create` returned a URL. Otherwise say the branch was pushed without a PR, and give the title/body and compare URL.

**16. Monitor CI** 🤖: `gh pr checks <pr> --watch` (or one-shot `gh pr view <pr> --json statusCheckRollup`). Report per check; surface failures, don't fix them.

**17. Comment on tracking issue** 🤖 (stopping point). Post the final summary with PR link and CI status: `gh issue comment <number> --body-file -`. Skip if no issue was found. **Never edit the issue body or tick its checkboxes** — that requires rewriting the whole body, risking clobbered concurrent edits; ticking stays manual.

## Final summary

Show in chat and post to the tracking issue:

```
## Release <version> prep complete — draft PR: <url>
[If no PR: "## Release <version> branch prepped — PR NOT created (<reason>)" + PR body for manual use.]

### CI status
- <check>: ✅/❌/⏳

### Version
- <last-tag> → <version> (<major|minor|patch> — <reason>)

### Drafted — review before finishing
- Changelog — closed issues/PRs since <last-tag>:
  - #123 "Title" — ✅ included (<sha>) / ⚠️ excluded (<reason>)
- Deploy notes: <summary>

### Test coverage (info)
- <%> (threshold <n>%)

### Needs your judgment
[Only items with findings.]
- Open milestone issues (+ "possibly post-release")
- Dependabot PRs to consider
- Migrations
- Settings sample gaps
- Python/JS deps needing release
- Docs/schema

### Skipped or failed
[Each with reason; omit section if none.]

### What to do next
1. Review the drafted changelog and deploy notes.
2. Resolve "Needs your judgment" items.
3. If an RC suffix was used, remove it after acceptance testing.
4. When CI and review pass: `git flow release finish <version>`, then push main, develop, and tags. Close the draft PR if it doesn't show as merged.
5. Complete the "after release" checklist (e.g. bump develop's version).
```

_Maintainers: bump `metadata.version` (semver) when changing this workflow — the PR footer reports it._

---
name: release-prep
description: Preps a Princeton-CDH-style Python release branch (git flow) and opens a PR for CI — version bump, changelog/deploy-notes drafts, Django/JS checks, dependency checks, tests/coverage. Stops before merge/tag/`git flow release finish`. Use whenever the user wants to cut, start, or prep a release, mentions "git flow release start", or references the software_release.md checklist.
---

# Release Prep

Automates the "release prep" half of Princeton-CDH's [software release checklist](https://github.com/Princeton-CDH/.github/blob/main/.github/ISSUE_TEMPLATE/software_release.md) for Python repos using git flow. Stops at opening a PR — never merges, tags, or runs `git flow release finish`; the "after release" checklist section is out of scope. Plain markdown + shell/git/gh — portable to any agentic tool (see `harness-adapters/`).

## Scope

🤖 auto · 📝 draft (needs review) · 🚩 flag (needs human judgment)

| # | Item | Handling |
|---|---|---|
| — | Determine next version | 🤖, prompts if ambiguous |
| 1 | Pull develop/main | 🤖 |
| 2 | `git flow release start` | 🤖 |
| 3 | Review DB migrations | 🚩 Django only |
| 4 | Bump version | 🤖 |
| 5 | Changelog | 📝 from git log + closed issues/PRs |
| 6 | Deploy notes | 📝 Django only |
| 7 | Local settings sample | 🚩 Django only |
| 8 | Internal Python deps need release/pin | 🚩 |
| 9 | Internal JS deps need release | 🚩 only if JS frontend |
| 10 | `npm audit fix` | 🤖 only if JS frontend |
| 11–12 | Tests + coverage | 🤖 info only — reported, not flagged |
| 13 | Docs/schema | 🚩 Django only |
| 14 | Lock deps | 🤖 (`pip freeze`/`uv lock --check`, see below) |
| 15 | `git flow release finish` | ❌ out of scope — human does this after review + CI |
| — | Push + open PR | 🤖 |
| — | Monitor CI checks | 🤖 (stopping point) |

Non-applicable items (not Django / no JS frontend) are simply omitted from the final summary — don't list them as "n/a".

## Setup

**Ask first:** repo path (or offer to clone); PR base branch (default `main`). Don't ask for version — step 1 determines it. Confirm `git status --porcelain` is clean before doing anything; if dirty, stop and ask rather than stash/discard.

**Resuming a release already in progress:** check `git branch --list 'release/*'` and `git ls-remote --heads origin 'release/*'` first. If a release branch exists, check it out (don't re-run `git flow release start`), diff it against develop/main to see what's already done, and report that before touching anything. If a PR's already open, jump to step 16.

**Project config** — look for `.release-prep.yml` at repo root first; any field it sets skips auto-detection for that item:
```yaml
django: false
js_frontend: false
dependency_manager: uv          # uv | pip
version_file: src/mypackage/__init__.py
changelog_file: CHANGELOG.md
deploy_notes_file: docs/deploynotes.rst
settings_sample_file: mypackage/settings/local_settings.py.sample
coverage_threshold: 95
pr_base_branch: main
```
If it doesn't exist, auto-detect (below) and offer to write it after a successful run — only if the user says yes, as its own commit. If it exists but looks stale (e.g. `django: false` with `manage.py` present), flag the mismatch rather than overriding it.

**Auto-detection** (whatever the config didn't set):
- Django: `manage.py` at root, or `django` in `requirements*.txt`/`pyproject.toml`/`setup.cfg` → steps 3, 6, 7, 13 apply.
- JS frontend: `package.json` with real `dependencies`/`scripts` (not just a stray config file) → steps 9, 10 apply.
- Dependency manager: `uv.lock` or `[tool.uv]` in `pyproject.toml` → uv; else pip. Determines step 14.

## Workflow

**1. Determine next version** (before any branch work). Find current version + last tag (`git describe --tags --abbrev=0` on `main`). Gather signal: `git log <last-tag>..develop --oneline --no-merges` plus closed issues/PRs (see step 5, pull labels). Classify: any breaking signal → major; any feature, no breaking → minor; only fixes/chores → patch. If the repo uses major.minor versioning (e.g. `0.5`→`0.6`) and signal is patch-only, or signal is genuinely mixed/unclear, **ask the user**. Otherwise state the version and reason in one line and continue.

**2. Sync branches**
```bash
git fetch --all --prune
git checkout develop && git pull
git checkout main && git pull
```

**3. Start release branch** — skip if resuming.
```bash
git flow release start <version>
```
Fall back to `git checkout -b release/<version> develop` if `git flow` isn't installed; note the fallback.

**4. Review migrations** (Django only)
```bash
git diff main...HEAD --stat -- '**/migrations/*.py'
```
List changed files in the summary; don't judge correctness.

**5. Bump version.** Use `version_file` from config if set; otherwise search, in order, and update **all** matches found (strip `-pre`/`-dev`/`.dev0`): `<package>/__init__.py` or `src/<package>/__init__.py` (`__version__`) → `pyproject.toml` (`version`) → `setup.cfg`/`setup.py`. Show the diff.

**6. Draft changelog** from two sources:
```bash
git log <last-tag>..HEAD --oneline --no-merges
gh issue list --state closed --milestone "<version>" --json number,title,labels   # if repo uses milestones
# else:
gh issue list --state closed --search "closed:>=<last-tag-date>" --json number,title,labels
gh pr list --state merged --search "merged:>=<last-tag-date>" --json number,title,labels
```
Cross-reference `#123`-style references in commits to avoid noise. Group Features/Fixes/Other by label or commit prefix. Insert at top of the changelog file (`CHANGELOG.rst`/`.md`/`docs/changelog.rst`, or `changelog_file` from config), marked `<!-- DRAFT — generated from git log + closed issues/PRs -->`. In the final summary, list every closed issue/PR the query found with ✅ included / ⚠️ excluded + reason, so the user can catch anything the heuristics missed.

**7. Draft deploy notes** (Django only). Find the file (`docs/deploynotes.rst`, `docs/deploy_notes.md`, or `deploy_notes_file` from config). Draft bullets from the diff: new settings not in the sample → "needs configuring"; new migrations → "run migrations"; new/changed management commands → possible one-off step. Mark as draft.

**8. Check local settings sample** (Django only). Diff the sample settings file against real settings for missing keys; list gaps.

**9. Check Python deps**
```bash
grep -E 'git\+|@ git|-e git' requirements*.txt pyproject.toml 2>/dev/null
```
Flag internal deps pinned to a branch/commit instead of a release.

**10. Check JS deps + audit** (only if JS frontend)
```bash
grep -E '"file:|"git\+|workspace:' package.json
npm audit fix   # not --force
```
Flag internal deps that look unreleased.

**11–12. Tests + coverage**
```bash
pytest --cov --cov-report=term-missing   # or: python manage.py test
```
Report pass/fail and coverage % in the summary as information, not as something requiring a decision — don't gate the PR on it or write tests unprompted. If it's below the configured threshold (`coverage_threshold`, default 95), just say so plainly alongside the number; it's context for the reviewer, not a flagged item.

**13. Docs/schema** (Django only). If migrations changed models and schema-diagram tooling exists (e.g. `django-extensions graph_models`), regenerate and commit. Otherwise flag for manual review.

**14. Lock dependencies.**
- pip: `pip freeze > requirements.lock && git add requirements.lock`
- uv: `uv lock --check`; if it fails, run `uv lock` and stage the result. (uv.lock is normally kept current every commit — don't `pip freeze` here.)

**15. Commit, push, open PR.** PR body must open with an AI-generated-content marker, then the same drafted-changelog/flagged-items content as the final summary (minus CI status, which isn't known yet):
```
:robot: This PR was prepared by an AI agent (release-prep skill). Review the drafted and flagged sections below before merging.

<changelog draft summary, issues included/excluded, deploy notes summary, flagged items, coverage %, next steps>
```
```bash
git push -u origin release/<version>
gh pr create --base <base-branch> --head release/<version> --title "Release <version>" --body "<body above>"
```
If `gh` is unavailable/unauthenticated, prepare the title/body and give the user the compare URL instead.

**16. Monitor CI** (stopping point).
```bash
gh pr checks <pr-number-or-branch> --watch
```
Or a one-shot `gh pr view <pr> --json statusCheckRollup` if watching isn't appropriate. Report pass/fail per check; don't fix failures unprompted — surface them for the user.

## Final summary

```
## Release <version> prep complete — PR opened: <url>

### CI status
- <check>: ✅/❌/⏳

### Version
- Last: <last-tag> → Next: <version> (<major|minor|patch> — <reason>)

### Drafted — review before merge
- Changelog (sources used): closed issues/PRs since <last-tag>:
  - #123 "Title" — ✅ included (commit <sha>) / ⚠️ excluded (<reason>)
- Deploy notes: <summary>   [omit this line entirely if not Django]

### Test coverage (info)
- <%> (threshold <n>%) — <above/below threshold, no action implied>

### Needs your judgment
[List only items that actually apply and have findings — e.g. migrations, local
settings gaps, unreleased internal deps, docs/schema flags. Omit any item that's
n/a for this repo (not Django, no JS frontend, nothing found) — don't list it as
"n/a", just leave it out entirely.]
- Migrations: <list>
- Local settings sample: <gaps>
- Python/JS deps needing release: <list>
- Docs/schema: <finding>

### What to do next
1. Review the drafted changelog and deploy notes above (and in the PR).
2. Resolve anything under "Needs your judgment."
3. Once CI passes and review looks good: merge the PR, then run `git flow release finish <version>` yourself (merges to main+develop, tags, cleans up the branch).
4. After that: push tags, and bump develop's version per the "after release" checklist.
```

## Notes

- Never force-push, rewrite shared history, or run `git flow release finish` — human calls those.
- If any 🤖 step fails, stop and report rather than pushing a broken branch.
- If an expected file (changelog, deploy notes, settings sample) isn't where expected, search briefly, then say so in the summary rather than skipping silently.

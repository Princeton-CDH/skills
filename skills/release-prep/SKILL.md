---
name: release-prep
description: Preps a Princeton-CDH-style Python release branch (git flow) and opens a PR for CI — version bump, changelog/deploy-notes drafts, Django/JS checks, dependency checks, tests/coverage. Stops before merge/tag/`git flow release finish`. Use whenever the user wants to cut, start, or prep a release, mentions "git flow release start", or references the software_release.md checklist.
license: Apache-2.0
metadata:
  author: princeton-cdh
  version: "1.1.0"
---

# Release Prep

Automates the "release prep" half of Princeton-CDH's [software release checklist](https://github.com/Princeton-CDH/.github/blob/main/.github/ISSUE_TEMPLATE/software_release.md) for Python repos using git flow. Stops at opening a draft PR (used to verify CI) — never merges, tags, or runs `git flow release finish`; the "after release" checklist section is out of scope. Plain markdown + shell/git/gh — portable to any agentic tool.

## Scope

🤖 auto · 📝 draft (needs review) · 🚩 flag (needs human judgment)

| Item                                            | Handling                                                               |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| Determine next version                          | 🤖, prompts if ambiguous                                               |
| Pull develop/main                               | 🤖                                                                     |
| `git flow release start`                        | 🤖                                                                     |
| Review DB migrations                            | 🚩 Django only                                                         |
| Bump version                                    | 🤖                                                                     |
| Changelog                                       | 📝 from git log + closed issues/PRs                                    |
| Open milestone issues, dependabot PRs           | 🚩                                                                     |
| Deploy notes                                    | 📝 Django only                                                         |
| Local settings sample                           | 🚩 Django only                                                         |
| Internal Python deps need release/pin           | 🚩                                                                     |
| Internal JS deps need release                   | 🚩 only if JS frontend                                                 |
| `npm audit fix`                                 | 🤖 only if JS frontend                                                 |
| Tests + coverage                                | 🤖 info only — reported, not flagged                                   |
| Docs/schema                                     | 🚩 Django only                                                         |
| Lock deps                                       | 🤖 (`pip freeze` / `uv lock --check`)                                  |
| `git flow release finish`                       | ❌ out of scope — human does this after review + CI                    |
| Push + open draft PR                            | 🤖                                                                     |
| Monitor CI checks                               | 🤖                                                                     |
| Post status update to tracking issue            | 🤖 best-effort, comment only — never edits checkboxes (stopping point) |

Items that don't apply to this repo's stack (not Django / no JS frontend) are simply omitted from the final summary — don't list them as "n/a". This is different from a step that _should_ have run but couldn't (see "Skipped or failed" below): those must always be reported.

## Pre-Workflow Setup

**Requirements:** `git` with `git-flow`; the GitHub CLI (`gh`), authenticated; `uv` if the repo uses it. Preflight with `gh auth status` before doing anything. If `gh` is missing or unauthenticated, stop and tell the user up front that the closed issue/PR changelog sources, PR creation, CI monitoring, and the tracking-issue comment all depend on it, and ask whether to continue with local-only steps.

**Ask first:** repo path (or offer to clone); PR base branch (default `main`). Don't ask for the version — the first workflow step determines it. Confirm `git status --porcelain` is clean before doing anything; if dirty, stop and ask rather than stash/discard.

**Resuming a release already in progress:** check `git branch --list 'release/*'` and `git ls-remote --heads origin 'release/*'` first. If a release branch exists, check it out (don't re-run `git flow release start`), diff it against develop/main to see what's already done, and report that before touching anything. If a PR is already open, skip to the CI-monitoring step.

**Project config** — look for `.release-prep.yml` at repo root first; any field it sets skips auto-detection for that item:

```yaml
django: false
js_frontend: false
dependency_manager: uv # uv | pip
version_file: src/mypackage/__init__.py
release_candidate_suffix: "" # optional, literal suffix e.g. "rc1" or "-rc1"; omit for final versions
test_command: "uv run pytest --cov" # optional override
changelog_file: CHANGELOG.md
deploy_notes_file: docs/deploynotes.rst
settings_sample_file: mypackage/settings/local_settings.py.sample
coverage_threshold: 95
pr_base_branch: main
```

If it doesn't exist, auto-detect (below) and offer to write it after a successful run — only if the user says yes, as its own commit. If it exists but looks stale (e.g. `django: false` with `manage.py` present), flag the mismatch rather than overriding it.

**Auto-detection** (whatever the config didn't set):

- Django: `manage.py` at root, or `django` in `requirements*.txt`/`pyproject.toml`/`setup.cfg`. Enables the Django-only steps.
- JS frontend: `package.json` with real `dependencies`/`scripts` (not just a stray config file). Enables the JS steps.
- Dependency manager: `uv.lock` or `[tool.uv]` in `pyproject.toml` → uv; else pip. Determines how tests are run and how dependencies are locked.

## Workflow

**1. Determine next version** (before any branch work). Find current version + last tag (`git describe --tags --abbrev=0` on `main`). Gather signal: `git log <last-tag>..develop --oneline --no-merges` plus closed issues/PRs and their labels (see the changelog step). Classify: any breaking signal → major; any feature, no breaking → minor; only fixes/chores → patch. If the repo uses major.minor versioning (e.g. `0.5`→`0.6`) and signal is patch-only, or signal is genuinely mixed/unclear, **ask the user**. Otherwise state the version and reason in one line and continue.

**2. Sync branches**

```bash
git fetch --all
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

**5. Bump version.** Use `version_file` from config if set; otherwise search, in order, and update **all** matches found (strip `-pre`/`-dev`/`.dev0`): `<package>/__init__.py` or `src/<package>/__init__.py` (`__version__`) → `pyproject.toml` (`version`) → `setup.cfg`/`setup.py`. If `release_candidate_suffix` is configured, append it literally; the human removes it later (see next steps). Show the diff.

**6. Draft changelog** from two sources:

```bash
git log <last-tag>..HEAD --oneline --no-merges
gh issue list --state closed --milestone "<version>" --json number,title,labels   # if repo uses milestones
# else:
gh issue list --state closed --search "closed:>=<last-tag-date>" --json number,title,labels
gh pr list --state merged --search "merged:>=<last-tag-date>" --json number,title,labels
```

Cross-reference `#123`-style references in commits to avoid noise. Group Features/Fixes/Other by label or commit prefix. Insert at top of the changelog file (`CHANGELOG.rst`/`.md`/`docs/changelog.rst`, or `changelog_file` from config), marked `<!-- DRAFT — generated from git log + closed issues/PRs -->`. Make updates consistent with existing changelog entries in style, tone, and formatting; don't overcomplicate and focus on user-facing features. In the final summary, list every closed issue/PR the query found with ✅ included / ⚠️ excluded + reason, so the user can catch anything the heuristics missed.

**7. Check open work** (flag only)

```bash
gh issue list --state open --milestone "<version>" --json number,title,labels   # if repo uses milestones
gh pr list --author "app/dependabot" --state open --json number,title
```

List open milestone issues as unresolved, **excluding** the release tracking issue itself and post-release tasks (e.g. titled "Deploy …" or labeled as deployment). If unsure whether an issue is post-release, list it separately as "possibly post-release" rather than as unresolved. List open dependabot PRs so the user can decide which to pull into this release; don't merge any.

**8. Draft deploy notes** (Django only). Find the file (`docs/deploynotes.rst`, `docs/deploy_notes.md`, or `deploy_notes_file` from config). Draft bullets from the diff: new settings not in the sample → "needs configuring"; new migrations → "run migrations"; new/changed management commands → possible one-off step. Mark as draft.

**9. Check local settings sample** (Django only). Diff the sample settings file against real settings for missing keys; list gaps. If no sample file exists, or the only one is test-only, skip quietly and omit it from the summary.

**10. Check Python deps**

```bash
grep -E 'git\+|@ git|-e git' requirements*.txt pyproject.toml 2>/dev/null
```

Flag internal deps pinned to a branch/commit instead of a release.

**11. Check JS deps + audit** (only if JS frontend)

```bash
grep -E '"file:|"git\+|workspace:' package.json
npm audit fix   # not --force
```

Flag internal deps that look unreleased.

**12. Tests + coverage.** Use `test_command` from config if set. Otherwise, in uv projects run tests through uv (`uv run pytest --cov --cov-report=term-missing`, or `uv run python manage.py test`); in pip projects use the active environment (`pytest --cov --cov-report=term-missing`, or `python manage.py test`). Report pass/fail and coverage % as information, not as something requiring a decision — don't gate the PR on it or write tests unprompted. If it's below the configured threshold (`coverage_threshold`, default 95), say so plainly alongside the number. If the tests can't be run (missing environment, wrong command), report that as skipped/failed with the reason — never imply they passed.

**13. Docs/schema** (Django only). If migrations changed models and schema-diagram tooling exists (e.g. `django-extensions graph_models`), regenerate and commit. Otherwise flag for manual review.

**14. Lock dependencies.**

- pip: `pip freeze > requirements.lock && git add requirements.lock`
- uv: `uv lock --check`; if it fails, run `uv lock` and stage the result. (uv.lock is normally kept current every commit — don't `pip freeze` here.)

**15. Commit, push, open draft PR.**

_Find the tracking issue first:_ `gh issue list --search "in:title Release <version>" --state open` (or a label like `release-checklist` if the repo uses one). If zero or multiple candidates, don't guess — omit the issue link and note it in the summary.

_Commit:_ one or a few logical commits, keeping the version bump, changelog, deploy notes, and lock file easy to review. Every commit message must include an `Assisted-by: <agent>:<model>` trailer, since these commits end up in the main history. If a pre-commit hook fails: if it only reformatted files, re-stage and retry once; otherwise stop and report the hook output. Never use `--no-verify`.

_PR body_ must open with an AI-generated-content marker, then the associated issue, then the same drafted-changelog/flagged-items content as the final summary (minus CI status, which isn't known yet), and close with a skill attribution:

```
:robot: This PR was prepared by an AI agent (release-prep skill) as a draft to verify CI. Review the drafted and flagged sections below before finishing the release.

Associated issue: Closes #<tracking-issue>   [omit if none found]

<changelog draft summary, issues included/excluded, deploy notes summary, flagged items, coverage %, next steps>

---
:robot: Generated by the `release-prep` skill v<metadata.version from frontmatter> — https://github.com/Princeton-CDH/skills/tree/main/skills/release-prep. Something look wrong? Flag it there.
```

```bash
git push -u origin release/<version>
gh pr create --draft --base <base-branch> --head release/<version> --title "Release <version>" --body "<body above>"
```

**Only say a PR was opened if `gh pr create` returned a URL.** If `gh` is unavailable, unauthenticated, or the command failed, say plainly that the branch was pushed but no PR was created, print the PR title/body for manual use, and give the compare URL.

**16. Monitor CI.**

```bash
gh pr checks <pr-number-or-branch> --watch
```

Or a one-shot `gh pr view <pr> --json statusCheckRollup` if watching isn't appropriate. Report pass/fail per check; don't fix failures unprompted — surface them for the user.

**17. Post status update to tracking issue** (best-effort, actual stopping point). Using the tracking issue found when opening the PR, post the final summary as a comment: `gh issue comment <number> --body-file -`, including the PR link and CI status. If no issue was found, skip and note it in the chat summary.

**Never edit the issue body / toggle its checkboxes.** GitHub has no per-checkbox API — "checking a box" means reading the whole body and writing it back, which risks silently clobbering a concurrent human edit and matching the wrong line if item wording has drifted from the template. A comment is append-only and carries no such risk, at the cost of not visually ticking the template's checkboxes — that stays a manual step for whoever reviews the release.

## Final summary

Post this as-is to the tracking issue and also show it in chat:

```
## Release <version> prep complete — draft PR: <url>
[If no PR was created: "## Release <version> branch prepped — PR NOT created (<reason>)", then include the PR body for manual use.]

### CI status
- <check>: ✅/❌/⏳

### Version
- Last: <last-tag> → Next: <version> (<major|minor|patch> — <reason>)

### Drafted — review before finishing the release
- Changelog (sources used): closed issues/PRs since <last-tag>:
  - #123 "Title" — ✅ included (commit <sha>) / ⚠️ excluded (<reason>)
- Deploy notes: <summary>   [omit this line entirely if not Django]

### Test coverage (info)
- <%> (threshold <n>%) — <above/below threshold, no action implied>

### Needs your judgment
[List only items that apply and have findings; omit anything that doesn't apply
to this repo or had nothing found — don't write "n/a".]
- Open milestone issues: <list; "possibly post-release" listed separately>
- Dependabot PRs to consider including: <list>
- Migrations: <list>
- Local settings sample: <gaps>
- Python/JS deps needing release: <list>
- Docs/schema: <finding>

### Skipped or failed
[Steps that should have run but couldn't — e.g. gh unauthenticated, tests couldn't
run, pre-commit hook failed — each with the reason. Omit the section if none.]

### What to do next
1. Review the drafted changelog and deploy notes above (and in the PR).
2. Resolve anything under "Needs your judgment."
3. If a release candidate suffix was used: after acceptance testing, remove it from the version.
4. Once CI passes and review looks good: run `git flow release finish <version>` yourself (merges to main and develop, tags, deletes the branch), then push main, develop, and tags. The draft PR should then show as merged; close it manually if it doesn't.
5. Then complete the "after release" checklist (e.g. bump develop's version).
```

## Notes

- Never force-push, rewrite shared history, or run `git flow release finish` — human calls those.
- If any 🤖 step fails, stop and report rather than pushing a broken branch.
- Never skip a step silently. A step that doesn't apply to the repo's stack is omitted from the summary; a step that should have run but couldn't goes under "Skipped or failed" with the reason. If an expected file (changelog, deploy notes, settings sample) isn't where expected, search briefly, then report it the same way.
- When editing this skill's own workflow, bump `metadata.version` in the frontmatter (semver) — the PR footer reports it for traceability.

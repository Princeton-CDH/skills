# release-prep

An agent skill that does the "release prep" half of the Princeton-CDH
[software release checklist](https://github.com/Princeton-CDH/.github/blob/main/.github/ISSUE_TEMPLATE/software_release.md)
for Python repos that use git flow. The instructions the agent follows are in
[`SKILL.md`](../skills/release-prep/SKILL.md); this file is for humans.

## What it does

- Works out the next version number (and asks you if it's ambiguous)
- Creates the release branch with `git flow release start`
- Bumps the version, drafts a changelog entry (from git history plus closed
  issues/PRs), and drafts deploy notes (Django only)
- Runs checks and flags what needs your judgment: migrations, local settings,
  unreleased internal dependencies, open milestone issues, dependabot PRs
- Runs tests and reports coverage (information only)
- Locks dependencies (`uv.lock` check or `requirements.lock`)
- Opens a **draft** PR so CI can run, watches the checks, and posts a status
  summary as a comment on the release tracking issue

## What it doesn't do

It never merges, tags, or runs `git flow release finish`, and it never edits
the checkboxes on the tracking issue. Those stay with you. The final summary
ends with a "What to do next" list.

## Requirements

- `git` with `git-flow`
- the GitHub CLI (`gh`), authenticated (`gh auth login`) — without it the
  skill can only do the local steps
- `uv`, if the repo uses it

## Optional config: `.release-prep.yml`

The skill auto-detects Django, a JS frontend, and uv vs. pip. If detection is
wrong or you want to pin settings once per repo, add a `.release-prep.yml` at
the repo root. Any field you set is used as-is; anything you leave out is
auto-detected. See the example with all supported fields in
[`SKILL.md`](../skills/release-prep/SKILL.md#setup) (step 5).

If there's no config file, the skill offers to write one after a successful run.

## Everything it generates is a draft

Review the changelog, deploy notes, and flagged items before finishing the
release. The PR and its commits are marked as AI-assisted, and the PR footer
records which skill version produced it. If something looks wrong, open an
issue on this repo.

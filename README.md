# common-workflows

Composite actions shared across the `lite-actions` repos, so a fix lands once
instead of five times.

| action | what it does |
| --- | --- |
| `shell-test` | runs the shell test suite |
| `shell-lint` | shellchecks `scripts/` and `tests/` |
| `conventional-validation` | validates commit messages and branch names |
| `changelog` | appends `CHANGELOG.md`, regenerates `RELEASE_NOTES.md`, lands both via an auto-merged PR |
| `publish` | tags `vX.Y.Z`, moves `vN`, cuts the GitHub Release that the Marketplace listing tracks |

Every input defaults to what the action repos already do, so a caller that
passes nothing keeps its current behaviour.

`publish` was called `release` until 2026-08-20. "Release" meant three things
across these repos — `release-notes` generates notes, `rust-release` builds
binaries, and this cuts a version — and the ambiguity had already produced a
wrong code comment. Callers invoke it from a `publish.yml` workflow.

It cuts a GitHub Release and never calls a Marketplace API, because there isn't
one: the first publish needs a human to accept the terms in the web UI, and
after that the listing tracks GitHub Releases on its own.

## Use

Each action lives at the repository root and is referenced by path:

```yaml
name: Lint
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: lite-actions/git-checkout@v1
      - uses: lite-actions/common-workflows/shell-lint@v1
```

The caller owns the job: `runs-on`, `permissions`, `concurrency` and the
checkout. These are composite actions, not reusable workflows, so they cannot
declare any of those themselves.

That is deliberate rather than a limitation to work around. Because the caller
names the job, the status check keeps its **bare name** — `lint`, not
`lint / lint`. Existing branch protection contexts (`validate`, `lint`, `test`)
keep working, so repos can migrate one at a time with no lockstep protection
edit.

### Checkout requirements

`changelog` and `publish` resolve "since the last release" from `vX.Y.Z` tags,
so they need history *and* tags:

```yaml
- uses: lite-actions/git-checkout@v1
  with:
    fetch-depth: 0
    fetch-tags: true
```

`git-checkout` appends `--no-tags` unless `fetch-tags: true`, **even at
`fetch-depth: 0`** — unlike `actions/checkout`, which implies tags at depth 0.
Getting this wrong silently versions from `0.0.0`. Both actions guard against
it: if the remote has version tags and the checkout has none, they fail with an
explicit message rather than producing wrong output. A repository before its
first release has no tags either way and is unaffected.

`conventional-validation` needs `fetch-depth: 0` so the commit range resolves.

### Two different titles

`changelog` writes two documents, and they take separate headings:

| input | goes to | writes |
| --- | --- | --- |
| `title` | `conventional-changelog` | the `CHANGELOG.md` heading, on first creation |
| `notes-title` | `release-notes` | the `RELEASE_NOTES.md` heading, every run |

Leave `notes-title` empty and `release-notes` falls back to the repository name
from `GITHUB_REPO`. Set it where the repo slug is not what readers should see —
`versioning-tests` wants "Versioning Tests", not `versioning-tests`.

### Merge method

The changelog PR is squash-merged by default. Pass `merge-method` where the
repository does not allow that:

```yaml
- uses: lite-actions/common-workflows/changelog@v1
  with:
    merge-method: rebase
```

**It must be a method the repository actually enables.** `gh pr merge` fails
outright rather than falling back, so a repo with squash merging turned off will
fail every changelog cycle on the merge step, after the PR has already been
opened and approved. Check with:

```bash
gh api repos/{owner}/{repo} --jq '{squash: .allow_squash_merge, rebase: .allow_rebase_merge, merge: .allow_merge_commit}'
```

### Secrets

Composite actions cannot read `secrets` directly; pass them as inputs.

```yaml
- uses: lite-actions/common-workflows/changelog@v1
  with:
    bot-token: ${{ secrets.CHANGELOG_BOT_TOKEN }}
    approve-token: ${{ secrets.CHANGELOG_APPROVE_TOKEN }}
```

Both are user PATs by necessity: `GITHUB_TOKEN` does not trigger required
checks on a pushed branch and cannot approve a PR at all, and a PR's author or
last pusher cannot self-approve — hence two distinct identities.

## What is not here

Workflows specific to a single action stay in that action's own repository. Two
of them cannot be centralised at all:

- `git-checkout`'s CI bootstraps from raw git and uses no action, which is what
  keeps it out of the dependency graph and makes its CI a genuine proof that the
  action replaces `actions/checkout`.
- `conventional-validator` validates its own PRs with `uses: ./` to dogfood the
  working tree rather than the last release.

Any repo that wants to test its own action rather than its last release has the
same constraint.

## Ranges

`changelog` runs two actions over deliberately different windows:

| action | range | window |
| --- | --- | --- |
| `conventional-changelog` | push `before`..`after` | this push only |
| `release-notes` | last `vX.Y.Z` tag..`HEAD` | since the last release |

`release-notes` is the sole authority on the version number —
`conventional-changelog` outputs only `changed` and `file`.

## Tags and releases

Releases are cut from `main` after the merge, never from a PR: squash and
rebase merges both rewrite the commit SHA, so a tag created at a PR head would
point at a commit that never lands. Tags are not branch-protected, which is what
lets `publish` run without a PR and approval.

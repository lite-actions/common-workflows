# lite-actions-workflows

Reusable GitHub Actions workflows shared across the `lite-actions` repos, so a
fix lands once instead of five times.

| workflow | trigger in the caller | what it does |
| --- | --- | --- |
| `test.yml` | `pull_request`, `push: main` | runs the shell test suite |
| `lint.yml` | `pull_request`, `push: main` | shellchecks `scripts/` and `tests/` |
| `conventional-validation.yml` | `pull_request` | validates commit messages and branch name |
| `changelog.yml` | `push: main` | appends `CHANGELOG.md`, regenerates `RELEASE_NOTES.md`, lands both via an auto-merged PR |
| `release.yml` | `workflow_dispatch` | tags `vX.Y.Z`, moves `vN`, publishes the GitHub Release |

Every input defaults to what the action repos already do, so a caller that
passes nothing keeps its current behaviour.

## Use

```yaml
name: Test
on:
  pull_request:
  push:
    branches: [main]
jobs:
  test:
    uses: lite-actions/lite-actions-workflows/.github/workflows/test.yml@v1
```

Triggers stay in the caller — `workflow_call` cannot declare them. That matters
most for `changelog.yml`, which needs its `paths-ignore` or the changelog PR's
own merge retriggers it in a loop:

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - CHANGELOG.md
      - RELEASE_NOTES.md
jobs:
  changelog:
    uses: lite-actions/lite-actions-workflows/.github/workflows/changelog.yml@v1
    secrets: inherit
```

## Before migrating a repo

**Required status check names change.** A job that calls a reusable workflow
reports its check as `<caller-job>/<called-job>`, not the bare job name — so
`validate` becomes `validate/validate`. The action repos require exactly
`validate`, `lint` and `test` on `main`, with `enforce_admins: true`. Update the
required contexts in the same change, or the branch blocks on checks that will
never report and there is no admin bypass. Naming the caller job to match does
not help; the called job name is always appended.

**Two workflows cannot be centralised.**

- `git-checkout`'s CI bootstraps from raw git and uses no action at all, which
  is what keeps it out of the dependency graph and makes its CI a genuine proof
  that the action replaces `actions/checkout`.
- `conventional-validator` validates its own PRs with `uses: ./` to dogfood the
  working tree. Inside a *called* workflow, `./` resolves to this repository,
  not the caller's, so it has to keep a local `conventional-validation.yml`.

Any repo that wants to test its own action rather than the last release has the
same constraint.

## Ranges

`changelog.yml` runs two actions over deliberately different windows:

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
lets `release.yml` run without a PR and approval.

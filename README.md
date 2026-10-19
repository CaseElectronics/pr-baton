# PR Assignee Baton

A GitHub composite action that automatically passes the PR "baton" between reviewers and authors using the assignee field to signal who currently needs to act.

- When a reviewer is requested, the PR is assigned to that reviewer.
- When a review is submitted, the PR is assigned back to the author.
- When a PR is opened as a draft, or converted from ready back to draft, it is assigned to the author.
- When a draft PR is marked ready for review, it is assigned to whoever is already requested as a reviewer.
- When a PR would be handed to a reviewer but has failing checks or a merge conflict, it stays with (or goes back to) the author instead.
- When a CI run fails on a PR's latest commit, the PR is assigned back to the author, even if it was already with the reviewer.

This makes it easy to see at a glance who is to act on a PR in the list of open PRs.

## Usage

Create a workflow file in your repository (e.g. `.github/workflows/pr-baton.yml`):

```yaml
name: PR Assignee Baton

on:
  pull_request: { types: [ review_requested, opened, converted_to_draft, ready_for_review ] }
  pull_request_review: { types: [ submitted ] }
  # Optional: hand the PR back to the author when CI fails. Pick the trigger(s)
  # that match your CI - see "CI failures" below.
  workflow_run: { workflows: [ CI ], types: [ completed ] }
  check_suite: { types: [ completed ] }
  status:

permissions:
  pull-requests: write
  issues: write
  contents: read
  checks: read
  statuses: read

jobs:
  assign-baton:
    runs-on: ubuntu-latest
    steps: [ { uses: CaseElectronics/pr-baton@v3 } ]
```

## Inputs

| Input          | Description                                                               | Required | Default               |
|----------------|---------------------------------------------------------------------------|----------|-----------------------|
| `github-token` | GitHub token with `pull-requests: write`, `issues: write`, `contents: read`, `checks: read` and `statuses: read` permissions. | No       | `${{ github.token }}` |

The default `github.token` is sufficient as long as your workflow grants the required permissions (see the example above).

## How it works

| Event                 | Trigger              | Action                                            |
|-----------------------|----------------------|---------------------------------------------------|
| `pull_request`        | `review_requested`   | Assigns the PR to the requested reviewer          |
| `pull_request`        | `ready_for_review`   | Assigns the PR to the already-requested reviewer  |
| `pull_request_review` | `submitted`          | Assigns the PR back to the PR author              |
| `pull_request`        | `opened` (draft)     | Assigns the PR to the PR author                   |
| `pull_request`        | `converted_to_draft` | Assigns the PR to the PR author                   |
| `workflow_run`        | `completed` (failed) | Assigns the PR back to the PR author              |
| `check_suite`         | `completed` (failed) | Assigns the PR back to the PR author              |
| `status`              | `failure` / `error`  | Assigns the PR back to the PR author              |

Before handing a PR to a reviewer (`review_requested`, `ready_for_review`), the action checks the PR's latest commit. If any check has failed, or the PR has a merge conflict with its base branch, the PR is assigned to the author instead.

### CI failures

Which trigger picks up a failed CI result depends on how your CI reports it:

- **GitHub Actions**: use `workflow_run`, and list the names of your CI workflows under `workflows`. GitHub does not send `check_suite` events for suites created by Actions, so `workflow_run` is the only way to see them.
- **Third-party CI using the Checks API** (e.g. CircleCI, Buildkite): use `check_suite`.
- **CI reporting commit statuses** (e.g. Jenkins): use `status`.

Keep only the triggers you need. `workflow_run` and `check_suite` only fire from the workflow file on the default branch.

**Edge cases:**
- If the requested reviewer is the GitHub Copilot bot or the reviewer field is empty (e.g. when a team is requested rather than an individual), the assignment step is skipped.
- `opened` only assigns when the PR is opened as a draft; non-draft PRs are left alone until a reviewer is requested.
- `ready_for_review` is a no-op when no reviewer has been requested yet.
- Bot authors (`dependabot[bot]`, `renovate[bot]`, …) cannot be assignees, so the author steps skip them rather than failing.
- A CI result only counts if it is for the PR's current head commit; results for commits that have since been pushed over are ignored.
- Checks still running when a review is requested don't block the handoff; if one of them fails later, the CI trigger hands the PR back.
- A merge conflict is only checked when the PR is handed to a reviewer. A conflict caused later by the base branch moving doesn't reassign the PR on its own (see below).

### Why merge conflicts aren't watched continuously

GitHub has no event for "this PR now has a merge conflict". A PR usually becomes conflicted because something else was merged into its base branch, and the only way to catch that would be to run on every `push` to the base branch and re-check every open PR against it. That costs a workflow run plus a mergeability check per open PR for every merge, for a situation that only matters once the PR is handed over.

So the handoff is the checkpoint instead: when the author requests a review, the PR has to be mergeable to go to the reviewer. If a conflict appears while the reviewer has it, it stays with the reviewer until it comes back to the author.

This could be added later as an opt-in `push` trigger if conflicts arising mid-review turn out to be a problem in practice.
- The action never hands a PR back to the reviewer automatically once CI is fixed; the author does that by re-requesting review.

## License

MIT

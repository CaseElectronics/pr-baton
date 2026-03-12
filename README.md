# PR Assignee Baton

A GitHub composite action that automatically passes the PR "baton" between reviewers and authors using the assignee field to signal who currently needs to act.

- When a reviewer is requested, the PR is assigned to that reviewer.
- When a review is submitted, the PR is assigned back to the author.

This makes it easy to see at a glance who is to act on a PR in the list of open PRs.

## Usage

Create a workflow file in your repository (e.g. `.github/workflows/pr-baton.yml`):

```yaml
name: PR Assignee Baton

on:
  pull_request: { types: [ review_requested ] }
  pull_request_review: { types: [ submitted ] }

permissions:
  pull-requests: write
  issues: write

jobs:
  assign-baton: { runs-on: ubuntu-latest, steps: [ { uses: CaseElectronics/pr-baton@v1 } ] }
```

## Inputs

| Input          | Description                                                               | Required | Default               |
|----------------|---------------------------------------------------------------------------|----------|-----------------------|
| `github-token` | GitHub token with `pull-requests: write` and `issues: write` permissions. | No       | `${{ github.token }}` |

The default `github.token` is sufficient as long as your workflow grants the required permissions (see the example above).

## How it works

| Event                 | Trigger            | Action                                   |
|-----------------------|--------------------|------------------------------------------|
| `pull_request`        | `review_requested` | Assigns the PR to the requested reviewer |
| `pull_request_review` | `submitted`        | Assigns the PR back to the PR author     |

**Edge cases:**
- If the requested reviewer is the GitHub Copilot bot or the reviewer field is empty (e.g. when a team is requested rather than an individual), the assignment step is skipped.

## License

MIT

# pr-gatory

A merge queue for GitHub Actions. PRs enter purgatory, only the worthy get merged.

Label a PR with `mergeme` and pr-gatory will:

1. **Rebase** the PR branch onto the latest base branch
2. **Wait** for all status checks to pass on the rebased code
3. **Squash merge** into the base branch
4. **Dequeue** with a comment if anything fails

Uses GitHub Actions [concurrency groups](https://docs.github.com/en/actions/using-jobs/using-concurrency) to serialize merges — only one PR is processed at a time, so each PR is always tested against the true latest state of your base branch.

## Usage

Add this workflow to your repository at `.github/workflows/merge-queue.yml`:

```yaml
name: Merge Queue

on:
  pull_request:
    types: [labeled]

# Only one merge at a time — this IS the queue
concurrency:
  group: merge-queue
  cancel-in-progress: false

jobs:
  merge:
    if: github.event.label.name == 'mergeme'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: opper-ai/pr-gatory-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

Then label any PR with `mergeme` to add it to the queue.

## How it works

```
PR labeled "mergeme"
        |
        v
  +-----------+     conflict    +----------+
  |  Rebase   | ------------->  | Dequeue  |
  |  onto     |                 | + comment|
  |  base     |                 +----------+
  +-----------+
        |
     success
        |
        v
  +-----------+     failure     +----------+
  |  Wait for | ------------->  | Dequeue  |
  |  status   |                 | + comment|
  |  checks   |    timeout      |          |
  +-----------+ ------------->  +----------+
        |
    all pass
        |
        v
  +-----------+
  |  Squash   |
  |  merge    |
  +-----------+
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `github-token` | GitHub token with repo permissions | `${{ github.token }}` |
| `label` | Label that triggers the merge queue | `mergeme` |
| `poll-interval` | Seconds between status check polls | `30` |
| `timeout` | Max seconds to wait for status checks | `1800` (30 min) |

## Outputs

| Output | Description |
|--------|-------------|
| `result` | `merged`, `failed`, or `timeout` |

## Requirements

- The repository must allow squash merges
- The token must have permission to push to the PR branch (for rebase) and merge PRs
- For repos with branch protection: use a GitHub App token or PAT instead of the default `GITHUB_TOKEN`

## Why not GitHub's built-in merge queue?

GitHub's native merge queue requires GitHub Enterprise or specific plan tiers. pr-gatory works on any plan, is fully transparent (it's just a workflow), and you control the behavior.

## License

MIT

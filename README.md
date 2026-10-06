# mnmal-ai/.github

Org-wide reusable GitHub Actions for the `mnmal-ai` organization.

## Available Actions

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `pr-review.yml` | `pull_request` | AI-powered code review with configurable expert perspectives |
| `issue-triage.yml` | `issues` | Auto-labels, adds context, and checks for duplicates |

Both are powered by [Claude](https://anthropic.com) via `anthropics/claude-code-action`.

## Adopting in Your Repo

### 1. Add the config file

Create `.github/ai-review.yml` in your repo. The minimal version uses all defaults:

```yaml
pr_review: {}
```

Full config with all options:

```yaml
pr_review:
  model: claude-sonnet-4-6        # optional
  ignore_paths:                    # optional
    - docs/**
    - "**/*.md"
  perspectives:                    # optional — defaults to Architecture/Testing/Security
    - name: My Expert
      focus: "What this expert looks for in the diff"

issue_triage:
  model: claude-sonnet-4-6        # optional
  repo_context: |                 # describe your repo in 1-3 sentences
    Your repo description here.
  areas:                          # module names for labeling and codebase search
    - core
    - api
    - cli
```

Omit `pr_review` to disable PR review. Omit `issue_triage` to disable issue triage.

### 2. Add the caller workflow

Create `.github/workflows/ai-actions.yml`:

```yaml
name: AI Actions
on:
  pull_request:
    types: [opened, synchronize]
  issues:
    types: [opened]

# The reusable jobs request these permissions; a caller must grant at least
# the same, or the run fails before any step (startup_failure).
permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write

jobs:
  pr-review:
    if: github.event_name == 'pull_request'
    uses: mnmal-ai/.github/.github/workflows/pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

  issue-triage:
    if: github.event_name == 'issues'
    uses: mnmal-ai/.github/.github/workflows/issue-triage.yml@main
    permissions:
      contents: read
      issues: write
      id-token: write
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Both workflows prefer `CLAUDE_CODE_OAUTH_TOKEN` (billed to the Claude subscription) when it is set, and fall back to `ANTHROPIC_API_KEY`. Pass whichever your repo has; passing both is fine.

### 3. Ask an org admin to add your repo to the secret allowlist

The `ANTHROPIC_API_KEY` org secret must be explicitly granted to your repo.
Contact an org admin or open an issue in this repo requesting access.

## Security Notes

- The `ANTHROPIC_API_KEY` is passed explicitly, not via `secrets: inherit`, to avoid leaking unrelated repo secrets.
- Workflows do not request `id-token: write` — OIDC tokens are not needed.
- Claude is instructed to read only `gh pr diff` output and ignore any instructions embedded in PR content.
- Keep the org secret on an explicit allowlist (not "All repositories") to control billing exposure.

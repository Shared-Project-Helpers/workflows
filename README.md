# Shared Workflows

Reusable GitHub Actions workflows for Claude-powered code review and fixes.

## Workflows

### Claude Code Review (`claude-review.yml`)

Automatically reviews pull requests using Claude.

**Usage:**

```yaml
# .github/workflows/review.yml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]

jobs:
  review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@claude'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      # Optional: customize the model
      model: 'claude-sonnet-4-20250514'
      # Optional: add project-specific review guidelines
      review_guidelines: |
        - Follow Kotlin coding conventions
        - Use coroutines for async operations
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-sonnet-4-20250514` | Claude model to use |
| `review_guidelines` | No | `''` | Additional project-specific guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |

**Outputs:**

| Output | Description |
|--------|-------------|
| `decision` | Review decision: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT` |

---

### Claude Fix Agent (`claude-fix.yml`)

Automatically applies fixes based on review feedback.

**Usage:**

```yaml
# .github/workflows/fix.yml
name: Fix Agent

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'PR number to fix'
        required: true
  issue_comment:
    types: [created]

jobs:
  fix:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@claude fix'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-sonnet-4-20250514` | Claude model to use |
| `max_tokens` | No | `8192` | Max tokens for fix response |
| `commit_author_name` | No | `Claude Fix Agent` | Git author name |
| `commit_author_email` | No | `claude-fix-agent@users.noreply.github.com` | Git author email |

**Outputs:**

| Output | Description |
|--------|-------------|
| `files_changed` | Number of files changed |

---

## Setup

1. Add `ANTHROPIC_API_KEY` to your repository secrets
2. Create wrapper workflows in your repo (see examples above)
3. Open a PR to test

## Triggering

- **Review:** Runs automatically on PR open/update, or comment `@claude`
- **Fix:** Runs when review requests changes, or comment `@claude fix`

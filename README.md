# Shared Workflows

Reusable GitHub Actions workflows for Claude-powered code review, automated fixes, and merge readiness checks.

## Workflows

### Claude Code Review (`claude-review.yml`)

Automatically reviews pull requests using Claude. On `REQUEST_CHANGES`, it can dispatch a fix workflow. On `APPROVE`, it can dispatch a merge readiness check.

**Usage:**

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

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
      APP_PRIVATE_KEY: ${{ secrets.MY_APP_PRIVATE_KEY }}
    with:
      reviewer_name: 'Thorin Oakenshield'
      app_id: ${{ vars.MY_APP_ID }}
      review_guidelines: |
        - Follow Kotlin coding conventions
        - Use coroutines for async operations
      auto_merge_label: 'auto-merge'
      auto_merge_method: 'squash'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-5-20251101` | Claude model to use |
| `review_guidelines` | No | `''` | Additional project-specific review guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `auto_merge_label` | No | `''` | Label that enables auto-merge on approval (empty = disabled) |
| `auto_merge_method` | No | `squash` | Merge method: `merge`, `squash`, or `rebase` |
| `reviewer_name` | No | `Claude` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow when review requests changes |
| `trigger_merge_check_on_approve` | No | `'true'` | Dispatch merge check workflow when review approves |
| `fix_workflow_id` | No | `claude-fix.yml` | Fix workflow filename to dispatch |
| `merge_check_workflow_id` | No | `claude-merge-check.yml` | Merge check workflow filename to dispatch |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key for Claude |
| `APP_PRIVATE_KEY` | No | GitHub App private key for custom bot identity |

**Outputs:**

| Output | Description |
|--------|-------------|
| `decision` | Review decision: `APPROVE`, `REQUEST_CHANGES`, or `COMMENT` |
| `pr_number` | Pull request number |
| `head_ref` | PR head branch ref |

---

### Claude Fix Agent (`claude-fix.yml`)

Automatically applies fixes based on review feedback. Also updates the PR branch if it is behind the base branch.

**Usage:**

```yaml
name: Fix Agent

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'PR number to fix'
        required: true
        type: string
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write

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
      PAT: ${{ secrets.PAT }}
      APP_PRIVATE_KEY: ${{ secrets.MY_APP_PRIVATE_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      fixer_name: 'Gloin'
      app_id: ${{ vars.MY_APP_ID }}
      commit_author_name: 'Gloin'
      commit_author_email: 'gloin@example.com'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-opus-4-5-20251101` | Claude model to use |
| `max_tokens` | No | `8192` | Max tokens for fix response |
| `fixer_name` | No | `Claude Fix Agent` | Display name for the fixer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `commit_author_name` | No | `Claude Fix Agent` | Git author name for fix commits |
| `commit_author_email` | No | `claude-fix-agent@users.noreply.github.com` | Git author email |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key for Claude |
| `PAT` | No | Personal Access Token for pushing (enables workflow re-triggers) |
| `APP_PRIVATE_KEY` | No | GitHub App private key for custom bot identity |

**Outputs:**

| Output | Description |
|--------|-------------|
| `files_changed` | Number of files changed |

---

### Claude Merge Readiness Check (`claude-merge-check.yml`)

Assesses whether a PR is ready to merge by checking CI status, review approvals, merge conflicts, and draft state. If the branch is behind the base, it can dispatch the fix workflow to update it.

**Usage:**

```yaml
name: Merge Readiness Check

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'PR number to check'
        required: true
        type: string
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write
  issues: write
  checks: write
  actions: write

jobs:
  check:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@balin'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-merge-check.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      checker_name: 'Balin'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to check |
| `model` | No | `claude-opus-4-5-20251101` | Claude model to use |
| `checker_name` | No | `Balin` | Display name for the checker persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_behind` | No | `'true'` | Dispatch fix workflow when branch is behind base |
| `fix_workflow_id` | No | `claude-fix.yml` | Fix workflow filename to dispatch |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key for Claude |
| `APP_PRIVATE_KEY` | No | GitHub App private key for custom bot identity |

**Outputs:**

| Output | Description |
|--------|-------------|
| `ready` | Whether the PR is ready to merge (`true`/`false`) |
| `blocking_issues` | Number of blocking issues found |
| `behind` | Whether the PR branch is behind the base (`true`/`false`) |
| `head_ref` | PR head branch ref |

---

## Pipeline Flow

By default, the workflows chain together automatically:

```
PR opened/updated
  -> claude-review.yml (Code Review)
       |
       |-- REQUEST_CHANGES -> claude-fix.yml (Apply Fixes) -> push triggers re-review
       |
       |-- APPROVE -> claude-merge-check.yml (Merge Readiness)
                        |
                        |-- branch behind -> claude-fix.yml (Update Branch)
                        |
                        |-- ready -> auto-merge (if label set)
```

Set `trigger_fix_on_request_changes: 'false'`, `trigger_merge_check_on_approve: 'false'`, or `trigger_fix_on_behind: 'false'` to disable any of these automatic dispatches.

## Setup

1. Add `ANTHROPIC_API_KEY` to your repository secrets
2. (Optional) Create a GitHub App for a custom bot identity and add `APP_PRIVATE_KEY` secret and `APP_ID` variable
3. (Optional) Add a `PAT` secret to enable fix commits that re-trigger CI workflows
4. Create caller workflows in your repo (see usage examples above)
5. Grant the required permissions listed in each example

## Triggering

- **Review**: Runs automatically on PR open/update, or comment `@claude` on a PR
- **Fix**: Dispatched automatically on `REQUEST_CHANGES`, or comment `@claude fix` on a PR
- **Merge Check**: Dispatched automatically on `APPROVE`, or comment `@balin` on a PR

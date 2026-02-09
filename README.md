# Shared Workflows

Reusable GitHub Actions workflows for Claude-powered code review, automated fixes, and merge readiness checks.

## Agent Roster

| Role | Workflow | Trigger |
|------|----------|---------|
| **General Reviewer** — Code quality, bugs, logic | `claude-review.yml` | PR open/sync, `@review` |
| **Performance Reviewer** — Performance, efficiency, cost | `claude-perf-review.yml` | PR open/sync, `@perf-review` |
| **Safety Reviewer** — Security, privacy, OWASP | `claude-safety-review.yml` | PR open/sync, `@safety-review` |
| **Test Reviewer** — Test coverage, test quality | `claude-test-review.yml` | PR open/sync, `@test-review` |
| **Fix Agent** — Fix General Reviewer's feedback | `claude-fix.yml` | Dispatched by General Reviewer, `@fix` |
| **Issue Fix Agent** — Fix all reviewers' feedback, create issues | `claude-issue-fix.yml` | Dispatched by other reviewers, `@issue-fix` |
| **Merge Checker** — Merge gate, verifies all approvals | `claude-merge-check.yml` | Dispatched by General Reviewer, `@merge-check` |

## Pipeline Flow

```
PR opened/updated
  ├──→ General Reviewer              ─┐
  ├──→ Performance Reviewer          ─┤  all four run in parallel
  ├──→ Safety Reviewer               ─┤
  └──→ Test Reviewer                 ─┘
         │
         ├─ General Reviewer REQUEST_CHANGES      → Fix Agent (simple fix)       → push re-triggers all
         ├─ Performance Reviewer REQUEST_CHANGES  → Issue Fix Agent (issue fix)  → push re-triggers all
         ├─ Safety Reviewer REQUEST_CHANGES       → Issue Fix Agent (issue fix)  → push re-triggers all
         ├─ Test Reviewer REQUEST_CHANGES         → Issue Fix Agent (issue fix)  → push re-triggers all
         │
         └─ General Reviewer APPROVE → Merge Checker
                                        ├─ checks: CI + all required reviewer approvals
                                        └─ ready → auto-merge (if 'auto-merge' label)
```

Issue Fix Agent creates GitHub Issues for critical problems outside PR scope (deferred items).

---

## Workflows

### General Reviewer (`claude-review.yml`)

Reviews pull requests for code quality, bugs, and logic. On `REQUEST_CHANGES`, dispatches Fix Agent. On `APPROVE`, dispatches Merge Checker.

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
       contains(github.event.comment.body, '@review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'General Reviewer'
      review_guidelines: |
        - Follow Kotlin coding conventions
        - Use coroutines for async operations
      auto_merge_label: 'auto-merge'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `review_guidelines` | No | `''` | Additional project-specific review guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `auto_merge_label` | No | `''` | Label that enables auto-merge on approval |
| `auto_merge_method` | No | `squash` | Merge method: `merge`, `squash`, or `rebase` |
| `reviewer_name` | No | `General Reviewer` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow on REQUEST_CHANGES |
| `trigger_merge_check_on_approve` | No | `'true'` | Dispatch merge check on APPROVE |
| `fix_workflow_id` | No | `claude-fix.yml` | Fix workflow filename to dispatch |
| `merge_check_workflow_id` | No | `claude-merge-check.yml` | Merge check workflow to dispatch |
| `create_tracked_issues` | No | `'true'` | Create GitHub Issues for non-blocking concerns |

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`

---

### Performance Reviewer (`claude-perf-review.yml`)

Reviews pull requests for performance, efficiency, and cost concerns. Categories: API efficiency, bandwidth, caching, algorithm complexity, memory, network, resource cleanup, cost optimization. On `REQUEST_CHANGES`, dispatches Issue Fix Agent.

**Usage:**

```yaml
jobs:
  perf-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@perf-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-perf-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'Performance Reviewer'
      perf_guidelines: |
        - Watch for N+1 query patterns
        - Ensure proper use of Kotlin coroutines
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `perf_guidelines` | No | `''` | Additional performance guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `reviewer_name` | No | `Performance Reviewer` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow on REQUEST_CHANGES |
| `fix_workflow_id` | No | `claude-issue-fix.yml` | Fix workflow filename to dispatch |

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`

---

### Safety Reviewer (`claude-safety-review.yml`)

Reviews pull requests for security, privacy, and safety concerns. Runs Semgrep SAST and GitLeaks scans before the Claude review. Covers OWASP Top 10, secrets, PII, injection, auth, encryption, and privacy-by-design. On `REQUEST_CHANGES`, dispatches Issue Fix Agent.

**Usage:**

```yaml
jobs:
  safety-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@safety-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-safety-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'Safety Reviewer'
      run_semgrep: 'true'
      run_gitleaks: 'true'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `safety_guidelines` | No | `''` | Additional safety guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `reviewer_name` | No | `Safety Reviewer` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow on REQUEST_CHANGES |
| `fix_workflow_id` | No | `claude-issue-fix.yml` | Fix workflow filename to dispatch |
| `run_semgrep` | No | `'true'` | Run Semgrep SAST scan |
| `run_gitleaks` | No | `'true'` | Run GitLeaks secrets scan |
| `semgrep_rules` | No | `auto` | Semgrep ruleset to use |

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`, `semgrep_findings`, `gitleaks_findings`

---

### Test Reviewer (`claude-test-review.yml`)

Reviews pull requests for test coverage, test quality, and missing tests. Detects test files and frameworks, optionally runs tests, then asks Claude to assess whether the changes have adequate test coverage. On `REQUEST_CHANGES`, dispatches Issue Fix Agent.

**Usage:**

```yaml
jobs:
  test-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@test-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-test-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'Test Reviewer'
      test_command: './gradlew test'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `test_guidelines` | No | `''` | Additional test review guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `reviewer_name` | No | `Test Reviewer` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow on REQUEST_CHANGES |
| `fix_workflow_id` | No | `claude-issue-fix.yml` | Fix workflow filename to dispatch |
| `test_command` | No | `''` | Command to run tests (empty = skip execution) |

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`, `tests_passed`

---

### Fix Agent (`claude-fix.yml`)

Applies fixes based on General Reviewer's feedback. Also updates the PR branch if it is behind the base branch.

**Usage:**

```yaml
jobs:
  fix:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@fix'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      fixer_name: 'Fix Agent'
      commit_author_name: 'Fix Agent'
      commit_author_email: 'fix-agent@users.noreply.github.com'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `max_tokens` | No | `8192` | Max tokens for fix response |
| `fixer_name` | No | `Fix Agent` | Display name for the fixer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `commit_author_name` | No | `Fix Agent` | Git author name |
| `commit_author_email` | No | `fix-agent@users.noreply.github.com` | Git author email |

**Secrets:** `ANTHROPIC_API_KEY` (required), `PAT` (optional), `APP_PRIVATE_KEY` (optional)

**Outputs:** `files_changed`

---

### Issue Fix Agent (`claude-issue-fix.yml`)

Collects feedback from ALL reviewers and applies fixes with priority ordering. Creates GitHub Issues for deferred items that cannot be fixed within the PR scope. Uses a concurrency group to prevent parallel runs on the same PR.

**Priority order:**
1. Critical security issues (Safety Reviewer)
2. Critical performance issues (Performance Reviewer)
3. Critical quality issues (General Reviewer)
4. Critical test issues (Test Reviewer)
5. Major issues in same order
6. Minor issues if straightforward

**Usage:**

```yaml
jobs:
  fix:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@issue-fix'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-issue-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      fixer_name: 'Issue Fix Agent'
      commit_author_name: 'Issue Fix Agent'
      commit_author_email: 'issue-fix-agent@users.noreply.github.com'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `max_tokens` | No | `8192` | Max tokens for fix response |
| `fixer_name` | No | `Issue Fix Agent` | Display name for the fixer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `commit_author_name` | No | `Issue Fix Agent` | Git author name |
| `commit_author_email` | No | `issue-fix-agent@users.noreply.github.com` | Git author email |
| `create_issues_for_deferred` | No | `'true'` | Create GitHub Issues for deferred items |
| `issue_labels` | No | `deferred,review-feedback` | Labels for created issues |

**Secrets:** `ANTHROPIC_API_KEY` (required), `PAT` (optional), `APP_PRIVATE_KEY` (optional)

**Outputs:** `files_changed`, `fixes_applied`, `fixes_deferred`, `issues_created`

---

### Merge Checker (`claude-merge-check.yml`)

Assesses whether a PR is ready to merge by checking CI status, review approvals, merge conflicts, draft state, and required reviewer approvals.

When `required_reviewer_names` is set, Merge Checker verifies each named reviewer has posted an APPROVE decision before allowing merge.

**Usage:**

```yaml
jobs:
  check:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@merge-check'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-merge-check.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      checker_name: 'Merge Checker'
      required_reviewer_names: 'Performance Reviewer,Safety Reviewer,Test Reviewer'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to check |
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `checker_name` | No | `Merge Checker` | Display name for the checker persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `required_reviewer_names` | No | `''` | Comma-separated required reviewer names |
| `trigger_fix_on_behind` | No | `'true'` | Dispatch fix workflow when branch is behind |
| `fix_workflow_id` | No | `claude-fix.yml` | Fix workflow to dispatch |

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `ready`, `blocking_issues`, `behind`, `head_ref`

---

## Setup

1. Add `ANTHROPIC_API_KEY` to your repository secrets
2. (Optional) Create a GitHub App for a custom bot identity and add `APP_PRIVATE_KEY` secret and `APP_ID` variable
3. (Optional) Add a `PAT` secret to enable fix commits that re-trigger CI workflows
4. Create caller workflows in your repo (see usage examples above)
5. Grant the required permissions listed in each example

## Triggering

- **General Reviewer**: Runs automatically on PR open/update, or comment `@review`
- **Performance Reviewer**: Runs automatically on PR open/update, or comment `@perf-review`
- **Safety Reviewer**: Runs automatically on PR open/update, or comment `@safety-review`
- **Test Reviewer**: Runs automatically on PR open/update, or comment `@test-review`
- **Fix Agent**: Dispatched automatically by General Reviewer on `REQUEST_CHANGES`, or comment `@fix`
- **Issue Fix Agent**: Dispatched automatically by other reviewers on `REQUEST_CHANGES`, or comment `@issue-fix`
- **Merge Checker**: Dispatched automatically by General Reviewer on `APPROVE`, or comment `@merge-check`

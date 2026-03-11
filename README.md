# Shared Workflows

Reusable GitHub Actions workflows for Claude-powered code review, automated fixes, and merge readiness checks.

## Architecture

```
Consumer repo (triggers + secrets)
  └──→ Agent wrapper (persona config, guidelines, settings)
         └──→ Core workflow (implementation)
```

Consumer repos are thin boilerplate — they define triggers and pass secrets. All agent configuration (persona names, review guidelines, merge settings, commit author info) lives in **agent wrapper** workflows in this repo. Core workflows contain the implementation.

## Agent Roster

| Persona | Wrapper | Core | Role |
|---------|---------|------|------|
| **Anthropic (Code Review)** | `anthropic-code-review.yml` | `claude-code-review.yml` | Minimal code review (claude-code-action) |
| **Thorin (Code Review)** | `thorin-code-review.yml` | `claude-code-review.yml` | Code quality, bugs, logic (claude-code-action) |
| **Thorin (Code Review)** | `thorin-review.yml` | `claude-review.yml` | Code quality, bugs, logic (Python SDK, deprecated) |
| **Dwalin (Performance)** | `dwalin-perf-review.yml` | `claude-perf-review.yml` | Performance, efficiency, cost |
| **Óin (Safety)** | `oin-safety-review.yml` | `claude-safety-review.yml` | Security, privacy, OWASP |
| **Dori (Test)** | `dori-test-review.yml` | `claude-test-review.yml` | Test coverage, test quality |
| **Glóin (Fix)** | `gloin-fix.yml` | `claude-fix.yml` | Fix General Reviewer's feedback |
| **Glóin (Fix)** | `gloin-issue-fix.yml` | `claude-issue-fix.yml` | Fix all reviewers' feedback, create issues |
| **Balin (Merge)** | `balin-merge-check.yml` | `claude-merge-check.yml` | Merge gate, verifies all approvals |
| **Nori (Test Runner)** | `nori-test-runner.yml` | `claude-test-runner.yml` | Autonomous test generation & execution |

## Pipeline Flow

```
PR opened/updated
  ├──→ Anthropic (Code Review)          ─┐
  ├──→ Thorin (Code Review)             ─┤
  ├──→ Dwalin (Performance Review)      ─┤  all five run in parallel
  ├──→ Óin (Safety Review)              ─┤
  └──→ Dori (Test Review)               ─┘
         │
         ├─ Thorin REQUEST_CHANGES      → Glóin Fix Agent          → push re-triggers all
         ├─ Dwalin REQUEST_CHANGES      → Glóin Issue Fix Agent   → push re-triggers all
         ├─ Óin REQUEST_CHANGES         → Glóin Issue Fix Agent   → push re-triggers all
         ├─ Dori REQUEST_CHANGES        → Glóin Issue Fix Agent   → push re-triggers all
         │
         └─ Thorin APPROVE → Balin (Merge Check)
                                ├─ checks: CI + all required reviewer approvals
                                └─ ready → auto-merge (if 'auto-merge' label)
```

Glóin Issue Fix Agent creates GitHub Issues for critical problems outside PR scope (deferred items).

---

## Agent Wrappers (consumer repos call these)

Wrappers embed all agent configuration and forward dynamic inputs (like `app_id`, `pr_number`) and secrets from the consumer.

### Anthropic (Code Review) — Code Review (`anthropic-code-review.yml`)

Wraps `claude-code-review.yml` using `anthropics/claude-code-action@v1`. Minimal prompt — no custom persona, just reviews for bugs, security issues, and correctness. Interactive follow-ups via `@anthropic`.

**Consumer usage:**

```yaml
name: Anthropic Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

concurrency:
  group: anthropic-review-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: false

jobs:
  review:
    if: |
      (github.event_name == 'pull_request' &&
       github.event.sender.login != 'github-actions[bot]') ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       !endsWith(github.event.sender.login, '[bot]') &&
       contains(github.event.comment.body, '@anthropic')) ||
      (github.event_name == 'pull_request_review_comment' &&
       !endsWith(github.event.sender.login, '[bot]') &&
       contains(github.event.comment.body, '@anthropic'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/anthropic-code-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      APP_PRIVATE_KEY: ${{ secrets.THORIN_APP_PRIVATE_KEY }}
    with:
      app_id: ${{ vars.THORIN_APP_ID }}
```

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

---

### Thorin (Code Review) — Code Review (`thorin-code-review.yml`)

Wraps `claude-code-review.yml` using `anthropics/claude-code-action@v1`. Full codebase access, interactive follow-ups via `@thorin`, progress tracking, and fix links.

**Consumer usage:**

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: write

concurrency:
  group: review-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: false

jobs:
  review:
    if: |
      (github.event_name == 'pull_request' &&
       github.event.sender.login != 'github-actions[bot]') ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       !endsWith(github.event.sender.login, '[bot]') &&
       contains(github.event.comment.body, '@thorin')) ||
      (github.event_name == 'pull_request_review_comment' &&
       !endsWith(github.event.sender.login, '[bot]') &&
       contains(github.event.comment.body, '@thorin'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/thorin-code-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      APP_PRIVATE_KEY: ${{ secrets.THORIN_APP_PRIVATE_KEY }}
    with:
      app_id: ${{ vars.THORIN_APP_ID }}
```

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

---

### Thorin (Code Review) — Code Review, deprecated (`thorin-review.yml`)

> **Deprecated**: Use `thorin-code-review.yml` instead. This wrapper uses the Python SDK approach (diff-only review). Kept for rollback.

Wraps `claude-review.yml`. Embeds: persona name, KMP review guidelines, auto-merge settings.

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`

---

### Dwalin (Performance) — Performance Review (`dwalin-perf-review.yml`)

Wraps `claude-perf-review.yml`. Embeds: persona name, KMP performance guidelines.

**Consumer usage:**

```yaml
jobs:
  perf-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@perf-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/dwalin-perf-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`

---

### Óin (Safety) — Safety & Privacy Review (`oin-safety-review.yml`)

Wraps `claude-safety-review.yml`. Embeds: persona name, safety guidelines, Semgrep + GitLeaks enabled.

**Consumer usage:**

```yaml
jobs:
  safety-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@safety-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/oin-safety-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`, `semgrep_findings`, `gitleaks_findings`

---

### Dori (Test) — Test Review (`dori-test-review.yml`)

Wraps `claude-test-review.yml`. Embeds: persona name, KMP test guidelines.

**Consumer usage:**

```yaml
jobs:
  test-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@test-review'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/dori-test-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Inputs:** `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `decision`, `pr_number`, `head_ref`, `tests_passed`

---

### Glóin (Fix) — Fix Agent (`gloin-fix.yml`)

Wraps `claude-fix.yml`. Embeds: persona name, commit author.

**Consumer usage:**

```yaml
jobs:
  fix:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@claude fix'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/gloin-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      PAT: ${{ secrets.PAT }}
      APP_PRIVATE_KEY: ${{ secrets.GLOIN_APP_PRIVATE_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      app_id: ${{ vars.GLOIN_APP_ID }}
```

**Inputs:** `pr_number` (required), `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `PAT` (optional), `APP_PRIVATE_KEY` (optional)

**Outputs:** `files_changed`

---

### Glóin (Fix) — Issue Fix Agent (`gloin-issue-fix.yml`)

Wraps `claude-issue-fix.yml`. Embeds: persona name, commit author.

**Consumer usage:**

```yaml
jobs:
  fix:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@issue-fix'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/gloin-issue-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      PAT: ${{ secrets.PAT }}
      APP_PRIVATE_KEY: ${{ secrets.GLOIN_APP_PRIVATE_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      app_id: ${{ vars.GLOIN_APP_ID }}
```

**Inputs:** `pr_number` (required), `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `PAT` (optional), `APP_PRIVATE_KEY` (optional)

**Outputs:** `files_changed`, `fixes_applied`, `fixes_deferred`, `issues_created`

---

### Balin (Merge) — Merge Readiness Check (`balin-merge-check.yml`)

Wraps `claude-merge-check.yml`. Embeds: persona name.

**Consumer usage:**

```yaml
jobs:
  check:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@balin'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/balin-merge-check.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      APP_PRIVATE_KEY: ${{ secrets.THORIN_APP_PRIVATE_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      required_reviewer_names: 'Dwalin (Performance),Óin (Safety),Dori (Test)'
      app_id: ${{ vars.THORIN_APP_ID }}
```

**Inputs:** `pr_number` (required), `required_reviewer_names` (optional), `app_id` (optional)

**Secrets:** `ANTHROPIC_API_KEY` (required), `APP_PRIVATE_KEY` (optional)

**Outputs:** `ready`, `blocking_issues`, `behind`, `head_ref`

---

### Nori (Test Runner) — Test Runner (`nori-test-runner.yml`)

Wraps `claude-test-runner.yml`. Embeds: persona name, commit author, KMP test guidelines. Provides KMP defaults for test_command, source_paths, test_paths, setup_java (overridable).

**Consumer usage:**

```yaml
name: Test Runner

on:
  workflow_dispatch:
  schedule:
    - cron: '0 6 * * 1'
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  test-runner-standalone:
    if: |
      github.event_name == 'workflow_dispatch' ||
      github.event_name == 'schedule'
    uses: Shared-Project-Helpers/workflows/.github/workflows/nori-test-runner.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      PAT: ${{ secrets.PAT }}

  test-runner-pr:
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request &&
      contains(github.event.comment.body, '@test-runner')
    uses: Shared-Project-Helpers/workflows/.github/workflows/nori-test-runner.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      PAT: ${{ secrets.PAT }}
    with:
      mode: 'pr'
      pr_number: ${{ github.event.issue.number }}
```

**Inputs:** `mode`, `pr_number`, `test_command`, `source_paths`, `test_paths`, `setup_java`, `app_id` (all optional with KMP defaults)

**Secrets:** `ANTHROPIC_API_KEY` (required), `PAT` (optional), `APP_PRIVATE_KEY` (optional)

**Outputs:** `tests_generated`, `tests_passed`, `pr_number`

---

## Core Workflows (reference)

Core workflows are called by wrappers, not directly by consumer repos. See wrapper docs above for consumer usage.

| Core Workflow | Inputs | Secrets | Outputs |
|--------------|--------|---------|---------|
| `claude-code-review.yml` | trigger_phrase, prompt, settings, app_id, additional_permissions | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | — |
| `claude-review.yml` *(deprecated)* | model, review_guidelines, diff_limit, excluded_files, auto_merge_label, auto_merge_method, reviewer_name, app_id, trigger_fix_on_request_changes, trigger_merge_check_on_approve, fix_workflow_id, merge_check_workflow_id, create_tracked_issues | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | decision, pr_number, head_ref |
| `claude-perf-review.yml` | model, perf_guidelines, diff_limit, excluded_files, reviewer_name, app_id, trigger_fix_on_request_changes, fix_workflow_id | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | decision, pr_number, head_ref |
| `claude-safety-review.yml` | model, safety_guidelines, diff_limit, excluded_files, reviewer_name, app_id, trigger_fix_on_request_changes, fix_workflow_id, run_semgrep, run_gitleaks, semgrep_rules | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | decision, pr_number, head_ref, semgrep_findings, gitleaks_findings |
| `claude-test-review.yml` | model, test_guidelines, diff_limit, excluded_files, reviewer_name, app_id, trigger_fix_on_request_changes, fix_workflow_id, test_command | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | decision, pr_number, head_ref, tests_passed |
| `claude-fix.yml` | model, pr_number, max_tokens, commit_author_name, commit_author_email, fixer_name, app_id | ANTHROPIC_API_KEY, PAT, APP_PRIVATE_KEY | files_changed |
| `claude-issue-fix.yml` | model, pr_number, max_tokens, commit_author_name, commit_author_email, fixer_name, app_id, create_issues_for_deferred, issue_labels | ANTHROPIC_API_KEY, PAT, APP_PRIVATE_KEY | files_changed, fixes_applied, fixes_deferred, issues_created |
| `claude-merge-check.yml` | model, pr_number, checker_name, app_id, required_reviewer_names, trigger_fix_on_behind, fix_workflow_id | ANTHROPIC_API_KEY, APP_PRIVATE_KEY | ready, blocking_issues, behind, head_ref |
| `claude-test-runner.yml` | mode, pr_number, model, max_iterations, test_command, source_paths, test_paths, test_guidelines, setup_java, setup_node, runner_name, commit_author_name, commit_author_email, branch_prefix, close_stale_prs, app_id | ANTHROPIC_API_KEY, PAT, APP_PRIVATE_KEY | tests_generated, tests_passed, pr_number |

---

## Setup

1. Add `ANTHROPIC_API_KEY` to your repository secrets
2. (Optional) Create GitHub Apps for custom bot identities and add `APP_PRIVATE_KEY` secrets and `APP_ID` variables
3. (Optional) Add a `PAT` secret to enable fix commits that re-trigger CI workflows
4. Create caller workflows in your repo using the agent wrapper examples above
5. Grant the required permissions listed in each example

## Adding a New Project

1. Create thin workflow files in your repo that call the agent wrappers (see consumer usage examples above)
2. Each file only needs: triggers (`on:`), permissions, job `if:` condition, `uses:` pointing to the wrapper, secrets, and `app_id`
3. All agent config (persona names, guidelines, merge settings) comes from the wrappers automatically

## Triggering

- **Anthropic (Code Review)**: Runs automatically on PR open/update, or comment `@anthropic` (interactive follow-ups supported)
- **Thorin (Code Review)**: Runs automatically on PR open/update, or comment `@thorin` (interactive follow-ups supported)
- **Dwalin (Performance)**: Runs automatically on PR open/update, or comment `@perf-review`
- **Óin (Safety)**: Runs automatically on PR open/update, or comment `@safety-review`
- **Dori (Test)**: Runs automatically on PR open/update, or comment `@test-review`
- **Glóin (Fix)**: Dispatched automatically by Thorin on `REQUEST_CHANGES`, or comment `@claude fix`
- **Glóin (Fix) Issue Fix**: Dispatched automatically by other reviewers on `REQUEST_CHANGES`, or comment `@issue-fix`
- **Balin (Merge)**: Dispatched automatically by Thorin on `APPROVE`, or comment `@balin`
- **Nori (Test Runner)**: Scheduled weekly, manual dispatch, or comment `@test-runner` on a PR

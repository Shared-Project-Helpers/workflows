# Shared Workflows

Reusable GitHub Actions workflows for Claude-powered code review, automated fixes, and merge readiness checks.

## Agent Roster

| Agent | Persona | Role | Workflow | Trigger |
|-------|---------|------|----------|---------|
| General Review | **Thorin** | Code quality, bugs, logic | `claude-review.yml` | PR open/sync, `@claude` |
| Performance Review | **Dwalin** | Performance, efficiency, cost | `claude-perf-review.yml` | PR open/sync, `@dwalin` |
| Safety & Privacy | **Oin** | Security, privacy, OWASP | `claude-safety-review.yml` | PR open/sync, `@oin` |
| Simple Fix | **Gloin** | Fix Thorin's feedback | `claude-fix.yml` | Dispatched by Thorin, `@claude fix` |
| Issue Fix & Manage | **Nori** | Fix any reviewer's feedback, create issues | `claude-issue-fix.yml` | Dispatched by Dwalin/Oin, `@nori` |
| Merge Readiness | **Balin** | Merge gate, verifies all approvals | `claude-merge-check.yml` | Dispatched by Thorin, `@balin` |

## Pipeline Flow

```
PR opened/updated
  ├──→ Thorin (general review)      ─┐
  ├──→ Dwalin (performance review)  ─┤  all three run in parallel
  └──→ Oin (safety/privacy review)  ─┘
         │
         ├─ Thorin REQUEST_CHANGES  → Gloin (simple fix) → push re-triggers all
         ├─ Dwalin REQUEST_CHANGES  → Nori (issue fix)   → push re-triggers all
         ├─ Oin REQUEST_CHANGES     → Nori (issue fix)   → push re-triggers all
         │
         └─ Thorin APPROVE → Balin (merge readiness)
                               ├─ checks: CI + Thorin + Dwalin + Oin approvals
                               └─ ready → auto-merge (if 'auto-merge' label)
```

Nori creates GitHub Issues for critical problems outside PR scope (deferred items).

---

## Workflows

### Claude Code Review (`claude-review.yml`) — Thorin

Automatically reviews pull requests using Claude for general code quality, bugs, and logic. On `REQUEST_CHANGES`, it dispatches Gloin (fix workflow). On `APPROVE`, it dispatches Balin (merge readiness check).

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
    with:
      reviewer_name: 'Thorin Oakenshield'
      review_guidelines: |
        - Follow Kotlin coding conventions
        - Use coroutines for async operations
      auto_merge_label: 'auto-merge'
      auto_merge_method: 'squash'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
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

### Claude Performance Review (`claude-perf-review.yml`) — Dwalin

Reviews pull requests for performance, efficiency, and cost concerns. Focuses on client-server efficiency, bandwidth, caching, algorithm complexity, memory, network, resource cleanup, and cost optimization. On `REQUEST_CHANGES`, dispatches Nori (issue fix workflow).

**Usage:**

```yaml
name: Performance Review

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
  perf-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@dwalin'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-perf-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'Dwalin'
      perf_guidelines: |
        - Watch for N+1 query patterns
        - Ensure proper use of Kotlin coroutines
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `perf_guidelines` | No | `''` | Additional project-specific performance guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `reviewer_name` | No | `Dwalin` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow when review requests changes |
| `fix_workflow_id` | No | `claude-issue-fix.yml` | Fix workflow filename to dispatch |

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

### Claude Safety & Privacy Review (`claude-safety-review.yml`) — Oin

Reviews pull requests for security, privacy, and safety concerns. Runs Semgrep SAST and GitLeaks scans before the Claude review, injecting tool results as structured context. Covers OWASP Top 10, secrets, PII, injection, auth, encryption, and privacy-by-design. On `REQUEST_CHANGES`, dispatches Nori (issue fix workflow).

**Usage:**

```yaml
name: Safety & Privacy Review

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
  safety-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       contains(github.event.comment.body, '@oin'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-safety-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      reviewer_name: 'Oin'
      run_semgrep: 'true'
      run_gitleaks: 'true'
      safety_guidelines: |
        - Never commit API keys or credentials
        - All PII must be encrypted at rest
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `safety_guidelines` | No | `''` | Additional project-specific safety guidelines |
| `diff_limit` | No | `50000` | Max diff size in bytes |
| `excluded_files` | No | `:!*.lock :!package-lock.json` | Git pathspec for excluded files |
| `reviewer_name` | No | `Oin` | Display name for the reviewer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `trigger_fix_on_request_changes` | No | `'true'` | Dispatch fix workflow when review requests changes |
| `fix_workflow_id` | No | `claude-issue-fix.yml` | Fix workflow filename to dispatch |
| `run_semgrep` | No | `'true'` | Run Semgrep SAST scan before Claude review |
| `run_gitleaks` | No | `'true'` | Run GitLeaks secrets scan before Claude review |
| `semgrep_rules` | No | `auto` | Semgrep ruleset to use |

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
| `semgrep_findings` | Number of Semgrep findings |
| `gitleaks_findings` | Number of GitLeaks findings |

---

### Claude Fix Agent (`claude-fix.yml`) — Gloin

Automatically applies fixes based on Thorin's review feedback. Also updates the PR branch if it is behind the base branch.

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
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      fixer_name: 'Gloin'
      commit_author_name: 'Gloin'
      commit_author_email: 'gloin@users.noreply.github.com'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-opus-4-6` | Claude model to use |
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

### Claude Issue Fix Agent (`claude-issue-fix.yml`) — Nori

Collects feedback from ALL reviewers (Thorin, Dwalin, Oin) and applies fixes with priority ordering. Creates GitHub Issues for deferred items that cannot be fixed within the PR scope. Uses a concurrency group to prevent parallel runs on the same PR.

**Priority order:**
1. Critical security issues (Oin)
2. Critical performance issues (Dwalin)
3. Critical quality issues (Thorin)
4. Major issues in same order
5. Minor issues if straightforward

**Usage:**

```yaml
name: Issue Fix Agent

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
       contains(github.event.comment.body, '@nori'))
    uses: Shared-Project-Helpers/workflows/.github/workflows/claude-issue-fix.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.inputs.pr_number || github.event.issue.number }}
      fixer_name: 'Nori'
      commit_author_name: 'Nori'
      commit_author_email: 'nori@users.noreply.github.com'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to fix |
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `max_tokens` | No | `8192` | Max tokens for fix response |
| `fixer_name` | No | `Nori` | Display name for the fixer persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `commit_author_name` | No | `Nori` | Git author name for fix commits |
| `commit_author_email` | No | `nori@users.noreply.github.com` | Git author email |
| `create_issues_for_deferred` | No | `'true'` | Create GitHub Issues for deferred items |
| `issue_labels` | No | `deferred,review-feedback` | Comma-separated labels for created issues |

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
| `fixes_applied` | Number of fixes successfully applied |
| `fixes_deferred` | Number of issues deferred to GitHub Issues |
| `issues_created` | Number of GitHub Issues created |

---

### Claude Merge Readiness Check (`claude-merge-check.yml`) — Balin

Assesses whether a PR is ready to merge by checking CI status, review approvals, merge conflicts, draft state, and required reviewer approvals. If the branch is behind the base, it can dispatch the fix workflow to update it.

**Required Approvals:** When `required_reviewer_names` is set, Balin checks that each named reviewer persona has posted an APPROVE decision in their review comments. This ensures all specialized reviewers (e.g., Dwalin for performance, Oin for safety) have signed off before merge.

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
      required_reviewer_names: 'Dwalin,Oin'
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pr_number` | Yes | - | Pull request number to check |
| `model` | No | `claude-opus-4-6` | Claude model to use |
| `checker_name` | No | `Balin` | Display name for the checker persona |
| `app_id` | No | `''` | GitHub App ID for custom bot identity |
| `required_reviewer_names` | No | `''` | Comma-separated list of required reviewer persona names (e.g., `Dwalin,Oin`) |
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

## Setup

1. Add `ANTHROPIC_API_KEY` to your repository secrets
2. (Optional) Create a GitHub App for a custom bot identity and add `APP_PRIVATE_KEY` secret and `APP_ID` variable
3. (Optional) Add a `PAT` secret to enable fix commits that re-trigger CI workflows
4. Create caller workflows in your repo (see usage examples above)
5. Grant the required permissions listed in each example

## Triggering

- **Thorin (Review)**: Runs automatically on PR open/update, or comment `@claude` on a PR
- **Dwalin (Perf Review)**: Runs automatically on PR open/update, or comment `@dwalin` on a PR
- **Oin (Safety Review)**: Runs automatically on PR open/update, or comment `@oin` on a PR
- **Gloin (Fix)**: Dispatched automatically by Thorin on `REQUEST_CHANGES`, or comment `@claude fix` on a PR
- **Nori (Issue Fix)**: Dispatched automatically by Dwalin/Oin on `REQUEST_CHANGES`, or comment `@nori` on a PR
- **Balin (Merge Check)**: Dispatched automatically by Thorin on `APPROVE`, or comment `@balin` on a PR

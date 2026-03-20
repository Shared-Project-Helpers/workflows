# CLAUDE.md

Reusable GitHub Actions workflows for Claude-powered CI/CD automation (code review, automated fixes, merge readiness checks).

## Project structure

### Agent wrappers (consumer repos call these)

- `.github/workflows/anthropic-code-review.yml` — Anthropic (Code Review): minimal claude-code-action code review wrapper
- `.github/workflows/thorin-review.yml` — Thorin (Code Review): Python SDK code review wrapper (deprecated)
- `.github/workflows/thorin-code-review.yml` — Thorin (Code Review): claude-code-action code review wrapper
- `.github/workflows/dwalin-perf-review.yml` — Dwalin (Performance): performance review wrapper
- `.github/workflows/oin-safety-review.yml` — Óin (Safety): safety & privacy review wrapper
- `.github/workflows/dori-test-review.yml` — Dori (Test): test review wrapper
- `.github/workflows/gloin-fix.yml` — Glóin (Fix): fix agent wrapper
- `.github/workflows/gloin-issue-fix.yml` — Glóin (Fix): issue fix agent wrapper
- `.github/workflows/balin-merge-check.yml` — Balin (Merge): merge readiness check wrapper
- `.github/workflows/bifur-combined-review.yml` — Bifur (Combined): combined perf/safety/test review wrapper
- `.github/workflows/nori-test-runner.yml` — Nori (Test Runner): test runner wrapper

### Core workflows (called by wrappers, not directly by consumers)

- `.github/workflows/claude-code-review.yml` — Code Review (Action): claude-code-action based review (full codebase access, interactive)
- `.github/workflows/claude-review.yml` — General Reviewer: automated PR code review (Python SDK, deprecated)
- `.github/workflows/claude-perf-review.yml` — Performance Reviewer: performance, efficiency, cost review
- `.github/workflows/claude-safety-review.yml` — Safety Reviewer: security & privacy review (with Semgrep + GitLeaks)
- `.github/workflows/claude-test-review.yml` — Test Reviewer: test coverage and quality review
- `.github/workflows/claude-fix.yml` — Fix Agent: applies fixes from General Reviewer's feedback
- `.github/workflows/claude-issue-fix.yml` — Issue Fix Agent: applies fixes from all reviewers, creates GitHub Issues for deferred items
- `.github/workflows/claude-merge-check.yml` — Merge Checker: merge readiness assessment (supports required reviewer approvals)
- `.github/workflows/claude-combined-review.yml` — Combined Reviewer: single-call perf/safety/test review (Sonnet 4.5)
- `.github/workflows/claude-test-runner.yml` — Test Runner: autonomous test generation, execution, and iteration

### Other

- `README.md` — Usage docs and configuration reference

## Tech stack

- **YAML**: GitHub Actions workflow definitions (primary)
- **Python 3.11**: Inline scripts calling the Anthropic API
- **JavaScript**: `actions/github-script` steps for GitHub API interaction
- **Bash**: Shell steps for git operations

## Key patterns

- **Three-layer architecture**: Consumer (triggers + secrets) → Wrapper (agent config) → Core (implementation)
- Wrappers embed agent persona, guidelines, and settings; consumers only provide `app_id` and secrets
- Core workflows are designed to be called via `uses:` from wrappers (reusable workflows)
- Authentication uses either `GITHUB_TOKEN` or GitHub App tokens (`create-github-app-token@v1`)
- Claude API calls use the `anthropic` Python SDK installed at runtime
- Agent roles (e.g., "Merge Checker") are configured per-workflow with custom display names
- Formal GitHub reviews (APPROVE/REQUEST_CHANGES) are submitted, not just comments

## Related

- `~/projects/dotfiles/docs/code-review-pipeline.md` — Pipeline architecture docs (agent roles, flow diagrams, branch protection, design decisions)

## Working with this repo

- No build step — all code lives in workflow YAML files
- Test changes by referencing a branch in a consumer repo's `uses:` directive
- Keep workflow inputs/outputs documented in README.md when adding or changing them

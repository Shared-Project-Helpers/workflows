# CLAUDE.md

Reusable GitHub Actions workflows for Claude-powered CI/CD automation (code review, automated fixes, merge readiness checks).

## Project structure

- `.github/workflows/claude-review.yml` — Automated PR code review using Claude
- `.github/workflows/claude-fix.yml` — Applies fixes based on review feedback
- `.github/workflows/claude-merge-check.yml` — "Balin" merge readiness assessment
- `README.md` — Usage docs and configuration reference

## Tech stack

- **YAML**: GitHub Actions workflow definitions (primary)
- **Python 3.11**: Inline scripts calling the Anthropic API
- **JavaScript**: `actions/github-script` steps for GitHub API interaction
- **Bash**: Shell steps for git operations

## Key patterns

- Workflows are designed to be called via `uses:` from consumer repos (reusable workflows)
- Authentication uses either `GITHUB_TOKEN` or GitHub App tokens (`create-github-app-token@v1`)
- Claude API calls use the `anthropic` Python SDK installed at runtime
- Bot personas (e.g., "Balin") are configured per-workflow with custom names/avatars
- Formal GitHub reviews (APPROVE/REQUEST_CHANGES) are submitted, not just comments

## Working with this repo

- No build step — all code lives in workflow YAML files
- Test changes by referencing a branch in a consumer repo's `uses:` directive
- Keep workflow inputs/outputs documented in README.md when adding or changing them

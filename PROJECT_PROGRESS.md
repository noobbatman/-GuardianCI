# GuardianCI Progress Log

This file records the work completed so far for GuardianCI on the target repository:
`noobbatman/document-intelligence-platform`.

## Target Repository

- GitHub: `https://github.com/noobbatman/document-intelligence-platform`
- Local path used during implementation: `C:\Users\Istiak\Desktop\2\improved`
- GuardianCI plan file: `C:\Users\Istiak\Desktop\Projects\GuardianCI\implementation_plan.md`

## Phase 1: Core Pipeline Skeleton

Implemented a GitHub Actions pipeline named `GuardianCI Phase 1`.

What was added:

- Parallel CI jobs:
  - `Tests and coverage`
  - `Docker smoke and integration tests`
  - `AI review stub`
- Deploy gate placeholder that only runs on push to `main`.
- Python setup with `uv`.
- Ruff lint and format checks.
- Pytest coverage gate with `COVERAGE_FAIL_UNDER=80`.
- Docker image build and Compose smoke test against the live API.
- Integration tests pointed at `DOCINTEL_BASE_URL`.
- Branch protection requiring the three main PR checks.

Key fixes:

- Excluded `tests/integration` from the normal unit-test job so live API tests only run in the Docker smoke job.
- Kept the coverage gate at 80 and added tests until local coverage reached about `82%`.
- Set job-level permissions instead of broad workflow permissions.

Proof:

- Passing PR was merged.
- Deliberately bad PR was blocked by CI and closed.

## Phase 2: Gemini Security Review Core

Replaced the no-op AI review stub with a real GuardianCI review script:
`scripts/guardianci_ai_review.py`.

What was added:

- Pull request diff extraction with `git diff origin/main...HEAD`.
- Security-relevant file filtering.
- High-risk path prioritization.
- Diff truncation budget.
- Local deterministic security preflight for:
  - hardcoded secrets/API keys
  - JWT `alg=none`
  - string-built SQL injection
  - `verify=False`
  - sensitive logging
- Gemini review with strict JSON output.
- Schema validation for findings.
- Inline GitHub review comments.
- `REQUEST_CHANGES` when CRITICAL findings are present.
- Quota/rate-limit fallback that lets the pipeline continue safely.

Important choices:

- Switched from Anthropic/Claude references to Gemini.
- Used `GEMINI_API_KEY`.
- Kept the job name `AI review stub` so branch protection did not break.
- Added `GUARDIANCI_GEMINI_ENABLED` to control model usage and protect API quota.

Proof:

- A deliberate security proof PR was blocked by GuardianCI.
- The bot posted inline findings and requested changes.

## Phase 3: Compliance Framework Mapping

Extended GuardianCI findings with compliance metadata.

What was added:

- New finding fields:
  - `frameworks`
  - `remediation_urgency`
- Compliance mapping for:
  - PCI-DSS
  - SOC 2
  - GDPR
- Top-level review summary with:
  - frameworks touched
  - severity counts
  - urgency counts
- Inline comments now include framework citations and remediation urgency.

Hardening follow-up:

- Consolidated duplicated hunk parsing into `parse_hunk_lines()`.
- Reduced diff context from `--unified=20` to `--unified=5`.
- Added logging when GitHub rejects inline comments and GuardianCI falls back to body-only review.
- Fixed hunk parsing for content that starts with `+++` or `---`.
- Added tests for the parser and PR review fallback behavior.

Proof:

- Phase 3 PR was merged.
- A follow-up hardening PR was merged.

## Phase 4: Auto-Fix Draft PRs

Implemented draft auto-fix PR support for CRITICAL findings.

What was added:

- `GUARDIANCI_AUTOFIX_ENABLED` flag.
- Focused Gemini auto-fix prompt for CRITICAL findings.
- Replacement of the vulnerable line range with the returned code block.
- Quick Python syntax check for changed `.py` files.
- Auto-fix branches named `guardianCI/fix-<short-sha>-<slug>`.
- Draft `[GuardianCI Auto-Fix] ...` PR creation.
- Original PR author requested as reviewer.
- `needs-human-review` label when syntax checks fail.
- Original PR comment linking to the draft fix PR.

Important hardening:

- Split `AI review stub` and `Auto-fix draft PR` into separate jobs.
- Kept the required AI review job at `contents: read`.
- Gave `contents: write` only to the separate auto-fix job.
- Added `--review-only` and `--auto-fix-only` script modes.
- Bounded the file context sent to Gemini for fixes.
- Rejected empty fenced replacements after stripping code fences.
- Skipped PR creation when Gemini produced no actual file changes.
- Sanitized commit messages and draft PR titles.

Model update:

- Changed default model to `gemma-4-31b-it`.
- Enabled live GuardianCI review and auto-fix by default unless repository variables override them.

Status:

- Phase 4 PR: `https://github.com/noobbatman/document-intelligence-platform/pull/7`
- At last check, PR #7 was still open.

## Phase 5: Security Posture Trending

Implemented metrics publishing for every GuardianCI review.

What was added:

- `scripts/guardianci_metrics.py`.
- `guardianci-review-result.json` artifact upload from the AI review job.
- Metrics job that publishes review records to the `guardianci-metrics` branch.
- `reviews/<timestamp>-<sha>.json` records.
- Aggregated `summary.json`.
- Generated `SECURITY_BADGE.md`.
- Static `index.html` dashboard.
- Badge color threshold tests.

Important hardening:

- Metrics artifact download now uses `continue-on-error: true`, so quiet/no-op PRs do not fail when no review result was uploaded.
- Metrics script remains standard-library only; the workflow notes that no dependency install is needed.

Status:

- Phase 5 branch: `guardianci/phase-5-security-metrics`
- Pushed commit: `5460558 Harden GuardianCI metrics artifact handling`

## Phase 6: False Positive Feedback Loop

Implemented `/fp` feedback support.

What was added:

- `scripts/guardianci_false_positive.py`.
- `.github/workflows/guardianci-false-positive.yml`.
- `/fp` handling for:
  - PR conversation comments (`issue_comment`)
  - inline review-thread replies (`pull_request_review_comment`)
- False-positive records stored on the `guardianci-metrics` branch in `exclusions.json`.
- Exclusion records include:
  - `file_pattern`
  - `issue_type`
  - `code_context_hash`
  - `dismissed_by`
  - `timestamp`
  - source PR/comment metadata
- GuardianCI review now loads `exclusions.json` before reviewing.
- Local deterministic findings are suppressed when they match a recorded exclusion.
- Gemini prompt receives a capped "known false positives" section.
- Monthly audit workflow reports active exclusions, using `GUARDIANCI_EXCLUSIONS_AUDIT_ISSUE` when configured.

Verification:

- Focused GuardianCI tests passed.
- Ruff check passed.
- Ruff format check passed.
- Full local coverage gate passed with `187` tests and about `82.30%` coverage.

Current branch:

- `guardianci/phase-6-false-positive-feedback`

## Phase 7: Deploy Pipeline and Notifications

Implemented the deployment pipeline branch.

What was added:

- `scripts/guardianci_deploy.py`.
- Replaced the deploy placeholder with a real `Deploy to Railway` job.
- Builds a Docker image tagged with the commit SHA.
- Pushes the image to GHCR.
- Triggers Railway via `RAILWAY_WEBHOOK_URL`.
- Optionally checks production readiness via `PRODUCTION_HEALTHCHECK_URL`.
- Records successful deploy metadata in `last-deploy.json` on the `guardianci-metrics` branch.
- Optionally triggers rollback via `RAILWAY_ROLLBACK_WEBHOOK_URL` when a deploy or health check fails.
- Sends Slack notifications via `SLACK_WEBHOOK_URL`; Slack failures are non-blocking.
- Added deploy helper tests for coverage parsing, security summary rendering, rollback payloads, and non-blocking Slack behavior.

Verification:

- Focused GuardianCI tests passed.
- Ruff check passed.
- Ruff format check passed.
- Full local coverage gate passed with `196` tests and about `82.30%` coverage.

Current branch:

- `guardianci/phase-7-deploy-notifications`

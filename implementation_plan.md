GuardianCI — Phase-by-Phase Implementation Plan
Phase 0 — Foundation (Day 1)
Goal: Everything in place before a single workflow file is written.

Steps:

Create the GuardianCI repository on GitHub (public, so GHCR is free)
Choose a target project to wire it onto — your AP Processor or Document Intelligence Platform (must already have a test suite and Docker setup)
Provision secrets in the target repo's GitHub Settings:
GEMINI_API_KEY (already exists in the repo — confirm it is present)
GITHUB_TOKEN (auto-provided by Actions, verify permissions)
RAILWAY_WEBHOOK_URL
SLACK_WEBHOOK_URL
Create the Railway project and link it to the target repo (manual deploy first, just to confirm it works)
Create a guardianci-metrics orphan branch in the target repo — this will be the storage layer for trending data, no code, just JSON files
Exit criteria: Railway returns 200 on health check from a manual deploy. All secrets exist. Metrics branch exists.

Phase 1 — Core Pipeline Skeleton (Days 2–3)
Goal: A working GitHub Actions workflow that runs tests and a Docker smoke test on every PR. No AI yet.

Steps:

Write the workflow file with 3 parallel jobs: test, docker-smoke, ai-review (stubbed as a no-op for now)
Configure the test job:
Set up Python environment
Install dependencies
Run pytest with --cov and fail if coverage < 80%
Upload coverage report as a workflow artifact
Configure the docker-smoke job:
Build the Docker image
Start the container with docker-compose up -d
Hit the health check endpoint with curl, fail if not 200
Run integration tests against the live container
Configure job dependency so deploy only triggers when all three jobs pass and the branch is main
Open a test PR to verify the pipeline runs end-to-end
Exit criteria: Green pipeline on a passing PR. Red pipeline (test failure) correctly blocks. Docker smoke test passes.

Phase 2 — AI Security Review Core (Days 4–6)
Goal: Gemini reviews every PR diff and posts structured findings as GitHub review comments. CRITICAL findings block merge.

Steps:

Write the ai-review job script (Python):
Use git diff origin/main...HEAD to extract only changed files and their diffs
Filter to relevant file types (.py, .ts, .env.*, config files)
Cap total diff size to a token budget (e.g., 8,000 tokens) — truncate with a note if exceeded
Use model: gemini-2.0-flash via the google-genai SDK (already a dependency in the project)
Write the Gemini security prompt — the fintech rubric:
Hardcoded secrets / API keys
HMAC signature bypass patterns
JWT algorithm confusion (alg: none)
SQL injection in raw SQLAlchemy queries
RBAC missing on new endpoints
PII logged in plaintext
Unvalidated Pydantic models on financial data
Define the required output schema — structured JSON array:
file, line_start, line_end, severity (CRITICAL | WARN | INFO), issue, suggested_fix
Parse Gemini's response and validate the JSON structure; on parse failure, post a fallback comment and exit cleanly (never crash the pipeline)
Use the GitHub API to post findings as PR review comments at the exact file + line position
If any finding is CRITICAL → submit a REQUEST_CHANGES review (blocks merge)
If findings are only WARN / INFO → submit an APPROVE or COMMENT review
Exit criteria: A deliberately vulnerable test PR gets CRITICAL findings posted as inline comments and is blocked from merging. A clean PR gets approved.

Phase 3 — Compliance Framework Mapping (Days 7–8)
Goal: Every finding is annotated with the specific regulatory clause it violates. No new jobs — this extends the Phase 2 prompt and output schema.

Steps:

Extend the Gemini prompt to include a compliance mapping instruction:
PCI-DSS 4.0 (payment card data, cryptography, access control clauses)
SOC 2 Type II (CC6, CC7, CC8 control categories)
GDPR Article references (Art. 25 data minimization, Art. 32 security of processing)
Extend the output schema with two new fields:
frameworks: ["PCI-DSS 6.4.1", "SOC 2 CC6.1"],
remediation_urgency: "before-merge" | "within-sprint" | "backlog"
Update the PR comment template to render the framework citations clearly — they should appear as a secondary block under each finding, not buried in prose
Add a summary comment at the top of the review that lists total findings by severity and a grouped list of all regulatory frameworks touched in this PR
Exit criteria: A PR touching auth or payment logic shows framework citations. A PR touching logging shows GDPR Art. 32. The summary comment renders correctly.

Phase 4 — Auto-Fix PRs for CRITICAL Findings (Days 9–12)
Goal: When Gemini finds a CRITICAL issue, it opens a companion draft PR with the fix already applied.

Steps:

After the main review pass, filter findings to CRITICAL only
For each CRITICAL finding, make a second Gemini call — a focused prompt that provides the specific file content and the finding, and asks only for the corrected code block (not a full review)
Apply the returned fix to a local copy of the file in the Actions runner
Commit the changes to a new branch named guardianCI/fix-<short-sha>-<slug>
Push the branch using the GITHUB_TOKEN
Open a draft PR via the GitHub API:
Title: [GuardianCI Auto-Fix] <issue summary>
Body: links back to the original PR, lists the finding, explains the change
Assigns the original PR author as reviewer
Post a comment on the original PR linking to the fix PR: "A fix has been prepared: PR #X"
Handle edge cases: if Gemini's fix introduces a syntax error (detected by a quick lint pass), mark the fix PR with a needs-human-review label instead of silently merging bad code
Exit criteria: A CRITICAL finding on a test PR results in a linked draft fix PR with a plausible correction. The fix PR is correctly linked in the original PR's comments.

Phase 5 — Security Posture Trending (Days 13–16)
Goal: Every review result is stored; a badge and summary report reflect the codebase's security trajectory over time.

Steps:

After each AI review job completes, serialize the findings to a JSON record:
{ pr_number, sha, timestamp, findings: [...], total_critical, total_warn, score }
Score formula: 100 - (critical * 20) - (warn * 5) — simple, explainable
Commit this record to the guardianci-metrics branch as reviews/<timestamp>-<sha>.json
Maintain a summary.json on the metrics branch that aggregates: rolling 30-day score, total PRs reviewed, top vulnerable modules (by finding count)
Generate a dynamic README badge by reading summary.json in the workflow and updating a SECURITY_BADGE.md file — use shields.io endpoint badge format
Build a lightweight static HTML dashboard (single file, no framework) that reads the JSON files and renders:
Score trend line (last 30 PRs)
Finding breakdown by severity
Top 5 files by finding frequency
Deploy this HTML file to GitHub Pages from the metrics branch
Exit criteria: After 3 test PRs, the dashboard shows a trend line. The README badge reflects the current score.

Phase 6 — False Positive Feedback Loop (Days 17–20)
Goal: Developers can flag false positives with /fp in a PR comment; those patterns are excluded from future reviews of the same codebase.

Steps:

Create a GitHub Actions workflow that triggers on issue_comment events (PR comments are issue comments in GitHub's API)
In that workflow, check if the comment body starts with /fp and was posted on a PR (not a standalone issue)
If yes, fetch the GuardianCI review comment it's replying to (via the GitHub API's comment thread structure)
Extract the finding's file path, pattern description, and the code context from that comment
Append a structured exclusion record to an exclusions.json file on the guardianci-metrics branch:
{ file_pattern, issue_type, code_context_hash, dismissed_by, timestamp }
Modify the AI review job to read exclusions.json at the start of each run and inject the exclusions into the Gemini prompt as a "known false positives in this codebase" section
Post a confirmation reply on the original comment: "GuardianCI: this pattern has been noted and will be excluded from future reviews."
Add a monthly audit step: a scheduled workflow that reports all active exclusions as a comment on a pinned issue, so the team can review and prune stale ones
Exit criteria: Flag a deliberate false positive with /fp. The next PR that triggers the same pattern does not get the finding. The exclusion appears in exclusions.json.

Phase 7 — Deploy Pipeline + Notifications (Days 21–22)
Goal: Merges to main (after all jobs pass) auto-deploy to Railway with Slack notification and rollback capability.

Steps:

Add the deploy job with needs: [test, docker-smoke, ai-review] and if: github.ref == 'refs/heads/main'
Tag and push the Docker image to GHCR: ghcr.io/<org>/<repo>:<sha>
Store the previous image tag before deploying (read from Railway API or a last-deploy.txt on the metrics branch)
Trigger Railway deploy via webhook; poll Railway's API until status is SUCCESS or FAILED (with a 5-minute timeout)
On Railway SUCCESS: hit the production health check endpoint to confirm the new version is live
On Railway FAILED or health check failure: re-trigger Railway deploy with the previous image tag (rollback); post a Slack alert with the rollback notice
On successful deploy, post the Slack summary:
✅ Deployed <repo> v<sha-short>
Tests: 24 passed | Coverage: 87%
Security: 0 critical | 2 warnings (PR #47)
Score: 91/100 ↑3 vs last week
Exit criteria: A merge to main triggers a deploy. A forced failure triggers rollback. Slack message arrives in both cases.

Phase 8 — Production Hardening (Days 23–25)
Goal: The pipeline is cost-controlled, observable, and documented enough for someone else to operate it.

Steps:

Cost controls: Add a pre-flight check before the Gemini API call — if the diff is > 600 lines, summarize which files changed and only send the highest-risk files (auth, payment, database) in full; send filenames only for the rest
Rate limiting: Add a check that skips the AI review if the same SHA was already reviewed (prevents re-runs from burning API quota)
Secret scanning: Add gitleaks or trufflehog as a lightweight pre-check before the Gemini review — catches obvious secrets cheaply without burning Gemini tokens
Observability: Log each Gemini API call's input token count, output token count, and cost estimate to the metrics branch
Workflow permissions audit: Lock down the GITHUB_TOKEN permissions in the workflow to the minimum required scopes (contents: write, pull-requests: write, issues: read)
Documentation: Write a SETUP.md that covers: adding secrets, wiring up a new repo, tuning the security rubric, reading the dashboard
Demo PR: Create a branch with 3-4 intentional vulnerabilities (HMAC bypass, hardcoded key, missing RBAC, SQL injection) and open a PR against the target repo — this is the live demo artifact
Exit criteria: The demo PR shows all four issue types caught, compliance citations present, a fix PR opened, and the posture dashboard updated. Cost per review is logged and < $0.01.

Milestone Summary
Phase	Days	Deliverable
0	1	Infra provisioned, secrets set, Railway live
1	2–3	Tests + Docker smoke running on every PR
2	4–6	Gemini reviewing diffs, blocking on CRITICAL
3	7–8	Compliance citations (PCI-DSS, SOC 2, GDPR) in every finding
4	9–12	Auto-fix draft PRs for CRITICAL findings
5	13–16	Trending dashboard + README badge live
6	17–20	/fp feedback loop wired up and tested
7	21–22	Auto-deploy to Railway + Slack + rollback
8	23–25	Cost controls, docs, live demo PR
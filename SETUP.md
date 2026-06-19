# GuardianCI Setup Guide

Step-by-step instructions for adding GuardianCI to an existing repository — GitHub or GitLab, any LLM provider.

---

## Prerequisites

- A GitHub or GitLab repository with CI/CD enabled
- Python 3.11+ (only needed locally to run tests; CI installs its own)
- An API key for your chosen LLM provider (see [Choosing a provider](#choosing-a-provider))
- Repository admin access (to add secrets and set required checks)

---

## Choosing a provider

GuardianCI works with any of the following out of the box:

| Provider | `GUARDIANCI_LLM_PROVIDER` | Model suggestion | Cost |
|---|---|---|---|
| Google Gemini | `gemini` | `gemini-2.0-flash` | Free tier available |
| OpenAI | `openai` | `gpt-4o-mini` | Low cost |
| Anthropic Claude | `anthropic` | `claude-haiku-4-5-20251001` | Low cost |
| Groq | `openai-compatible` | `llama-3.3-70b-versatile` | Free |
| Azure OpenAI | `openai-compatible` | `gpt-4o` | Enterprise pricing |
| Ollama (self-hosted) | `openai-compatible` | `llama3.2` | Free, offline |
| Any OpenAI-compatible | `openai-compatible` | — | Varies |

Get keys:
- **Gemini:** [Google AI Studio](https://aistudio.google.com) — free tier, no credit card
- **OpenAI:** [platform.openai.com](https://platform.openai.com)
- **Anthropic:** [console.anthropic.com](https://console.anthropic.com)
- **Groq:** [console.groq.com](https://console.groq.com) — free

---

## GitHub setup

### Step 1 — Copy the files

Copy these files from this repository into your own:

**Scripts** → place in `scripts/` at your repo root:
- `scripts/guardianci_ai_review.py`
- `scripts/guardianci_metrics.py`
- `scripts/guardianci_false_positive.py`

**Dependencies** → place in `requirements/` at your repo root (the workflow installs from these):
- `requirements/base.txt`
- `requirements/dev.txt` _(only needed if you run tests locally)_

**Workflows** → place in `.github/workflows/`:
- `.github/workflows/guardianci.yml`
- `.github/workflows/guardianci-false-positive.yml`

Your repo structure should look like:

```
your-repo/
├── scripts/
│   ├── guardianci_ai_review.py
│   ├── guardianci_metrics.py
│   ├── guardianci_false_positive.py
│   └── ... (your existing scripts)
├── requirements/
│   ├── base.txt              ← required: CI installs from this
│   └── dev.txt               ← optional: for local development
├── .github/
│   └── workflows/
│       ├── guardianci.yml
│       ├── guardianci-false-positive.yml
│       └── ... (your existing workflows)
└── ... (rest of your repo)
```

> **Important:** `requirements/base.txt` must be present. The workflow runs
> `uv pip install --system -r requirements/base.txt` — it will fail if the file
> is missing.

Commit and push to a branch (not main yet — you'll open a PR to test it).

---

### Step 2 — Create the metrics branch

GuardianCI stores trending data in a dedicated orphan branch. Run this once:

```bash
git checkout --orphan guardianci-metrics
git rm -rf .
git commit --allow-empty -m "chore: init guardianci-metrics branch"
git push origin guardianci-metrics
git checkout main
```

This branch will accumulate JSON files over time — one per reviewed PR, plus a `summary.json` with rolling 30-day counts and an `index.html` dashboard.

---

### Step 3 — Add the LLM API key

1. Go to your repo on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Name: `GUARDIANCI_LLM_API_KEY`
5. Value: your API key from the provider you chose above
6. Click **Add secret**

---

### Step 4 — Set repository variables

These variables configure which provider and model to use. Skip any you want to leave at the default.

1. On the same **Secrets and variables → Actions** page, click the **Variables** tab
2. Add any of the following:

| Variable | Default | Description |
|---|---|---|
| `GUARDIANCI_LLM_PROVIDER` | `gemini` | `gemini` / `openai` / `anthropic` / `openai-compatible` |
| `GUARDIANCI_LLM_MODEL` | `gemini-2.0-flash` | Model name to use. Must match the provider's API. |
| `GUARDIANCI_LLM_BASE_URL` | _(provider default)_ | Required for `openai-compatible` providers (Groq, Azure, Ollama, etc.) |
| `GUARDIANCI_AI_ENABLED` | `true` | Set `false` to disable LLM calls (pattern scan only — no API cost) |
| `GUARDIANCI_AUTOFIX_ENABLED` | `true` | Set `false` to disable auto-fix draft PRs |

**Provider-specific base URLs** (set as `GUARDIANCI_LLM_BASE_URL`):

| Provider | Base URL |
|---|---|
| Groq | `https://api.groq.com/openai/v1` |
| Azure OpenAI | `https://<resource>.openai.azure.com/openai/deployments/<deployment>` |
| Ollama (local) | `http://localhost:11434/v1` |
| Mistral | `https://api.mistral.ai/v1` |

---

### Step 5 — (Optional) Configure metrics webhook

If you want GuardianCI to push every review result to Datadog, Grafana, Splunk, or a custom API:

1. Click **New repository secret**
2. Name: `GUARDIANCI_METRICS_WEBHOOK_URL`
3. Value: the full URL that should receive POST requests

Optionally, add `GUARDIANCI_METRICS_WEBHOOK_SECRET` (any string) — GuardianCI will sign every payload with HMAC-SHA256 in the `X-GuardianCI-Signature-256` header so your receiver can verify authenticity.

Webhook payload shape (stable across versions):

```json
{
  "event": "guardianci.review.completed",
  "schema_version": 1,
  "review": { "...per-PR result fields..." },
  "summary": { "...rolling 30-day aggregates..." }
}
```

Webhook failures are logged but never fail the CI job — git-branch storage continues to work independently.

---

### Step 6 — Make the AI review a required check

This is what gives GuardianCI its teeth — CRITICAL findings block the merge button.

1. Go to **Settings** → **Branches**
2. Click **Edit** next to your main branch rule (or create one if you don't have one)
3. Under **Require status checks to pass before merging**, search for and add:
   - `AI security review`
4. Save

Now any PR with CRITICAL findings cannot be merged until the issues are resolved or the finding is dismissed with `/fp`.

---

### Step 7 — Open a test PR

Create a branch, make any change, and open a PR against main. Within a few minutes you should see:

- A **GuardianCI** check appear in the PR checks list
- A review comment from `github-actions[bot]` with the summary (CRITICAL/WARN/INFO counts)
- Inline comments at any flagged lines
- (If findings exist) a draft auto-fix PR linked in the comments

If no issues are found in your test change, the `AI security review` check passes and the summary says `CRITICAL: 0 WARN: 0`.

---

### Step 8 — Test with an intentional finding (optional)

To confirm the full flow works end-to-end, temporarily add this to any Python file in your PR:

```python
# TEST — remove before merge
SECRET_KEY = "hardcoded-secret-do-not-ship"
```

GuardianCI should flag it as CRITICAL with a PCI-DSS citation and block the merge. Then delete the line, push again — the check should go green.

---

## GitLab setup

Copy `.gitlab/guardianci.gitlab-ci.yml` from this repo into your GitLab repository (as `.gitlab-ci.yml` or include it from your existing pipeline).

### Required CI/CD variables

Set these under **Settings → CI/CD → Variables**:

| Variable | Description |
|---|---|
| `GUARDIANCI_LLM_API_KEY` | API key for your chosen LLM provider |
| `GITLAB_TOKEN` | Project or Group access token with `api` scope — required to post MR notes (CI_JOB_TOKEN lacks this permission) |

### Optional CI/CD variables

| Variable | Default | Description |
|---|---|---|
| `GUARDIANCI_LLM_PROVIDER` | `gemini` | Same provider options as GitHub |
| `GUARDIANCI_LLM_MODEL` | `gemini-2.0-flash` | Model name |
| `GUARDIANCI_LLM_BASE_URL` | _(provider default)_ | Custom endpoint for `openai-compatible` providers |
| `GUARDIANCI_AI_ENABLED` | `true` | Set `false` for pattern scan only |
| `GUARDIANCI_VCS_PLATFORM` | `gitlab` | Pre-set in the template; no need to change |

The template runs only on `merge_request_event` so it does not trigger on ordinary branch pushes.

---

## How reviews work

### Severity levels

| Level | Meaning | Blocks merge? |
|---|---|---|
| CRITICAL | Exploitable now: credentials, injection, auth bypass | Yes |
| WARN | Elevated risk: weak config, disabled security controls | No |
| INFO | Best-practice notes | No |

### Remediation urgency

| Tag | Meaning |
|---|---|
| `before-merge` | Fix it before this PR lands |
| `within-sprint` | Fix within the current sprint |
| `backlog` | Track it, not urgent |

### Large diffs

PRs adding more than 600 lines send only high-risk file paths (auth, payments, secrets, security config) to the LLM. Local pattern matching still runs on all files. The review summary notes which files were skipped so nothing is silently dropped.

### SHA deduplication

If the same commit SHA is pushed again (force-push to same commit), GuardianCI detects the cached result and skips the LLM call. This avoids duplicate billing on re-runs.

---

## False positive workflow

If GuardianCI flags something that isn't a real issue:

1. Any repo OWNER, MEMBER, or COLLABORATOR can comment on the inline finding: `/fp <reason>`
   - Example: `/fp Test fixture — not real credentials, never deployed`
2. The `guardianci-false-positive` workflow records the dismissal in the metrics branch
3. The finding is excluded from future LLM prompts for the same pattern
4. A monthly audit issue lists all active exclusions for review

---

## Metrics

After each reviewed PR, GuardianCI writes files to the `guardianci-metrics` branch:

```
reviews/
  {timestamp}-{sha12}.json   # per-PR review result
summary.json                  # rolling 30-day aggregates
SECURITY_BADGE.md             # badge with current score
index.html                    # self-contained HTML dashboard
exclusions/
  {pattern-hash}.json         # false-positive exclusions
```

You can query this branch directly, link the badge in your main README, or set `GUARDIANCI_METRICS_WEBHOOK_URL` to stream results into any observability platform.

---

## Local development

To run GuardianCI's test suite locally or contribute to the project:

```bash
# Install dev dependencies (includes ruff + pytest)
uv pip install --system -r requirements/dev.txt

# Run tests
pytest

# Run the linter
ruff check scripts/ tests/

# Auto-fix lint issues and reformat
ruff check --fix scripts/ tests/ && ruff format scripts/ tests/
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full contribution guide and [SECURITY.md](SECURITY.md) for the vulnerability reporting policy and known detection limits.

---

## Troubleshooting

**The `AI security review` check never appears**
- Check that the workflow file is in the branch being reviewed (not just main)
- Check that GitHub Actions is enabled for the repo (Settings → Actions → General)

**LLM API call failed: 503 / UNAVAILABLE**
- Transient provider error. Re-run the failed job from the Actions tab.

**Could not parse LLM response as JSON**
- The provider returned a malformed or empty response (quota or rate-limit). Re-run the `auto-fix` job or reduce the model to a smaller/faster variant.

**False-positive workflow does nothing when I comment `/fp`**
- The commenter must have OWNER, MEMBER, or COLLABORATOR association on the repo
- The comment must be on a PR (issue comments on non-PR issues are ignored)
- Check the Actions tab for the `GuardianCI False Positive Feedback` run and look at its logs

**I want pattern-scan only (no LLM API cost)**
- Set `GUARDIANCI_AI_ENABLED` to `false`
- GuardianCI still runs all local pattern checks and reports findings, but makes no API calls

**GitLab: inline comments are posting as a bot but without MR approval ability**
- `CI_JOB_TOKEN` can only post notes, not approve MRs. This is expected — use a `GITLAB_TOKEN` with `api` scope if you need approval integration.

**Webhook never fires**
- Check that `GUARDIANCI_METRICS_WEBHOOK_URL` is set on the repository secret (not a plain variable — secrets are redacted in logs so you won't see the value echoed)
- Webhook errors are printed to the metrics job log but do not fail the job; check the metrics step output in the Actions run

**Switching from Gemini to another provider**
- Set `GUARDIANCI_LLM_PROVIDER` and `GUARDIANCI_LLM_API_KEY` (+ `GUARDIANCI_LLM_BASE_URL` for `openai-compatible`)
- Old `GEMINI_API_KEY` and `GUARDIANCI_GEMINI_ENABLED` variables still work as fallbacks — no need to delete them

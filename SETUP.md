# GuardianCI Setup Guide

Step-by-step instructions for adding GuardianCI to an existing GitHub repository.

---

## Prerequisites

- A GitHub repository with GitHub Actions enabled
- Python 3.11+ (only needed locally to test scripts; CI installs its own)
- A Gemini API key — get one free at [Google AI Studio](https://aistudio.google.com)
- Repository admin access (to add secrets and set required checks)

---

## Step 1 — Copy the files

Copy these files from this repository into your own:

**Scripts** → place in `scripts/` at your repo root:
- `scripts/guardianci_ai_review.py`
- `scripts/guardianci_metrics.py`
- `scripts/guardianci_false_positive.py`

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
├── .github/
│   └── workflows/
│       ├── guardianci.yml
│       ├── guardianci-false-positive.yml
│       └── ... (your existing workflows)
└── ... (rest of your repo)
```

Commit and push to a branch (not main yet — you'll open a PR to test it).

---

## Step 2 — Create the metrics branch

GuardianCI stores trending data in a dedicated orphan branch. Run this once:

```bash
git checkout --orphan guardianci-metrics
git rm -rf .
git commit --allow-empty -m "chore: init guardianci-metrics branch"
git push origin guardianci-metrics
git checkout main
```

This branch will accumulate JSON files over time — one per reviewed PR, plus a `summary.json` with rolling counts.

---

## Step 3 — Add the Gemini API key secret

1. Go to your repo on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Name: `GEMINI_API_KEY`
5. Value: your Gemini API key from Google AI Studio
6. Click **Add secret**

---

## Step 4 — (Optional) Set repository variables

These variables let you tune GuardianCI behavior without editing the workflow file.

1. On the same **Secrets and variables → Actions** page, click the **Variables** tab
2. Add any of the following:

| Variable | Recommended value | Effect |
|---|---|---|
| `GEMINI_MODEL` | `gemini-2.0-flash` | Which Gemini model to use. `gemini-2.0-flash` is fast and cheap. Use `gemini-1.5-pro` for more thorough analysis on sensitive repos. |
| `GUARDIANCI_GEMINI_ENABLED` | `true` | Set to `false` to disable Gemini calls entirely (pattern-scan only, no API cost). Useful for testing the workflow itself. |
| `GUARDIANCI_AUTOFIX_ENABLED` | `true` | Set to `false` to disable auto-fix draft PRs. |

If you leave these unset, the workflow defaults are used (`gemini-2.0-flash`, enabled for both).

---

## Step 5 — Make the AI review a required check

This is what gives GuardianCI its teeth — CRITICAL findings block the merge button.

1. Go to **Settings** → **Branches**
2. Click **Edit** next to your main branch rule (or create one if you don't have one)
3. Under **Require status checks to pass before merging**, search for and add:
   - `AI security review`
4. Save

Now any PR with CRITICAL findings cannot be merged until the issues are resolved or the finding is dismissed with `/fp`.

---

## Step 6 — Open a test PR

Create a branch, make any change, and open a PR against main. Within a few minutes you should see:

- A **GuardianCI** check appear in the PR checks list
- A review comment from `github-actions[bot]` with the summary (CRITICAL/WARN/INFO counts)
- Inline comments at any flagged lines
- (If findings exist) a draft auto-fix PR linked in the comments

If no issues are found in your test change, the `AI security review` check passes and the summary says `CRITICAL: 0 WARN: 0`.

---

## Step 7 — Test with an intentional finding (optional)

To confirm the full flow works end-to-end, temporarily add this to any Python file in your PR:

```python
# TEST — remove before merge
SECRET_KEY = "hardcoded-secret-do-not-ship"
```

GuardianCI should flag it as CRITICAL with a PCI-DSS citation and block the merge. Then delete the line, push again — the check should go green.

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

PRs adding more than 600 lines send only high-risk file paths (auth, payments, secrets, security config) to Gemini. Local pattern matching still runs on all files. The review summary notes which files were skipped so nothing is silently dropped.

---

## False positive workflow

If GuardianCI flags something that isn't a real issue:

1. Any repo OWNER, MEMBER, or COLLABORATOR can comment on the inline finding: `/fp <reason>`
   - Example: `/fp Test fixture — not real credentials, never deployed`
2. The `guardianci-false-positive` workflow records the dismissal in the metrics branch
3. The finding is excluded from future Gemini prompts for the same pattern
4. A monthly audit issue lists all active exclusions for review

---

## Metrics

After each reviewed PR, GuardianCI writes a JSON file to the `guardianci-metrics` branch:

```
reviews/
  {repo}-{pr}-{sha12}.json   # per-PR review result
summary.json                  # rolling 30-day counts
exclusions/
  {pattern-hash}.json         # false-positive exclusions
```

You can query this branch directly or build a dashboard on top of it. No external database or service is required.

---

## Troubleshooting

**The `AI security review` check never appears**
- Check that the workflow file is on the branch being reviewed (not just main)
- Check that GitHub Actions is enabled for the repo (Settings → Actions → General)

**Gemini review failed: 503 UNAVAILABLE**
- This is a transient Gemini API error. Re-run the failed job from the Actions tab.

**GuardianCI could not prepare an auto-fix PR: Could not parse Gemini response as JSON**
- Gemini returned an empty response (usually a quota or rate-limit issue). The review itself still posted correctly. Re-run the `auto-fix` job or disable auto-fix if quota is limited.

**False-positive workflow does nothing when I comment `/fp`**
- The commenter must have OWNER, MEMBER, or COLLABORATOR association on the repo
- The comment must be on a PR (issue comments on non-PR issues are ignored)
- Check the Actions tab for the `GuardianCI False Positive Feedback` run and look at its logs

**I want pattern-scan only (no Gemini API cost)**
- Set the `GUARDIANCI_GEMINI_ENABLED` repository variable to `false`
- GuardianCI will still run all local pattern checks and report findings, but will not call the Gemini API

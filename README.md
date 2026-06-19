# GuardianCI

AI-powered security compliance review for every pull request. GuardianCI runs Gemini against your PR diff, posts inline findings with compliance citations, blocks merges on critical issues, and tracks your security posture over time.

---

## What it does

When a PR is opened or updated, GuardianCI:

1. **Pre-scans for secrets** with gitleaks before Gemini ever sees the diff
2. **Sends the diff to Gemini** with a security-focused prompt covering hardcoded credentials, injection flaws, auth bypasses, and insecure transport
3. **Posts inline PR comments** at the exact lines where findings occur, each with a suggested fix and compliance citation
4. **Blocks the merge** if any CRITICAL findings are present — the job exits 1 and GitHub's required-check gate holds
5. **Opens a draft auto-fix PR** with Gemini-generated remediation patches for CRITICAL issues
6. **Persists metrics** to a dedicated `guardianci-metrics` branch — JSON files tracking finding counts, severity breakdown, and framework coverage over time
7. **Handles false positives** via `/fp <reason>` comments — authorized users can dismiss findings, which are recorded and audited monthly

---

## What it catches

GuardianCI checks every changed file against these categories, with compliance citations attached to each finding:

| Category | Severity | Frameworks |
|---|---|---|
| Hardcoded secrets / API keys | CRITICAL | PCI-DSS 6.4.3, SOC 2 CC6.1, GDPR Art. 32 |
| SQL / command injection | CRITICAL | PCI-DSS 6.2.4, SOC 2 CC6.1, GDPR Art. 32 |
| JWT algorithm confusion (alg=none) | CRITICAL | SOC 2 CC6.1, GDPR Art. 32 |
| Insecure deserialization | CRITICAL | PCI-DSS 6.2.4, SOC 2 CC6.1 |
| Disabled TLS verification | WARN | SOC 2 CC6.7, GDPR Art. 32 |
| Overly permissive CORS | WARN | SOC 2 CC6.6 |
| Debug/verbose logging in prod paths | WARN | SOC 2 CC7.2 |
| Missing input validation | WARN | PCI-DSS 6.2.4, GDPR Art. 5(1)(f) |

Findings are tagged `before-merge`, `within-sprint`, or `backlog` by remediation urgency.

---

## Demo

A PR with intentional vulnerabilities produces output like this:

```
GuardianCI Gemini compliance review completed.

Frameworks touched: GDPR Art. 32, PCI-DSS 6.2.4, PCI-DSS 6.4.3, SOC 2 CC6.1, SOC 2 CC6.7

CRITICAL: 8  WARN: 3  INFO: 0
before-merge: 8  within-sprint: 3  backlog: 0

CRITICAL findings block this PR until fixed.
```

Each finding posts as an inline review comment at the exact vulnerable line:

```
GuardianCI CRITICAL

SQL injection vulnerability via f-string interpolation of the 'document_id'
parameter into a raw SQL query.

Suggested fix: Use parameterized queries or an ORM to safely handle user input.
  Example: db.execute("SELECT * FROM documents WHERE id = ?", (document_id,))

Frameworks: PCI-DSS 6.2.4, SOC 2 CC6.1, GDPR Art. 32
Remediation urgency: before-merge
```

---

## Architecture

```
pull_request event
       │
       ▼
┌─────────────────────────────────────┐
│  ai-review job                      │
│  1. gitleaks secret pre-scan        │
│  2. Gemini diff analysis            │
│  3. Post inline PR comments         │
│  4. Exit 1 if CRITICAL found        │
└────────────┬────────────────────────┘
             │ artifact: review-result.json
     ┌───────┴───────┐
     ▼               ▼
┌─────────┐   ┌──────────────┐
│ metrics │   │  auto-fix    │
│  job    │   │  job         │
│         │   │              │
│ Persist │   │ Draft PR     │
│ JSON to │   │ with Gemini  │
│ metrics │   │ patches for  │
│ branch  │   │ CRITICALs    │
└─────────┘   └──────────────┘

issue_comment or pull_request_review_comment event (/fp <reason>)
       │
       ▼
┌─────────────────────────────────────┐
│  guardianci-false-positive workflow  │
│  Record exclusion to metrics branch  │
│  Monthly audit issue posted          │
└─────────────────────────────────────┘
```

---

## Large-diff cost controls

PRs adding more than 600 lines send only high-risk file paths to Gemini (auth, payments, security, secrets, config). Local pattern matching still runs on every file. Skipped files are noted in the review summary so nothing is silently ignored.

## SHA deduplication

Before calling Gemini, GuardianCI checks whether this exact commit SHA has already been reviewed (stored in the metrics branch). Identical re-pushes skip the Gemini call and reuse the cached result.

---

## Quick start

**Prerequisites:** Python 3.11+, a Gemini API key, a GitHub repo with Actions enabled.

### 1. Copy the files

```
your-repo/
├── scripts/
│   ├── guardianci_ai_review.py
│   ├── guardianci_metrics.py
│   └── guardianci_false_positive.py
└── .github/
    └── workflows/
        ├── guardianci.yml
        └── guardianci-false-positive.yml
```

Copy `scripts/guardianci_*.py` from this repo into your `scripts/` folder.  
Copy `.github/workflows/guardianci.yml` and `.github/workflows/guardianci-false-positive.yml` into your `.github/workflows/`.

### 2. Create the metrics branch

```bash
git checkout --orphan guardianci-metrics
git rm -rf .
git commit --allow-empty -m "chore: init guardianci-metrics branch"
git push origin guardianci-metrics
git checkout main
```

### 3. Add secrets and variables

In your repo → **Settings → Secrets and variables → Actions**:

| Name | Type | Value |
|---|---|---|
| `GEMINI_API_KEY` | Secret | Your Gemini API key from Google AI Studio |

Optional variables (can be left unset to use defaults):

| Name | Type | Default | Description |
|---|---|---|---|
| `GEMINI_MODEL` | Variable | `gemini-2.0-flash` | Gemini model to use |
| `GUARDIANCI_GEMINI_ENABLED` | Variable | `true` | Set to `false` to run pattern-scan only |
| `GUARDIANCI_AUTOFIX_ENABLED` | Variable | `true` | Set to `false` to disable auto-fix PRs |

### 4. Make `ai-review` a required check

In your repo → **Settings → Branches → main** → require status check `AI security review`.

Open a PR — GuardianCI will review it automatically.

Full adoption guide: [SETUP.md](SETUP.md)

---

## Configuration

All configuration is through environment variables set in the workflow or as GitHub Actions secrets/variables. No config file is needed.

| Variable | Default | Description |
|---|---|---|
| `GEMINI_API_KEY` | — | Required. Gemini API key. |
| `GEMINI_MODEL` | `gemini-2.0-flash` | Model to use for analysis. |
| `GUARDIANCI_GEMINI_ENABLED` | `true` | Disable to run local pattern scan only (no API cost). |
| `GUARDIANCI_AUTOFIX_ENABLED` | `true` | Whether to open auto-fix draft PRs. |
| `GUARDIANCI_REVIEW_RESULT_PATH` | `guardianci-review-result.json` | Path for the review result artifact. |
| `GUARDIANCI_METRICS_BRANCH` | `guardianci-metrics` | Branch where metrics JSON files are stored. |

---

## False positive handling

Any authorized repo member (OWNER, MEMBER, or COLLABORATOR) can dismiss a GuardianCI finding by commenting `/fp <reason>` on the inline comment. The dismissal is recorded in the metrics branch and excluded from future reviews of the same pattern.

A monthly audit issue is posted automatically listing all active exclusions so nothing goes unreviewed indefinitely.

---

## Requirements

- Python 3.11+
- `google-genai` and `requests` Python packages (installed by the workflow)
- A Gemini API key (free tier available at [Google AI Studio](https://aistudio.google.com))
- GitHub Actions enabled on your repository

---

## License

MIT

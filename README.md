# GuardianCI

AI-powered security compliance review for every pull request. GuardianCI reviews your PR diff with any major LLM, posts inline findings with compliance citations, blocks merges on critical issues, and tracks your security posture over time — on GitHub and GitLab.

---

## What it does

When a PR or merge request is opened or updated, GuardianCI:

1. **Pre-scans for secrets** with gitleaks before any LLM sees the diff
2. **Sends the diff to your LLM** (Gemini, OpenAI, Anthropic, Groq, Ollama, Azure, or any OpenAI-compatible endpoint) with a security-focused prompt covering hardcoded credentials, injection flaws, auth bypasses, and insecure transport
3. **Posts inline comments** at the exact lines where findings occur, each with a suggested fix and compliance citation
4. **Blocks the merge** if any CRITICAL findings are present — the job exits 1 and your branch protection gate holds
5. **Opens a draft auto-fix PR** with AI-generated remediation patches for CRITICAL issues
6. **Persists metrics** to a dedicated `guardianci-metrics` branch — JSON files tracking finding counts, severity breakdown, and framework coverage over time
7. **Pushes metrics to any webhook** (Datadog, Grafana, Splunk, or a custom API) after every review — signed with HMAC-SHA256 so receivers can verify authenticity
8. **Handles false positives** via `/fp <reason>` comments — authorized users can dismiss findings, which are recorded and audited monthly

---

## Supported platforms

| VCS | Trigger | Inline comments |
|---|---|---|
| **GitHub** | `pull_request` event | Inline review comments via GitHub Review API |
| **GitLab** | `merge_request_event` | Inline discussions via GitLab MR Discussions API |

---

## Supported LLM providers

| Provider | `GUARDIANCI_LLM_PROVIDER` | Notes |
|---|---|---|
| Google Gemini | `gemini` | Default. Free tier at Google AI Studio. |
| OpenAI | `openai` | `gpt-4o-mini` is a cost-effective starting point. |
| Anthropic Claude | `anthropic` | `claude-haiku-4-5` is fast and cheap. |
| Groq | `openai-compatible` | Free inference for Llama and Mixtral models. |
| Azure OpenAI | `openai-compatible` | Set `GUARDIANCI_LLM_BASE_URL` to your deployment endpoint. |
| Ollama (self-hosted) | `openai-compatible` | Zero cost, fully offline — ideal for air-gapped environments. |
| Mistral, Together, any OpenAI-compatible | `openai-compatible` | Point `GUARDIANCI_LLM_BASE_URL` at the provider base URL. |

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
| Debug / verbose logging in prod paths | WARN | SOC 2 CC7.2 |
| Missing input validation | WARN | PCI-DSS 6.2.4, GDPR Art. 5(1)(f) |

Findings are tagged `before-merge`, `within-sprint`, or `backlog` by remediation urgency.

---

## Demo

A PR with intentional vulnerabilities produces output like this:

```
GuardianCI AI compliance review completed.

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
pull_request / merge_request event
             │
             ▼
┌────────────────────────────────────┐
│  ai-review job                     │
│  1. gitleaks secret pre-scan       │
│  2. LLM diff analysis              │
│     (Gemini / OpenAI / Anthropic   │
│      / Groq / Ollama / Azure / …)  │
│  3. Post inline PR/MR comments     │
│  4. Exit 1 if CRITICAL found       │
└────────────┬───────────────────────┘
             │ artifact: review-result.json
     ┌───────┴────────┐
     ▼                ▼
┌──────────┐   ┌──────────────┐
│ metrics  │   │  auto-fix    │
│ job      │   │  job         │
│          │   │              │
│ Git branch   │ Draft PR     │
│ storage  │   │ with AI      │
│    +     │   │ patches for  │
│ Webhook  │   │ CRITICALs    │
│ push     │   │              │
└──────────┘   └──────────────┘

issue_comment / note event  (/fp <reason>)
             │
             ▼
┌────────────────────────────────────┐
│  false-positive workflow           │
│  Record exclusion to metrics branch│
│  Monthly audit issue posted        │
└────────────────────────────────────┘
```

---

## Large-diff cost controls

PRs adding more than 600 lines send only high-risk file paths to the LLM (auth, payments, security, secrets, config). Local pattern matching still runs on every file. Skipped files are noted in the review summary so nothing is silently ignored.

## SHA deduplication

Before calling the LLM, GuardianCI checks whether this exact commit SHA has already been reviewed (stored in the metrics branch). Identical re-pushes skip the LLM call and reuse the cached result.

---

## Quick start

**GitHub:** See [SETUP.md](SETUP.md) for the full step-by-step guide.

**GitLab:** See `.gitlab/guardianci.gitlab-ci.yml` in this repo — copy it to your repo as `.gitlab-ci.yml` and set the CI/CD variables.

**Prerequisites:** Python 3.11+, an API key for your chosen LLM, and a GitHub or GitLab repo with CI/CD enabled.

### 1. Copy the files

```
your-repo/
├── scripts/
│   ├── guardianci_ai_review.py
│   ├── guardianci_metrics.py
│   └── guardianci_false_positive.py
├── requirements/
│   └── base.txt               ← required: CI installs from this
└── .github/
    └── workflows/
        ├── guardianci.yml
        └── guardianci-false-positive.yml
```

Copy `scripts/guardianci_*.py`, the `requirements/` directory, and the workflow files into your repo at the same paths.  
For GitLab, copy `.gitlab/guardianci.gitlab-ci.yml` to your repo instead (the `requirements/` directory is still needed).

### 2. Create the metrics branch (GitHub only)

```bash
git checkout --orphan guardianci-metrics
git rm -rf .
git commit --allow-empty -m "chore: init guardianci-metrics branch"
git push origin guardianci-metrics
git checkout main
```

### 3. Add secrets and variables

In your repo → **Settings → Secrets and variables → Actions**:

| Name | Type | Description |
|---|---|---|
| `GUARDIANCI_LLM_API_KEY` | Secret | API key for your chosen LLM provider |

Optional (leave unset to use the listed defaults):

| Name | Type | Default | Description |
|---|---|---|---|
| `GUARDIANCI_LLM_PROVIDER` | Variable | `gemini` | `gemini` / `openai` / `anthropic` / `openai-compatible` |
| `GUARDIANCI_LLM_MODEL` | Variable | `gemini-2.0-flash` | Model name, e.g. `gpt-4o-mini`, `claude-haiku-4-5` |
| `GUARDIANCI_LLM_BASE_URL` | Variable | _(provider default)_ | Custom base URL for OpenAI-compatible endpoints |
| `GUARDIANCI_AI_ENABLED` | Variable | `true` | Set `false` to run pattern-scan only (no LLM cost) |
| `GUARDIANCI_AUTOFIX_ENABLED` | Variable | `true` | Set `false` to disable auto-fix draft PRs |
| `GUARDIANCI_METRICS_WEBHOOK_URL` | Secret | _(none)_ | POST metrics JSON to Datadog / Grafana / Splunk / custom API |
| `GUARDIANCI_METRICS_WEBHOOK_SECRET` | Secret | _(none)_ | Sign webhook payloads with HMAC-SHA256 |

Backward-compatible aliases still work: `GEMINI_API_KEY`, `GEMINI_MODEL`, `GUARDIANCI_GEMINI_ENABLED`.

### 4. Make `ai-review` a required check

In your repo → **Settings → Branches → main** → require status check `AI security review`.

Open a PR — GuardianCI will review it automatically.

Full adoption guide: [SETUP.md](SETUP.md)

---

## Configuration reference

All configuration is through environment variables set in the workflow or as GitHub Actions secrets/variables. No config file is needed.

### LLM provider

| Variable | Default | Description |
|---|---|---|
| `GUARDIANCI_LLM_PROVIDER` | `gemini` | LLM backend: `gemini`, `openai`, `anthropic`, `openai-compatible` |
| `GUARDIANCI_LLM_MODEL` | `gemini-2.0-flash` | Model name passed to the provider |
| `GUARDIANCI_LLM_API_KEY` | — | API key. Falls back to `GEMINI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` |
| `GUARDIANCI_LLM_BASE_URL` | _(provider default)_ | Override the API base URL (required for OpenAI-compatible providers) |
| `GUARDIANCI_AI_ENABLED` | `true` | Set `false` to skip LLM calls and run local pattern scan only |

### Behavior

| Variable | Default | Description |
|---|---|---|
| `GUARDIANCI_AUTOFIX_ENABLED` | `true` | Open a draft auto-fix PR for CRITICAL findings |
| `GUARDIANCI_REVIEW_RESULT_PATH` | `guardianci-review-result.json` | Path for the review result artifact |
| `GUARDIANCI_METRICS_BRANCH` | `guardianci-metrics` | Branch where metrics JSON files are stored |

### Metrics webhook

| Variable | Default | Description |
|---|---|---|
| `GUARDIANCI_METRICS_WEBHOOK_URL` | _(none)_ | POST the review result to any HTTP endpoint after each review |
| `GUARDIANCI_METRICS_WEBHOOK_SECRET` | _(none)_ | Sign the payload with HMAC-SHA256 (`X-GuardianCI-Signature-256` header) |

### GitLab-specific

| Variable | Description |
|---|---|
| `GUARDIANCI_VCS_PLATFORM` | Set to `gitlab` to force GitLab mode (auto-detected from `CI_MERGE_REQUEST_IID`) |
| `GITLAB_TOKEN` | Project access token with `api` scope (CI_JOB_TOKEN lacks MR note permissions) |

---

## False positive handling

Any authorized repo member (OWNER, MEMBER, or COLLABORATOR) can dismiss a GuardianCI finding by commenting `/fp <reason>` on the inline comment. The dismissal is recorded in the metrics branch and excluded from future reviews of the same pattern.

A monthly audit issue is posted automatically listing all active exclusions so nothing goes unreviewed indefinitely.

---

## Metrics webhook payload

When `GUARDIANCI_METRICS_WEBHOOK_URL` is set, GuardianCI POSTs this JSON after every review:

```json
{
  "event": "guardianci.review.completed",
  "schema_version": 1,
  "review": { "...per-PR result..." },
  "summary": { "...rolling 30-day aggregates..." }
}
```

Verify authenticity on the receiver side:

```python
import hashlib, hmac
expected = hmac.new(secret.encode(), request.body, hashlib.sha256).hexdigest()
assert request.headers["X-GuardianCI-Signature-256"] == f"sha256={expected}"
```

---

## Requirements

- Python 3.11+
- An API key for your chosen LLM provider
- GitHub Actions or GitLab CI/CD enabled on your repository
- The `requirements/base.txt` file from this repo (the workflow installs deps from it)

Runtime dependencies are declared in `pyproject.toml` and pinned in `requirements/base.txt`. No additional install step is needed — the workflow handles it.

---

## Project files

| File | Purpose |
|---|---|
| `pyproject.toml` | Project metadata, dependency spec, ruff + pytest config |
| `requirements/base.txt` | Version-constrained runtime deps (installed by CI) |
| `requirements/dev.txt` | Dev deps: adds ruff + pytest |
| `LICENSE` | MIT |
| `SECURITY.md` | Vulnerability reporting policy and known detection limits |
| `CONTRIBUTING.md` | Dev setup, test/lint commands, PR guidelines |
| `CHANGELOG.md` | Version history |
| `.github/CODEOWNERS` | Code ownership for review assignment |

---

## License

MIT — see [LICENSE](LICENSE)

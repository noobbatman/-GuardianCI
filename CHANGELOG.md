# Changelog

All notable changes to GuardianCI are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions follow [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

### Planned — v0.2.0
- Split `guardianci_ai_review.py` into a proper package (`guardianci/vcs/`, `guardianci/llm/`, `guardianci/scan/`) to address the single-2200-line-module concern
- AST-based local scanning (Semgrep rules) to replace regex heuristics for Python and JavaScript
- Auto-updating LLM pricing table fetched from provider APIs (removes hardcoded drift risk)
- `uv.lock` fully pinned lockfile with hash verification

---

## [0.1.0] — 2026-06-19

### Added
- **Multi-provider LLM support**: `GUARDIANCI_LLM_PROVIDER` dispatches to Gemini, OpenAI, Anthropic, or any OpenAI-compatible endpoint (Groq, Mistral, Azure OpenAI, Ollama)
- **GitLab support**: `_gitlab_context()`, `_gitlab_post_review()` — inline discussions via MR Discussions API; auto-detected from `CI_MERGE_REQUEST_IID`
- **GitLab CI template**: `.gitlab/guardianci.gitlab-ci.yml` ready-to-use pipeline
- **Metrics webhook push**: `push_webhook()` in `guardianci_metrics.py` — POST every review result to any HTTP endpoint (Datadog, Grafana, Splunk, custom API), with HMAC-SHA256 signing and exponential-backoff retry; non-fatal on failure
- **Extended LLM pricing table**: `estimate_llm_cost()` covers Gemini, OpenAI (GPT-4o family), Anthropic (Haiku/Sonnet/Opus), Groq / free-tier models
- **`_call_openai_compatible()`**: REST-only implementation (no SDK) for OpenAI and OpenAI-compatible providers
- **`_call_anthropic()`**: REST-only implementation for Anthropic Claude (Messages API)
- **Auto-fix for all providers**: `call_llm_fix()` dispatcher + `_call_openai_compatible_fix()` + `_call_anthropic_fix()`
- **Prompt injection mitigation**: diff wrapped in `<guardianciDiff>` XML delimiters
- **`pyproject.toml`**: project metadata, optional dependency groups, ruff + pytest configuration
- **`requirements/`**: version-constrained `base.txt` and `dev.txt`; instructions for generating a full lockfile
- **Tests in CI**: new `test.yml` workflow runs `pytest` on every push and PR
- **`LICENSE`** (MIT), **`SECURITY.md`** (policy + known detection limits), **`CONTRIBUTING.md`**, **`.github/CODEOWNERS`**, **`CHANGELOG.md`**
- **`__version__ = "0.1.0"`** in `guardianci_ai_review.py`

### Changed
- `call_gemini()` → `_call_gemini()` (public alias preserved)
- `github_context()` → `_github_context()` (public alias preserved)
- `estimate_gemini_cost()` → `estimate_llm_cost()` (alias preserved)
- `_GEMINI_PRICING` → `_LLM_PRICING` (alias preserved)
- `_github_post()` → `_vcs_post()` (alias preserved)
- `github_headers()` → `_github_headers()` (alias preserved)
- Step label "Run Gemini security review" → "Run GuardianCI AI security review"
- `GUARDIANCI_GEMINI_ENABLED` → `GUARDIANCI_AI_ENABLED` (old name still accepted)
- `GEMINI_API_KEY` / `GEMINI_MODEL` still accepted as fallbacks

### Security
- All GitHub Actions `uses:` steps pinned to full commit SHAs (SLSA Level 2)
- Gitleaks pre-scan runs before any LLM sees the diff

---

## [0.0.1] — 2024 (initial release)

Initial implementation:
- Gemini-only LLM provider
- GitHub-only VCS support
- Inline PR comments with compliance citations (GDPR, PCI-DSS, SOC 2)
- CRITICAL/WARN/INFO severity with `before-merge` / `within-sprint` / `backlog` urgency
- Auto-fix draft PR generation
- `guardianci-metrics` git branch for trend storage
- False-positive `/fp` workflow with monthly audit
- Gitleaks secret pre-scan
- Large-diff cost controls (600-line threshold, high-risk path filtering)
- SHA deduplication (skips re-review of identical commits)

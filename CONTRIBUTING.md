# Contributing to GuardianCI

Thank you for taking the time to contribute. This guide covers everything you need to go from zero to a merged pull request.

---

## Table of contents

1. [Dev setup](#dev-setup)
2. [Running tests and lint](#running-tests-and-lint)
3. [How to add a new LLM provider](#how-to-add-a-new-llm-provider)
4. [How to add a new detection pattern](#how-to-add-a-new-detection-pattern)
5. [How to add a new VCS platform](#how-to-add-a-new-vcs-platform)
6. [Updating the lockfile](#updating-the-lockfile)
7. [Making a good PR](#making-a-good-pr)
8. [Reporting bugs](#reporting-bugs)
9. [Code structure](#code-structure)

---

## Dev setup

**Requirements:** Python 3.11+, [uv](https://docs.astral.sh/uv/)

```bash
git clone https://github.com/noobbatman/-GuardianCI.git
cd -GuardianCI

# Install dev dependencies from the hash-pinned lockfile
uv pip install --system -r requirements/dev.lock
```

---

## Running tests and lint

```bash
# Full test suite
pytest

# Single test
pytest tests/test_guardianci.py::test_local_scan_detects_hardcoded_api_key_python

# Lint
ruff check scripts/ tests/

# Auto-fix lint issues
ruff check --fix scripts/ tests/

# Check formatting
ruff format --check scripts/ tests/

# Fix formatting
ruff format scripts/ tests/
```

All tests make zero network calls — no LLM API key is needed.

---

## How to add a new LLM provider

This is the most common contribution. The pattern is consistent — each provider needs four things added to [scripts/guardianci_ai_review.py](scripts/guardianci_ai_review.py).

### Step 1 — Add the provider constant (line ~35)

```python
LLM_COHERE = "cohere"   # use lowercase, matches GUARDIANCI_LLM_PROVIDER value
```

### Step 2 — Implement `_call_<provider>()`  (near line ~1116)

Model it on `_call_openai_compatible()` or `_call_anthropic()`. The function must:
- Accept `(diff_text, *, truncated, model, exclusions, skipped_files)`
- Return `(parsed_payload, usage_dict)` where `parsed_payload` is the result of `parse_json_response()` and `usage_dict` is `{"input_tokens": int, "output_tokens": int}`
- Read the API key from `os.getenv("GUARDIANCI_LLM_API_KEY") or os.getenv("COHERE_API_KEY")`
- Read the base URL from `os.getenv("GUARDIANCI_LLM_BASE_URL", "https://api.cohere.ai")`
- Use `build_review_prompt()` to generate the prompt text

```python
def _call_cohere(
    diff_text: str,
    *,
    truncated: bool,
    model: str,
    exclusions: list[FalsePositiveExclusion] | None = None,
    skipped_files: list[str] | None = None,
) -> tuple[Any, dict[str, Any]]:
    api_key = os.getenv("GUARDIANCI_LLM_API_KEY") or os.getenv("COHERE_API_KEY", "")
    if not api_key:
        raise RuntimeError("No API key found. Set GUARDIANCI_LLM_API_KEY.")
    base_url = os.getenv("GUARDIANCI_LLM_BASE_URL", "https://api.cohere.ai")
    prompt = build_review_prompt(diff_text, truncated=truncated, exclusions=exclusions, skipped_files=skipped_files)

    resp = requests.post(
        f"{base_url}/v2/chat",
        headers={"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"},
        json={"model": model, "messages": [{"role": "user", "content": prompt}]},
        timeout=120,
    )
    resp.raise_for_status()
    data = resp.json()
    content = data["message"]["content"][0]["text"]
    usage = data.get("usage", {})
    return parse_json_response(content), {
        "input_tokens": usage.get("billed_units", {}).get("input_tokens", 0),
        "output_tokens": usage.get("billed_units", {}).get("output_tokens", 0),
    }
```

### Step 3 — Wire it into `call_llm()` (line ~1032)

```python
def call_llm(diff_text, *, truncated, model, provider=LLM_GEMINI, exclusions=None, skipped_files=None):
    p = provider.lower()
    if p == LLM_ANTHROPIC:
        return _call_anthropic(...)
    if p in (LLM_OPENAI, LLM_OPENAI_COMPAT):
        return _call_openai_compatible(...)
    if p == LLM_COHERE:                        # ← add this block
        return _call_cohere(...)
    return _call_gemini(...)                   # default
```

### Step 4 — Implement `_call_<provider>_fix()` and wire into `call_llm_fix()` (line ~1364)

Same pattern — takes `(file_path, file_text, finding, *, model)` and returns the patched file content as a string.

### Step 5 — Add to the pricing table (line ~48)

```python
_LLM_PRICING: dict[str, tuple[float, float]] = {
    ...
    "command-r-plus": (2.50, 10.00),   # USD per 1M input / output tokens
    "command-r": (0.15, 0.60),
    ...
}
```

Use `(0.0, 0.0)` if pricing is unknown or free. This is used only for cost logging — it never affects billing.

### Step 6 — Update docs

- Add the provider to the table in `README.md` (Supported LLM providers)
- Add a quick-start example to `.gitlab/guardianci.gitlab-ci.yml`
- Add a `GUARDIANCI_LLM_BASE_URL` entry to `SETUP.md` if it needs a custom endpoint

---

## How to add a new detection pattern

Local patterns run on every diff before the LLM is called. They are fast regex checks in `local_security_findings()` ([line 754](scripts/guardianci_ai_review.py#L754)).

### Step 1 — Add the regex and finding

Inside `local_security_findings()`, after the existing patterns:

```python
# Example: detect eval() on user-controlled input
eval_re = re.compile(r"\beval\s*\(\s*(?:request|input|params|user)", re.IGNORECASE)

for path, patch in patches:
    for line_no, raw_line in added_lines_with_numbers(patch):
        if not raw_line.startswith("+"):
            continue
        line = raw_line[1:]
        if eval_re.search(line):
            add_if_not_excluded(
                Finding(
                    file=path,
                    line_start=line_no,
                    line_end=line_no,
                    severity="CRITICAL",
                    issue="eval() called on what appears to be user-controlled input — remote code execution risk.",
                    suggested_fix="Replace eval() with a safe parser (ast.literal_eval, json.loads, or a purpose-built parser).",
                    frameworks=("PCI-DSS 6.2.4", "SOC 2 CC6.1", "GDPR Art. 32"),
                    remediation_urgency="before-merge",
                ),
                line,
            )
```

### Step 2 — Write two tests

In `tests/test_guardianci.py`, following the existing pattern:

```python
def test_local_scan_detects_eval_on_user_input() -> None:
    patch = _added(['    result = eval(request.body)'], start=12)
    findings = review.local_security_findings([_patch("app/handler.py", patch)])
    assert any("eval" in f.issue for f in findings)


def test_local_scan_does_not_flag_safe_eval() -> None:
    patch = _added(['    result = ast.literal_eval(safe_string)'], start=1)
    findings = review.local_security_findings([_patch("app/utils.py", patch)])
    assert not any("eval" in f.issue for f in findings)
```

Run `pytest` to confirm both pass before opening the PR.

---

## How to add a new VCS platform

GuardianCI auto-detects GitHub vs GitLab. Adding a third platform (e.g. Bitbucket) requires four things:

1. **Add a platform constant** near `VCS_GITHUB = "github"` (line ~43)
2. **Add `_<platform>_context()`** — reads the platform's CI env vars and returns a context dict with at minimum `repo`, `pr_number`, `pr_url`, `sha`, `token`, `platform`
3. **Add `_<platform>_post_review()`** — posts the summary comment and inline findings via the platform's REST API
4. **Wire both into `get_vcs_context()`** (detection) and **`post_review()`** (dispatch)
5. **Add a CI template** — e.g. `.bitbucket/guardianci.bitbucket-pipelines.yml`

---

## Updating the lockfile

When you add or change a dependency in `pyproject.toml` or `requirements/base.txt`, regenerate the lockfile:

```bash
# Runtime lockfile
uv pip compile pyproject.toml --extra gemini -o requirements/base.lock --generate-hashes --no-header

# Dev lockfile
uv pip compile pyproject.toml --extra dev --extra gemini -o requirements/dev.lock --generate-hashes --no-header
```

Commit both `.lock` files alongside the `pyproject.toml` / `requirements/*.txt` change.

---

## Making a good PR

- **One concern per PR.** A new provider and a refactor in the same PR are hard to review.
- **Tests come with every change.** New behaviour without tests will not be merged.
- **Backward-compatible.** If you rename a function or env var, keep an alias. See the `call_gemini = _call_gemini` pattern.
- **No new mandatory runtime dependencies** without a discussion issue first.
- **Update CHANGELOG.md** for any user-visible change.

Use the [PR template](.github/pull_request_template.md) — it has a checklist for each change type.

---

## Reporting bugs

Open a [GitHub issue](https://github.com/noobbatman/-GuardianCI/issues/new/choose) using the bug report template. For security vulnerabilities, see [SECURITY.md](SECURITY.md) — do **not** open a public issue.

---

## Code structure

| File | Responsibility |
|---|---|
| `scripts/guardianci_ai_review.py` | Main script: diff parsing, LLM dispatch, VCS posting, local scan |
| `scripts/guardianci_metrics.py` | Metrics: git-branch storage + webhook push |
| `scripts/guardianci_false_positive.py` | `/fp` comment handler and monthly audit |
| `.github/workflows/guardianci.yml` | GitHub CI: review → metrics → auto-fix jobs |
| `.github/workflows/guardianci-false-positive.yml` | False-positive workflow |
| `.gitlab/guardianci.gitlab-ci.yml` | GitLab CI template |
| `tests/test_guardianci.py` | Unit tests (zero network calls) |
| `requirements/base.lock` | Hash-pinned runtime lockfile |
| `requirements/dev.lock` | Hash-pinned dev lockfile |

**Planned for v0.2.0:** split `guardianci_ai_review.py` into a package (`guardianci/vcs/`, `guardianci/llm/`, `guardianci/scan/`). If you want to coordinate on that refactor, open an issue first.

---

## Good first issues

| Task | Difficulty | Where to look |
|---|---|---|
| Add a detection pattern for a new vulnerability class | Low | `local_security_findings()` + two tests |
| Add a new OpenAI-compatible provider to the pricing table | Low | `_LLM_PRICING` dict |
| Improve an existing error message | Low | Search for `raise RuntimeError` or `print(` |
| Add a provider that uses a non-OpenAI API shape | Medium | `_call_<provider>()` + `_call_<provider>_fix()` |
| Add Bitbucket Pipelines support | High | New platform context + post functions + CI template |

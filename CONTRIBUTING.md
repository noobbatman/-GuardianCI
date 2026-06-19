# Contributing to GuardianCI

Thank you for taking the time to contribute.

---

## Development setup

**Requirements:** Python 3.11+, [uv](https://docs.astral.sh/uv/) (recommended) or pip.

```bash
# Clone and enter the repo
git clone https://github.com/noobbatman/-GuardianCI.git
cd -GuardianCI

# Install development dependencies
uv pip install --system -r requirements/dev.txt

# Verify everything works
pytest          # runs the test suite
ruff check scripts/ tests/     # linter
ruff format --check scripts/ tests/   # formatter
```

---

## Running tests

```bash
pytest                      # all tests, short tracebacks
pytest -v                   # verbose mode
pytest tests/test_guardianci.py::test_local_scan_detects_hardcoded_api_key_python  # single test
```

The test suite (`tests/test_guardianci.py`) covers core logic without making any network calls, GitHub API calls, or LLM API calls. Keep it that way — all external calls must be mocked.

---

## Linting and formatting

GuardianCI uses [ruff](https://docs.astral.sh/ruff/) for both linting and formatting. CI enforces both.

```bash
ruff check scripts/ tests/         # check for lint errors
ruff check --fix scripts/ tests/   # auto-fix where possible
ruff format scripts/ tests/        # reformat in place
ruff format --check scripts/ tests/ # check without modifying
```

---

## Making changes

1. Fork the repo and create a feature branch from `main`
2. Write tests for any new behaviour (the test suite must stay green)
3. Ensure `ruff check` and `ruff format --check` pass before pushing
4. Open a pull request — GuardianCI will review its own PR (dogfooding)

### What makes a good PR

- **One concern per PR.** A bugfix and a refactor in the same PR are hard to review.
- **Tests come with the change.** Adding a new finding pattern? Add a test that triggers it and one that doesn't.
- **Backward compatibility.** GuardianCI is used as a copypasted workflow file. If you rename a function or env var, keep an alias. See the existing `call_gemini = _call_gemini` pattern.
- **No new mandatory dependencies.** Optional dependencies (like `google-genai`) are fine. Runtime-mandatory new packages require a discussion first.

---

## Reporting bugs

Open a [GitHub issue](https://github.com/noobbatman/-GuardianCI/issues/new) with:

- What you expected to happen
- What actually happened
- The relevant portion of the workflow log (redact secrets)
- Your `GUARDIANCI_LLM_PROVIDER` and model name

For security vulnerabilities, see [SECURITY.md](SECURITY.md) — do **not** open a public issue.

---

## Code structure

| File | Responsibility |
|---|---|
| `scripts/guardianci_ai_review.py` | Main review script: diff parsing, LLM dispatch, VCS posting |
| `scripts/guardianci_metrics.py` | Metrics: git-branch storage + webhook push |
| `scripts/guardianci_false_positive.py` | `/fp` comment handler and monthly audit |
| `.github/workflows/guardianci.yml` | GitHub CI: review → metrics → auto-fix jobs |
| `.github/workflows/guardianci-false-positive.yml` | False-positive workflow |
| `.gitlab/guardianci.gitlab-ci.yml` | GitLab CI template |
| `tests/test_guardianci.py` | Unit tests (no network calls) |

**Planned for v0.2:** Split `guardianci_ai_review.py` (~2 200 lines) into a proper package (`guardianci/vcs/`, `guardianci/llm/`, `guardianci/scan/`). If you want to help with that refactor, open an issue to coordinate.

---

## License

By contributing you agree that your contributions will be licensed under the MIT License.

## What does this PR do?

<!-- One paragraph summary. Link to the issue it closes: "Closes #123" -->

## Type of change

- [ ] Bug fix
- [ ] New LLM provider
- [ ] New VCS platform
- [ ] New detection pattern
- [ ] Performance / cost improvement
- [ ] Documentation
- [ ] Other (describe below)

## Checklist

- [ ] Tests added or updated for every changed behaviour
- [ ] `pytest` passes locally (`pytest tests/`)
- [ ] `ruff check` and `ruff format --check` pass (`ruff check scripts/ tests/ && ruff format --check scripts/ tests/`)
- [ ] No new mandatory runtime dependencies introduced without discussion
- [ ] Backward-compatible: old env var names / function names still work (or breakage is intentional and documented)
- [ ] CHANGELOG.md updated if this is a user-visible change

## If adding a new LLM provider

- [ ] `_call_<provider>()` implemented and wired into `call_llm()`
- [ ] `_call_<provider>_fix()` implemented and wired into `call_llm_fix()`
- [ ] Provider constant added (`LLM_<PROVIDER> = "<provider>"`)
- [ ] Pricing entry added to `_LLM_PRICING` (use `(0.0, 0.0)` if free / unknown)
- [ ] Provider listed in `.gitlab/guardianci.gitlab-ci.yml` quick-start comments
- [ ] Provider listed in `README.md` supported providers table

## If adding a new detection pattern

- [ ] Pattern added to `local_security_findings()` with severity, issue text, frameworks, and urgency
- [ ] Test that **triggers** the pattern
- [ ] Test that **does not trigger** (avoid false positives on safe equivalent code)

## Testing notes

<!-- Describe how you tested this. Paste relevant test output if useful. -->

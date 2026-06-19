# Security Policy

## Supported versions

| Version | Supported |
|---|---|
| `main` branch | Yes — security fixes land here first |
| Older tagged releases | No |

GuardianCI does not yet use stable semver releases. All production users should track `main`.

---

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Email **security@guardianci.dev** (or open a [GitHub private security advisory](https://github.com/noobbatman/-GuardianCI/security/advisories/new)) with:

- A description of the vulnerability
- Steps to reproduce it
- The impact you expect if exploited
- Any suggested fix (optional)

You will receive an acknowledgement within 48 hours and a resolution timeline within 7 days.

---

## Known detection limits

GuardianCI is a defence-in-depth layer, not a complete security solution. Users and adopters should be aware of the following inherent constraints:

### Regex-based local scanning

The `local_security_findings()` function uses regular-expression heuristics, not an AST parser. This means:

- **False negatives are possible.** A finding may be missed if the code is structured in a way the pattern does not match (e.g., multi-line string concatenation, obfuscated identifiers).
- **False positives are possible.** The patterns match text — they cannot tell whether a matched value is actually reachable, deployed, or exploitable.
- Patterns are not language-aware. A YAML config and a Python string are treated the same way.

For higher-assurance scanning, combine GuardianCI with a dedicated SAST tool (Semgrep, CodeQL, Bandit) that performs AST analysis.

### LLM-based analysis

- **The LLM may hallucinate.** Findings from the AI review should be treated as high-confidence suggestions, not guaranteed facts. Each finding is reviewed by a human before merge.
- **Prompt injection is possible.** A PR author could craft diff content that attempts to manipulate the LLM's output. GuardianCI wraps the diff in XML delimiters (`<guardianciDiff>`) and instructs the model to treat the content as untrusted, but this is a mitigation, not a complete defence.
- **The LLM only sees changed lines.** It does not see the full file context, so findings that depend on understanding call graphs, class hierarchies, or data flow across files will be missed.
- **Token truncation.** Very large diffs are truncated to fit the model's context window. Files above the threshold are flagged in the review summary, but their content is not sent to the LLM.

### LLM cost estimates

The `estimate_llm_cost()` function uses a hardcoded pricing table. Provider pricing changes frequently — treat cost estimates as approximations and verify against your provider's current billing page.

### What GuardianCI is not

- Not a runtime WAF or RASP
- Not a penetration test replacement
- Not a compliance certification (findings cite frameworks as guidance, not as audit evidence)
- Not a substitute for security-trained code review on high-sensitivity changes

---

## Supply-chain security

GuardianCI's own CI follows SLSA Level 2 practices:

- All GitHub Actions `uses:` steps are pinned to full commit SHAs (not floating tags)
- The workflow itself (`guardianci.yml`) runs with minimal permissions (`contents: read`, elevated only where required)
- Secret scanning (gitleaks) runs before the LLM ever sees diff content
- Dependency versions are constrained in `requirements/` and `pyproject.toml`

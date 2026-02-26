---
description: Triage SAST scan results from a Semgrep JSON report, classifying each finding as a true or false positive with explanation and remediation advice. Saves a full triage-report.md to the project root.
allowed-tools: Read, Write, Bash(semgrep:*)
argument-hint: <sast-results.json> <repo-path> [repo-context-description]
---

## Your Task
Triage all SAST findings in the provided Semgrep scan report.

**Arguments provided:** $ARGUMENTS

Parse them as:
1. **Arg 1** — path to Semgrep JSON results file
2. **Arg 2** — path to the scanned code repository
3. **Arg 3+** (optional) — context about the repository (purpose, stack, hosting, internal/external)

---

## Triage Workflow

1. **Read the results file** (Arg 1) and parse all findings from the `results` array in the Semgrep JSON.
2. **Sort findings** by severity: Critical → High → Medium → Low.
3. **For each finding**:
   a. Note the file path, line number, rule ID, and vulnerability description.
   b. Read the flagged source file from Arg 2 and examine ±25 lines around the flagged line for context.
   c. Trace the data flow: does user-controlled input actually reach the vulnerable sink without adequate sanitization?
   d. Consider any encoding, validation, parameterized queries, or framework-level protections present.
   e. Factor in any repository context provided (Arg 3).
   f. Assign a verdict: **TRUE POSITIVE** or **FALSE POSITIVE**, with a confidence level (High / Medium / Low).
4. **Write** the complete triage report to `triage-report.md` in the current directory using the format below.

---

## Output: triage-report.md Format

```
# SAST Triage Report

**Date:** [current date]
**Tool:** Semgrep
**Repository:** [Arg 2]
**Total Findings:** N | **True Positives:** X | **False Positives:** Y | **Confidence Breakdown:** ...

---

## Executive Summary
[2-3 sentence summary of the scan findings, key risk areas, and recommended priorities]

---

## Summary Table

| # | File | Vulnerability Type | Severity | Verdict | Confidence |
|---|------|--------------------|----------|---------|------------|
| 1 | path/to/File.java:42 | SQL Injection | High | ✅ TRUE POSITIVE | High |
| 2 | ... | ... | ... | ❌ FALSE POSITIVE | Medium |

---

## Detailed Findings

### Finding #1: [Vulnerability Type] — [Severity]
- **File:** `path/to/File.java:42`
- **Rule:** `semgrep-rule-id`
- **Verdict:** ✅ TRUE POSITIVE / ❌ FALSE POSITIVE
- **Confidence:** High / Medium / Low
- **Explanation:** [2-4 sentences referencing specific code: why user input reaches the sink, or why this is a false alarm]
- **Remediation:** [If TRUE POSITIVE: specific, actionable fix. If FALSE POSITIVE: brief rationale for dismissal]

---
[repeat for each finding]
```

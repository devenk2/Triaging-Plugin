---
description: Triage SAST scan results from a Semgrep JSON report, classifying each finding as a true or false positive with explanation and remediation advice. Optionally limit analysis to the first N findings. Saves triage-report.md to the project root.
allowed-tools: Read, Write, Bash(semgrep:*)
argument-hint: <sast-results.json> <repo-path> [repo-context-description] [limit]
---

## Your Task
Triage all SAST findings in the provided Semgrep scan report.

**Arguments provided:** $ARGUMENTS

Parse them as:
1. **Arg 1** — path to Semgrep JSON results file
2. **Arg 2** — path to the scanned code repository
3. **Arg 3** (optional) — context about the repository (purpose, stack, hosting, internal/external)
4. **Arg 4** (optional) — maximum number of findings to analyze (integer, e.g. `20`)

If Arg 4 is present and valid (>0), use it as `limit`. Otherwise, process all findings.

---

## Triage Workflow

1. **Read the results file** (Arg 1) and parse all findings from the `results` array in the Semgrep JSON.
2. **Sort findings** by severity: Critical → High → Medium → Low.
3. **Apply limit (if provided)**: if `limit` is set, analyze only the first `limit` findings from the sorted list.
4. **For each selected finding**:
   a. Note the file path, line number, rule ID, and vulnerability description.
   b. Read the flagged source file from Arg 2 and examine ±25 lines around the flagged line for context.
   c. Trace the data flow: does user-controlled input actually reach the vulnerable sink without adequate sanitization?
   d. Consider any encoding, validation, parameterized queries, or framework-level protections present.
   e. Factor in any repository context provided (Arg 3).
   f. Assign a verdict: **TRUE POSITIVE** or **FALSE POSITIVE**, with a confidence level (High / Medium / Low).
5. **Write** the triage report to `triage-report.md` in the current directory using the format below.

---

## Output: triage-report-{datetime}.md Format

Directions:
- Tag the report output with ISO 8601 datetime in the filename (e.g., `triage-report-2026-02-27T10-34-02.md`)
- Place the report in `triage-reports/{repository-name}/` (extract repository name from Arg 2)
- Create the directory if it doesn't exist


```
# SAST Triage Report

**Date:** [current date]
**Tool:** Semgrep
**Repository:** [Arg 2]
**Total Findings in Scan:** N | **Findings Analyzed:** M | **Limit:** [all or number] | **True Positives:** X | **False Positives:** Y | **Confidence Breakdown:** ...

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

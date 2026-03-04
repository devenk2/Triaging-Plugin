---
description: Triage SAST scan results from a Semgrep JSON report, classifying each finding as a true, false positive, or needing human review with explanation and remediation advice. Optionally limit analysis to the first N findings. Saves triage-report.md to the project root.
allowed-tools: Read, Write, Bash(semgrep:*)
argument-hint: <sast-results.json> <repo-path> [repo-context-or-file] [limit]
---

## Your Task
Triage all SAST findings in the provided Semgrep scan report.

**Arguments provided:** $ARGUMENTS

Parse them as:
1. **Arg 1** — path to Semgrep JSON results file
2. **Arg 2** — path to the scanned code repository
3. **Arg 3** (optional) — context about the repository as either an inline text description OR a path to a text file (.txt, .md, .json) containing the description. If a file path, read the file contents and use that as context.
4. **Arg 4** (optional) — maximum number of findings to analyze (integer, e.g. `20`)

If Arg 4 is present and valid (>0), use it as `limit`. Otherwise, process all findings.

---

## Vulnerability Type Guidance

Before analyzing each finding, consult the vulnerability guidance lookup table in `.claude/vulnerability-guidance.json` and apply the specific true positive / false positive indicators for that vulnerability type. This external reference makes it easy to expand the guidance with new vulnerability types without modifying the triage logic.

---

## Confidence Scoring Rubric

Replace traditional "High/Medium/Low" with a percentage (0–100%) calculated by sum of these factors:

| Factor | Points |
|--------|--------|
| Clear source of untrusted input identified | +25% |
| Direct, unbroken data flow to vulnerable sink traced | +25% |
| No sanitization, validation, or parameterized query found | +20% |
| Vulnerability class matches known-exploitable pattern (per guidance table) | +15% |
| Repository context confirms external exposure / high-risk environment | +10% |
| Ambiguity in data flow (partial or unclear trace) | −15% |
| Possible but unconfirmed protection in upstream/downstream code | −10% |
| Framework or library may provide implicit protection (unclear if applied) | −10% |

**Rules:**
- Sum factors to calculate initial score
- Cap at **95%** (never 100% — static analysis cannot guarantee runtime exploitability)
- Floor at **5%** (never 0% — some vulnerability findings always carry minimal risk)
- Round to nearest integer (e.g., 87%)

---

## Verdict Options

Assign one of three verdicts to each finding:

1. **✅ TRUE POSITIVE** — The vulnerability is real and exploitable; requires remediation.
2. **❌ FALSE POSITIVE** — The finding is a false alarm and can be safely dismissed.
3. **⚠️ NEEDS HUMAN REVIEW** — The verdict cannot be determined statically; human judgment required.

### When to Use "NEEDS HUMAN REVIEW"

Use this verdict if any of the following apply:
- Data flow is partially traceable but requires runtime/deployment context (e.g., method calls that may or may not sanitize depending on runtime config)
- A protection exists but its effectiveness is ambiguous (custom validator, non-standard library)
- Confidence score falls between 35–60% after rubric scoring
- The vulnerability is high-severity but context suggests possible mitigation that cannot be verified statically
- Flagged code is generated/templated and the generator logic is outside the scan scope
- Framework behavior is unclear or depends on version-specific features

---

## Triage Workflow

1. **Process Arg 3 (repo context)**:
   - If Arg 3 is provided, determine if it's a file path or inline text:
     - If it looks like a file path (contains `.txt`, `.md`, `.json` extension OR `Read` tool can access it), read the file contents as the repo context
     - Otherwise, use Arg 3 directly as inline context text
   - Store the context for use in each finding analysis

2. **Read the results file** (Arg 1) and parse all findings from the `results` array in the Semgrep JSON.

3. **Sort findings** by severity: Critical → High → Medium → Low.

4. **Apply limit (if provided)**: if `limit` is set, analyze only the first `limit` findings from the sorted list.

5. **For each selected finding**:
   a. Note the file path, line number, rule ID, and vulnerability description.
   b. **Look up the vulnerability type** in `.claude/vulnerability-guidance.json` and note the key indicators.
   c. Read the flagged source file from Arg 2 and examine ±25 lines around the flagged line for context.
   d. Trace the data flow: does user-controlled input actually reach the vulnerable sink without adequate sanitization?
   e. Check against the guidance JSON's false positive patterns to rule out common false alarms.
   f. Consider any encoding, validation, parameterized queries, or framework-level protections present.
   g. Factor in the repository context (processed from Arg 3).
   h. **Assign a verdict** (✅ TRUE POSITIVE / ❌ FALSE POSITIVE / ⚠️ NEEDS HUMAN REVIEW) and calculate **confidence percentage** using the rubric above.

6. **Write** the triage report to `triage-report.md` in the current directory using the format below.

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
**Total Findings in Scan:** N | **Findings Analyzed:** M | **Limit:** [all or number] | **True Positives:** X | **False Positives:** Y | **Needs Human Review:** Z

---

## Executive Summary
[2-3 sentence summary of the scan findings, key risk areas, and recommended priorities]

---

## Summary Table

| # | File | Vulnerability Type | Severity | Verdict | Confidence |
|---|------|--------------------|----------|---------|------------|
| 1 | path/to/File.java:42 | SQL Injection | High | ✅ TRUE POSITIVE | 87% |
| 2 | path/to/File.java:55 | XSS | Medium | ❌ FALSE POSITIVE | 12% |
| 3 | path/to/File.java:73 | Path Traversal | High | ⚠️ NEEDS HUMAN REVIEW | 48% |

---

## Detailed Findings

### Finding #1: [Vulnerability Type] — [Severity]
- **File:** `path/to/File.java:42`
- **Rule:** `semgrep-rule-id`
- **Verdict:** ✅ TRUE POSITIVE / ❌ FALSE POSITIVE / ⚠️ NEEDS HUMAN REVIEW
- **Confidence:** [percentage, e.g., 87%]
- **Explanation:** [2-4 sentences referencing specific code and the guidance JSON indicators: why user input reaches the sink, false positive patterns ruled out, or what ambiguity prevents a definitive verdict]
- **Action:** [If TRUE POSITIVE: specific, actionable remediation. If FALSE POSITIVE: brief rationale for dismissal. If NEEDS HUMAN REVIEW: specific questions or information needed to resolve the ambiguity, e.g., "Determine if the validator at line 42 applies to all callers of this method"]

---
[repeat for each finding]
```

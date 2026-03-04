# SAST Triage Plugin

## Overview

This repository contains a Claude Code plugin for efficiently triaging static application security testing (SAST) scan results. A common problem in application security today is that SAST tools can generate thousands of “findings,” which need to be triaged as false positives (FP) or true positives (TP).

AppSec teams often face bandwidth and resource constraints when analyzing these findings, while development teams face pressure to release features quickly. This plugin automates the triage process by leveraging Claude's security expertise, allowing security teams to classify findings as exploitable vulnerabilities or false alarms with detailed explanations and remediation advice.

## The Problem

Manual SAST triage is time-consuming:
1. Export scan results from the SAST tool
2. Open and review the code repository
3. Prioritize findings by severity (Critical → High → Medium → Low)
4. For each finding, determine if it's exploitable in context
5. Document findings with explanations and fixes
6. Communicate results to the development team

This plugin accelerates steps 3-6 by automating intelligent analysis of each finding.

## What This Plugin Does

The plugin includes two components:

### 1. **Slash Command: `/triage-sast`**
Explicit command for programmatic SAST triage with specific parameters.

**Usage:**
```bash
/triage-sast <sast-results.json> <repo-path> [repo-context] [limit]
```

### 2. **Skill: `sast-triage`**
Auto-triggered skill that activates when you discuss SAST findings conversationally (e.g., “triage these findings”, “is this a false positive?”).

## Setup

### Prerequisites
- SAST scan results in **Semgrep JSON format** (other formats supported with conversion)
- The **source code repository** that was scanned
- **Claude Code** installed

### Generating Semgrep Results

If you don't have SAST results yet, generate them using Semgrep:

```bash
# Install Semgrep (if needed)
brew install semgrep
# OR: pip install semgrep

# Run Semgrep on your Java code
semgrep --config=p/java /path/to/code --json > sast-results.json
```

For other languages, replace `p/java` with the appropriate ruleset (e.g., `p/python`, `p/javascript`, `p/security-audit`).

## Quick Start

### Example Command

```bash
/triage-sast sast-results.json BenchmarkJava-master/ “Java EE web application, OWASP Benchmark test suite, intentionally vulnerable code for security tool evaluation” 20
```

### Argument Breakdown

| Argument | Type | Example | Description |
|----------|------|---------|-------------|
| **Arg 1** | Path | `sast-results.json` | Path to Semgrep JSON report file. Can be relative or absolute. |
| **Arg 2** | Path | `BenchmarkJava-master/` | Path to the scanned source code repository root. The plugin reads source files from here to analyze findings in context. |
| **Arg 3** | String or File Path (optional) | `”Java EE web application...”` or `context.md` | Context about the repository to inform triage decisions. Can be either: (1) **inline text** with purpose/function, tech stack (languages/frameworks), deployment (internal/external/cloud), security notes, or (2) **path to a file** (.txt, .md, .json) containing the context. File paths are automatically detected and read. Helps distinguish between exploitable vulnerabilities and false positives based on actual application behavior. |
| **Arg 4** | Integer (optional) | `20` | Maximum number of findings to analyze. Limits analysis to the first N findings (sorted by severity). Useful for large scans. Omit to analyze all findings. |

#### Detailed Example Walkthrough

```bash
/triage-sast sast-results.json BenchmarkJava-master/ “Java EE web application, OWASP Benchmark test suite, intentionally vulnerable code for security tool evaluation” 20
```

**Arg 1: `sast-results.json`**
- The Semgrep scan output in JSON format
- Contains 1,909 findings across the codebase
- Located in the project root

**Arg 2: `BenchmarkJava-master/`**
- Root directory of the source code that was scanned
- Plugin reads flagged files from here to examine code context (±25 lines)
- Helps trace data flow and identify sanitization/validation

**Arg 3: Context String**
```
“Java EE web application, OWASP Benchmark test suite,
intentionally vulnerable code for security tool evaluation”
```
- **Java EE web application** → Framework: Spring/Java servlets, likely handles HTTP requests
- **OWASP Benchmark test suite** → This is intentional vulnerable code (not production)
- **Intentionally vulnerable** → Expected to have security issues; used for tool evaluation
- Helps the plugin understand that all findings are expected to be true positives

**Arg 4: `20`**
- Limits triage to the first 20 findings (highest severity)
- Scans often have hundreds/thousands of findings; limiting is practical for initial review
- Results are sorted by severity: Critical → High → Medium → Low
- Omitting this argument analyzes all findings

## Output

The plugin generates a comprehensive triage report and saves it to:

```
triage-reports/{repository-name}/triage-report-{ISO-8601-timestamp}.md
```

**Example path:**
```
triage-reports/BenchmarkJava-master/triage-report-2026-02-27T10-41-49.md
```

### Report Contents

Each report includes:

1. **Metadata**
   - Scan date and tool (Semgrep)
   - Repository path
   - Total findings in scan, findings analyzed, limit applied
   - True Positives, False Positives, Needs Human Review counts

2. **Executive Summary**
   - 2-3 sentence overview of key risk areas
   - Vulnerability type breakdown
   - Recommendations for remediation priority

3. **Summary Table**
   - Quick reference with file, line number, vulnerability type, severity
   - Verdict (✅ TRUE POSITIVE / ❌ FALSE POSITIVE / ⚠️ NEEDS HUMAN REVIEW)
   - **Confidence (percentage 0–100%)** — Evidence-based scoring using the confidence rubric

4. **Detailed Findings**
   - For each finding:
     - File path and line number
     - Semgrep rule ID
     - Verdict (TP/FP/NHR) with percentage confidence
     - Explanation citing specific code evidence and vulnerability guidance indicators
     - Actionable remediation (for TP), dismissal rationale (for FP), or investigation guidance (for NHR)

## Triage Methodology

For each finding, the plugin:

1. **Looks up vulnerability guidance** — Consults `.claude/vulnerability-guidance.json` for vulnerability-type-specific indicators (true positive patterns, false positive patterns)
2. **Reads the flagged code** and surrounding context (±25 lines)
3. **Traces the data flow** — Does user-controlled input actually reach the vulnerable sink?
4. **Checks for protections** — Encoding, validation, parameterized queries, framework guards (compared against FP patterns)
5. **Considers context** — Application purpose, framework capabilities, deployment model
6. **Assigns verdict** — ✅ TRUE POSITIVE (exploitable), ❌ FALSE POSITIVE (safe), or ⚠️ NEEDS HUMAN REVIEW (ambiguous)
7. **Calculates confidence** — Evidence-based percentage (0–100%) using the confidence scoring rubric:
   - Clear input source: +25%
   - Direct data flow to sink: +25%
   - No sanitization: +20%
   - Matches vulnerability pattern: +15%
   - External exposure: +10%
   - Capped at 95%, floored at 5%

## Vulnerability Guidance and Confidence Scoring

### Vulnerability Guidance Table (`.claude/vulnerability-guidance.json`)

The plugin uses a centralized JSON lookup table containing vulnerability-type-specific guidance:

```json
{
  "SQL Injection": {
    "true_positive_indicators": [...],
    "false_positive_patterns": [...]
  },
  "Path Traversal": { ... },
  "Cross-Site-Scripting (XSS)": { ... },
  ...
}
```

**Vulnerability types covered:**
- SQL Injection, Path Traversal, Cross-Site-Scripting (XSS), Command Injection
- Deserialization, LDAP Injection, XXE (XML External Entity), SSRF
- Cryptographic Issues, Insecure Hashing Algorithm, XPath Injection, and more

**Why this matters:** Instead of using subjective confidence levels, the plugin references concrete TP/FP indicators for each vulnerability class. This makes triage decisions more consistent and evidence-based.

### Confidence Scoring Rubric

Confidence is calculated as a percentage (0–100%) by summing evidence factors:

| Factor | Points | Explanation |
|--------|--------|-------------|
| Clear source of untrusted input | +25% | HTTP parameter, header, cookie, user session |
| Direct data flow to sink | +25% | Unbroken chain from input to vulnerable function |
| No sanitization/validation | +20% | No encoding, filtering, or parameterized queries |
| Matches vulnerability pattern | +15% | Code pattern matches known-exploitable indicators |
| External exposure | +10% | Application exposed to untrusted users (e.g., internet-facing) |
| Data flow ambiguity | −15% | Partial trace, unclear protection effectiveness |
| Possible upstream protection | −10% | Unconfirmed protection in calling code |
| Framework implicit protection | −10% | Framework may auto-protect but not certain |

**Result:** Sum factors, cap at 95% (static analysis certainty limit), floor at 5%.

**Example:** A direct SQL injection (no PreparedStatement) = +25% (input) +25% (flow) +20% (no sanitization) +15% (pattern) +10% (external) = 95%

### Verdict Types

- **✅ TRUE POSITIVE (TP):** Real, exploitable vulnerability. Requires remediation.
- **❌ FALSE POSITIVE (FP):** False alarm. Safe to dismiss with documented rationale.
- **⚠️ NEEDS HUMAN REVIEW (NHR):** Cannot determine statically. Confidence typically 35–60%. Requires investigation of specific ambiguous aspects.

## Common Use Cases

### Review High-Priority Findings (Inline Context)
```bash
/triage-sast scan-results.json ./src/ “Production Node.js API, externally facing” 50
```
Analyze the 50 highest-severity findings from a production Node.js API.

### Review High-Priority Findings (File-Based Context)
```bash
/triage-sast scan-results.json ./src/ context.md 50
```
Analyze the 50 highest-severity findings using context from `context.md` (useful for complex or lengthy context descriptions).

### Full Scan Triage
```bash
/triage-sast semgrep-output.json ./code/ “Internal Python data pipeline”
```
Triage all findings (no limit) from an internal Python application.

### Conversational Analysis
Simply paste findings in chat:
> “I got these SQL injection warnings in our Flask app. Are they real?”

The `sast-triage` skill will auto-trigger and analyze them.

## Integration with Development Workflow

1. **Developer runs SAST:** `semgrep --config=p/java . --json > sast-results.json`
2. **Security analyst triages:** `/triage-sast sast-results.json ./ “context”`
3. **Review report:** Open `triage-reports/` and review findings
4. **Communicate:** Share true positives with developers for remediation
5. **Track progress:** Timestamped reports allow tracking of remediation over time

## Tips for Best Results

- **Provide context (inline or file):** Detailed repository context helps distinguish FP from TP. Use a `.md` or `.txt` file for complex context descriptions
- **Use limits on large scans:** Analyze highest-severity findings first with the limit parameter
- **Review percentage confidence:** Findings with 35–60% confidence may warrant “NEEDS HUMAN REVIEW” verdicts. Investigate the specific questions in the report
- **Understand the guidance table:** Check `.claude/vulnerability-guidance.json` to see what the plugin looks for when triaging each vulnerability type
- **Keep reports organized:** The timestamped directory structure lets you track trends and remediation progress over time
- **Re-scan after fixes:** Run triage on updated code to verify remediation and see confidence scores improve

## What's New (v2.0)

The plugin has been enhanced with four major features:

### 1. **Percentage-Based Confidence** (Not High/Medium/Low)
Instead of subjective confidence levels, each finding receives a percentage score (0–100%) calculated from objective evidence factors. This makes confidence assessments more transparent and defensible.

### 2. **File-Based Context Support**
Arg 3 now accepts file paths (.txt, .md, .json) in addition to inline text. Automatically detects and reads the file, so you can maintain detailed context descriptions without command-line length constraints.

```bash
/triage-sast sast-results.json repo/ context-production-java.md 50
```

### 3. **Centralized Vulnerability Guidance**
A JSON lookup table (`.claude/vulnerability-guidance.json`) contains vulnerability-type-specific true positive and false positive patterns. This ensures consistent, evidence-based triaging across all findings without modifying the triage logic.

**Current coverage:** SQL Injection, Path Traversal, XSS, Command Injection, LDAP Injection, XXE, SSRF, Cryptographic Issues, Insecure Hashing, XPath Injection, and more.

### 4. **"Needs Human Review" Verdict**
A third verdict option for ambiguous findings where confidence is 35–60% or data flow cannot be fully determined statically. Includes specific investigation guidance for the reviewer.

```
⚠️ NEEDS HUMAN REVIEW — Determine if the validator at line 42 applies to all callers
```

## Inputs and Outputs Summary

### Inputs
- **SAST results:** Semgrep JSON format (other tools can be converted)
- **Source code:** Scanned repository for code context analysis
- **Repository context:** Purpose, tech stack, deployment, security notes

### Outputs
- **Markdown report** with triaged findings
- **True/False verdicts** for each finding
- **Explanations** of triage decisions with code context
- **Remediation advice** for true positives
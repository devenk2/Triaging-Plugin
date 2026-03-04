# GitHub Copilot SAST Triage Agent — Design & Implementation Plan

## 1. Problem Statement & Why Copilot

Our Claude Code SAST triage plugin delivers accurate, automated vulnerability triage — but many clients don't have Claude Code. Most of these clients **are already on GitHub** with active Copilot subscriptions.

[GitHub Copilot CLI](https://github.blog/changelog/2026-02-25-github-copilot-cli-is-now-generally-available/) reached general availability in February 2026 with full agentic capabilities, [custom agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents), and [multi-model support](https://github.blog/ai-and-ml/github-copilot/whats-new-with-github-copilot-coding-agent/) (Claude, GPT, Gemini). This means we can deliver the **same triage methodology** as the Claude Code plugin — without writing any application code. The agent definition file IS the implementation.

**Key advantages of the Copilot approach:**

- **Zero custom code** — No Python, no dependencies, no Docker. The agent is a markdown file.
- **Multi-model** — Clients choose Claude Sonnet 4.6, GPT-5.3-Codex, GPT-4.1, or Gemini 3 Pro.
- **Three usage modes** — Interactive CLI, background cloud delegation, and GitHub Issue automation.
- **Built-in security scanning** — Copilot's [native code/secret/dependency scanning](https://github.blog/ai-and-ml/github-copilot/whats-new-with-github-copilot-coding-agent/) complements SAST triage.
- **Enterprise-ready** — SSO, audit logs, RBAC, and data residency inherited from GitHub Business/Enterprise tiers.
- **No new vendor** — Clients already paying for Copilot; no procurement needed.

---

## 2. Architecture — Mapping Claude Code to Copilot

### 2.1 Component Mapping

| Claude Code Component | File | Copilot Equivalent | File |
|---|---|---|---|
| Slash command | `.claude/commands/triage-sast.md` | Custom agent | `.github/agents/sast-triage.agent.md` |
| Auto-triggered skill | `.claude/skills/sast-triage/SKILL.md` | Agent auto-selection (via `description`) | Same agent file |
| Vulnerability guidance | `.claude/vulnerability-guidance.json` | Vulnerability guidance | `.github/vulnerability-guidance.json` |
| `CLAUDE.md` (project context) | `CLAUDE.md` | Repository instructions | `.github/copilot-instructions.md` |
| N/A | N/A | Cloud agent environment | `.github/workflows/copilot-setup-steps.yml` |
| N/A | N/A | Reusable prompt template | `.github/prompts/triage-batch.prompt.md` |

### 2.2 How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                   User Invocation                            │
│                                                              │
│  CLI:   copilot "@sast-triage analyze sast-results.json"    │
│  Cloud: & @sast-triage analyze sast-results.json            │
│  Issue: Assign to @copilot with triage instructions          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Copilot Agent Runtime                            │
│                                                              │
│  1. Loads sast-triage.agent.md (methodology + rubric)        │
│  2. Reads copilot-instructions.md (repo context)             │
│  3. Reads vulnerability-guidance.json (TP/FP indicators)     │
│  4. Parses Semgrep JSON results file                         │
│  5. For each finding:                                        │
│     a. Reads flagged source file (±25 lines)                 │
│     b. Looks up vuln type in guidance JSON                   │
│     c. Traces data flow, checks protections                  │
│     d. Assigns verdict + confidence score                    │
│  6. Writes triage-report-{timestamp}.md                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Output                                  │
│                                                              │
│  CLI mode:   Report written to triage-reports/               │
│  Cloud mode: PR opened with report as new file               │
│  Issue mode: PR opened, linked to original issue             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Repository Structure

Files to add to a client's repository:

```
.github/
├── agents/
│   └── sast-triage.agent.md           # Custom agent definition (core deliverable)
├── vulnerability-guidance.json         # TP/FP indicators per vuln type (reused from plugin)
├── copilot-instructions.md            # Repository-level context for all Copilot interactions
├── prompts/
│   └── triage-batch.prompt.md         # Reusable prompt for standardized invocations
└── workflows/
    └── copilot-setup-steps.yml        # Cloud agent environment setup
```

**Total footprint:** 5 files, ~25KB. No source code, no build system, no dependencies.

---

## 4. Custom Agent Design: `sast-triage.agent.md`

This is the core deliverable. The full agent file is shown below.

### 4.1 Design Constraints

- **30,000 character limit** on markdown instructions per [GitHub docs](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- **Tools**: `read`, `edit`, `search`, `execute` — agent needs to read source files, parse JSON, and write the report
- **No network tools needed** — all data is local (Semgrep JSON + source code)
- The vulnerability guidance JSON is referenced by path rather than inlined to save prompt space

### 4.2 Complete Agent File

```markdown
---
name: sast-triage
description: >
  SAST vulnerability triage agent. Analyzes Semgrep scan results against source code
  to classify each finding as a true positive, false positive, or needing human review.
  Use when triaging security scan results, reviewing SAST findings, assessing whether
  a vulnerability is exploitable, or classifying code scanning alerts. Triggers on
  phrases like "triage these findings", "is this a false positive", "review SAST results",
  "analyze this security scan", or "which findings should I fix".
tools:
  - read
  - edit
  - search
  - execute
---

# SAST Triage Agent

You are an expert application security analyst with deep knowledge of OWASP Top 10
vulnerability patterns, secure coding practices, and SAST tool behavior.

## Your Task

When invoked, you will triage SAST findings from a Semgrep JSON report. Parse the
user's request to identify:

1. **Results file** — path to the Semgrep JSON results file
2. **Repository path** — path to the scanned source code (defaults to repo root)
3. **Repository context** — optional description of the application (inline text or
   path to a .txt/.md file)
4. **Limit** — optional max number of findings to analyze

## Vulnerability Type Guidance

Before analyzing each finding, read the vulnerability guidance lookup table at
`.github/vulnerability-guidance.json` and apply the specific true positive / false
positive indicators for that vulnerability type. This file contains per-vulnerability-type
indicators for: SQL Injection, Path Traversal, XSS, Command Injection, Deserialization,
LDAP Injection, XXE, SSRF, XPath Injection, Cryptographic Issues, Insecure Hashing, etc.

## Confidence Scoring Rubric

Calculate a confidence percentage (0–100%) by summing these factors:

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

## Verdict Options

Assign one of three verdicts to each finding:

1. **✅ TRUE POSITIVE** — The vulnerability is real and exploitable; requires remediation.
2. **❌ FALSE POSITIVE** — The finding is a false alarm and can be safely dismissed.
3. **⚠️ NEEDS HUMAN REVIEW** — The verdict cannot be determined statically; human judgment required.

### When to Use "NEEDS HUMAN REVIEW"

Use this verdict if any of the following apply:
- Data flow is partially traceable but requires runtime/deployment context
- A protection exists but its effectiveness is ambiguous
- Confidence score falls between 35–60% after rubric scoring
- The vulnerability is high-severity but context suggests possible mitigation that
  cannot be verified statically
- Flagged code is generated/templated and the generator logic is outside the scan scope
- Framework behavior is unclear or depends on version-specific features

## Triage Workflow

1. **Process context**: If the user provides repository context, determine if it's a
   file path or inline text. If a file path, read it. Store context for all analyses.

2. **Read the results file** and parse all findings from the `results` array in the
   Semgrep JSON.

3. **Sort findings** by severity: ERROR (High) → WARNING (Medium) → INFO (Low).

4. **Apply limit**: If the user specified a limit, analyze only the first N findings
   from the sorted list.

5. **For each finding**:
   a. Note the file path, line number, rule ID, and vulnerability description.
   b. Read `.github/vulnerability-guidance.json` and look up the vulnerability type.
   c. Read the flagged source file and examine ±25 lines around the flagged line.
   d. Trace the data flow: does user-controlled input reach the vulnerable sink
      without adequate sanitization?
   e. Check against the guidance JSON's false positive patterns.
   f. Consider encoding, validation, parameterized queries, or framework protections.
   g. Factor in the repository context.
   h. Assign a verdict and calculate confidence percentage using the rubric.

6. **Write the triage report** using the format below.

## Output Format

Write the report to `triage-reports/{repository-name}/triage-report-{ISO-8601-datetime}.md`.
Create the directory if it doesn't exist.

Use this exact format:

# SAST Triage Report

**Date:** [current date]
**Tool:** Semgrep
**Repository:** [repository path]
**Total Findings in Scan:** N | **Findings Analyzed:** M | **Limit:** [all or number]
**True Positives:** X | **False Positives:** Y | **Needs Human Review:** Z

---

## Executive Summary
[2-3 sentence summary of scan findings, key risk areas, and recommended priorities]

---

## Summary Table

| # | File | Vulnerability Type | Severity | Verdict | Confidence |
|---|------|--------------------|----------|---------|------------|
| 1 | path/to/File.java:42 | SQL Injection | High | ✅ TRUE POSITIVE | 87% |
| 2 | path/to/File.java:55 | XSS | Medium | ❌ FALSE POSITIVE | 12% |

---

## Detailed Findings

### Finding #1: [Vulnerability Type] — [Severity]
- **File:** `path/to/File.java:42`
- **Rule:** `semgrep-rule-id`
- **Verdict:** ✅ TRUE POSITIVE / ❌ FALSE POSITIVE / ⚠️ NEEDS HUMAN REVIEW
- **Confidence:** [percentage]
- **Explanation:** [2-4 sentences with code evidence and guidance indicators]
- **Action:** [Remediation if TP, rationale if FP, questions to resolve if NEEDS REVIEW]

[repeat for each finding]
```

### 4.3 Character Count Estimate

The agent file above is approximately **4,500 characters** — well within the 30,000 character limit. This leaves ~25,000 characters of headroom for future enhancements (additional vulnerability types, more detailed examples, edge case handling).

---

## 5. Vulnerability Guidance File

The existing `.claude/vulnerability-guidance.json` is copied verbatim to `.github/vulnerability-guidance.json`. No modifications needed — the content is model-agnostic.

The agent reads this file at runtime for each finding to look up per-vulnerability-type true positive indicators and false positive patterns.

**Source:** `.claude/vulnerability-guidance.json` (150 lines, ~4KB)
**Destination:** `.github/vulnerability-guidance.json`

---

## 6. Three Usage Modes

### 6.1 Mode 1: Copilot CLI (Interactive)

The analyst runs Copilot CLI locally with the agent:

```bash
# Basic triage — all findings
copilot "@sast-triage analyze sast-results.json for src/"

# With context and limit
copilot "@sast-triage triage sast-results.json for BenchmarkJava-master/ context: 'Java EE web app, OWASP Benchmark, externally accessible on port 8443' limit: 20"

# Autopilot mode — no approval prompts
copilot --autopilot "@sast-triage analyze sast-results.json for src/ limit: 50"
```

**Behavior:** The agent reads files locally, produces the report, and writes it to `triage-reports/`. The analyst sees progress in real-time and can approve/reject individual steps (unless in autopilot mode).

### 6.2 Mode 2: Cloud Delegation (Background)

Prefix with `&` to delegate to a cloud-hosted coding agent:

```bash
# Delegate to cloud — analyst continues working locally
& @sast-triage analyze sast-results.json for src/ context: "Production API" limit: 100
```

**Behavior:** Copilot spins up a GitHub Actions-powered environment (configured via `copilot-setup-steps.yml`), clones the repo, runs the triage, and opens a pull request with the report file. The analyst gets a notification when complete.

### 6.3 Mode 3: GitHub Issue Assignment (Fully Autonomous)

Create a GitHub Issue and assign it to `@copilot`:

```markdown
**Title:** Triage SAST scan results from latest Semgrep run

**Body:**
@sast-triage

Please triage the findings in `sast-results.json` against the source code in `src/`.

Context: This is a production Java Spring Boot API serving external traffic on port 443.
Limit: 50 highest-severity findings.

Save the report to `triage-reports/`.
```

**Behavior:** The Copilot coding agent picks up the issue, runs the triage in a cloud environment, opens a PR with the report, and links it to the original issue.

---

## 7. Cloud Agent Environment Setup

The cloud coding agent needs `jq` (for JSON parsing) available in its environment. Create `.github/workflows/copilot-setup-steps.yml`:

```yaml
name: "Copilot Setup Steps"

on: workflow_dispatch

jobs:
  copilot-setup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y -qq jq

      - name: Verify tools
        run: |
          jq --version
          echo "Environment ready for SAST triage agent"
```

> **Note:** This workflow runs automatically when the Copilot coding agent starts a cloud session. The `jq` tool helps the agent parse large Semgrep JSON files via shell commands if needed, though the agent can also parse JSON directly through its `read` tool.

---

## 8. Reusable Prompt Files

Create `.github/prompts/triage-batch.prompt.md` for standardized, repeatable invocations:

```markdown
---
description: "Run SAST triage on Semgrep scan results"
---

@sast-triage

Triage the SAST findings in `{{ results_file }}` against the source code in `{{ repo_path }}`.

**Repository context:** {{ context }}

**Limit:** Analyze the {{ limit }} highest-severity findings.

Sort findings by severity (High → Medium → Low) and save the report to
`triage-reports/{{ repo_name }}/`.
```

Usage from Copilot CLI:

```bash
copilot --prompt .github/prompts/triage-batch.prompt.md
```

---

## 9. Model Selection Guide

Copilot allows users to select their backing model via the model picker. Recommendations for SAST triage:

| Model | Accuracy | Cost | Speed | Best For |
|-------|----------|------|-------|----------|
| **Claude Sonnet 4.6** | Highest | Moderate | Moderate | Production triage where accuracy is critical |
| **Claude Opus 4.6** | Highest | High | Slow | Complex cross-file vulnerability chains |
| **GPT-4.1** | High | Low | Fast | Cost-sensitive clients, large batch triage |
| **GPT-5.3-Codex** | High | Moderate | Moderate | Clients on ChatGPT Pro subscriptions |
| **Gemini 3 Pro** | High | Low | Fast | Google-ecosystem clients |
| **Claude Haiku 4.5** | Moderate | Very Low | Very Fast | Quick first-pass triage, initial screening |

**Default recommendation:** **Claude Sonnet 4.6** for the best accuracy-to-cost ratio on security analysis tasks. Fall back to **GPT-4.1** for budget-constrained clients.

### How to Set the Model

- **Copilot CLI:** Use the model picker in the TUI, or set via config
- **Cloud agent:** Configure in repository settings under Copilot → Model preferences
- **Per-agent default:** Add `model: claude-sonnet-4-6` to the agent's YAML frontmatter (IDE environments only)

---

## 10. Implementation Phases

### Phase 1: Core Agent + Local Testing (~3-5 days)

| Task | Description |
|------|-------------|
| Create `sast-triage.agent.md` | Adapt methodology from Claude Code plugin files into Copilot agent format |
| Copy `vulnerability-guidance.json` | Place at `.github/vulnerability-guidance.json` |
| Create `copilot-instructions.md` | Write repository-level context instructions |
| Local CLI testing | Test agent against `sast-results.json` + `BenchmarkJava-master/` using Copilot CLI |
| Report validation | Compare output format and verdicts against Claude Code plugin baseline |
| Iterate on prompt | Refine agent instructions based on output quality |

**Deliverable:** Working agent that produces accurate triage reports via Copilot CLI.

### Phase 2: Cloud Delegation + CI/CD (~3-5 days)

| Task | Description |
|------|-------------|
| Create `copilot-setup-steps.yml` | Cloud agent environment setup |
| Test cloud delegation | Verify `&` prefix triggers cloud triage and opens PR |
| Test Issue assignment | Create test issue, assign to `@copilot`, verify PR output |
| Create reusable prompt | `.github/prompts/triage-batch.prompt.md` |
| Multi-model testing | Run same 20 findings through Claude Sonnet, GPT-4.1, and Gemini 3 Pro; compare verdicts |
| Document model recommendations | Update model selection guide with empirical accuracy data |

**Deliverable:** All three usage modes working end-to-end with validated model recommendations.

### Phase 3: Client Deployment Kit (~2-3 days)

| Task | Description |
|------|-------------|
| Template repository | Create a template repo that clients can fork or copy files from |
| README documentation | Setup guide, usage examples for all three modes, model selection, troubleshooting |
| Client checklist | Step-by-step adoption guide (see Section 14) |
| Internal demo | Walkthrough for the team showing all modes and model options |

**Deliverable:** Client-ready deployment package.

---

## 11. Advantages Over Standalone Python CLI

| Dimension | Standalone Python CLI | Copilot Custom Agent |
|---|---|---|
| **Code to maintain** | ~1,500 lines across 7 Python modules | 0 lines — markdown file only |
| **Dependencies** | Python 3.10+, openai/anthropic/genai SDKs, pyyaml, tenacity | None — Copilot handles everything |
| **Installation** | `pip install` or Docker | Copy 5 files into `.github/` |
| **API key management** | Client configures env vars per provider | Handled by Copilot subscription |
| **Model updates** | Update SDK, test compatibility | Automatic — Copilot updates models |
| **File exploration** | Static ±25 line window | Dynamic — agent can read additional files |
| **Security scanning** | SAST only (Semgrep) | SAST + secret scanning + dependency checks |
| **CI/CD integration** | Custom pipeline scripts | Native GitHub Actions |
| **Enterprise governance** | Build your own | SSO, audit logs, RBAC inherited |
| **Multi-environment** | Terminal only | CLI, VS Code, IntelliJ, GitHub web |
| **Concurrency** | Custom asyncio implementation | Cloud delegation with parallel agents |
| **Maintenance burden** | Ongoing: SDK updates, bug fixes, new features | Near-zero: update markdown as needed |

---

## 12. Limitations & Mitigations

| Limitation | Impact | Mitigation |
|---|---|---|
| **File-scoped exploration** | Copilot may not trace complex cross-file data flows as deeply as Claude Code | Agent instructions explicitly direct it to follow imports and read related files; use Claude Sonnet/Opus as backing model for best code reasoning |
| **30,000 char prompt limit** | Cannot inline the full vulnerability guidance JSON | Reference the file by path; agent reads it at runtime |
| **Custom agents are new** | Less mature than Claude Code's skill/command system; potential breaking changes | Pin agent behavior via clear, deterministic instructions; monitor GitHub changelog for updates |
| **Cloud agent cold start** | GitHub Actions environment setup adds ~30-60s latency | Acceptable for background delegation; use CLI mode for interactive speed |
| **No structured output enforcement** | Unlike Azure OpenAI's JSON mode, Copilot doesn't force structured responses | Agent instructions specify exact report format; self-review catches format deviations |
| **Copilot subscription required** | Not free — requires Copilot Business ($19/user/month) or Enterprise ($39/user/month) | Most target clients already have Copilot; cost is incremental to existing subscription |
| **Model availability varies by tier** | Some models (Opus) may require higher-tier subscriptions | Document tier requirements; recommend Sonnet as default (available on all paid tiers) |

---

## 13. Testing & Validation Strategy

### 13.1 Functional Testing

| Test | Method | Success Criteria |
|------|--------|------------------|
| **Agent loads correctly** | Invoke `@sast-triage` in Copilot CLI | Agent recognized, description displayed |
| **Guidance JSON read** | Triage a SQL injection finding | Agent references guidance JSON indicators in explanation |
| **Report format** | Run triage on 10 findings | Report matches expected markdown structure (header, summary table, detailed findings) |
| **Verdict accuracy** | Run against OWASP Benchmark (known ground truth) | ≥80% agreement with Claude Code plugin verdicts |
| **Confidence scoring** | Review 20 findings | Confidence percentages follow rubric (5-95% range, integer values) |
| **Limit enforcement** | Request limit of 5 on a 100-finding scan | Exactly 5 findings analyzed, sorted by severity |
| **Context processing** | Provide context as inline text AND as file path | Both modes produce context-aware verdicts |

### 13.2 Multi-Mode Testing

| Mode | Test | Success Criteria |
|------|------|------------------|
| **CLI interactive** | `copilot "@sast-triage analyze sast-results.json for BenchmarkJava-master/ limit: 10"` | Report written to `triage-reports/`, analyst sees progress |
| **CLI autopilot** | Same command with `--autopilot` | Runs without approval prompts, report produced |
| **Cloud delegation** | `& @sast-triage analyze sast-results.json for src/ limit: 10` | PR opened with report file within ~5 minutes |
| **Issue assignment** | Create issue, assign to `@copilot` | PR opened, linked to issue, report in PR |

### 13.3 Multi-Model Comparison

Run the same 20 findings through each supported model and compare:

| Metric | Target |
|--------|--------|
| Verdict agreement rate (vs Claude Code baseline) | ≥80% |
| Confidence score variance across models | ≤15% average deviation |
| Report format compliance | 100% (all models produce valid format) |
| Average time per finding (CLI mode) | <30 seconds |

---

## 14. Client Deployment Checklist

### Prerequisites

- [ ] GitHub repository with source code
- [ ] Copilot Business or Enterprise subscription
- [ ] Semgrep scan results in JSON format (`semgrep --json > sast-results.json`)

### Setup Steps

1. **Copy agent files** into the client's repository:
   ```bash
   mkdir -p .github/agents .github/prompts .github/workflows
   # Copy the 5 files from the deployment kit
   ```

2. **Customize `copilot-instructions.md`** with repository-specific context:
   - Application type (web API, microservice, monolith)
   - Tech stack (Spring Boot, Django, Express, etc.)
   - Deployment model (internal, external, cloud provider)
   - Known security controls (WAF, API gateway, auth framework)

3. **Generate Semgrep results** (if not already available):
   ```bash
   semgrep --config=p/java src/ --json > sast-results.json
   # Replace p/java with appropriate ruleset for your language
   ```

4. **Test locally with Copilot CLI**:
   ```bash
   copilot "@sast-triage analyze sast-results.json for src/ limit: 5"
   ```

5. **Review the generated report** in `triage-reports/` — verify verdicts and format.

6. **Select preferred model** via Copilot model picker (recommend Claude Sonnet 4.6).

7. **(Optional) Test cloud delegation**:
   ```bash
   & @sast-triage analyze sast-results.json for src/ limit: 20
   ```

8. **(Optional) Test Issue-based workflow** — create an issue with triage instructions and assign to `@copilot`.

### Post-Setup

- [ ] Share the `triage-reports/` directory structure with the security team
- [ ] Document the model choice and rationale for the team
- [ ] Set up a recurring workflow: scan → triage → review → remediate
- [ ] (Optional) Extend `vulnerability-guidance.json` with org-specific patterns

---

## 15. Cross-Reference: Standalone Python CLI vs. Copilot Agent

For clients who cannot use Copilot (no GitHub, self-hosted Git, air-gapped environments), refer to the standalone Python CLI plan in [sast-triage-standalone-plan.md](sast-triage-standalone-plan.md). The two approaches are complementary:

| Scenario | Recommended Approach |
|---|---|
| Client on GitHub with Copilot subscription | **Copilot Custom Agent** (this plan) |
| Client on GitHub without Copilot | Standalone Python CLI |
| Client on GitLab / Bitbucket / self-hosted | Standalone Python CLI |
| Air-gapped / no cloud access | Standalone Python CLI with local LLM |
| Client wants CI/CD pipeline integration on GitHub | Copilot cloud delegation OR Python CLI in GitHub Actions |
| Client needs maximum triage accuracy | Claude Code plugin (original) |

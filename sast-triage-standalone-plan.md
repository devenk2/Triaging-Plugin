# Standalone SAST Triage Tool — Design & Implementation Plan

## 1. Problem Statement

Our Claude Code SAST triage plugin accelerates vulnerability triage by analyzing Semgrep findings against source code and classifying them as true positives, false positives, or needing human review. However, many clients do not have access to Claude Code. We need a **standalone, LLM-agnostic Python CLI tool** that delivers the same triage capability using Azure OpenAI, Google Gemini, or Anthropic Claude APIs — selectable at runtime.

---

## 2. Why Claude Code Has a Structural Advantage for This Use Case

Before discussing the standalone tool, it's important to understand what Claude Code provides out of the box — and what gaps the standalone tool must fill when using other providers.

### 2.1 Native Tool Use and Agentic File Access

Claude Code operates as a full **agentic coding environment** — it doesn't just generate text, it reads files, writes files, runs shell commands, and navigates codebases autonomously. For SAST triage, this means:

- **Dynamic code exploration**: When analyzing a finding, Claude Code can autonomously read the flagged file, then follow imports, trace method calls across files, and inspect framework configurations — all without pre-scripting these steps. A standalone script must pre-extract code context (±25 lines) and hope it's sufficient.
- **Adaptive investigation depth**: If a finding is ambiguous, Claude Code can decide to read additional files (e.g., a custom validator class, a framework config, an upstream caller). The standalone tool gets one shot per finding — whatever context you put in the prompt is all the LLM sees.

### 2.2 Skills and Slash Commands (Zero-Setup Workflow)

The plugin's two-component design (auto-triggered skill + explicit slash command) means users get SAST triage without installing anything beyond Claude Code:

- `/triage-sast` provides a structured, repeatable interface
- The `sast-triage` skill auto-triggers on conversational phrases ("is this a false positive?")
- No API keys to configure, no Python environment to set up, no dependencies to install

The standalone tool requires Python 3.10+, SDK packages, API key configuration, and familiarity with CLI arguments — a higher barrier to entry.

### 2.3 MCP (Model Context Protocol) Integration

Claude Code connects to external systems via [MCP](https://modelcontextprotocol.io/) — databases, GitHub, Jira, Slack, Sentry, and custom internal APIs. For SAST triage, this enables workflows like:

- Automatically creating Jira tickets for true positive findings
- Posting triage summaries to a Slack channel
- Cross-referencing findings against an internal vulnerability database
- Querying deployment metadata to assess exposure

The standalone tool would need custom integrations for each of these, built from scratch per client.

### 2.4 Agent Teams and Sub-Agents

Claude Code's [Agent Teams](https://www.anthropic.com/news/claude-4) capability allows multiple specialized agents to coordinate on complex tasks. For a large scan (1000+ findings), you could theoretically have:

- A **triage agent** analyzing individual findings
- A **code exploration agent** investigating complex data flows
- A **report agent** synthesizing results

This multi-agent coordination is built into Claude Code. The standalone tool must implement its own concurrency model (asyncio) which is functional but lacks the dynamic task delegation.

### 2.5 Context Window and Model Quality

Claude Opus 4.6 offers a 1M-token context window (beta), which means for large codebases, the model can hold significantly more code context in a single session. Combined with Anthropic's strong performance on code reasoning benchmarks, this gives Claude Code an accuracy edge — particularly for complex data flow tracing across multiple files.

Other models (GPT-4o, Gemini 2.5 Pro) are capable but may require more aggressive context windowing and prompt engineering to achieve comparable accuracy on nuanced triage decisions.

### 2.6 Summary: What the Standalone Tool Must Compensate For

| Claude Code Advantage | Standalone Tool Compensation |
|---|---|
| Dynamic file access (read any file on demand) | Pre-extract ±25 lines of context; accept reduced trace depth |
| Adaptive investigation (follow imports, trace calls) | Larger context windows; include import chains in prompt |
| Zero-setup (skill/command system) | CLI with clear docs; Docker for easy distribution |
| MCP integrations (Jira, Slack, GitHub) | Custom post-triage scripts or webhook integrations |
| Agent Teams (multi-agent coordination) | asyncio concurrency; sequential pipeline |
| 1M-token context window | Chunk findings; summarize large files |

---

## 3. Improving Implementations with Provider-Specific Tooling

While the standalone Python CLI is the most portable approach, clients already invested in specific AI ecosystems can enhance the triage experience using tools native to their stack.

### 3.1 OpenAI Ecosystem: Codex

[OpenAI Codex](https://openai.com/codex/) is a coding agent that runs locally from your terminal or in cloud sandboxes. It provides capabilities that closely mirror what Claude Code offers for this use case:

**How Codex could enhance the standalone tool:**

- **Cloud sandbox execution**: Codex runs each task in an [isolated, network-disabled container](https://developers.openai.com/codex/security/) preloaded with your repository. You could submit the entire triage job as a Codex task — it would autonomously read the Semgrep JSON, explore source files, and produce the report, similar to how the Claude Code plugin works.
- **Codex CLI (local mode)**: The [Codex CLI](https://developers.openai.com/codex/cli/) runs locally with approval controls (Suggest / Auto Edit / Full Auto). A client could use the standalone tool's prompts and methodology but execute them through Codex CLI for interactive, approval-gated triage.
- **Parallel task execution**: Codex supports running multiple tasks simultaneously in separate cloud sandboxes. For large scans, you could split findings into batches and triage them in parallel Codex tasks, then merge the reports.
- **Session resume**: Codex stores transcripts locally, allowing analysts to resume a triage session — useful for multi-day triage of large scans.
- **Non-interactive / CI mode**: Codex supports [`codex e`](https://developers.openai.com/codex/cli/reference/) for non-interactive execution with JSONL output, making it viable for CI/CD pipeline integration.

**Limitation**: Codex cloud sandboxes disable network access during execution, which prevents real-time integration with external services (Jira, Slack) during the triage run. Post-triage integrations would need to happen outside the sandbox.

**Recommended model**: `gpt-5.3-codex` for accuracy; `gpt-5-codex-mini` for cost-sensitive clients needing ~4x more throughput.

### 3.2 Google Ecosystem: Gemini CLI + Antigravity

Google offers two relevant tools: the [Gemini CLI](https://developers.google.com/gemini-code-assist/docs/gemini-cli) (open-source terminal agent) and [Antigravity](https://www.baytechconsulting.com/blog/google-antigravity-ai-ide-2026) (agent-first IDE).

**Gemini CLI:**

- **ReAct loop with file tools**: Gemini CLI uses a reason-and-act loop with built-in tools and MCP server support. This means it can dynamically read source files during triage, similar to Claude Code — not limited to pre-extracted context.
- **Free tier available**: 60 requests/min and 1,000 requests/day with a personal Google account, using Gemini 2.5 Pro/Flash models with a 1M-token context window. This makes it the most cost-effective option for small teams or proof-of-concept deployments.
- **MCP support**: Gemini CLI supports local and remote MCP servers, enabling the same kind of external integrations (GitHub, Jira) that Claude Code provides.
- **Agent mode**: Gemini Code Assist's [agent mode](https://developers.google.com/gemini-code-assist/docs/agent-mode) can plan and execute multi-step tasks with human approval gates — a client could frame SAST triage as an agent mode task in VS Code or IntelliJ.

**Antigravity (caution advised):**

- Google's agent-first IDE could theoretically run the triage workflow as an autonomous agent task. However, Antigravity is still in early preview with [documented security vulnerabilities](https://bdtechtalks.substack.com/p/antigravity-prompt-injection-vulnerability) — including prompt injection attacks that can exfiltrate source code and credentials.
- **Not recommended for security-sensitive work** like SAST triage until the platform matures. Source code is transmitted to the cloud, and the agent has been shown to take unsafe autonomous actions (e.g., `chmod -R 777` to resolve permission errors).
- For regulated industries (healthcare, finance), Antigravity should be treated as a public cloud environment and **should not be used for codebases containing PII or secrets**.

**Recommendation**: Use **Gemini CLI** for clients in the Google ecosystem. Avoid Antigravity for SAST triage until its security posture improves.

### 3.3 Microsoft Ecosystem: Azure AI Foundry

For enterprise clients already on Azure:

- [Azure AI Foundry](https://azure.microsoft.com/en-us/products/ai-foundry) provides a platform for building and deploying AI applications with enterprise-grade security — managed virtual networks, private endpoints, and RBAC with Microsoft Entra ID.
- **Agent Service**: Azure AI Foundry's Agent Service can be used to build a custom SAST triage agent that runs within the client's Azure tenant, keeping all data (source code, findings) within their security boundary. This addresses the data residency concerns that may prevent some clients from using cloud-hosted tools like Claude Code or Codex.
- **Model flexibility**: Foundry supports Azure OpenAI models (GPT-4o, GPT-4.1), open-source models (Llama, Mistral), and third-party models — clients can choose based on accuracy/cost/compliance requirements.
- **Integration with Azure DevOps**: Natural fit for CI/CD pipelines already in Azure DevOps; the triage tool can be invoked as a pipeline task with results published as pipeline artifacts.

**Recommendation**: Use **Azure AI Foundry** for clients who need data residency guarantees and Azure DevOps integration.

### 3.4 Provider Comparison Matrix

| Capability | Claude Code | OpenAI Codex | Gemini CLI | Azure AI Foundry |
|---|---|---|---|---|
| Agentic file access | Native | Cloud sandbox | ReAct loop | Custom agent |
| Dynamic code exploration | Full | Full (sandboxed) | Full | Build custom |
| Multi-agent coordination | Agent Teams | Parallel sandboxes | Agent mode | Agent Service |
| MCP / external integrations | Native MCP | Limited (no network in sandbox) | MCP support | Azure services |
| Data residency control | Local execution | Cloud (OpenAI-managed) | Cloud (Google) | Client's Azure tenant |
| CI/CD integration | CLI / hooks | `codex e` + Autofix | CLI | Azure DevOps native |
| Free tier | No | No (ChatGPT subscription) | Yes (1K req/day) | Pay-as-you-go |
| Security maturity | High | High (sandboxed) | Moderate | High (enterprise) |
| Best for | Teams using Claude | Teams using OpenAI | Cost-sensitive / Google shops | Regulated industries |

---

## 4. Goals

| Goal | Detail |
|------|--------|
| **Feature parity** | Same triage methodology, confidence rubric, verdict types, and report format as the Claude Code plugin |
| **Multi-LLM support** | Azure OpenAI (GPT-4o, GPT-4.1), Google Gemini (2.0 Flash/Pro), Anthropic Claude — configurable via CLI flag or env var |
| **Zero IDE dependency** | Runs as a standalone CLI; no VS Code or Claude Code required |
| **CI/CD friendly** | Can be invoked in pipelines (GitHub Actions, Azure DevOps, Jenkins) with exit codes based on finding severity |
| **Distributable** | Installable via `pip install` or runnable as a single Docker container |

---

## 5. Architecture

### 5.1 High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     CLI Entry Point                      │
│                      (triage.py)                         │
│  Parses args, orchestrates workflow, writes report       │
└──────────┬──────────────────────────────────┬────────────┘
           │                                  │
           ▼                                  ▼
┌─────────────────────┐          ┌─────────────────────────┐
│   Findings Parser   │          │   Source Code Reader     │
│  (parser.py)        │          │  (code_reader.py)        │
│                     │          │                          │
│  - Parse Semgrep    │          │  - Read flagged file     │
│    JSON format      │          │  - Extract ±25 lines     │
│  - Sort by severity │          │  - Return code snippet   │
│  - Apply limit      │          │                          │
└─────────┬───────────┘          └────────────┬────────────┘
          │                                   │
          ▼                                   ▼
┌─────────────────────────────────────────────────────────┐
│                   Triage Engine                          │
│                 (triage_engine.py)                        │
│                                                          │
│  For each finding:                                       │
│  1. Look up vuln type in guidance JSON                   │
│  2. Build prompt (system + finding + code context)       │
│  3. Call LLM via abstraction layer                       │
│  4. Parse structured response (verdict, confidence, etc) │
│  5. Collect results                                      │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 LLM Client Abstraction                   │
│                   (llm_client.py)                         │
│                                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │ Azure OpenAI │ │   Gemini     │ │  Anthropic Claude│ │
│  │   Backend    │ │   Backend    │ │    Backend       │ │
│  └──────────────┘ └──────────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Report Generator                        │
│                (report_generator.py)                      │
│                                                          │
│  - Build summary table                                   │
│  - Build detailed findings sections                      │
│  - Write timestamped markdown file                       │
│  - Optionally output JSON for CI/CD integration          │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Project Structure

```
sast-triage-tool/
├── pyproject.toml                 # Package config, dependencies, entry point
├── Dockerfile                     # Container distribution
├── README.md                      # Usage documentation
├── config/
│   └── vulnerability-guidance.json  # Reused from existing plugin (unchanged)
├── src/
│   └── sast_triage/
│       ├── __init__.py
│       ├── cli.py                 # CLI argument parsing (argparse)
│       ├── parser.py              # Semgrep JSON parsing + severity sorting
│       ├── code_reader.py         # Source file reading + context extraction
│       ├── triage_engine.py       # Orchestrates per-finding analysis
│       ├── llm_client.py          # LLM abstraction (Azure OpenAI / Gemini / Claude)
│       ├── prompts.py             # System prompt + per-finding prompt templates
│       └── report_generator.py    # Markdown + JSON report output
└── tests/
    ├── test_parser.py
    ├── test_code_reader.py
    ├── test_triage_engine.py
    ├── test_report_generator.py
    └── fixtures/
        ├── sample_semgrep_results.json
        └── sample_source_files/
```

---

## 6. Component Details

### 6.1 CLI Interface (`cli.py`)

```
Usage:
  sast-triage <results-file> <repo-path> [options]

Positional Arguments:
  results-file          Path to Semgrep JSON results file
  repo-path             Path to the scanned source code repository

Options:
  --provider            LLM provider: azure-openai | gemini | claude  (default: azure-openai)
  --model               Model name override (e.g., gpt-4o, gemini-2.0-flash, claude-sonnet-4-6)
  --context             Repository context string OR path to a .txt/.md file
  --limit N             Max number of findings to analyze
  --output-dir          Output directory for report (default: ./triage-reports/<repo-name>/)
  --output-format       Output format: markdown | json | both  (default: markdown)
  --concurrency N       Number of parallel LLM calls (default: 5)
  --verbose             Enable detailed logging
  --config              Path to config file (YAML) for defaults and API keys
```

**Example invocations:**

```bash
# Azure OpenAI
sast-triage sast-results.json ./myapp --provider azure-openai --context "Production Java API" --limit 50

# Gemini
sast-triage sast-results.json ./myapp --provider gemini --model gemini-2.0-pro --context context.md

# Claude
sast-triage sast-results.json ./myapp --provider claude --limit 20

# In CI/CD (JSON output for downstream processing)
sast-triage sast-results.json ./src --provider azure-openai --output-format json --limit 100
```

### 6.2 Findings Parser (`parser.py`)

Responsibilities:
- Load and validate the Semgrep JSON structure (expects a `results` array)
- Normalize each finding into an internal dataclass:

```python
@dataclass
class Finding:
    id: int
    rule_id: str
    file_path: str
    line_start: int
    line_end: int
    severity: str          # ERROR (Critical/High), WARNING (Medium), INFO (Low)
    message: str
    snippet: str           # Semgrep-provided code snippet
    metadata: dict         # Extra metadata from Semgrep (CWE, OWASP, etc.)
```

- Map Semgrep severity levels: `ERROR` → High, `WARNING` → Medium, `INFO` → Low
- Sort findings: High → Medium → Low
- Apply `--limit` truncation after sorting

### 6.3 Source Code Reader (`code_reader.py`)

Responsibilities:
- Given a file path and line number, read the source file from the repository
- Extract a context window of ±25 lines around the flagged line
- Return the snippet with line numbers for prompt inclusion
- Handle missing files gracefully (log warning, skip finding)

### 6.4 LLM Client Abstraction (`llm_client.py`)

A unified interface that delegates to provider-specific implementations:

```python
class LLMClient(Protocol):
    def analyze(self, system_prompt: str, user_prompt: str) -> str:
        """Send a prompt and return the LLM response text."""
        ...
```

**Provider implementations:**

| Provider | SDK | Auth Mechanism |
|----------|-----|----------------|
| Azure OpenAI | `openai` (v1+) | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_ENDPOINT` + `AZURE_OPENAI_DEPLOYMENT` env vars |
| Google Gemini | `google-genai` | `GOOGLE_API_KEY` env var |
| Anthropic Claude | `anthropic` | `ANTHROPIC_API_KEY` env var |

**Key design decisions:**
- Use **structured output / JSON mode** where supported (Azure OpenAI, Gemini) to get reliable parsed responses. For Claude, use a prompt-enforced JSON block.
- Implement **retry with exponential backoff** for rate limits (429) and transient errors.
- Support **async concurrency** via `asyncio.gather()` to parallelize per-finding LLM calls (configurable via `--concurrency`).

### 6.5 Prompt Engineering (`prompts.py`)

Two prompt components, extracted and adapted from the existing Claude Code plugin:

#### System Prompt (constant per run)

Derived from [SKILL.md](.claude/skills/sast-triage/SKILL.md). Contains:
- Role definition ("expert application security analyst")
- The full confidence scoring rubric (points-based, 5%–95% range)
- Verdict definitions (TRUE POSITIVE, FALSE POSITIVE, NEEDS HUMAN REVIEW)
- "NEEDS HUMAN REVIEW" criteria
- Required output schema (JSON):

```json
{
  "verdict": "TRUE POSITIVE | FALSE POSITIVE | NEEDS HUMAN REVIEW",
  "confidence": 87,
  "explanation": "...",
  "action": "..."
}
```

#### Per-Finding User Prompt (varies per finding)

Constructed from [triage-sast.md](.claude/commands/triage-sast.md) methodology:

```
## Vulnerability Guidance for "{vuln_type}"
True positive indicators: [from guidance JSON]
False positive patterns: [from guidance JSON]

## Repository Context
{context_string}

## Finding
- Rule: {rule_id}
- Severity: {severity}
- File: {file_path}:{line}
- Message: {message}

## Source Code (lines {start}-{end})
```java
{code_with_line_numbers}
```

Analyze this finding. Respond with JSON only.
```

### 6.6 Triage Engine (`triage_engine.py`)

Orchestrates the full workflow:

```
1. Load vulnerability guidance JSON
2. Parse Semgrep results → sorted Finding list
3. Process repo context (inline string or file read)
4. For each finding (with concurrency):
   a. Read source code context via code_reader
   b. Look up vuln type in guidance JSON
   c. Build per-finding prompt
   d. Call LLM client
   e. Parse JSON response → TriageResult dataclass
   f. Handle parse failures (retry once with corrective prompt, else mark as NEEDS HUMAN REVIEW)
5. Collect all TriageResults
6. Pass to report generator
```

```python
@dataclass
class TriageResult:
    finding: Finding
    verdict: str           # "TRUE POSITIVE" | "FALSE POSITIVE" | "NEEDS HUMAN REVIEW"
    confidence: int        # 5-95
    explanation: str
    action: str
```

### 6.7 Report Generator (`report_generator.py`)

Produces output matching the existing plugin format:

**Markdown output** — Same structure as current plugin:
- Header with metadata (date, tool, repo, counts)
- Executive summary (generated from aggregate stats, not LLM-generated)
- Summary table (file, vuln type, severity, verdict, confidence %)
- Detailed findings sections
- Timestamped filename: `triage-report-{ISO8601}.md`
- Placed in `triage-reports/{repo-name}/`

**JSON output** (new, for CI/CD):
```json
{
  "metadata": { "date": "...", "repo": "...", "total": 100, "analyzed": 50 },
  "summary": { "true_positives": 30, "false_positives": 15, "needs_review": 5 },
  "findings": [
    {
      "id": 1, "rule_id": "...", "file": "...", "line": 42,
      "severity": "High", "verdict": "TRUE POSITIVE",
      "confidence": 87, "explanation": "...", "action": "..."
    }
  ]
}
```

**CI/CD exit codes:**
- `0` — No true positives found
- `1` — True positives found (fail the pipeline)
- `2` — Tool error

---

## 7. Reused Assets (No Changes Needed)

These files from the existing plugin are model-agnostic and can be copied directly:

| Asset | Source | Usage |
|-------|--------|-------|
| Vulnerability guidance | `.claude/vulnerability-guidance.json` | Lookup table for per-vuln-type TP/FP indicators |
| Triage methodology | `.claude/commands/triage-sast.md` | Adapted into system prompt |
| Confidence rubric | `.claude/skills/sast-triage/SKILL.md` | Embedded in system prompt |
| Report format | `.claude/commands/triage-sast.md` | Template for report generator |

---

## 8. Configuration

Support a `sast-triage.yaml` config file for defaults (overridden by CLI flags):

```yaml
provider: azure-openai
model: gpt-4o

azure_openai:
  endpoint: https://myorg.openai.azure.com/
  deployment: gpt-4o
  api_version: "2024-12-01-preview"

gemini:
  model: gemini-2.0-flash

claude:
  model: claude-sonnet-4-6

concurrency: 5
default_context: "Production Java web application"
output_format: markdown
```

API keys are **never** stored in the config file — always sourced from environment variables.

---

## 9. Dependencies

```
# Core
openai>=1.0              # Azure OpenAI SDK
google-genai>=1.0         # Google Gemini SDK
anthropic>=0.40           # Anthropic Claude SDK

# Utilities
pyyaml                   # Config file parsing
tenacity                 # Retry logic with backoff
rich                     # CLI progress bars and formatting (optional)
```

Python version: **3.10+** (for dataclasses, match statements, modern typing)

---

## 10. Implementation Phases

### Phase 1: Core Pipeline (MVP) — ~1-2 sprints

| Task | Description |
|------|-------------|
| Project scaffolding | Set up `pyproject.toml`, directory structure, dev tooling (ruff, pytest) |
| `parser.py` | Semgrep JSON parsing, severity sorting, limit |
| `code_reader.py` | File reading with ±25 line context extraction |
| `prompts.py` | System prompt + per-finding prompt templates from existing plugin |
| `llm_client.py` — Azure OpenAI only | Single provider implementation with structured output |
| `triage_engine.py` | Synchronous per-finding loop (no concurrency yet) |
| `report_generator.py` | Markdown output matching existing format |
| `cli.py` | Basic CLI with argparse |
| Unit tests | Parser, code reader, report generator |
| Manual validation | Run against `BenchmarkJava-master/` with existing `sast-results.json`, compare output to Claude Code plugin results |

**Deliverable:** Working CLI that triages Semgrep results using Azure OpenAI and outputs a markdown report.

### Phase 2: Multi-LLM + Concurrency — ~1 sprint

| Task | Description |
|------|-------------|
| `llm_client.py` — Gemini backend | Add Google Gemini provider |
| `llm_client.py` — Claude backend | Add Anthropic Claude provider |
| Async concurrency | `asyncio` parallel LLM calls with configurable concurrency |
| Retry logic | Exponential backoff for rate limits and transient errors |
| Config file support | YAML config loading |
| Integration tests | Test each provider against sample findings |

**Deliverable:** Tool supports all three LLM providers with parallel processing.

### Phase 3: Distribution + CI/CD — ~1 sprint

| Task | Description |
|------|-------------|
| JSON output format | Machine-readable output for pipeline integration |
| Exit codes | `0` / `1` / `2` based on findings |
| `Dockerfile` | Containerized distribution |
| PyPI packaging | `pip install sast-triage` |
| GitHub Actions example | Sample workflow YAML for CI integration |
| Documentation | README with setup, usage, examples per provider |

**Deliverable:** Production-ready tool installable via pip or Docker, with CI/CD integration examples.

### Phase 4: Enhancements (Future) — Backlog

| Task | Description |
|------|-------------|
| Additional SAST formats | Support Checkmarx, Fortify, SonarQube JSON/XML export |
| Caching | Cache LLM responses by finding hash to avoid re-analyzing unchanged findings |
| Incremental triage | Compare against previous report, only triage new/changed findings |
| Web UI | Simple Flask/FastAPI dashboard for viewing and filtering reports |
| Cost tracking | Log token usage and estimated cost per run |
| Custom guidance | Let users extend `vulnerability-guidance.json` with org-specific patterns |

---

## 11. Testing Strategy

| Layer | Approach |
|-------|----------|
| **Unit tests** | Parser, code reader, report generator — no LLM calls, use fixtures |
| **Prompt tests** | Validate prompt construction against expected templates |
| **Integration tests** | Mock LLM responses, verify end-to-end pipeline produces correct report |
| **Accuracy validation** | Run against OWASP Benchmark (known ground truth) with each provider, compare TP/FP rates to the Claude Code plugin baseline |
| **Provider parity** | Same 20 findings through all 3 providers — compare verdict agreement rate |

---

## 12. Risk & Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| LLM response format inconsistency | Parsing failures, missing verdicts | Enforce JSON schema in prompt; retry with corrective prompt on parse failure; fallback to NEEDS HUMAN REVIEW |
| Rate limiting on large scans | Slow or failed runs | Configurable concurrency; exponential backoff via `tenacity`; `--limit` flag |
| Model quality variance across providers | Different TP/FP accuracy | Validate against OWASP Benchmark ground truth; document per-provider accuracy; recommend minimum model tier |
| Cost on large scans (1000+ findings) | Unexpectedly high API bills | `--limit` flag; cost estimation before run (`--dry-run`); log token usage |
| API key management in CI/CD | Secret exposure | Document env var approach; never store keys in config files; support Azure Managed Identity |

---

## 13. Cost Estimates (Per Run)

Rough estimates for triaging 100 findings (~800 tokens input + ~200 tokens output per finding):

| Provider | Model | Est. Cost (100 findings) |
|----------|-------|--------------------------|
| Azure OpenAI | GPT-4o | ~$0.50–$1.00 |
| Azure OpenAI | GPT-4.1 | ~$0.40–$0.80 |
| Google Gemini | 2.0 Flash | ~$0.05–$0.10 |
| Google Gemini | 2.0 Pro | ~$0.25–$0.50 |
| Anthropic | Claude Sonnet 4.6 | ~$0.60–$1.20 |

*Costs vary by prompt length, code context size, and response verbosity. Recommend Gemini 2.0 Flash for cost-sensitive clients; GPT-4o or Claude Sonnet for accuracy-sensitive clients.*

---

## 14. Success Criteria

1. **Feature parity** — Produces reports indistinguishable in structure from the Claude Code plugin
2. **Accuracy** — ≥85% verdict agreement with Claude Code plugin on OWASP Benchmark test set
3. **Performance** — Triages 100 findings in <5 minutes with concurrency enabled
4. **Portability** — Runs on macOS, Linux, Windows; Python 3.10+
5. **Adoption** — Client can go from `pip install` to first report in <10 minutes

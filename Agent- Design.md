# GitHub Secrets Scanner Agent — Architecture & Design

> **Viewing Mermaid diagrams**
> - **VS Code**: install [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid), then open Preview (`Ctrl+Shift+V`)
> - **GitHub**: renders automatically in `.md` files
> - **Online**: paste any diagram block at [mermaid.live](https://mermaid.live)

---

## 1. System Overview

```mermaid
flowchart TD
    DEV["👤 Developer\nprovides: repo name + scan mode\n(full repo scan OR specific PR number)"]

    subgraph BACKEND["Python Backend"]
        direction TB
        GRAPH["LangGraph StateGraph\n4 nodes"]
        LLM["LLM Service\nClaude · GPT · Groq"]
        AUTH["GitHub App Auth\nJWT → Installation Token"]
    end

    subgraph MCP["GitHub MCP Server (Docker · stdio transport)"]
        DOCKER["ghcr.io/github/github-mcp-server\nToolsets: repos, pull_requests"]
    end

    subgraph GITHUB["GitHub (READ-ONLY)"]
        REPO["Repository\n• File tree\n• File contents\n• PR diff"]
    end

    subgraph OUTPUT["Output"]
        CHAT["Chat UI\nStructured findings report\nper finding: file · line · severity · action"]
    end

    DEV -->|"repo + mode + optional branch/PR number"| GRAPH
    GRAPH -->|"structured output"| LLM
    GRAPH -->|"token request"| AUTH
    AUTH -->|"GITHUB_PAT (installation token)"| DOCKER
    GRAPH -->|"stdio subprocess"| DOCKER
    DOCKER -->|"GitHub API (read-only)"| REPO
    REPO -->|"file contents / diff"| DOCKER
    DOCKER -->|"file_entries"| GRAPH
    GRAPH -->|"findings report"| CHAT
```

**What each layer does:**

| Layer | Technology | Role |
|-------|-----------|------|
| Chat UI | IE Platform MessageList | Developer triggers agent, receives findings report |
| API entry point | FastAPI `POST /api/agent/secrets-scanner` | Receives request, starts LangGraph run, streams progress via WebSocket |
| Agent orchestration | LangGraph `StateGraph` | Wires 4 nodes, manages state, handles conditional routing |
| LLM | Claude / GPT / Groq (via `LLM_PROVIDER`) | Confirmation pass — removes false positives, assigns severity |
| GitHub integration | GitHub MCP Server (Docker, stdio) | Reads file tree, file contents, PR diff — no write operations |
| Auth | GitHub App (`PyJWT` → installation token) | Generates short-lived tokens from PEM private key |
| Structured output | Pydantic (`SecretFinding`, `ScanReport`) | Enforces severity + secret_type before description is written |

> ⚠️ **Read-only constraint:** This agent performs zero write operations to GitHub. It cannot create PRs, post comments, or modify any file. It reads and reports only.

---

## 2. LangGraph Graph — Exact Node Order

```mermaid
flowchart TD
    START(["START"]) --> N0

    N0["Node 0 · validate_inputs\nCheck repo format (owner/repo)\nCheck scan_mode is 'repo' or 'pr'\nIf mode=pr, check pr_number provided\nDefault branch to main\nGenerate conversation_id"]

    N0 -->|"error"| END1(["END"])
    N0 -->|"ok"| N1

    N1["Node 1 · fetch_code_from_github\nMode: repo → fetch file tree\n  then read high-risk files only\n  (.env*, config*, *.yaml, *.json, *.toml, source files)\nMode: pr → fetch changed file list\n  then read full contents of each changed file\nFilter: skip binary, images, lock files, files > 500KB\nNormalise into file_entries list"]

    N1 -->|"error (repo not found / auth fail / rate limit)"| END2(["END"])
    N1 -->|"no scannable files"| END3(["END"])
    N1 -->|"file_entries ready"| N2

    N2["Node 2 · scan_for_secrets\nPass 1 — Regex: apply patterns across all file_entries\n  AWS keys, GitHub tokens, PEM keys, JWT secrets,\n  API key patterns, DB connection strings, config passwords\nRecord each match: file path, line number, pattern type,\n  redacted value (first 4 + **** + last 4)\nPass 2 — LLM confirmation: send all regex matches to LLM\n  LLM removes clear false positives\n  LLM assigns/confirms severity per finding\nOutputs confirmed_findings list"]

    N2 -->|"no findings"| N3_CLEAN

    N3_CLEAN["Node 3 · generate_findings_report\n(clean path)\nReturns: No hardcoded secrets detected\nin the scanned files."]

    N3_CLEAN --> END4(["END"])

    N2 -->|"findings confirmed"| N3

    N3["Node 3 · generate_findings_report\n(findings path)\nSorts findings: Critical first → High → Medium → Low\nBuilds summary header: total count + severity breakdown\nPer finding: file path · line number · secret type\n  severity · redacted value · recommended action\nReturns structured report to Chat UI"]

    N3 --> END5(["END"])
```

**Routing summary:**

| After node | Routes to |
|-----------|-----------|
| `validate_inputs` | `fetch_code_from_github` or END (on error) |
| `fetch_code_from_github` | `scan_for_secrets` or END (error / no scannable files) |
| `scan_for_secrets` | `generate_findings_report` (always — clean or findings) |
| `generate_findings_report` | END |

---

## 3. State Schema (`SecretsScannerState`)

```mermaid
classDiagram
    class SecretsScannerState {
        +conversation_id: str
        +repository: str
        +scan_mode: Literal[repo, pr]
        +branch: str
        +pr_number: int
        +file_entries: List~FileEntry~
        +regex_matches: List~RegexMatch~
        +confirmed_findings: List~SecretFinding~
        +report: ScanReport
        +error: str
        +response_type: Literal[findings, clean, error]
        +messages: List
    }

    class FileEntry {
        +path: str
        +content: str
        +source_mode: repo|pr
        +size_bytes: int
        +language: str
    }

    class RegexMatch {
        +file_path: str
        +line_number: int
        +pattern_type: str
        +matched_line: str
        +redacted_value: str
    }

    class SecretFinding {
        +file_path: str
        +line_number: int
        +secret_type: str
        +severity: critical|high|medium|low
        +redacted_value: str
        +context_snippet: str
        +recommended_action: str
        +reasoning: str
    }

    class ScanReport {
        +total_files_scanned: int
        +total_findings: int
        +severity_breakdown: Dict
        +readiness_verdict: clean|review_required
        +findings: List~SecretFinding~
        +summary_text: str
    }

    SecretsScannerState --> FileEntry
    SecretsScannerState --> RegexMatch
    SecretsScannerState --> SecretFinding
    SecretsScannerState --> ScanReport
```

---

## 4. Two-Mode Fetch Strategy

```mermaid
sequenceDiagram
    participant Node1 as fetch_code_from_github
    participant MCP as GitHub MCP (read-only)
    participant GH as GitHub API

    alt scan_mode = repo
        Node1->>MCP: get file tree (owner/repo, branch)
        MCP->>GH: GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1
        GH-->>MCP: full file tree
        MCP-->>Node1: list of all file paths

        loop for each high-risk file path
            Node1->>MCP: get file contents (path)
            MCP->>GH: GET /repos/{owner}/{repo}/contents/{path}
            GH-->>MCP: file content (base64)
            MCP-->>Node1: decoded content
        end

    else scan_mode = pr
        Node1->>MCP: list changed files (pr_number)
        MCP->>GH: GET /repos/{owner}/{repo}/pulls/{pr_number}/files
        GH-->>MCP: list of changed files + patch
        MCP-->>Node1: changed file list

        loop for each changed file
            Node1->>MCP: get file contents (path)
            MCP->>GH: GET /repos/{owner}/{repo}/contents/{path}
            GH-->>MCP: full file content
            MCP-->>Node1: decoded content
        end
    end

    Note over Node1: Filter out: binary, images, lock files, files > 500KB
    Note over Node1: Normalise all into file_entries list regardless of mode
```

**Why two modes, why normalise:**

| | `repo` mode | `pr` mode |
|-|------------|----------|
| When to use | Full audit of a codebase | Checking what just changed |
| What it fetches | File tree → filter to high-risk files | PR changed files list → all of them |
| GitHub API calls | 1 tree call + N file reads | 1 files call + N file reads |
| Misses | Changes not in high-risk file types | Secrets in files not touched by this PR |
| Catches | Anything that was always there | Anything introduced or modified in this PR |

After fetching, both modes produce the **same `file_entries` list**. The scan node never needs to know which mode was used — it always receives the same data structure.

---

## 5. Regex Pass + LLM Confirmation Pass

```mermaid
sequenceDiagram
    participant N2 as scan_for_secrets
    participant RE as Regex Engine (in-memory)
    participant LLM as LLM (Claude / GPT / Groq)

    N2->>RE: all file_entries content
    RE-->>N2: regex_matches (all pattern hits, unfiltered)

    Note over RE,N2: Zero API cost. Zero latency. Runs in memory.
    Note over RE,N2: Will over-flag — placeholder values included.

    N2->>LLM: regex_matches + file context + system prompt
    Note over N2,LLM: System prompt instructs LLM to:\n1. Remove clear false positives\n2. Assign confirmed severity\n3. Add reasoning

    LLM-->>N2: confirmed_findings (false positives removed, severity set)

    Note over N2: confirmed_findings written to state
```

**Regex patterns applied (Pass 1):**

| Secret Type | Pattern Logic | Example Match |
|------------|--------------|---------------|
| AWS Access Key | `AKIA` followed by 16 uppercase alphanumeric chars | `AKIAIOSFODNN7EXAMPLE` |
| AWS Secret Key | 40-char mixed alphanumeric after `aws_secret` keyword | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| GitHub PAT | `ghp_` or `github_pat_` prefix | `ghp_abc123...` |
| PEM Private Key | `-----BEGIN ... PRIVATE KEY-----` header | Full PEM block |
| JWT Secret | `jwt_secret`, `jwt_key` followed by a value | `jwt_secret = "s3cr3t!"` |
| Generic API Key | `api_key`, `apikey`, `api-key` followed by 20+ char value | `api_key = "sk-abc123..."` |
| DB Connection String | Protocol + credentials embedded in URL | `postgres://user:pass@host/db` |
| Config Password | `password =`, `passwd =`, `secret =` followed by value | `password = "hunter2"` |

**LLM confirmation pass (Pass 2) — what it removes:**

| False Positive Type | Example | LLM removes? |
|--------------------|---------|-------------|
| Placeholder values | `api_key = "your-api-key-here"` | ✅ Yes |
| Documentation examples | `password = "example_password"` in README | ✅ Yes |
| Test fixture values | `secret = "test123"` inside test file | ✅ Yes |
| Environment variable references | `password = os.getenv("DB_PASS")` | ✅ Yes |
| Clearly dummy data | `token = "REPLACE_WITH_YOUR_TOKEN"` | ✅ Yes |
| Real-looking credential | `password = "Tr0ub4dor&3"` in config | ❌ Keep — flag it |

---

## 6. Secret Types & Severity

**Severity thresholds:**

| Severity | Assigned when | Recommended action |
|----------|--------------|-------------------|
| `critical` | Private keys, auth tokens, certificates — immediate account takeover risk | Rotate immediately + scrub from git history |
| `high` | AWS credentials, database passwords, OAuth secrets — service-level compromise risk | Rotate immediately + remove from codebase |
| `medium` | Generic API keys, JWT secrets, third-party service keys | Move to environment variable or secrets manager |
| `low` | Config passwords with weak or obvious values, internal tool keys | Move to environment variable; review necessity |

**Readiness verdict (from `generate_findings_report`):**

| Verdict | Condition |
|---------|-----------|
| `✅ CLEAN` | 0 confirmed findings |
| `⚠️ REVIEW REQUIRED` | ≥ 1 finding of any severity |

> There is no "blocked" tier — this agent does not gate deployments. It reports. Action is taken by the developer.

---

## 7. Anti-False-Positive Rules

The LLM prompt enforces hard rules to prevent noise:

| Rule | What it eliminates |
|------|-------------------|
| Only flag lines with actual credential values, not variable declarations | `api_key: str` (type annotation only) flagged as secret |
| Placeholder detection: `your-`, `example-`, `REPLACE_`, `<...>` patterns → always remove | Documentation and template false positives |
| Environment variable references: `os.getenv(...)`, `process.env.X`, `${VAR}` → always remove | Config files that correctly use env vars |
| Test file heuristic: files in `tests/`, `test_`, `_test`, `spec/`, `__mocks__/` → reduce severity one tier | Test fixtures inflated to critical |
| `.env.example` files → reduce severity one tier, note it is a template | Template files flagged same as live secrets |
| Dead assignment (value assigned but never used in same file) → `low` not `high` | Unreachable code inflated to high |
| Entry-point/demo files with inline instructions → no flags | Tutorial code and quickstart files |

**Structured output enforces commitment:** LLM must fill `secret_type` and `severity` fields *before* writing `description` and `reasoning`. This prevents vague descriptions being stamped `critical` retroactively.

**Value redaction rule:** every finding redacts the matched value to first 4 chars + `****` + last 4 chars. Long enough for the developer to identify which credential it is. Short enough that the full secret never appears in chat logs or state traces.

---

## 8. What the Developer Sees

**Chat UI — live progress (WebSocket):**
```
●  Validating inputs...
●  Fetching code from GitHub · repo mode · branch: main
●  5 high-risk files identified

●  Scanning for secrets...
   Pass 1 — Regex: 4 potential matches found
   Pass 2 — LLM confirmation: 2 confirmed · 2 false positives removed

●  Generating report...
```

**Chat UI — findings report (example with findings):**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Secrets Scan Report · owner/repo · main
  2 findings  ·  1 Critical · 1 High · 0 Medium
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  🔴 CRITICAL · Private Key
  File:     config/deploy.pem  ·  Line 1
  Secret:   -----BEGIN RSA PR****KEY-----
  Action:   Rotate this credential immediately and remove
            from repository history (git filter-branch or BFG)

  🔴 HIGH · AWS Access Key
  File:     scripts/deploy.sh  ·  Line 14
  Secret:   AKIA****MPLE
  Action:   Rotate this credential immediately and remove
            from codebase. Verify no unauthorised AWS API
            calls have been made.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Chat UI — clean report:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Secrets Scan Report · owner/repo · PR #42
  ✅  No hardcoded secrets detected in the scanned files.
  (8 files scanned · 3 regex matches · all confirmed as
  false positives)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 9. File Structure

```
agents/GitHub-Secrets-Scanner/
├── backend/
│   ├── app/
│   │   ├── graph.py                     ← StateGraph: wires all nodes, run_scan()
│   │   ├── config.py                    ← Settings (LLM_PROVIDER, file size limits, etc.)
│   │   ├── logging.py                   ← Structured logging (never logs code content or tokens)
│   │   ├── main.py                      ← FastAPI app
│   │   ├── nodes/
│   │   │   ├── state.py                 ← SecretsScannerState, SecretFinding, FileEntry schemas
│   │   │   ├── validate_inputs.py       ← Node 0
│   │   │   ├── fetch_code_from_github.py← Node 1 (repo + pr modes)
│   │   │   ├── scan_for_secrets.py      ← Node 2 (regex pass + LLM confirmation)
│   │   │   └── generate_findings_report.py ← Node 3 (clean + findings paths)
│   │   ├── prompts/
│   │   │   └── scan_prompt.py           ← LLM confirmation system + user prompt builder
│   │   ├── services/
│   │   │   ├── llm.py                   ← LLM provider factory (Claude/GPT/Groq)
│   │   │   ├── regex_patterns.py        ← All regex pattern definitions (config, not hardcoded)
│   │   │   ├── mcp/client.py            ← MCP client (shared with Code Reviewer)
│   │   │   └── github/auth.py           ← GitHub App JWT + installation token (shared)
│   │   └── routers/scan.py              ← FastAPI /api/agent/secrets-scanner endpoint
│   └── start_backend.py                 ← FastAPI startup
├── SETUP.md
└── Docs/
    └── architecture.md                  ← This file
```

---

## 10. What This Agent Does NOT Do

These are explicit design constraints — not missing features, but intentional boundaries:

| Not in scope | Why |
|-------------|-----|
| Write to GitHub (comments, PRs, commits) | Read-only by design — agent observes, developer acts |
| Scan git history (commits before current state) | Separate security layer — access logs and git history audits are out of scope for v1 |
| Auto-rotate or revoke credentials | Too high risk for an automated agent to act on without human confirmation |
| Block deployments or enforce policy | This agent reports. CI/CD policy enforcement is a separate concern |
| Full repository content scan (all files) | Token cost control — high-risk file types only in repo mode |

---

*Last updated: 2026-03-03*

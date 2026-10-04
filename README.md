<!--
  SentrySleuth — Autonomous Incident Response Engine
  Generated for hackathon judging. Judges: you can run this project blindly by
  following the "Installation" and "Running" sections exactly.
-->

# 🛡️ SentrySleuth — Autonomous Incident Response Engine

> **Autonomous Incident Response · Dual-Agent Peer-Reviewed Remediation · Isolated Sandboxes · Automated Post-Mortems**

SentrySleuth is a self-healing engineering platform. Point it at a broken repository — as a
`.zip` upload or a Git URL — tell it where the bug is, and it will autonomously:

1. **Ingest** your code into an isolated, disposable sandbox workspace.
2. Run an **autonomous two-agent LLM loop** — a *Lead Investigator* (Agent A) that inspects
   the code and drafts a fix, and a *Senior QA Reviewer* (Agent B) that must approve it.
3. **Verify** the fix by executing **your own test suite** inside the sandbox.
4. Produce a **unified diff** of every change and a formal **enterprise Post-Mortem (RCA)** report.

Every step streams live to a dark, glassmorphic **Mission Control** dashboard over WebSockets.

---

## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [How It Works (Architecture)](#-how-it-works-architecture)
3. [Incident Lifecycle](#-incident-lifecycle)
4. [Feature Highlights](#-feature-highlights)
5. [Tech Stack & Dependencies](#-tech-stack--dependencies)
6. [Prerequisites](#-prerequisites)
7. [Repository Structure](#-repository-structure)
8. [Configuration (API Key & Provider)](#-configuration-api-key--provider)
9. [Installation](#-installation)
10. [Running the Application (Linux & macOS)](#-running-the-application-linux--macos)
11. [Triggering a Test Incident (Zipped Repo)](#-triggering-a-test-incident-zipped-repo)
12. [Private Repositories (Access Tokens)](#-private-repositories-access-tokens)
13. [Backend API Reference](#-backend-api-reference)
14. [WebSocket Telemetry Events](#-websocket-telemetry-events)
15. [Troubleshooting](#-troubleshooting)
16. [Security Notes](#-security-notes)

---

## 🔭 Project Overview

Modern incident response is slow: a crash fires, someone gets paged, they context-switch,
they bisect the code, they write a fix, they cross their fingers, and they ship. **SentrySleuth
automates that entire loop** and leaves an audit trail behind.

There are three pillars:

### 1. Autonomous Incident Response
You report an incident (file, line, error signature, and whether it is a *reliability defect*
or a *security vulnerability*). SentrySleuth dispatches an AI remediation loop that runs
unattended and reports back over a live telemetry channel.

### 2. Multi-Agent Peer-Reviewed Loop
A single LLM guessing at a fix is risky. SentrySleuth uses **two distinct agents with
adversarial roles**:

| Agent | Role | Responsibility |
| --- | --- | --- |
| **Agent A — Lead Investigator** | Propose | Inspects git blame + source, identifies the root cause, and drafts a full-file patch. It is **read-only** (`git_blame_inspector`, `source_reader`) and never writes to disk. |
| **Agent B — Senior QA Reviewer** | Approve | Receives Agent A's proposed patch **plus a generated diff** and reviews it for correctness, edge cases, regression risk, and security hygiene. It must end with `VERDICT: APPROVED` or `VERDICT: REJECTED`. |

If Agent B **rejects**, its structured critiques are fed back to Agent A, which re-patches.
If Agent B **approves**, *only then* is the patch written to disk. This loop runs for up to
`MAX_REVIEW_ROUNDS` (default **3**) rounds. If no passing patch is approved, the workspace is
**restored to its original state**.

### 3. Isolated Sandbox
Every incident gets its own throwaway workspace under `server/workspaces/<id>`. Zip uploads are
extracted with **zip-slip protection**; Git clones run with `GIT_TERMINAL_PROMPT=0` and a
disabled credential store. All file tooling is routed through a single path guard
(`resolveInsideWorkspace`) so an agent can never touch anything outside its sandbox.

---

## 🧠 How It Works (Architecture)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  BROWSER — React 19 "Mission Control" (Vite dev server)   http://localhost:5173│
│  • Incident configuration form   • Live telemetry terminal   • Unified diff    │
│  • Enterprise artifact downloads (patched .zip / RCA .html)                    │
└───────────────▲───────────────────────────────────────────┬───────────────────┘
                │ ws://localhost:3001/stream                 │ REST (HTTP/JSON)
                │ live telemetry (info/alert/investigator/   │ ingest · trigger
                │ reviewer/success/error)                    │ diff · rca · download
┌───────────────┴───────────────────────────────────────────▼───────────────────┐
│  FASTIFY BACKEND (Node.js / ESM)                          http://localhost:3001│
│                                                                               │
│  POST /ingest/zip   POST /ingest/git   POST /trigger                          │
│  GET  /diff/:id     GET  /rca/:id      GET  /download/:id                     │
│                                                                               │
│   ┌────────────────────────────┐   ┌──────────────────────────────────────┐   │
│   │  DUAL-AGENT ORCHESTRATOR   │   │  REPORTING                           │   │
│   │  (server/agent/index.js)   │   │  (server/reporting/)                 │   │
│   │                            │   │                                      │   │
│   │  Agent A ──draft──► Agent B│   │  gitDiff.js     → unified diff       │   │
│   │  (read-only)  ◄──critique─ │   │  rcaGenerator.js→ INCIDENT_REPORT    │   │
│   │        │  (up to 3 rounds) │   │                    .md + .html       │   │
│   │        ▼  APPROVED         │   └──────────────────────────────────────┘   │
│   └────────┬───────────────────┘                                              │
│            │ patch applied + tests executed                                    │
│   ┌────────▼───────────────────────────────────────────────────────────────┐   │
│   │  MCP TOOL SERVER (stdio) — server/index.js                             │   │
│   │  git_blame_inspector · source_reader · source_patcher                  │   │
│   │  sandbox_test_runner (Node) · python_test_runner (Python)              │   │
│   └────────┬───────────────────────────────────────────────────────────────┘   │
└────────────┼───────────────────────────────────────────────────────────────────┘
             ▼
   server/workspaces/<id>/   ← isolated, git-baselined, disposable sandbox
```

**Back end (`server/`)** — a Fastify 5 server (ESM) exposing the REST API and the WebSocket
telemetry channel, orchestrating the dual-agent loop, managing sandbox workspaces, and
rendering RCA artifacts. The MCP tool server (`server/index.js`) is a standalone stdio MCP
server exposing the five tools above.

**Front end (`client/`)** — a React 19 + Vite 8 single-page "Mission Control" dashboard with a
glassmorphic dark theme and Framer Motion animations. It drives ingestion, streams telemetry,
renders the unified diff, and downloads the artifacts.

---

## 🔄 Incident Lifecycle

1. **Ingest** — `POST /ingest/zip` (multipart `.zip`) or `POST /ingest/git` (public or
   token-authenticated clone) creates `server/workspaces/<id>` and returns
   `{ workspaceId, workspacePath }`.
2. **Baseline** — Zip imports are `git init` + committed so `git blame`/`git diff` work; Git
   imports already have history.
3. **Trigger** — `POST /trigger` is called with
   `{ workspacePath, file, line, errorType, mode, runtime, testCommand }`.
4. **Agent A (Investigator)** inspects blame + source and drafts a `<proposed_patch>` (never writes).
5. **Agent B (Reviewer)** reviews the patch + diff → `VERDICT: APPROVED` or `VERDICT: REJECTED`
   (rejects are fed back to Agent A; up to 3 rounds).
6. **Apply & Verify** — on approval the patch is written via `source_patcher`, then the runtime's
   test runner executes (`sandbox_test_runner` for Node, `python_test_runner` for Python).
7. **Report** — a unified diff is captured and an enterprise Post-Mortem
   (`INCIDENT_REPORT.md` + `INCIDENT_REPORT.html`) is written into the workspace.
8. **Broadcast** — a `success` telemetry event
   (`Mission Complete: Code verified and RCA generated.`) carrying the `workspaceId` is pushed
   to every connected dashboard.

If no passing patch is approved, the original file is **restored** and an `error` event is broadcast.

---

## ✨ Feature Highlights

- 🤖 **Two adversarial agents** with a mandatory review gate — a rejected patch is *never* written.
- 🌐 **Multi-language**: Node.js (npm/Jest) **and** Python (pytest / `python -m unittest`).
- 🛡️ **Security triage mode**: prompts the investigator to hunt attack vectors (injection,
  insecure regex/ReDoS, buffer risks, insecure crypto, path traversal) and remediate defensively
  **without breaking the interface contract**.
- 📦 **Two ingestion paths**: `.zip` upload (zip-slip protected) or Git clone
  (GitHub / GitLab / Bitbucket, public or private with a Personal Access Token).
- 🧪 **Execution-verified fixes**: a patch only counts as done when *your* test suite passes.
- 📊 **Live telemetry**: color-badged `INVESTIGATOR`, `QA REVIEWER`, `ALERT`, `SUCCESS`, `SYSTEM` events.
- 🔀 **Zero-dependency unified diff viewer** with line-number gutters and add/delete highlighting.
- 📄 **Automated Post-Mortem**: a polished standalone HTML + Markdown RCA with five sections.
- 🧊 **Premium UI**: glassmorphism, an animated mesh gradient, spring entrance animations,
  a pulsing radar beacon, and a blinking terminal cursor.

---

## 🧱 Tech Stack & Dependencies

### Backend (`server/`) — required dependencies

| Package | Version | Purpose |
| --- | --- | --- |
| `fastify` | ^5.12.5 | HTTP server |
| `@fastify/cors` | ^11.3.0 | CORS |
| `@fastify/websocket` | ^11.3.3 | WebSocket telemetry (`/stream`) |
| `@fastify/multipart` | ^10.1.2 | Streaming `.zip` uploads |
| `adm-zip` | ^0.6.1 | Zip extraction + workspace packaging |
| `zod` | ^4.6.5 | MCP tool schema validation |
| `@modelcontextprotocol/sdk` | ^1.32.0 | MCP stdio tool server |
| `@cline/sdk` | ^0.0.90 | LLM agent runtime (Agent A + Agent B) |

### Frontend (`client/`) — required dependencies

| Package | Version | Purpose |
| --- | --- | --- |
| `react` / `react-dom` | ^19.2.8 | UI runtime |
| `framer-motion` | ^14.0.0 | Animations (springs, hover glows) |
| `vite` | ^8.3.0 | Dev server + bundler |
| `@vitejs/plugin-react` | ^6.1.1 | React fast refresh |
| `eslint` (+ `@eslint/js`, `globals`, `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`) | ^10.x | Linting |

> The UI additionally loads the **Plus Jakarta Sans** and **JetBrains Mono** web fonts from Google
> Fonts (declared in `client/index.html`). An internet connection is required on first load, or the
> browser falls back to system fonts.

---

## ✅ Prerequisites

| Tool | Minimum | Tested With | Notes |
| --- | --- | --- | --- |
| **Node.js** | ≥ 20 LTS | **v26.10.0** | ESM + global `fetch`/`WebSocket`/`FormData` are used, so a modern runtime is required |
| **npm** | ≥ 9 | **11.19.1** | Ships with Node |
| **Git** | ≥ 2.30 | **2.56.0** | Required for clone / blame / diff |
| **Python 3** *(optional)* | ≥ 3.9 | 3.12 | Only for the **Python (Pytest)** runtime; `pytest` or `python -m unittest` |
| **zip** *(optional)* | any | — | Only needed if you build the sample `.zip` on the command line |

Verify your environment:

```bash
node -v      # e.g. v26.10.0
npm -v       # e.g. 11.19.1
git --version
```

> **OS support:** Linux and macOS natively. On Windows, use **WSL2** (Ubuntu) and follow the Linux steps.

---

## 📂 Repository Structure

```
sentrysleuth-demo/
├── server/                         # ── Backend (Fastify 5, ESM) ──
│   ├── server.js                   # HTTP + WebSocket server, all REST routes
│   ├── index.js                    # MCP tool server (stdio): 5 tools
│   ├── agent/
│   │   └── index.js                # ⚙️  Dual-agent orchestrator — PROVIDER_ID / MODEL_ID / API_KEY live here
│   ├── tools/                      # Sandboxed tool implementations
│   │   ├── gitBlame.js             #   git blame (read-only)
│   │   ├── sourceReader.js         #   read source file (read-only)
│   │   ├── sourcePatcher.js        #   write patched content
│   │   ├── testRunner.js           #   Node test runner (npm test)
│   │   ├── pythonRunner.js         #   Python test runner (pytest / unittest)
│   │   └── workspacePath.js        #   resolveInsideWorkspace() path guard
│   ├── reporting/
│   │   ├── gitDiff.js              # Unified diff capture
│   │   └── rcaGenerator.js         # INCIDENT_REPORT.md + INCIDENT_REPORT.html
│   ├── workspaces/                 # 🗂  Auto-created sandbox root (server/workspaces/<id>)
│   ├── package.json
│   └── node_modules/               # (created by npm install)
│
├── client/                         # ── Frontend (React 19 + Vite 8) ──
│   ├── index.html                  # Entry HTML + Google Fonts
│   ├── vite.config.js
│   ├── package.json
│   └── src/
│       ├── main.jsx                # React entry
│       ├── App.jsx                 # Mission Control dashboard (form, terminal, diff, downloads)
│       ├── App.css                 # Glassmorphism theme + animations
│       └── index.css               # Base styles
│
└── demo-repo/                      # Legacy sample repository (not required at runtime)
```

---

## 🔑 Configuration (API Key & Provider)

SentrySleuth talks to an LLM through the **`@cline/sdk`** agent runtime. All of the
configuration lives in **one file**:

> ### 📍 `server/agent/index.js` — lines **14–16**

Open it and edit these three constants:

```js
// server/agent/index.js  (~line 14)
const PROVIDER_ID = 'cline';
const MODEL_ID = 'anthropic/claude-sonnet-5';
const API_KEY = 'sk_...your-provider-key-here...';
```

| Constant | What to put there |
| --- | --- |
| **`PROVIDER_ID`** | The provider identifier — see the table below. |
| **`MODEL_ID`** | The model id. For the `cline` provider this **must** be in `modelType/model` form (e.g. `anthropic/claude-sonnet-5`). |
| **`API_KEY`** | Your provider API key (a quoted string). |

#### Supported provider examples

| `PROVIDER_ID` | Provider | Example `MODEL_ID` | Typical key format |
| --- | --- | --- | --- |
| `cline` | Cline (usage-billed) | `anthropic/claude-sonnet-5` | `sk_...` |
| `anthropic` | Anthropic | `claude-sonnet-4-5` | `sk-ant-...` |
| `openai` | OpenAI | `gpt-4o` | `sk-...` |
| `gemini` | Google Gemini API | `gemini-2.5-flash` | `AIza...` |

> ⚠️ **Gotchas learned in testing**
> - Google's provider id is **`gemini`**, **not** `google` (using `google` raises *"Unknown or disabled provider"*).
> - The **`cline`** provider requires the `modelType/model` format in `MODEL_ID`
>   (`claude-3-5-sonnet-latest` alone will fail with *"invalid model format. Expected format: modelType/model."*).
> - If a model id is rejected, the error message tells you which model to use instead — paste that into `MODEL_ID`.

**Do not commit a real key.** Keep it local, or inject it at runtime (see below).

### Environment variables

SentrySleuth **requires no environment variables** to run. Two notes for advanced setups:

- If you prefer not to hard-code the key, replace the literal in `server/agent/index.js` with
  `process.env.LLM_API_KEY` and export it before starting the backend:
  ```bash
  export LLM_API_KEY="sk-..."     # macOS/Linux
  ```
- The backend hard-codes **port 3001**; the frontend targets `http://localhost:3001` and
  `ws://localhost:3001/stream` (see `SERVER_HTTP` / `SERVER_WS` at the top of `client/src/App.jsx`).
  If you change the port, update both.

---

## 📦 Installation

From the **project root** (`sentrysleuth-demo/`), install each workspace's dependencies.
These commands are identical on **Linux and macOS**.

```bash
# 1. Install backend dependencies
cd server
npm install

# 2. Install frontend dependencies
cd ../client
npm install
```

> If you plan to run the **Python** runtime, make sure `python3` and (optionally) `pytest` are on
> your `PATH`:
> ```bash
> python3 --version
> # optional:
> python3 -m pip install --user pytest
> ```

Before continuing, open **`server/agent/index.js`** and set your `PROVIDER_ID`, `MODEL_ID`, and
`API_KEY` (see [Configuration](#-configuration-api-key--provider)).

---

## 🚀 Running the Application (Linux & macOS)

You need **two terminals** — one for the backend, one for the frontend.

### Terminal 1 — Start the backend (port 3001)

```bash
cd server
node server.js
```

Expected output:

```
🚀 SentrySleuth Backend is LIVE at http://localhost:3001
📂 Workspaces root: /absolute/path/to/sentrysleuth-demo/server/workspaces
```

> The `server/workspaces/` directory is created automatically on first boot.

### Terminal 2 — Start the frontend (Vite dev server, port 5173)

```bash
cd client
npm run dev
```

Expected output:

```
  VITE v8.x.x  ready in ~200 ms
  ➜  Local:   http://localhost:5173/
```

### Open the dashboard

- **macOS**
  ```bash
  open http://localhost:5173
  ```
- **Linux**
  ```bash
  xdg-open http://localhost:5173
  ```

You should see the glassmorphic **Mission Control** dashboard with a
**`LINK ONLINE`** beacon in the top-right (the WebSocket connected successfully).

### Health check (optional)

With the backend running, this proves the API is alive (a `404` is the *correct* response for a
non-existent workspace):

```bash
curl -i http://localhost:3001/diff/does-not-exist
# → HTTP/1.1 404 Not Found   {"status":"failed","error":"Workspace not found."}
```

### Stopping

Press `Ctrl+C` in each terminal. To reclaim port 3001 if a stale process lingers:

- **macOS / Linux**
  ```bash
  lsof -ti:3001 | xargs kill -9
  ```

---

## 🧪 Triggering a Test Incident (Zipped Repo)

This is the fastest way to see the whole pipeline. It uses a **zero-dependency** sample project so
`npm test` works inside the sandbox without any extra install.

### Step 1 — Create a small buggy repository

**macOS / Linux** (identical commands):

```bash
mkdir -p /tmp/sample-buggy-repo/src
cd /tmp/sample-buggy-repo

cat > package.json <<'JSON'
{
  "name": "sample-buggy-repo",
  "version": "1.0.0",
  "private": true,
  "scripts": { "test": "node test.js" }
}
JSON

cat > src/cart.js <<'JS'
module.exports = (cart) => cart.reduce((acc, item) => acc + item.price, 0);
JS

cat > test.js <<'JS'
const total = require('./src/cart.js');
try {
  const result = total(undefined);              // crash: cart is undefined
  if (result !== 0) { console.error('FAIL: expected 0, got', result); process.exit(1); }
  console.log('PASS: defensive guard present');
} catch (error) {
  console.error('FAIL:', error.message);
  process.exit(1);
}
JS
```

This repo **fails its own test**: calling `total(undefined)` throws because of the missing guard.

### Step 2 — Zip the repository contents

```bash
cd /tmp/sample-buggy-repo
zip -r /tmp/sample-buggy-repo.zip .
# or from Finder on macOS: right-click the folder → Compress
```

> The archive must contain `package.json`, `test.js`, and `src/` at its **root**
> (that is what `zip -r file.zip .` produces).

### Step 3 — Trigger the incident from the dashboard (recommended)

1. Open **http://localhost:5173**.
2. Under **Incident Reporting**, choose the **Upload .ZIP** tab.
3. Click the dropzone and select **`/tmp/sample-buggy-repo.zip`**.
4. Fill in the incident details:

   | Field | Value |
   | --- | --- |
   | **Target File Path** | `src/cart.js` |
   | **Line Number** | `1` |
   | **Error Message** | `TypeError: Cannot read properties of undefined (reading 'reduce')` |
   | **Runtime** | `Node.js (Jest)` |
   | **Incident Mode** | `Reliability Defect` |
   | **Custom Test Command** | *(leave empty)* |

5. Click **Deploy Agent**.

Watch the **Live Telemetry Terminal**:

```
[ALERT]        🚨 INCIDENT TRIGGERED: src/cart.js (Line 1) [reliability/node]
[INVESTIGATOR] 🔍 Agent A (Lead Investigator) dispatched
[REVIEWER]     🧪 Agent B (Senior QA Reviewer) standing by.
[INVESTIGATOR] 📝 Agent A drafting a patch (round 1/3)...
[REVIEWER]     ❌ Agent B REJECTED round 1: …
[INVESTIGATOR] 📤 Agent A submitted a patch … (round 2)
[REVIEWER]     ✅ Agent B APPROVED the patch (round 2).
[INVESTIGATOR] 🛠️ Successfully updated src/cart.js (source_patcher)
[REVIEWER]     🟢 sandbox_test_runner finished with exit code 0.
[REVIEWER]     📄 Post-mortem compiled: INCIDENT_REPORT.md + INCIDENT_REPORT.html
[SUCCESS]      Mission Complete: Code verified and RCA generated. (1791…-…)
```

On `SUCCESS` the dashboard reveals the **Changed Code** diff viewer and the two
**Enterprise Artifacts** download buttons.

### Step 4 — (Optional) Watch telemetry from the CLI

In a separate terminal *before* you trigger:

```bash
node -e "const ws=new WebSocket('ws://localhost:3001/stream');ws.onmessage=e=>console.log(e.data)"
```

### Step 5 — (Optional) Drive it entirely from the command line

```bash
# 5a. Ingest the zip → copy the resulting workspacePath
curl -s -F "file=@/tmp/sample-buggy-repo.zip" http://localhost:3001/ingest/zip
# → {"workspaceId":"1791…","workspacePath":"/…/server/workspaces/1791…"}

# 5b. Trigger the agent (paste the workspacePath from above)
curl -s -X POST http://localhost:3001/trigger \
  -H 'Content-Type: application/json' \
  -d '{"workspacePath":"/…/server/workspaces/1791…","file":"src/cart.js","line":1,"errorType":"TypeError","mode":"reliability","runtime":"node"}'
# → {"status":"Investigating..."}

# 5c. After the SUCCESS event, download the artifacts
curl -OJ "http://localhost:3001/download/1791…"   # patched codebase (.zip)
curl -OJ "http://localhost:3001/rca/1791…"        # RCA report (.html)
```

### Step 6 — Inspect the results

Inside `server/workspaces/<workspaceId>/` you will find:

```
src/cart.js              ← the fixed file (guard added)
INCIDENT_REPORT.md       ← Markdown post-mortem
INCIDENT_REPORT.html     ← standalone, styled HTML post-mortem
.git/                    ← baseline git history (blame/diff support)
```

### Using a Jest project instead

If your payload uses Jest (`"test": "jest"` + `devDependencies`), the sandbox needs the
dependencies installed before the tests can run. Either **bundle `node_modules/` in the zip**, or
install them once in the workspace:

```bash
cd server/workspaces/<workspaceId>
npm install
```

---

## 🔐 Private Repositories (Access Tokens)

SentrySleuth can clone **private** repositories using a **Personal Access Token (PAT)**.

**In the dashboard:** choose the **Public Git Repo URL** tab and fill in:

| Field | Value |
| --- | --- |
| **Repository URL** | e.g. `https://github.com/company/private-repo.git` |
| **Git Access Token (Optional for Private Repos)** | your token (masked input, e.g. `ghp_…`) |

Supported hosts over **HTTPS**: **GitHub**, **GitLab**, and **Bitbucket**.

**How it is handled securely:**
- The URL is rewritten **in memory only** to `https://<TOKEN>@host/owner/repo.git`.
- The clone runs with `git -c credential.helper= clone …` (no OS keychain) and
  `GIT_TERMINAL_PROMPT=0` (never blocks on a prompt).
- After cloning, the remote URL is reset to the clean URL so the token is **not** left in `.git/config`.
- The token is **never** logged, returned to the client, or broadcast over the WebSocket.

**Via the API:**

```bash
curl -s -X POST http://localhost:3001/ingest/git \
  -H 'Content-Type: application/json' \
  -d '{"repoUrl":"https://github.com/company/private-repo.git","gitToken":"ghp_xxxxxxxxxxxx"}'
```

**Authentication errors** return `HTTP 401` with a clear message:

- No token, repo looks private: *"This repository looks private (or does not exist). Provide a valid
  Git Access Token to clone private repositories."*
- Token present but rejected: *"The provided Git Access Token was rejected. Check that it is valid
  and has read access to this repository."*

---

## 📡 Backend API Reference

Base URL: **`http://localhost:3001`**

| Method | Path | Body | Returns |
| --- | --- | --- | --- |
| `POST` | `/ingest/zip` | `multipart/form-data` with field **`file`** (a `.zip`) | `{ workspaceId, workspacePath }` |
| `POST` | `/ingest/git` | JSON `{ repoUrl, gitToken? }` | `{ workspaceId, workspacePath }` |
| `POST` | `/trigger` | JSON (see below) | `{ status: "Investigating..." }` |
| `GET` | `/download/:workspaceId` | — | `application/zip` attachment (patched workspace) |
| `GET` | `/diff/:workspaceId` | — | `{ diff: string, filesChanged: string[] }` |
| `GET` | `/rca/:workspaceId` | — | `text/html` or `text/markdown` attachment (RCA report) |
| `GET` | `/stream` | — | **WebSocket** telemetry channel |

### `POST /trigger` body

```jsonc
{
  "workspacePath": "/abs/path/server/workspaces/<id>", // required
  "file": "src/math.js",                               // file with the bug
  "line": 4,                                           // offending line number
  "errorType": "TypeError: ...",                        // error signature
  "mode": "reliability",                                // "reliability" | "security"
  "runtime": "node",                                    // "node" | "python"
  "testCommand": ""                                     // optional, e.g. "pytest tests/test_auth.py"
}
```

Notes:
- `mode: "security"` switches Agent A into **security-triage** mode (attack-vector hunting + defensive remediation).
- `runtime: "python"` makes the loop verify with **`python_test_runner`** instead of `sandbox_test_runner`.
- The endpoint returns immediately; progress arrives over the WebSocket.

### Status codes

| Code | Meaning |
| --- | --- |
| `200` | Success |
| `400` | Bad request (missing/invalid fields, unsafe URL, bad archive) |
| `401` | Git authentication required / token rejected |
| `404` | Workspace or report not found |

---

## 📶 WebSocket Telemetry Events

Connect to **`ws://localhost:3001/stream`**. Every frame is a JSON object `{ type, message }`
(success frames also carry `workspaceId`):

| `type` | Meaning |
| --- | --- |
| `info` | System/status (e.g. *Connected to SentrySleuth Server!*, workspace staged) |
| `alert` | Incident triggered |
| `investigator` | **Agent A** (Lead Investigator) — dispatching, drafting, applying, re-patching |
| `reviewer` | **Agent B** (Senior QA Reviewer) — reviewing, **approved/rejected**, test result, RCA compiled |
| `success` | Mission complete → `{ type: "success", message: "...", workspaceId }` |
| `error` | The mission failed (with a human-readable reason) |

Minimal client:

```bash
node -e "const ws=new WebSocket('ws://localhost:3001/stream');ws.onmessage=e=>console.log(JSON.parse(e.data))"
```

---

## 🛠 Troubleshooting

| Symptom | Cause & Fix |
| --- | --- |
| **"Unknown or disabled provider"** | `PROVIDER_ID` is wrong. For Google use **`gemini`** (not `google`). Check the spelling in `server/agent/index.js`. |
| **"invalid model format. Expected format: modelType/model."** | The `cline` provider needs `MODEL_ID` as `type/model`, e.g. `anthropic/claude-sonnet-5`. |
| **"models/… is not found / no longer available to new users"** | The model id was retired. Copy the model the error recommends into `MODEL_ID`. |
| **"Authentication failed" / token errors** | Your `API_KEY` is missing/invalid, or (for private repos) the Git Access Token is wrong. |
| **"This repository looks private…"** | Add a **Git Access Token** for private repos (see [Private Repositories](#-private-repositories-access-tokens)). |
| **Frontend badge shows `LINK OFFLINE`** | The backend isn't running, or it's on a different port. Start it with `node server.js` and confirm port **3001**. |
| **`jest: not found` inside a sandbox** | Install deps in the workspace (`cd server/workspaces/<id> && npm install`) or bundle `node_modules/` in your `.zip`. |
| **`EADDRINUSE: address already in use :::3001`** | A stale server is running: `lsof -ti:3001 \| xargs kill -9`. |
| **Mission ends with `error`** | Expected when the reviewer rejects every candidate, or the LLM call fails. The workspace is restored; check the terminal output for the reason. |

---

## 🔒 Security Notes

- **Secrets stay in memory.** The LLM API key lives in `server/agent/index.js` (never transmitted to
  the browser). Git tokens are injected into the clone URL only for the duration of one `git clone`
  and are redacted from all logs, HTTP responses, and WebSocket frames.
- **Sandboxed file access.** Every tool routes paths through `resolveInsideWorkspace()`, which blocks
  path traversal, so agents cannot read or write outside their workspace.
- **Zip-slip protection.** Archive extraction validates every entry stays inside the target workspace.
- **No shell interpolation for git.** Only allow-listed URL schemes are accepted, and unsafe
  metacharacters are rejected before the URL reaches `git`.
- **Disposable workspaces.** Each incident gets an isolated directory under `server/workspaces/`
  that can be deleted at any time.
- **Production hardening (out of scope for the demo):** the backend enables permissive CORS
  (`origin: '*'`) and the API key is a plain literal — put both behind a secret manager and tighten
  CORS before deploying publicly.

---

## ✅ Judges' Quick-Start Checklist

```bash
# 0) Prereqs: node >= 20, npm, git  (see Prerequisites)

# 1) Install
cd server && npm install && cd ../client && npm install && cd ..

# 2) Configure  → edit server/agent/index.js (PROVIDER_ID / MODEL_ID / API_KEY)

# 3) Run backend (terminal 1)
cd server && node server.js

# 4) Run frontend (terminal 2)
cd client && npm run dev
#   → open http://localhost:5173  (macOS: open http://localhost:5173 | Linux: xdg-open ...)

# 5) Trigger a sample incident — see "Triggering a Test Incident (Zipped Repo)"
#    Upload a .zip, target the buggy file + line, click "Deploy Agent".

# 6) Watch → SUCCESS → inspect the diff viewer and download the .zip + RCA .html
```

**What to look for:** an `alert → investigator → reviewer → success` telemetry sequence, Agent B
approving (and, on the first attempt, rejecting) a patch, tests going green (`exit code 0`), and the
generated `INCIDENT_REPORT.html`.

---

## 📝 License

Provided as-is for demonstration / hackathon purposes.

## 🙌 Acknowledgements

Built with [Fastify](https://fastify.dev/), [React](https://react.dev/), [Vite](https://vite.dev/),
[Framer Motion](https://www.framer.com/motion/), [adm-zip](https://github.com/cthackers/adm-zip),
the [Model Context Protocol SDK](https://modelcontextprotocol.io/), and the
[Cline SDK](https://cline.bot/).








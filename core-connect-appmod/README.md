# Connect AppMod MCP Server

> � **Download:** [di-appmod-mcp.beta.v1.0](https://github.com/PreciselyData/precisely-mcp-servers/releases/tag/di-appmod-mcp.beta.v1.0)

A [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server that evaluates whether a mainframe DFSORT or Syncsort sort step is supported by the Precisely Connect AppMod components (**DMXMMSRT** and **DMXMFSRT**). Each tool call takes one extracted sort step, invokes DMXMMSRT for syntax validation, and returns a support classification derived from the exit code plus normalized stderr diagnostics.

---

## Overview

```
MCP Client ──stdio (JSON-RPC)──► Connect AppMod MCP Server ──► DMXMMSRT (subprocess) ──► exit code + stderr
```

### Tool

| Tool | Description |
|------|-------------|
| `check_sort_support` | Submit a single extracted DFSORT or Syncsort step; receive `supported`, `not_supported`, or `indeterminate` based on the DMXMMSRT exit code, plus normalized stderr diagnostics |

---

## Prerequisites

1. **`uv` is installed** — MCP clients invoke `uvx --from <wheel>` to launch the server on demand. `uvx` creates a temporary isolated environment, installs the wheel, and runs the server automatically; no separate installation step is required. Install `uv` from [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/) if not already present.

2. **The wheel file is available** at a known absolute path on the machine where the MCP client runs.

3. **The DMXMMSRT executable is installed** and accessible on the machine where the MCP server runs. The server **fails fast at startup** if `DMXMMSRT_EXECUTABLE` is not set or not executable.

   The latest GA release of Connect (10.0.2), which includes DMXMMSRT, does not yet contain the enhancements required by this MCP server. To try this feature before a GA release is available, email ban.tran@precisely.com to request a pre-release version.

---

## Configuration

The server is configured entirely via environment variables.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DMXMMSRT_EXECUTABLE` | **Yes** | — | Absolute path to the DMXMMSRT executable, e.g. `/opt/precisely/dmxmmsrt/bin/dmxmmsrt` |
| `DMXMMSRT_TIMEOUT_SECONDS` | No | `30` | Maximum seconds to wait for DMXMMSRT to complete per sort step |
| `DMXMMSRT_LOG_LEVEL` | No | `INFO` | Python logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

> **Startup fail-fast**: The server exits immediately with a clear error message if `DMXMMSRT_EXECUTABLE` is unset or points to a path that is not executable. Run the server in a terminal (see [Manual Installation (for Debugging)](#manual-installation-for-debugging)) to see the error before configuring an MCP client.

---

## Tool Reference: `check_sort_support`

Submit a single extracted sort step and receive a support classification.

### Input

| Field | Type | Required | Description |
|---|---|---|---|
| `sort_step` | string | yes | Complete text of one DFSORT or Syncsort step (≤ 1 MB). Client is responsible for extracting individual steps from JCL source. |
| `correlation_id` | string | no | Opaque identifier echoed unchanged in the response for batch result mapping |
| `generate_syntax` | boolean | no | When `true`, DMXMMSRT returns the converted DMExpress SDK syntax in `syntax_file_content` for supported steps. Has no effect if the step is not supported. |

### Output

| Field | Description |
|---|---|
| `support_status` | `supported` (exit 0), `not_supported` (exit 16), or `indeterminate` (all other exits, timeout, not available) |
| `diagnostics` | Normalized stderr messages — empty when DMXMMSRT emits no output |
| `execution_outcome` | Exit code, classification, timeout flag, and duration in seconds |
| `syntax_file_content` | DMExpress SDK syntax content when `generate_syntax` is `true` and the step is supported; `null` or empty string otherwise |
| `correlation_id` | Echoed from request; `null` if not provided |

### Status Reference

| `support_status` | Exit code | `classification` | Meaning |
|---|---|---|---|
| `supported` | `0` | `success` | Step is fully supported by DMXMMSRT |
| `not_supported` | `16` | `unsupported` | Step contains constructs DMXMMSRT cannot convert |
| `indeterminate` | other | `error` | Exit code indicates an unexpected error |
| `indeterminate` | — | `timeout` | DMXMMSRT exceeded `DMXMMSRT_TIMEOUT_SECONDS` |
| `indeterminate` | — | `not_available` | `DMXMMSRT_EXECUTABLE` is not configured |

### Syntax Generation

When `generate_syntax: true` is passed, DMXMMSRT returns the converted DMExpress SDK syntax as a string in `syntax_file_content`. The caller can save this content to a `.sdk` file for use with DMExpress or for reference.

> **Note on record attributes:** JCL sort steps often omit record attributes (RECFM, LRECL, etc.) for input and/or output files. In those cases, DMXMMSRT derives them from sort control statements or uses defaults. In production deployments, accurate attributes are retrieved from the map file (DMXMMSRT) or the catalog (DMXMFSRT).

### Examples

#### Supported — with SDK syntax

**Call:**
```json
{
  "name": "check_sort_support",
  "arguments": {
    "sort_step": "//SORT01  EXEC PGM=SORT\n//SORTIN  DD DSN=MY.INPUT.DATASET,DISP=SHR\n//SORTOUT DD DSN=MY.OUTPUT.DATASET,DISP=(,CATLG,DELETE),\n//           DCB=(RECFM=FB,LRECL=80)\n//SYSIN   DD *\n  SORT FIELDS=COPY\n/*",
    "generate_syntax": true,
    "correlation_id": "step-01"
  }
}
```

**Response:**
```json
{
  "support_status": "supported",
  "diagnostics": [
    ...
  ],
  "failure_reasons": [],
  "execution_outcome": {
    "exit_code": 0,
    "classification": "success",
    "timed_out": false,
    "duration_seconds": 0.15
  },
  "correlation_id": "step-01",
  "syntax_file_content": "/copy\n/infile             MY.INPUT.DATASET fixed 80\n/outfile            MY.OUTPUT.DATASET fixed 80\n/statistics\n"
}
```

#### Not supported

**Call:**
```json
{
  "name": "check_sort_support",
  "arguments": {
    "sort_step": "//SORT01  EXEC PGM=SORT\n//SORTIN  DD DSN=MY.INPUT.DATASET,DISP=SHR\n//SORTOUT DD DSN=MY.OUTPUT.DATASET,DISP=(,CATLG,DELETE),\n//           DCB=(RECFM=FB,LRECL=10)\n//SYSIN   DD *\n  SORT FIELDS=(1,10,CH,A)\n  INREC OVERLAY=(11:21,10)\n/*",
    "correlation_id": "step-42"
  }
}
```

**Response:**
```json
{
  "support_status": "not_supported",
  "diagnostics": [
    ...
    4: {
      "category": "unsupported_syntax",
      "severity": "error",
      "message": "Connect : (CEEXFIELDS) expecting FIELDS keyword",
      "source": "stderr",
      "token": "OVERLAY",
      "location": null
    }
  ],
  "failure_reasons": ["Connect : (CEEXFIELDS) expecting FIELDS keyword => found 'OVERLAY'"],
  "execution_outcome": {
    "exit_code": 16,
    "classification": "unsupported",
    "timed_out": false,
    "duration_seconds": 0.42
  },
  "correlation_id": "step-42",
  "syntax_file_content": null
}
```

---

## MCP Client Quick Start

The primary configuration snippet for all clients uses `uvx --from` with the wheel file. Use the **absolute path** to the wheel.

**macOS/Linux:**
```json
{
  "command": "uvx",
  "args": ["--from", "/absolute/path/to/di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
  "env": {
    "DMXMMSRT_EXECUTABLE": "/opt/precisely/dmxmmsrt/bin/dmxmmsrt"
  }
}
```

**Windows:**
```json
{
  "command": "uvx",
  "args": ["--from", "C:\\absolute\\path\\to\\di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
  "env": {
    "DMXMMSRT_EXECUTABLE": "C:\\Precisely\\AppMod\\dmxmmsrt.exe"
  }
}
```

For detailed per-client configuration (VS Code / GitHub Copilot, IntelliJ, JetBrains AI Assistant, Microsoft Visual Studio), see [deployment-clients.md](deployment-clients.md).

### Claude Desktop

Open or create the Claude Desktop config file:

| Platform | Path |
|----------|------|
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |

Add the following (merging with any existing `mcpServers` entries):

**macOS/Linux:**
```json
{
  "mcpServers": {
    "di-appmod-mcp": {
      "command": "uvx",
      "args": ["--from", "/absolute/path/to/di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
      "env": {
        "DMXMMSRT_EXECUTABLE": "/opt/precisely/dmxmmsrt/bin/dmxmmsrt"
      }
    }
  }
}
```

**Windows:**
```json
{
  "mcpServers": {
    "di-appmod-mcp": {
      "command": "uvx",
      "args": ["--from", "C:\\absolute\\path\\to\\di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
      "env": {
        "DMXMMSRT_EXECUTABLE": "C:\\Precisely\\AppMod\\dmxmmsrt.exe"
      }
    }
  }
}
```

Fully quit Claude Desktop and relaunch after saving.

### Verifying the Setup

Once Claude Desktop is configured and relaunched, you can test the MCP server directly from the chat window — no additional tooling required. Paste a JCL sort step and ask Claude to check it.

**You:**

> Can you check if this sort step is supported?
>
> ```
> //SORT01  EXEC PGM=SORT
> //SORTIN  DD DSN=MY.INPUT.DATASET,DISP=SHR
> //SORTOUT DD DSN=MY.OUTPUT.DATASET,DISP=(,CATLG,DELETE),DCB=(RECFM=FB,LRECL=80)
> //SYSIN   DD *
>   SORT FIELDS=COPY
> /*
> ```

**Claude:**

> I'll check that now using the `check_sort_support` tool.
>
> *(Tool: check\_sort\_support)*
>
> **Result: Supported**
>
> DMXMMSRT exits with code 0 and emits no diagnostics — the step is fully supported by DMXMMSRT. If you also want to see the converted DMExpress SDK syntax, ask me to rerun with syntax generation enabled.

If the `check_sort_support` tool does not appear in Claude's response, the MCP server did not start correctly. See [Troubleshooting](#troubleshooting) for diagnosis steps.

---

## Batch JCL Sort-Support Analysis

Beyond single-step checks, the MCP server can be used with an **AI agent** (GitHub Copilot Chat or Claude Desktop) to analyse entire JCL libraries at scale. The agent extracts every sort step from a folder of JCL files, calls `check_sort_support` for each step in parallel batches, and produces a structured Markdown report.

### Additional Files Required

The wheel alone is not sufficient for batch analysis. The following files must also be present in the workspace:

| File | Purpose |
|------|---------|
| `.github/prompts/sort-support-analysis.prompt.md` | Workflow instructions for the AI agent (required by both clients) |
| `.github/copilot-instructions.md` | VS Code GitHub Copilot only — tells Copilot to invoke the prompt automatically |
| `scripts/extract_complete_sort_steps.py` | Extracts sort steps from JCL source files into a run directory |
| `scripts/build_test_results.py` | Assembles `test-results.json` from batch MCP results |
| `scripts/generate_report.py` | Generates the Markdown report from `test-results.json` |
| `scripts/report_token_usage.py` | Reports token usage statistics for the run |

These files are distributed separately alongside the wheel.

### Workspace Layout

Place the files in the workspace root before running the analysis:

```
<workspace>/
├── .github/
│   ├── copilot-instructions.md          ← VS Code Copilot only
│   └── prompts/
│       └── sort-support-analysis.prompt.md
├── scripts/
│   ├── extract_complete_sort_steps.py
│   ├── build_test_results.py
│   ├── generate_report.py
│   └── report_token_usage.py
└── reports/                             ← created automatically on first run
```

Configure the MCP server for this workspace using one of the client configurations in [deployment-clients.md](deployment-clients.md).

### Running the Analysis

The workflow can be driven by either **GitHub Copilot in VS Code** or **Claude Desktop**.

#### GitHub Copilot (VS Code)

`copilot-instructions.md` tells Copilot to follow `sort-support-analysis.prompt.md` automatically when an analysis is requested. With both files in place, switch Copilot Chat to **Agent mode** and enter:

> "Run sort-support analysis on `C:\path\to\jcl\source`"

To also generate DMExpress SDK syntax files for all supported steps:

> "Run sort-support analysis on `C:\path\to\jcl\source` with syntax generation"

#### Claude Desktop

Claude Desktop does not read workspace instruction files. Load the workflow instructions using one of the options below, then send the analysis request.

**Option A — Claude Project (recommended for repeated use)**

Create a Claude Project at [claude.ai/projects](https://claude.ai/projects). Open *Project Instructions* and paste the contents of `sort-support-analysis.prompt.md`. Every conversation in the project will then have the workflow available without any extra setup.

**Option B — Paste into conversation (one-off use)**

Start a new conversation, paste the full contents of `sort-support-analysis.prompt.md` as the opening message, then follow with the analysis request.

For either option, send:

> "Run sort-support analysis on `C:\path\to\jcl\source`"

#### What the agent does

Regardless of client, the agent will:

1. Extract all SORT/ICEMAN/ICETOOL steps from the JCL source directory into a timestamped run directory (`reports/sort-support-run-<timestamp>/`)
2. Call `check_sort_support` for each step in parallel batches of ~10
3. Write per-step stderr logs and (when requested) `.sdk` syntax files after each batch
4. Build `test-results.json`
5. Generate `sort-support-results.md`
6. Report token usage and total elapsed time

### Run Directory Contents

| Path | Description |
|------|-------------|
| `extracted-steps.json` | All extracted sort steps with metadata |
| `temp-steps/*.jcl` | Individual JCL file per step |
| `stderr/*.stderr.log` | DMXMMSRT stderr output per step |
| `syntax/*.sdk` | Generated DMExpress SDK syntax for supported steps (when requested) |
| `test-results.json` | Full results array, one entry per step |
| `sort-support-results.md` | Markdown report |

### Example Report

The report is written to `sort-support-results.md` in the run directory. In the actual generated report, every JCL file path, SDK syntax file, and stderr log appears as a **clickable relative link** that opens the corresponding file directly in your editor or browser. The example below shows the same link formatting — in this README the links are placeholders and clicking one will scroll back to this section heading rather than opening a file.

#### Run Summary

- Total sort steps: 20
- Successful sort steps: 17 (85.0%)
- Failed sort steps: 3 (15.0%)

#### Successful RC=0 Sort Steps

| Step reference | JCL file | Syntax file |
| --- | --- | --- |
| MYJOB_001_STEP010 | [temp-steps/MYJOB_001_STEP010.jcl](#example-report) | [syntax/MYJOB_001_STEP010.sdk](#example-report) |
| MYJOB_002_STEP020 | [temp-steps/MYJOB_002_STEP020.jcl](#example-report) | [syntax/MYJOB_002_STEP020.sdk](#example-report) |
| MYJOB_003_STEP010 | [temp-steps/MYJOB_003_STEP010.jcl](#example-report) | [syntax/MYJOB_003_STEP010.sdk](#example-report) |
| MYJOB_004_STEP020 | [temp-steps/MYJOB_004_STEP020.jcl](#example-report) | [syntax/MYJOB_004_STEP020.sdk](#example-report) |
| MYJOB_005_STEP010 | [temp-steps/MYJOB_005_STEP010.jcl](#example-report) | [syntax/MYJOB_005_STEP010.sdk](#example-report) |
| MYJOB_006_STEP030 | [temp-steps/MYJOB_006_STEP030.jcl](#example-report) | [syntax/MYJOB_006_STEP030.sdk](#example-report) |
| MYJOB_007_STEP010 | [temp-steps/MYJOB_007_STEP010.jcl](#example-report) | [syntax/MYJOB_007_STEP010.sdk](#example-report) |
| MYJOB_008_STEP020 | [temp-steps/MYJOB_008_STEP020.jcl](#example-report) | [syntax/MYJOB_008_STEP020.sdk](#example-report) |
| MYJOB_009_STEP010 | [temp-steps/MYJOB_009_STEP010.jcl](#example-report) | [syntax/MYJOB_009_STEP010.sdk](#example-report) |
| MYJOB_010_STEP020 | [temp-steps/MYJOB_010_STEP020.jcl](#example-report) | [syntax/MYJOB_010_STEP020.sdk](#example-report) |
| MYJOB_011_STEP010 | [temp-steps/MYJOB_011_STEP010.jcl](#example-report) | [syntax/MYJOB_011_STEP010.sdk](#example-report) |
| MYJOB_012_STEP030 | [temp-steps/MYJOB_012_STEP030.jcl](#example-report) | [syntax/MYJOB_012_STEP030.sdk](#example-report) |
| MYJOB_013_STEP010 | [temp-steps/MYJOB_013_STEP010.jcl](#example-report) | [syntax/MYJOB_013_STEP010.sdk](#example-report) |
| MYJOB_014_STEP020 | [temp-steps/MYJOB_014_STEP020.jcl](#example-report) | [syntax/MYJOB_014_STEP020.sdk](#example-report) |
| MYJOB_015_STEP010 | [temp-steps/MYJOB_015_STEP010.jcl](#example-report) | [syntax/MYJOB_015_STEP010.sdk](#example-report) |
| MYJOB_016_STEP020 | [temp-steps/MYJOB_016_STEP020.jcl](#example-report) | [syntax/MYJOB_016_STEP020.sdk](#example-report) |
| MYJOB_017_STEP010 | [temp-steps/MYJOB_017_STEP010.jcl](#example-report) | [syntax/MYJOB_017_STEP010.sdk](#example-report) |

#### Grouped Failed Steps

| Failure reason | JCL sort step reference | Failed logs if any | JCL file | Executed step % |
| --- | --- | --- | --- | --- |
| Connect : (CEEXIN) expecting a valid integer => found 'SFF' | MYJOB_018_STEP010 | [stderr/MYJOB_018_STEP010.stderr.log](#example-report) | [temp-steps/MYJOB_018_STEP010.jcl](#example-report) | 5.00% |
| Connect : (CEEXFIELDS) expecting FIELDS keyword => found 'OVERLAY'<br>Connect : (CEIGPARM) parameter is ignored => found 'REMOVECC' | MYJOB_019_STEP020 | [stderr/MYJOB_019_STEP020.stderr.log](#example-report) | [temp-steps/MYJOB_019_STEP020.jcl](#example-report) | 5.00% |
| ICETOOL is not supported by DMXMMSRT | MYJOB_020_STEP010 | [stderr/MYJOB_020_STEP010.stderr.log](#example-report) | [temp-steps/MYJOB_020_STEP010.jcl](#example-report) | 5.00% |

#### Detailed Failed Steps

| JCL sort step reference | Summarized failure reasons parsed from the stderr log | Link to stderr log | Link to the JCL file |
| --- | --- | --- | --- |
| MYJOB_018_STEP010 | Connect : (CEEXIN) expecting a valid integer => found 'SFF' | [stderr/MYJOB_018_STEP010.stderr.log](#example-report) | [temp-steps/MYJOB_018_STEP010.jcl](#example-report) |
| MYJOB_019_STEP020 | Connect : (CEEXFIELDS) expecting FIELDS keyword => found 'OVERLAY'<br>Connect : (CEIGPARM) parameter is ignored => found 'REMOVECC' | [stderr/MYJOB_019_STEP020.stderr.log](#example-report) | [temp-steps/MYJOB_019_STEP020.jcl](#example-report) |
| MYJOB_020_STEP010 | ICETOOL is not supported by DMXMMSRT | [stderr/MYJOB_020_STEP010.stderr.log](#example-report) | [temp-steps/MYJOB_020_STEP010.jcl](#example-report) |

---

## Troubleshooting

**Server doesn't appear in Claude Desktop**
- Verify the config file exists and the JSON is valid (no missing commas, no unescaped backslashes).
- Fully quit Claude Desktop using File → Quit (macOS) or system tray → Quit (Windows) and relaunch — closing the window is not enough.
- Confirm `uvx` is on PATH: run `uvx --version` in your terminal. If missing, install [uv](https://docs.astral.sh/uv/getting-started/installation/).

**`DMXMMSRT_EXECUTABLE` error on startup**
- The server exits immediately with an error if the env var is not set or the path is not executable.
- Run the server directly in a terminal to see the full error message (see [Manual Installation (for Debugging)](#manual-installation-for-debugging)).
- Verify the path: `ls -la /opt/precisely/dmxmmsrt/bin/dmxmmsrt` (macOS/Linux) or `Test-Path "C:\..."` (Windows).

**All results return `indeterminate`**
- Check whether DMXMMSRT exits with codes other than 0 or 16. The `execution_outcome.exit_code` field in the response shows the actual exit code.
- A value of `null` with `timed_out: true` means the process was killed. Increase `DMXMMSRT_TIMEOUT_SECONDS`.

**Tools not listed in the client at all**
- The server process did not start. Check client logs.
- Run the server manually in a terminal to surface startup errors.

**Server crashes silently / MCP client sees no tools**
- Ensure nothing in the environment writes to stdout. The stdio MCP transport uses stdout exclusively for JSON-RPC; any stray output corrupts the protocol.
- MCP clients suppress server stderr, so startup errors are invisible in the client. Run the server directly (see below) to see full error output.

**`uvx` not found**
- Install `uv` following the [official installation guide](https://docs.astral.sh/uv/getting-started/installation/). `uvx` is included with `uv`.

**`uvx --from <wheel>` fails with path error**
- Use an **absolute path** to the wheel file. Relative paths may not resolve correctly depending on how the MCP client launches the process.

---

## Manual Installation (for Debugging)

> **Not required for MCP client integration.** MCP clients launch the server directly via `uvx --from <wheel>` — no separate installation step is needed. Use manual installation only when you need to run the server outside of an MCP client to diagnose a problem.

Running the server directly lets you see stderr in your terminal — MCP clients swallow stderr, so startup errors are invisible in the client.

### Option 1 — `uv tool install` from wheel (recommended)

```bash
uv tool install di_appmod_mcp-{version}-py3-none-any.whl
```

**macOS/Linux:**
```bash
export DMXMMSRT_EXECUTABLE=/opt/precisely/dmxmmsrt/bin/dmxmmsrt
di-appmod-mcp
```

**Windows (PowerShell):**
```powershell
$env:DMXMMSRT_EXECUTABLE = "C:\Precisely\AppMod\dmxmmsrt.exe"
di-appmod-mcp
```

### Option 2 — `pip install` into a virtual environment

```bash
python -m venv .venv-debug

# macOS/Linux
.venv-debug/bin/pip install di_appmod_mcp-{version}-py3-none-any.whl
DMXMMSRT_EXECUTABLE=/opt/precisely/dmxmmsrt/bin/dmxmmsrt .venv-debug/bin/di-appmod-mcp

# Windows
.venv-debug\Scripts\pip install di_appmod_mcp-{version}-py3-none-any.whl
$env:DMXMMSRT_EXECUTABLE = "C:\Precisely\AppMod\dmxmmsrt.exe"
.venv-debug\Scripts\di-appmod-mcp
```

The server starts listening on stdin/stdout. It will log to stderr and exit immediately with an error message if `DMXMMSRT_EXECUTABLE` is not configured correctly. Use [MCP Inspector](https://github.com/modelcontextprotocol/inspector) (`npx @modelcontextprotocol/inspector`) to connect and test tools interactively.

---

## See Also

- [deployment-clients.md](deployment-clients.md) — Per-client deployment guide (VS Code, JetBrains, Visual Studio)
- [Model Context Protocol documentation](https://modelcontextprotocol.io/) — MCP specification and client guides

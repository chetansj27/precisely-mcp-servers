# Deploying Connect AppMod MCP Server

This guide covers deploying the Connect AppMod MCP server across supported MCP clients using a pre-built wheel artifact. For the Claude Desktop quick-start, see the [Claude Desktop](README.dist.md#claude-desktop) section in the distribution README.

In all configuration snippets below, replace `{version}` with the actual wheel version (e.g. `0.1.0`), and replace the `DMXMMSRT_EXECUTABLE` path with the absolute path to DMXMMSRT on the target machine.

Supported clients with dedicated instructions:
- [VS Code / GitHub Copilot](#vs-code--github-copilot)
- [JetBrains AI Assistant](#jetbrains-ai-assistant-intellij-idea-and-other-jetbrains-ides) (all platforms; IDE 2025.2+)
- [IntelliJ / GitHub Copilot](#intellij--github-copilot)
- [Microsoft Visual Studio](#microsoft-visual-studio) (Visual Studio 2022 17.14+ / 2026 with GitHub Copilot)

---

## Prerequisites

Before configuring any MCP client, ensure the following are in place:

1. **`uv` is installed** — MCP clients invoke `uvx --from <wheel>` to launch the server on demand. `uvx` creates a temporary isolated environment, installs the wheel, and runs the server automatically; no separate installation step is required. Install `uv` from [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/) if not already present.

2. **The wheel file is available** at a known absolute path on the machine where the MCP client runs. Each client config section below uses this path in the `--from` argument.

3. **DMXMMSRT is installed** and the path is known. The server exits immediately at startup if `DMXMMSRT_EXECUTABLE` is not set or not executable.

   The latest GA release of Connect (10.0.2), which includes DMXMMSRT, does not yet contain the enhancements required by this MCP server. To try this feature before a GA release is available, email ban.tran@precisely.com to request a pre-release version.

---

## VS Code / GitHub Copilot

GitHub Copilot in VS Code reads MCP server configurations from a `.vscode/mcp.json` file in your workspace root.

### Configuration

Create or open `.vscode/mcp.json` in your workspace root and add:

```json
{
    "servers": {
        "di-appmod-mcp": {
            "type": "stdio",
            "command": "uvx",
            "args": ["--from", "/absolute/path/to/di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
            "env": {
                "DMXMMSRT_EXECUTABLE": "/opt/precisely/dmxmmsrt/bin/dmxmmsrt"
            }
        }
    }
}
```

> **Windows path**: Replace the `--from` value with `C:\\absolute\\path\\to\\di_appmod_mcp-{version}-py3-none-any.whl` (double backslashes in JSON). Set `DMXMMSRT_EXECUTABLE` to `C:\\Precisely\\AppMod\\dmxmmsrt.exe`.

### Starting the server

After saving `.vscode/mcp.json`:

1. Reload the VS Code window: `Ctrl+Shift+P` (or `Cmd+Shift+P` on macOS) → **Developer: Reload Window**.
2. VS Code will show a **trust dialog** on first use — review the server configuration and confirm trust. The server will not start until you do.
3. Once trusted, VS Code starts the server automatically when Copilot needs its tools. To manually start, stop, or restart at any time, run **MCP: List Servers** from the Command Palette, select `di-appmod-mcp`, and choose **Start** or **Restart**.

> **If the server does not appear or start**: run **MCP: List Servers**, select `di-appmod-mcp`, and choose **Start**. If trust was previously declined, run **MCP: Reset Trust** from the Command Palette and reload the window to be prompted again.

> **Agent mode required**: MCP tools are only available in **Agent mode**. In the Copilot Chat panel, switch the mode selector to **Agent** before invoking any tools; **Ask** is generally the default mode.

For full MCP documentation in VS Code see [Use MCP servers in VS Code](https://code.visualstudio.com/docs/copilot/chat/mcp-servers).

### Log location

Open the Output panel (`Ctrl+Shift+U`) and select **GitHub Copilot** or **MCP** from the dropdown to see server startup and tool-execution logs.

---

## IntelliJ / GitHub Copilot

GitHub Copilot in IntelliJ IDEA and other JetBrains IDEs reads MCP server configurations from a `mcp.json` file on disk.

### Configuration

**Config file location:**

| Platform | Path |
|----------|------|
| **Windows** | `%LOCALAPPDATA%\github-copilot\intellij\mcp.json` |
| **macOS** | `~/Library/Application Support/github-copilot/intellij/mcp.json` |
| **Linux** | `~/.config/github-copilot/intellij/mcp.json` |

If the file does not exist, create it. Add a `servers` block (merging with any existing entries):

```json
{
  "servers": {
    "di-appmod-mcp": {
      "type": "stdio",
      "command": "uvx",
      "args": ["--from", "/absolute/path/to/di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
      "env": {
        "DMXMMSRT_EXECUTABLE": "/opt/precisely/dmxmmsrt/bin/dmxmmsrt"
      }
    }
  }
}
```

> **Windows path**: Replace the `--from` value with `C:\\absolute\\path\\to\\di_appmod_mcp-{version}-py3-none-any.whl`. Set `DMXMMSRT_EXECUTABLE` to `C:\\Precisely\\AppMod\\dmxmmsrt.exe`.

### Starting the server

After saving the config file, restart the IDE. The most reliable way to start the MCP server and confirm its tools are available is:

1. Open the GitHub Copilot Chat panel.
2. Switch to **Agent mode** using the mode selector at the bottom of the chat input (the server is only accessible in Agent mode).
3. Click the **Configure tools** button that appears in the chat input toolbar once Agent mode is active. This triggers GitHub Copilot to connect to the MCP server and load its tools.
4. Confirm `di-appmod-mcp` and its tools are listed. If the server is not connected, check the Copilot output log for startup errors.

> **Agent mode required**: MCP tools are not available in Ask or Edit mode.

### Log location

Check the GitHub Copilot output panel: **Help → Show Log in Explorer** (Windows) / **Show Log in Finder** (macOS). Open `idea.log` and search for lines containing `copilot` or `MCP`.

---

## JetBrains AI Assistant (IntelliJ IDEA and other JetBrains IDEs)

> **Minimum IDE version**: JetBrains AI Assistant MCP client support requires **IntelliJ IDEA 2025.2** or later. Verify your IDE version under Help → About.

### Configuration

#### Step 1 — Open the MCP settings

Navigate to: **Settings → Tools → AI Assistant → Model Context Protocol (MCP)**

#### Step 2 — Add the server

1. Click the **+** button to add a new server entry.
2. Paste the following JSON into the input dialog:

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

> **Windows**: Replace the `--from` value and `DMXMMSRT_EXECUTABLE` with Windows paths (double backslashes in JSON).

3. Set the **Server level** to **Global** or **Project** as appropriate.
4. Click **OK** → **Apply**.

### Starting the server

JetBrains IDEs start the MCP server process automatically when AI Assistant first invokes a tool. To manually start or restart:

1. Go to **Settings → Tools → AI Assistant → Model Context Protocol (MCP)**.
2. Find the `di-appmod-mcp` entry. Click **Start** (▶) if stopped.
3. A green status indicator confirms the server is connected and ready.

For full documentation see the [JetBrains AI Assistant — MCP Servers](https://www.jetbrains.com/help/idea/mcp-servers.html) help page.

### Log location

**Help → Show Log in Explorer** (Windows) / **Show Log in Finder** (macOS) → look for an `mcp/` subfolder.

---

## Microsoft Visual Studio

> **Minimum version**: Visual Studio 2022 version 17.14 or later (or Visual Studio 2026) with the **GitHub Copilot** extension installed. Visual Studio is Windows-only.

GitHub Copilot in Visual Studio reads MCP server configurations from `.mcp.json` files (note the leading dot).

### Config file locations

| Priority | Path | Scope |
|----------|------|-------|
| 1 | `%USERPROFILE%\.mcp.json` | User-global — applies to all solutions for the current user |
| 2 | `<SOLUTIONDIR>\.mcp.json` | Solution-level — suitable for source control |

### Configuration

Create or open the config file and add:

```json
{
    "servers": {
        "di-appmod-mcp": {
            "type": "stdio",
            "command": "uvx",
            "args": ["--from", "C:\\absolute\\path\\to\\di_appmod_mcp-{version}-py3-none-any.whl", "di-appmod-mcp"],
            "env": {
                "DMXMMSRT_EXECUTABLE": "C:\\Precisely\\AppMod\\dmxmmsrt.exe"
            }
        }
    }
}
```

### Starting the server

After saving `.mcp.json`:

1. Restart Visual Studio (required for config changes to take effect).
2. Open the GitHub Copilot Chat panel (**View → GitHub Copilot Chat**) and switch to **Agent mode**.
3. Click the **Select tools and skills** button in the Copilot chat toolbar. A tools panel opens listing all registered MCP servers and their tools.
4. If `di-appmod-mcp` shows a stopped or disconnected state, click **Start** (or **Restart**) next to it.

For full documentation see [Use MCP servers in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers).

### Log location

**View → Output** → select **GitHub Copilot** from the dropdown.

---

## Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DMXMMSRT_EXECUTABLE` | **Yes** | — | Absolute path to the DMXMMSRT executable. The server exits at startup if not set or not executable. |
| `DMXMMSRT_TIMEOUT_SECONDS` | No | `30` | Maximum seconds to wait for DMXMMSRT to complete per sort step. |
| `DMXMMSRT_LOG_LEVEL` | No | `INFO` | Python logging level: `DEBUG`, `INFO`, `WARNING`, or `ERROR`. |

---

## Verification

After configuring any client:

1. Open the AI chat panel and switch to **Agent mode**.
2. Ask the assistant to check a simple sort step, e.g.:
   > "Check if this sort step is supported: `SORT FIELDS=(1,10,CH,A) END`"
3. A response containing `support_status` confirms the server is connected and working.
4. **If the call fails**:
   - Tools not listed → the server process did not start. Check client logs.
   - `DMXMMSRT_EXECUTABLE` error → the env var is missing or the path is wrong. Run the server in a terminal to see the startup error.
   - `indeterminate` with `not_available` classification → the server started but `DMXMMSRT_EXECUTABLE` was not set at the time it ran the tool (env var may not have been inherited).

---

## Troubleshooting

### Server not appearing in the client's tool list

**Symptom**: The client connects but no tools appear.

**Fix**:
1. Check config JSON syntax — paste into `python -m json.tool` or an online JSON linter.
2. Verify `uvx` is on PATH: `uvx --version` in a terminal.
3. Run the server directly in a terminal to see startup errors (see the distribution README's manual installation section).

### `DMXMMSRT_EXECUTABLE` not found or not executable

**Symptom**: Server exits immediately; tools not available.

**Fix**:
- Verify the path exists and is executable: `ls -la /path/to/dmxmmsrt` (macOS/Linux) or `Test-Path "C:\path\to\dmxmmsrt.exe"` (Windows).
- Ensure the environment variable is visible to the MCP client process, not just your terminal session.

### All results return `indeterminate`

**Symptom**: Every `check_sort_support` call returns `indeterminate` with classification `error`.

**Fix**:
- Check `execution_outcome.exit_code` in the response for the actual exit code.
- Exit codes other than `0` (supported) and `16` (not supported) map to `indeterminate`. This may indicate a DMXMMSRT configuration issue, not an MCP issue.
- A `null` exit code with `timed_out: true` means the process was killed — increase `DMXMMSRT_TIMEOUT_SECONDS`.

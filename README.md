# WSL + Windows Workspace Connector

[English](README.md) | [Português](README.PT.md)

[![CI](https://github.com/denerbatista/conector-wsl/actions/workflows/ci.yml/badge.svg)](https://github.com/denerbatista/conector-wsl/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/claude-wsl-terminal-connector.svg)](https://www.npmjs.com/package/claude-wsl-terminal-connector)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Node](https://img.shields.io/badge/node-%3E%3D20-43853d.svg)](package.json)
[![MCPB](https://img.shields.io/badge/MCPB-0.3-0099ff.svg)](manifest.json)

A local MCP connector for Claude Desktop / Cowork that gives Claude **controlled access to your machine**: WSL terminal, Windows PowerShell, persistent sessions, opening files/URLs in Windows, and filesystem access within the folders you allow. When the client is **Claude Code** (which already has a terminal), it enters **silent mode** and uses no extra context — it can be bundled in a plugin.

## Why this connector

- **Zero config.** Install and it just works — the distro, Linux user, and Windows user are detected automatically.
- **Path sandbox.** Every file and terminal operation is validated against a list of `allowed_roots` (`/home/<you>` and `/mnt/c/Users/<you>` by default).
- **Sessions preserve cwd.** `cd` and exported variables persist between commands in the same session.
- **Cross-host file access.** Access files on WSL and Windows without the UNC-loop error.
- **Two sides, one sandbox.** WSL (bash) and Windows (PowerShell) commands use the same `allowed_roots` list; paths can be `C:\...`, `\\wsl.localhost\...`, `/mnt/c/...`, or `/home/...`.
- **Silent mode.** Detects Claude Code through the MCP handshake and reports status only — plugins can declare it without incurring CLI context costs.

Documentation: [denerbatista.github.io/conector-wsl](https://denerbatista.github.io/conector-wsl/) · Privacy policy: [privacy](https://denerbatista.github.io/conector-wsl/privacidade.html)

## Installation

### Option 1 — Claude plugin (Cowork / Claude Code)

Install the **conector-wsl** plugin from Claude's plugin directory. It declares a local MCP server (`node ${CLAUDE_PLUGIN_ROOT}/src/index.js`), and Claude installs the dependencies from `package-lock.json` (`@modelcontextprotocol/sdk` and `zod`). Node.js 20 or later must be installed on the machine.

To test from the repository in Claude Code:

```bash
claude --plugin-dir ./conector-wsl
```

### Option 2 — Via npm

```bash
npm install -g claude-wsl-terminal-connector
```

The installer detects your OS and automatically adds the connector to `claude_desktop_config.json`. Just **restart Claude Desktop**.

Works on WSL, native Windows, macOS, and Linux.

### Option 3 — Claude Desktop / Cowork via .mcpb

1. Download `conector-wsl.mcpb` from the [latest release](https://github.com/denerbatista/conector-wsl/releases/latest).
2. Drag it into Claude Desktop, or open **Settings → Extensions → Install from file**.
3. Restart Claude Desktop.

### Optional configuration

All fields are optional — the connector detects everything automatically. To override a value:

| Field                         | Auto-detected default                        | When to set it                          |
| ----------------------------- | -------------------------------------------- | --------------------------------------- |
| `default_cwd`                 | `/home/<linux-user>`                         | To start in a different folder          |
| `allowed_roots`               | `/home/<linux-user>:/mnt/c/Users/<win-user>` | To allow more or fewer directories       |
| `wsl_distro`                  | Distro marked with `*` in `wsl --list --verbose` | To select one of multiple distros    |
| `timeout_ms`                  | `120000`                                     | For longer or shorter-running commands  |
| `mode`                        | `auto`                                       | To force `full` or `silent`              |
| `WSL_CONNECTOR_PREWARM` (env) | empty                                        | MCPs to prewarm (`pkg@ver ...`)          |

## Available tools

| Tool                     | Description                                                                    | Hint        |
| ------------------------ | ------------------------------------------------------------------------------ | ----------- |
| `connector_status`       | Shows active config, detected distro, roots, and auto-detection flags.         | readOnly    |
| `list_allowed_roots`     | Lists the allowed roots.                                                       | readOnly    |
| `list_directory`         | Lists files and folders in an allowed directory.                              | readOnly    |
| `get_path_info`          | Gets a path's type, size, and dates.                                           | readOnly    |
| `read_text_file`         | Reads a UTF-8 text file, subject to a size limit.                              | readOnly    |
| `write_text_file`        | Creates or overwrites a UTF-8 text file.                                       | destructive |
| `create_directory`       | Creates a directory (recursive by default).                                   | write       |
| `run_wsl_command`        | Runs a one-off command in a new shell.                                         | destructive |
| `start_wsl_session`      | Starts a persistent session (preserves cwd and exported variables).           | write       |
| `run_in_wsl_session`     | Runs a command in a persistent session.                                       | destructive |
| `close_wsl_session`      | Ends a session.                                                                | write       |
| `run_windows_command`    | Runs PowerShell on Windows in a new shell (`cwd` can be Windows or Linux).     | destructive |
| `start_windows_session`  | Starts a persistent PowerShell session (preserves cwd).                        | write       |
| `run_in_windows_session` | Runs a command in a PowerShell session.                                        | destructive |
| `close_windows_session`  | Ends the PowerShell session.                                                  | write       |
| `open_in_windows`        | Opens a file, folder, or URL in Windows (default app or the specified `app`).  | write       |

The `*_windows_*` tools and `open_in_windows` are available when PowerShell is found: on native Windows or WSL with interop (`/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe`). Override with `WSL_CONNECTOR_POWERSHELL`.

## Modes: full vs. silent

| MCP client (`clientInfo.name`) | Mode   | Tools                                         |
| ------------------------------ | ------ | --------------------------------------------- |
| Claude Desktop / Cowork / other clients | full | all                                      |
| `claude-code`                  | silent | `connector_status` and `list_allowed_roots` only |

Claude Code runs inside WSL and already has Bash — these tools would be redundant and use context in every session. Force a mode with `WSL_CONNECTOR_MODE=full|silent` (environment variable or the "Mode" field in the `.mcpb` config).

### Use in a Claude Code / Cowork plugin

**Recommended — bundled (no npx, no cold start):** build the bundle and copy it into the plugin.

```bash
npm run bundle                      # dist/wsl-connector.mjs (~740 KB, ESM, Node >= 20)
cp dist/wsl-connector.mjs <plugin>/vendor/
```

```json
{
  "mcpServers": {
    "wsl-workspace": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/vendor/wsl-connector.mjs"],
      "env": { "WSL_CONNECTOR_PREWARM": "chrome-devtools-mcp@1.8.0" }
    }
  }
}
```

`WSL_CONNECTOR_PREWARM` (optional) lists other plugin `npx` MCPs that the connector warms in the background on the first start in full mode — prevents Cowork from timing out after 30 seconds while they start with a cold cache.

**Alternative — via npx** (downloads from the network; with a cold cache, startup may exceed Cowork's timeout):

```json
{
  "wsl-workspace": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "claude-wsl-terminal-connector@0.6.0"]
  }
}
```

In Cowork (Windows), it enters full mode based on the platform; in Claude Code (WSL), it enters silent mode.

## How it works

```
Claude Desktop  --stdio-->  node src/index.js
                                 |
                                 +-- detect.js   (distro, Linux user, Windows user)
                                 +-- config.js   (resolveConfig: env > detection)
                                 +-- wsl.js      (spawn wsl.exe -d <distro> -- bash -lc ...)
                                 +-- filesystem  (read/write/list via toHostPath)
                                 +-- sessions    (markers to capture cwd + state)
```

The `filesystem.toHostPath` module converts `/mnt/<letter>/...` paths to `<Letter>:\...` (native Windows paths, without UNC), resolving the classic `EPERM` error under `/mnt/c/`.

## Development

```bash
git clone https://github.com/denerbatista/conector-wsl
cd conector-wsl
npm install

npm test           # vitest
npm run lint       # eslint
npm run format     # prettier --write
npm run package    # creates conector-wsl-X.Y.Z.mcpb
```

Requirements: Node 20+.

## What the connector executes and accesses

So you know exactly what runs on your machine:

- **Processes it starts:** `wsl.exe --exec bash -lc <command>` (on Windows) or `/bin/bash -lc <command>` (on Linux/WSL) for WSL tools, and `powershell.exe` for Windows tools. Commands are those Claude requests through the tools, always with `cwd` inside `allowed_roots`.
- **Files:** Reads and writes only within `allowed_roots`. Paths outside them are rejected.
- **Opening in Windows:** `open_in_windows` uses the Windows default app (or the specified app) to open a file, folder, or URL requested by Claude.
- **Network:** The connector makes no network requests. The only exception is optional: if you set `WSL_CONNECTOR_PREWARM`, it runs `npx -y <package@version> --help` to warm the npm cache for those packages. The plugin does not set this variable.
- **npm installation (Option 2):** `postinstall` adds the connector entry to Claude Desktop's `claude_desktop_config.json`; `preuninstall` removes it. They only act on global installations (`npm install -g`); when the plugin installs dependencies, they do nothing.
- **Telemetry:** None.

## Known limitations

- Does not create a real interactive TTY.
- WSL sessions preserve `cwd` and **exported** variables. Aliases, shell functions, and unexported variables do not persist.
- Windows (PowerShell) sessions preserve only `cwd`.
- Files are treated as UTF-8 text. For binaries, use `run_wsl_command` with `cat`, `cp`, or `mv`.
- `open_in_windows` and the PowerShell tools require Windows or WSL with interop; they are not available on plain Linux.

## Privacy policy

This connector runs **entirely on your local machine**. It does not collect, transmit, or store personal data on external servers.

- **Data collection:** None. No telemetry, analytics, or usage data is sent anywhere.
- **Data access:** The connector reads and writes files only within the `allowed_roots` directories you configure (default: the WSL home and Windows user folder).
- **Sharing with third parties:** None. All operations stay on your machine.
- **Data retention:** No data is persisted beyond the current Claude Desktop session. Session state exists only in memory.
- **Network access:** None. The connector communicates exclusively with Claude Desktop over local stdio — it makes no network requests.
- **Contact:** For questions or concerns, open an issue at [github.com/denerbatista/conector-wsl](https://github.com/denerbatista/conector-wsl/issues).

Full version: [denerbatista.github.io/conector-wsl/privacidade.html](https://denerbatista.github.io/conector-wsl/privacidade.html).

## License

[MIT](LICENSE) © Dener Batista

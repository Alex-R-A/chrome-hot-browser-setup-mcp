# Chrome DevTools MCP Setup for Claude Code

Connect Claude Code to a Chrome browser instance via the Chrome DevTools Protocol (CDP).

## Architecture Overview

Two components are involved:

1. **Chrome's CDP debug port** (e.g., 9222): Exposes browser state over WebSocket
2. **Chrome DevTools MCP server** (`chrome-devtools-mcp`): Communicates with Claude Code over stdio, connects outbound to Chrome's debug port

The MCP server uses stdio to talk to Claude Code. It does not listen on a TCP port itself.

## Prerequisites

- Chrome 136+ (or Chromium-based browser)
- Claude Code CLI installed
- Node.js/npx available

## Critical Limitation: Chrome 136+ Security

Since Chrome 136, the `--remote-debugging-port` flag is **silently ignored** when `--user-data-dir` points to Chrome's default profile location. This is a security hardening measure.

**What this means in practice:**

- You **cannot** use your existing Chrome profile (with saved logins, cookies, extensions) with remote debugging
- The debug instance must use a **separate, fresh profile directory**
- **You will need to manually log in** to any websites you want to automate in the debug instance
- Workarounds like symlinks do not work (Chrome resolves the path before checking)
- Pointing `--user-data-dir` at your actual profile directory (e.g., `~/Library/Application Support/Google/Chrome`) will start Chrome but the debug port will not be exposed

**Tested and confirmed non-working approaches:**

```bash
# DOES NOT WORK - default profile location, debug port silently ignored
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="/Users/you/Library/Application Support/Google/Chrome" \
  --profile-directory="Default"

# DOES NOT WORK - symlink to default profile, Chrome resolves path
ln -s "/Users/you/Library/Application Support/Google/Chrome" ~/.chrome-debug-link
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-debug-link" \
  --profile-directory="Default"
```

**Future workaround (not yet available in stable):**

The MCP's `--autoConnect` flag can attach to a running Chrome session with your existing profile, but it requires `chrome://inspect/#remote-debugging` UI, which is only available in Chrome 145+ beta/canary channels.

## Part 1: Launch Chrome with Remote Debugging

Since we're using a separate `--user-data-dir`, the debug Chrome instance runs independently. Your normal Chrome can stay open.

### Step 1: Create Fresh Profile Directory

**macOS/Linux:**
```bash
mkdir -p "$HOME/.chrome-debug-profile"
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.chrome-debug-profile"
```

### Step 2: Launch Chrome with Debug Port

#### macOS

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-debug-profile" \
  --profile-directory="Default"
```

#### Linux

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.chrome-debug-profile" \
  --profile-directory="Default"
```

#### Windows (PowerShell)

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="$env:USERPROFILE\.chrome-debug-profile" `
  --profile-directory="Default"
```

#### Windows (CMD)

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --remote-debugging-port=9222 ^
  --user-data-dir="%USERPROFILE%\.chrome-debug-profile" ^
  --profile-directory="Default"
```

### Step 3: Verify Debug Port is Exposed

**macOS/Linux:**
```bash
curl http://127.0.0.1:9222/json/version
```

**Windows (PowerShell):**
```powershell
Invoke-RestMethod http://127.0.0.1:9222/json/version
```

**Expected response:**

```json
{
  "Browser": "Chrome/143.0.7499.147",
  "Protocol-Version": "1.3",
  "User-Agent": "Mozilla/5.0 ...",
  "V8-Version": "14.3.127.17",
  "WebKit-Version": "537.36 (...)",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."
}
```

**If no response or connection refused:**

- Chrome is not exposing CDP
- Verify `--user-data-dir` points to a non-default path (not your actual Chrome profile)
- Check port 9222: `lsof -i :9222` (macOS/Linux) or `netstat -an | findstr 9222` (Windows)

### Flag Reference

| Flag | Purpose |
|------|---------|
| `--remote-debugging-port=9222` | Exposes CDP on port 9222 |
| `--user-data-dir=<path>` | Root profile store. **Must be non-default** for debug port to work (Chrome 136+) |
| `--profile-directory=<name>` | Profile within user-data-dir (e.g., "Default", "Profile 1") |

## Part 2: Configure MCP in Claude Code

Claude Code MCP configuration is **different** from Claude Desktop's `claude_desktop_config.json`.

| Scope | File | Use Case |
|-------|------|----------|
| Project-shared | `.mcp.json` at repo root | Commit to repo, shared with team. Everyone needs Chrome running on the same port. |
| User/local | `~/.claude.json` | Personal config not shared with team. Managed via `claude mcp` CLI. |

**Which option to use:**
- **Option A (CLI)**: Easiest. Adds to your personal `~/.claude.json` for current project.
- **Option B (.mcp.json)**: Share with team via git. Everyone can use the MCP.
- **Option C (manual)**: Same as Option A but manual editing instead of CLI.

### Option A: Via Claude Code CLI (Recommended)

The `--` separator is **critical** to prevent Claude's flag parser from consuming the server's arguments:

```bash
claude mcp add --transport stdio chrome-devtools -- \
  npx -y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222
```

This writes to `~/.claude.json` under the current project's settings.

**Important:** After adding the MCP, you must **restart Claude Code** for the new server to be available. The `/mcp` command will show "No MCP servers configured" until you restart.

### Option B: Project .mcp.json (Shared with Team)

Create `.mcp.json` at your repository root:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "chrome-devtools-mcp@latest",
        "--browser-url=http://127.0.0.1:9222"
      ],
      "env": {}
    }
  }
}
```

**Important:** After creating or modifying `.mcp.json`, **restart Claude Code** for changes to take effect.

### Option C: User ~/.claude.json (Manual Edit)

Edit `~/.claude.json` and add to your project's `mcpServers` object:

```json
{
  "projects": {
    "/path/to/your/project": {
      "mcpServers": {
        "chrome-devtools": {
          "type": "stdio",
          "command": "npx",
          "args": [
            "-y",
            "chrome-devtools-mcp@latest",
            "--browser-url=http://127.0.0.1:9222"
          ],
          "env": {}
        }
      }
    }
  }
}
```

**Important:** After modifying `~/.claude.json`, **restart Claude Code** for changes to take effect.

### Configuration Field Reference

| Field | Value | Purpose |
|-------|-------|---------|
| `type` | `"stdio"` | MCP communicates via stdin/stdout with Claude Code |
| `command` | `"npx"` | Package runner |
| `args[0]` | `"-y"` | Auto-confirm npx install prompt (prevents hang on first run) |
| `args[1]` | `"chrome-devtools-mcp@latest"` | The MCP package |
| `args[2]` | `"--browser-url=http://127.0.0.1:9222"` | Connect to existing Chrome's CDP endpoint |

### Common Configuration Mistakes

**args must be an array of separate tokens:**

Wrong (all args as single string):
```json
"args": ["-y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222"]
```

Correct (each arg is separate array element):
```json
"args": ["-y", "chrome-devtools-mcp@latest", "--browser-url=http://127.0.0.1:9222"]
```

**Both flag formats work:**
- `--browser-url=http://127.0.0.1:9222`
- `--browserUrl=http://127.0.0.1:9222`

**Without `--browser-url`:** The MCP launches its own Chrome instance with "controlled by automated test software" banner instead of connecting to your running instance.

### Avoid --mcp-config CLI Flag

The `--mcp-config` CLI flag has known issues (v1.0.73+) where it misparses arguments and fails schema validation. Use `claude mcp add` or manual file editing instead.

### Removing an MCP Server

To remove the chrome-devtools MCP:

```bash
claude mcp remove chrome-devtools
```

Then restart Claude Code.

## Part 3: Verify MCP Connection in Claude Code

After restarting Claude Code:

1. Run `/mcp` to see available MCP servers
2. The chrome-devtools server should be listed
3. Claude can now use `mcp__chrome-devtools__*` tools

**Test the connection:** Ask Claude to list browser pages. Claude will call `mcp__chrome-devtools__list_pages` and return the open tabs in the debug Chrome instance.

## Part 4: Future - Auto-Connect to Existing Session

Chrome DevTools MCP supports an `--autoConnect` flag to attach to a running Chrome session with your existing profile (preserving logins, cookies, etc.).

**Requirements:**
- Chrome 145+ with `chrome://inspect/#remote-debugging` UI enabled
- This UI is currently only available in Chrome beta/canary channels

**Status:** Not available in stable Chrome releases yet.

### Setup (When Available)

1. In Chrome 145+ beta/canary, go to `chrome://inspect/#remote-debugging` and enable remote debugging
2. Configure MCP with `--autoConnect`:

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "chrome-devtools-mcp@latest",
        "--autoConnect"
      ],
      "env": {}
    }
  }
}
```

Chrome will prompt for permission when MCP attempts to connect.

## Part 5: Shell Aliases (macOS/Linux)

Add to `~/.zshrc` or `~/.bashrc`. Chrome executable paths:
- **macOS:** `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`
- **Linux:** `google-chrome` or `/usr/bin/google-chrome`

### Basic Alias (macOS)

```bash
alias chrome-debug="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome --remote-debugging-port=9222 --user-data-dir=\"\$HOME/.chrome-debug-profile\" --profile-directory=\"Default\""
```

### Function with Verification

```bash
# Set CHROME_PATH for your OS
CHROME_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"  # macOS
# CHROME_PATH="google-chrome"  # Linux

chrome-debug() {
  local port="${1:-9222}"

  if lsof -i :"$port" >/dev/null 2>&1; then
    echo "Port $port already in use"
    return 1
  fi

  "$CHROME_PATH" \
    --remote-debugging-port="$port" \
    --user-data-dir="$HOME/.chrome-debug-profile" \
    --profile-directory="Default" &>/dev/null &

  sleep 2

  if curl -s "http://127.0.0.1:$port/json/version" >/dev/null; then
    echo "Chrome ready on port $port"
  else
    echo "Chrome started but debug port not exposed - check --user-data-dir path"
    return 1
  fi
}
```

Usage: `chrome-debug` (default port 9222) or `chrome-debug 9223` (custom port).

## Security Considerations

While the debug port is open, **any local process can connect and control the browser**, including:
- Reading cookies and session tokens
- Accessing page content
- Executing JavaScript
- Navigating to URLs

**Recommendations:**
- Use the debug instance only for automation work, not sensitive browsing
- Close the debug Chrome when not in use
- The fresh profile approach provides some isolation since it has no saved credentials

## Troubleshooting

### Chrome starts but debug port not exposed

The `--user-data-dir` path resolves to Chrome's default profile location. Use the recommended path `$HOME/.chrome-debug-profile` (or `%USERPROFILE%\.chrome-debug-profile` on Windows).

### /mcp shows "No MCP servers configured"

You added the MCP config but didn't restart Claude Code. Exit and restart.

### MCP hangs on first use

Missing `-y` flag in npx args. Without it, npx prompts for install confirmation and blocks.

### MCP launches new browser instead of connecting

Missing `--browser-url` argument. The MCP defaults to launching its own Chrome.

### Connection refused when verifying

Chrome not running, or running on different port. Verify with:

**macOS/Linux:**
```bash
lsof -i :9222
curl http://127.0.0.1:9222/json/version
```

**Windows (PowerShell):**
```powershell
netstat -an | findstr 9222
Invoke-RestMethod http://127.0.0.1:9222/json/version
```

### Profile lock conflict

Only occurs if you accidentally point `--user-data-dir` at your normal Chrome profile. With the recommended fresh profile path (`$HOME/.chrome-debug-profile`), this won't happen.

### Port already in use

Kill existing process:

**macOS/Linux:**
```bash
lsof -ti :9222 | xargs kill
```

**Windows (PowerShell):**
```powershell
Get-NetTCPConnection -LocalPort 9222 | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

## Complete Workflow Summary

1. **Create profile directory** (one-time, see Part 1 Step 1 for your OS):
   ```bash
   mkdir -p "$HOME/.chrome-debug-profile"
   ```

2. **Start Chrome with debug port** (your normal Chrome can stay open). See Part 1 for your OS. macOS example:
   ```bash
   /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
     --remote-debugging-port=9222 \
     --user-data-dir="$HOME/.chrome-debug-profile" \
     --profile-directory="Default"
   ```

3. **Verify CDP endpoint** (macOS/Linux: `curl`, Windows: `Invoke-RestMethod`):
   ```bash
   curl http://127.0.0.1:9222/json/version
   ```

4. **Add MCP to Claude Code (one-time):**
   ```bash
   claude mcp add --transport stdio chrome-devtools -- \
     npx -y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222
   ```

5. **Restart Claude Code** to load the MCP server

6. **Verify in Claude Code:**
   - Run `/mcp` to see chrome-devtools server listed
   - Ask Claude to list browser pages to test connection

## References

- [Chrome DevTools MCP GitHub](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [Chrome Remote Debugging Security Changes (Chrome 136)](https://developer.chrome.com/blog/remote-debugging-port)
- [Chrome DevTools MCP Auto-Connect](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Claude Code Settings](https://code.claude.com/docs/en/settings)
- [Claude Code MCP Documentation](https://code.claude.com/docs/en/mcp)

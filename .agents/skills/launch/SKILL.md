---
name: launch
description: Launch and automate Electron desktop apps (VS Code, Slack, Discord, Figma, Notion, Spotify, etc.) using agent-browser via Chrome DevTools Protocol. Use when the user needs to interact with an Electron app, automate a desktop app, connect to a running app, control a native app, or test an Electron application. Triggers include "automate Slack app", "control VS Code", "interact with Discord app", "test this Electron app", "connect to desktop app", or any task requiring automation of a native Electron application.
allowed-tools: Bash(agent-browser:*), Bash(npx agent-browser:*)
---

# Electron App Automation

Automate any Electron desktop app using agent-browser. Electron apps are built on Chromium and expose a Chrome DevTools Protocol (CDP) port that agent-browser can connect to, enabling the same snapshot-interact workflow used for web pages.

## Prerequisites

- **`agent-browser` must be installed.** It's available in the project's devDependencies — run `npm install` if you haven't already.
- **`code-insiders` is required.** This extension uses 58 proposed VS Code APIs and targets `vscode ^1.110.0-20260223`. VS Code Stable will **not** activate it — you must use VS Code Insiders.
- **The extension must be compiled first.** Use `npm run compile` for a one-shot build, or `npm run watch` for iterative development.
- **CSS selectors are internal implementation details.** Selectors like `.interactive-input-part`, `.monaco-editor`, and `.view-line` are VS Code internals that may change across versions. If automation breaks after a VS Code update, re-snapshot and check for selector changes.

## Core Workflow

1. **Launch** the Electron app with remote debugging enabled
2. **Connect** agent-browser to the CDP port
3. **Snapshot** to discover interactive elements
4. **Interact** using element refs
5. **Re-snapshot** after navigation or state changes

```bash
# Launch an Electron app with remote debugging
open -a "Slack" --args --remote-debugging-port=9222

# Connect agent-browser to the app
agent-browser connect 9222

# Standard workflow from here
agent-browser snapshot -i
agent-browser click @e5
agent-browser screenshot slack-desktop.png
```

## Launching Electron Apps with CDP

Every Electron app supports the `--remote-debugging-port` flag since it's built into Chromium.

### macOS

```bash
# Slack
open -a "Slack" --args --remote-debugging-port=9222

# VS Code
open -a "Visual Studio Code" --args --remote-debugging-port=9223

# Discord
open -a "Discord" --args --remote-debugging-port=9224

# Figma
open -a "Figma" --args --remote-debugging-port=9225

# Notion
open -a "Notion" --args --remote-debugging-port=9226

# Spotify
open -a "Spotify" --args --remote-debugging-port=9227
```

### Linux

```bash
slack --remote-debugging-port=9222
code --remote-debugging-port=9223
discord --remote-debugging-port=9224
```

### Windows

```bash
"C:\Users\%USERNAME%\AppData\Local\slack\slack.exe" --remote-debugging-port=9222
"C:\Users\%USERNAME%\AppData\Local\Programs\Microsoft VS Code\Code.exe" --remote-debugging-port=9223
```

**Important:** If the app is already running, quit it first, then relaunch with the flag. The `--remote-debugging-port` flag must be present at launch time.

## Connecting

```bash
# Connect to a specific port
agent-browser connect 9222

# Or use --cdp on each command
agent-browser --cdp 9222 snapshot -i

# Auto-discover a running Chromium-based app
agent-browser --auto-connect snapshot -i
```

After `connect`, all subsequent commands target the connected app without needing `--cdp`.

## Tab Management

Electron apps often have multiple windows or webviews. Use tab commands to list and switch between them:

```bash
# List all available targets (windows, webviews, etc.)
agent-browser tab

# Switch to a specific tab by index
agent-browser tab 2

# Switch by URL pattern
agent-browser tab --url "*settings*"
```

## Common Patterns

### Inspect and Navigate an App

```bash
open -a "Slack" --args --remote-debugging-port=9222
sleep 3  # Wait for app to start (adjust timing as needed)
# If connect fails, retry:
# for i in 1 2 3 4 5; do agent-browser connect 9222 2>/dev/null && break || sleep 2; done
agent-browser connect 9222
agent-browser snapshot -i
# Read the snapshot output to identify UI elements
agent-browser click @e10  # Navigate to a section
agent-browser snapshot -i  # Re-snapshot after navigation
```

### Take Screenshots of Desktop Apps

```bash
agent-browser connect 9222
agent-browser screenshot app-state.png
agent-browser screenshot --full full-app.png
agent-browser screenshot --annotate annotated-app.png
```

### Extract Data from a Desktop App

```bash
agent-browser connect 9222
agent-browser snapshot -i
agent-browser get text @e5
agent-browser snapshot --json > app-state.json
```

### Fill Forms in Desktop Apps

```bash
agent-browser connect 9222
agent-browser snapshot -i
agent-browser fill @e3 "search query"
agent-browser press Enter
agent-browser wait 1000
agent-browser snapshot -i
```

### Run Multiple Apps Simultaneously

Use named sessions to control multiple Electron apps at the same time:

```bash
# Connect to Slack
agent-browser --session slack connect 9222

# Connect to VS Code
agent-browser --session vscode connect 9223

# Interact with each independently
agent-browser --session slack snapshot -i
agent-browser --session vscode snapshot -i
```

## Color Scheme

Playwright overrides the color scheme to `light` by default when connecting via CDP. To preserve dark mode:

```bash
agent-browser connect 9222
agent-browser --color-scheme dark snapshot -i
```

> **Note:** `--color-scheme dark` only affects CSS media queries (`prefers-color-scheme`). It does **not** change VS Code's internal color theme. To change the actual VS Code theme, use the settings or command palette.

Or set it globally:

```bash
AGENT_BROWSER_COLOR_SCHEME=dark agent-browser connect 9222
```

## Launching VS Code Extensions for Debugging

To debug a VS Code extension via agent-browser, launch VS Code Insiders with `--extensionDevelopmentPath` pointing to your extension source and `--remote-debugging-port` for CDP. Use `--user-data-dir` to avoid conflicting with an already-running VS Code instance.

```bash
# Build the extension first (from the repo root)
npm run compile

# Launch VS Code Insiders with the extension and CDP
code-insiders \
  --extensionDevelopmentPath="$PWD" \
  --remote-debugging-port=9223 \
  --user-data-dir=/tmp/vscode-ext-debug

# Wait for VS Code to start (adjust timing as needed)
sleep 8
# If connect fails, retry:
# for i in 1 2 3 4 5; do agent-browser connect 9223 2>/dev/null && break || sleep 2; done
agent-browser connect 9223
agent-browser snapshot -i
```

**Key flags:**
- `--extensionDevelopmentPath=<path>` — loads your extension from source (must be compiled first). Use `$PWD` when running from the repo root.
- `--remote-debugging-port=9223` — enables CDP (use 9223 to avoid conflicts with other apps on 9222)
- `--user-data-dir=<path>` — uses a separate profile so it starts a new process instead of sending to an existing VS Code instance

**Without `--user-data-dir`**, VS Code detects the running instance, forwards the args to it, and exits immediately — you'll see "Sent env to running instance. Terminating..." and CDP never starts.

> **⚠️ Authentication Warning:** Launching with `--user-data-dir=/tmp/...` creates a fresh profile with **no authentication**. The agent will hit a "Sign in to use Copilot" flow. To avoid this:
> - **Use a persistent user-data-dir** that's already authenticated (e.g., `~/.vscode-ext-debug` instead of `/tmp/...`)
> - **Or sign in manually** before running automation — launch once, sign in through the UI, then subsequent launches with the same `--user-data-dir` will retain the session

## Interacting with Monaco Editor (Chat Input, Code Editors)

VS Code uses Monaco Editor for all text inputs including the Copilot Chat input. Monaco editors appear as textboxes in the accessibility snapshot but require specific agent-browser commands to interact with.

### What Works

#### `type @ref` — The Best Approach

The `type` command with a snapshot ref handles focus and input in one step:

```bash
# Snapshot to find the chat input ref
agent-browser snapshot -i
# Look for: textbox "The editor is not accessible..." [ref=e51]

# Type directly using the ref — handles focus automatically
agent-browser type @e51 "Hello from George!"

# Send the message
agent-browser press Enter

# Wait for the response to complete before re-snapshotting
agent-browser wait 5000
agent-browser snapshot -i
```

This is the simplest and most reliable method. It works for both the main editor chat input and the sidebar chat panel.

#### `keyboard type` / `keyboard inserttext` — After Focus

If focus is already on a Monaco editor, `keyboard type` and `keyboard inserttext` both work:

```bash
# Focus first (via type @ref, or JS mouse events, or a prior interaction)
agent-browser type @e51 ""
# Then keyboard type works for subsequent input
agent-browser keyboard type "More text here"
agent-browser keyboard inserttext "And this too"
```

#### `press` — Individual Keystrokes

Always works when focus is on a Monaco editor. Useful for special keys and keyboard shortcuts:

```bash
# Select all
# macOS:
agent-browser press Meta+a
# Linux / Windows:
agent-browser press Control+a

agent-browser press Backspace    # Delete selection
agent-browser press Enter        # Send message / new line

# Send to new chat
# macOS:
agent-browser press Meta+Shift+Enter
# Linux / Windows:
agent-browser press Control+Shift+Enter
```

### What Does NOT Work

| Method | Result | Reason |
|--------|--------|--------|
| `click @ref` | "Element blocked by another element" | Monaco overlays a transparent div over the textarea |
| `fill @ref "text"` | "Element not found or not visible" | The textbox is not a standard fillable input |
| `document.execCommand("insertText")` via `eval` | No effect | Monaco intercepts and discards execCommand |
| Setting `textarea.value` + dispatching `input` event via `eval` | No effect | Monaco doesn't read from the textarea's value property |

### Fallback: Focus via JavaScript Mouse Events

If `type @ref` doesn't work (e.g., ref is stale), you can focus the editor via JavaScript:

```bash
agent-browser eval '
(() => {
  const inputPart = document.querySelector(".interactive-input-part");
  const editor = inputPart.querySelector(".monaco-editor");
  const rect = editor.getBoundingClientRect();
  const x = rect.x + rect.width / 2;
  const y = rect.y + rect.height / 2;
  editor.dispatchEvent(new MouseEvent("mousedown", { bubbles: true, clientX: x, clientY: y }));
  editor.dispatchEvent(new MouseEvent("mouseup", { bubbles: true, clientX: x, clientY: y }));
  editor.dispatchEvent(new MouseEvent("click", { bubbles: true, clientX: x, clientY: y }));
  return "activeElement: " + document.activeElement?.className;
})()'

# After JS focus, keyboard type and press work
agent-browser keyboard type "Text after JS focus"
```

After JS mouse events, `document.activeElement` becomes a `DIV` with class `native-edit-context` — this is VS Code's native text editing surface.

### Verifying Text in Monaco

Monaco renders text in `.view-line` elements, not the textarea:

```bash
agent-browser eval '
(() => {
  const inputPart = document.querySelector(".interactive-input-part");
  return Array.from(inputPart.querySelectorAll(".view-line")).map(vl => vl.textContent).join("|");
})()'
```

### Clearing Monaco Input

```bash
# macOS:
agent-browser press Meta+a
# Linux / Windows:
agent-browser press Control+a

agent-browser press Backspace
```

## Troubleshooting

### "Connection refused" or "Cannot connect"

- Make sure the app was launched with `--remote-debugging-port=NNNN`
- If the app was already running, quit and relaunch with the flag
- Check that the port isn't in use by another process: `lsof -i :9222`

### App launches but connect fails

- Wait a few seconds after launch before connecting (`sleep 3`)
- Some apps take time to initialize their webview

### Elements not appearing in snapshot

- The app may use multiple webviews. Use `agent-browser tab` to list targets and switch to the right one
- Use `agent-browser snapshot -i -C` to include cursor-interactive elements (divs with onclick handlers)

### Cannot type in input fields

- Try `agent-browser keyboard type "text"` to type at the current focus without a selector
- Some Electron apps use custom input components; use `agent-browser keyboard inserttext "text"` to bypass key events
- **For Monaco Editor inputs (VS Code, GitHub Desktop):** Standard `click` and `fill` don't work — see the "Interacting with Monaco Editor" section above for the full compatibility matrix. In summary: `type @ref` is the best approach on Insiders; individual `press` commands work everywhere; `keyboard type` and `keyboard inserttext` work after focus is established.

## Cleanup / Disconnect

```bash
# Disconnect agent-browser from the current session
agent-browser close

# The VS Code (or other Electron app) instance continues running
```

To fully stop, quit the Electron app separately (e.g., `Cmd+Q` / `Alt+F4`, or kill the process).

## Supported Apps

Any app built on Electron works, including:

- **Communication:** Slack, Discord, Microsoft Teams, Signal, Telegram Desktop
- **Development:** VS Code, GitHub Desktop, Postman, Insomnia
- **Design:** Figma, Notion, Obsidian
- **Media:** Spotify, Tidal
- **Productivity:** Todoist, Linear, 1Password

If an app is built with Electron, it supports `--remote-debugging-port` and can be automated with agent-browser.

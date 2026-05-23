---
name: chrome-devtools
description: Uses Chrome DevTools via CLI for efficient debugging, troubleshooting and browser automation. Use when debugging web pages, automating browser interactions, analyzing performance, or inspecting network requests.
---

The `chrome-devtools` CLI lets you interact with the browser from your terminal using Bash tool.

## Setup

_Note: If this is your very first time using the CLI, install globally first (one-time):_

```sh
npm i chrome-devtools-mcp@latest -g
chrome-devtools status # check if install worked
```

## AI Workflow

1. **Execute**: Run tools directly via Bash (e.g., `chrome-devtools list_pages`). The background server starts implicitly; **do not** run `start`/`status`/`stop` before each use.
2. **Inspect**: Use `take_snapshot` to get an element `<uid>`.
3. **Act**: Use `click`, `fill`, etc. State persists across commands.

Snapshot example:

```
uid=1_0 RootWebArea "Example Domain" url="https://example.com/"
  uid=1_1 heading "Example Domain" level="1"
```

## Core Concepts

**Browser lifecycle**: Browser starts automatically on first CLI call using a persistent Chrome profile. Use `chrome-devtools start --help` to see configuration options (e.g., `--categoryExtensions` to enable extensions).
**Page selection**: Tools operate on the currently selected page. Use `list_pages` to see available pages, then `select_page` to switch context.
**Element interaction**: Use `take_snapshot` to get page structure with element `uid`s. Each element has a unique `uid` for interaction. If an element isn't found, take a fresh snapshot.

## Command Usage

```sh
chrome-devtools <tool> [arguments] [flags]
```

Use `--help` on any command. Output defaults to Markdown, use `--output-format=json` for JSON.

## Input Automation (<uid> from snapshot)

```bash
chrome-devtools take_snapshot                                        # Get UIDs for elements
chrome-devtools take_snapshot --help                                 # Help for any command
chrome-devtools click "id"                                           # Click element
chrome-devtools click "id" --dblClick true --includeSnapshot true    # Double click + snapshot
chrome-devtools drag "src" "dst"                                     # Drag element
chrome-devtools fill "id" "text"                                     # Type text into input
chrome-devtools fill "id" "text" --includeSnapshot true              # Fill + snapshot
chrome-devtools handle_dialog accept                                 # Handle browser dialog
chrome-devtools handle_dialog dismiss --promptText "hi"              # Dismiss with prompt text
chrome-devtools hover "id"                                           # Hover element
chrome-devtools press_key "Enter"                                    # Press key
chrome-devtools press_key "Control+A" --includeSnapshot true         # Press key + snapshot
chrome-devtools type_text "hello"                                    # Type text into focused input
chrome-devtools type_text "hello" --submitKey "Enter"                # Type + submit
chrome-devtools upload_file "id" "file.txt"                          # Upload file
```

## Navigation

```bash
chrome-devtools list_pages                                           # List open pages
chrome-devtools select_page 1                                        # Select page by index
chrome-devtools select_page 1 --bringToFront true                    # Select + bring to front
chrome-devtools navigate_page --url "https://example.com"            # Navigate to URL
chrome-devtools navigate_page --type "reload" --ignoreCache true     # Reload ignoring cache
chrome-devtools navigate_page --type "back"                          # Navigate back
chrome-devtools new_page "https://example.com"                       # Open new page
chrome-devtools new_page "https://example.com" --background true     # Open in background
chrome-devtools close_page 1                                         # Close page by index
```

## Emulation

```bash
chrome-devtools emulate --networkConditions "Offline"                # Emulate network
chrome-devtools emulate --colorScheme "dark" --viewport "1920x1080"  # Dark mode + viewport
chrome-devtools emulate --cpuThrottlingRate 4                        # CPU throttling
chrome-devtools resize_page 1920 1080                                # Resize window
```

## Performance

```bash
chrome-devtools performance_start_trace true false                   # Start performance trace
chrome-devtools performance_stop_trace                               # Stop trace
chrome-devtools performance_stop_trace --filePath "t.json"           # Stop + save to file
chrome-devtools performance_analyze_insight "1" "LCPBreakdown"       # Analyze insight
chrome-devtools take_memory_snapshot "./snap.heapsnapshot"            # Memory snapshot
```

## Network

```bash
chrome-devtools list_network_requests                                # List all requests
chrome-devtools list_network_requests --pageSize 50 --pageIdx 0      # With pagination
chrome-devtools list_network_requests --resourceTypes Fetch           # Filter by type
chrome-devtools get_network_request                                  # Get selected request
chrome-devtools get_network_request --reqid 1 --requestFilePath r.md # Get by id + save
```

## Debugging & Inspection

```bash
chrome-devtools evaluate_script "() => document.title"               # Evaluate JS
chrome-devtools evaluate_script "(a) => a.innerText" --args 1_4      # JS with UID args
chrome-devtools list_console_messages                                # List console messages
chrome-devtools list_console_messages --types error                  # Filter by type
chrome-devtools get_console_message 1                                # Get message by ID
chrome-devtools take_screenshot                                      # Screenshot viewport
chrome-devtools take_screenshot --fullPage true --filePath "s.png"   # Full page screenshot
chrome-devtools take_screenshot --uid "id" --filePath "s.png"        # Element screenshot
chrome-devtools take_snapshot --verbose true --filePath "s.txt"      # Verbose snapshot to file
chrome-devtools lighthouse_audit --mode "navigation"                 # Lighthouse audit
```

## Extensions

```bash
chrome-devtools list_extensions                                      # List installed extensions
chrome-devtools install_extension "/path/to/extension"               # Install extension
chrome-devtools uninstall_extension "extension_id"                   # Uninstall extension
chrome-devtools reload_extension "extension_id"                      # Reload extension
chrome-devtools trigger_extension_action "extension_id"              # Trigger extension action
```

## Workflow Patterns

### Before interacting with a page

1. Navigate: `chrome-devtools navigate_page --url "..."` or `chrome-devtools new_page "..."`
2. Snapshot: `chrome-devtools take_snapshot` to understand page structure
3. Interact: Use element `uid`s from snapshot for `click`, `fill`, etc.

### Efficient data retrieval

- Use `--filePath` for large outputs (screenshots, snapshots, traces)
- Use pagination (`--pageIdx`, `--pageSize`) and filtering (`--resourceTypes`, `--types`) to minimize data
- Set `--includeSnapshot false` on input actions unless you need updated page state

### Tool selection

- **Automation/interaction**: `take_snapshot` (text-based, faster, better for automation)
- **Visual inspection**: `take_screenshot` (when user needs to see visual state)
- **Additional details**: `evaluate_script` for data not in accessibility tree

## Service Management

```bash
chrome-devtools start   # Start or restart
chrome-devtools status  # Check if running
chrome-devtools stop    # Stop service
```

## Troubleshooting

If there are errors launching Chrome, refer to https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/main/docs/troubleshooting.md.

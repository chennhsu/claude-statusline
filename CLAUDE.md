# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

claude-statusline is an npm package (`@kamranahmedse/claude-statusline`) that provides a customizable status line for Claude Code CLI. It displays real-time model info, context window usage, rate limits, git branch, session duration, and effort level. Zero runtime dependencies.

## Architecture

Two files do all the work:

- **`bin/install.js`** — Node.js CLI entry point (`npx` target). Handles install/uninstall: checks for required system tools (jq, curl, git), copies statusline.sh to `~/.claude/statusline.sh`, and configures `~/.claude/settings.json` with a `statusLine` command entry. Supports `--uninstall` flag to restore previous config from backup.

- **`bin/statusline.sh`** — Bash script invoked by Claude Code on each status line render. Reads JSON context from stdin, extracts model/context/session data via `jq`, fetches rate limit data from `https://api.anthropic.com/api/oauth/usage` (with 60s file cache at `/tmp/claude/statusline-usage-cache.json`), and outputs formatted terminal text with RGB color codes.

### statusline.sh data flow

1. Claude Code pipes JSON context (model, context window, session, cwd) to stdin
2. Script extracts fields with `jq` and computes context usage percentage
3. Reads effort level from `~/.claude/settings.json`
4. Resolves OAuth token (env var → macOS Keychain → credentials file → Linux secret-tool)
5. Fetches/caches usage data from Anthropic API
6. Outputs: Line 1 (model, context %, directory/branch, session timer, effort) + rate limit bars (current 5h, weekly 7d, extra usage if enabled)

### Platform compatibility

The script handles macOS vs Linux differences for `date` commands (`date -j` vs `date -d`) and `stat` (`stat -f` vs `stat -c`). Both paths must be maintained.

## Development

No build step, test suite, or linter. To test changes:

```bash
# Test the installer
node bin/install.js
node bin/install.js --uninstall

# Test the statusline script directly with sample JSON input
echo '{"model":{"display_name":"Claude Sonnet 4"},"context_window":{"context_window_size":200000,"current_usage":{"input_tokens":50000,"cache_creation_input_tokens":0,"cache_read_input_tokens":0}},"cwd":"/tmp","session":{"start_time":"2025-01-01T00:00:00Z"}}' | bash bin/statusline.sh
```

## Publishing

```bash
npm publish --access public
```

## Key conventions

- Colors use RGB escape sequences (`\033[38;2;R;G;Bm`), not 256-color or named ANSI codes
- Progress bars use `●` (filled) and `○` (empty) characters
- Token counts format as raw number, `Xk`, or `X.Xm` depending on magnitude
- Usage percentages color-code: green (<50%), orange (50-69%), yellow (70-89%), red (90%+)

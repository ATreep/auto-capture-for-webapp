# auto-capture-for-webapp

Take screenshots for your Web app automatically using Playwright MCP.

## Installation

```bash
# Add the plugin marketplace
/plugin marketplace add ATreep/auto-capture-for-webapp

# Install the plugin
/plugin install auto-capture-for-webapp
```

## Prerequisites

The `take-screenshots` skill requires the Playwright MCP server:

- **Playwright MCP** — `npx @anthropic/mcp-playwright`

If it is not available, the skill will report the error and stop.

## Skills

### take-screenshots

Capture viewport screenshots of web applications using MCP browser tools. The skill enforces read-only, single-threaded screenshot capture with no data modification.

**What it does:**
- Detects available MCP tools (Playwright MCP only)
- Captures viewport screenshots at 1920×1200 resolution for all specified pages
- Takes accessibility snapshots and verifies each screenshot's content matches expectations
- Produces a structured screenshot report listing successes and failures
- Rejects non-web projects (CLI apps, desktop apps) with a clear error

**Key constraints:**
- MCP tools only — no Playwright/Puppeteer/Selenium scripts
- Read-only — never clicks delete, submit, save, or modifies data
- Single-thread — processes pages sequentially, never dispatches parallel sub-agents
- Viewport-only — one screenshot per page at 1920×1200, no full-page or multi-position captures
- Always verifies — checks snapshot content after every screenshot to confirm the correct page was captured

**Usage:** Just ask Claude to take screenshots:

> "Take screenshots of the login page, dashboard, and settings page from http://localhost:3000"

> "Screenshot all pages in the admin section and save to ./docs/screenshots/"

## License

MIT

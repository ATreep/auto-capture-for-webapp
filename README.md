# auto-capture-for-webapp

Take screenshots for your Web app automatically using Playwright MCP or Chrome DevTools MCP.

## Installation

```bash
# Add the plugin marketplace
/plugin marketplace add ATreep/auto-capture-for-webapp

# Install the plugin
/plugin install auto-capture-for-webapp
```

## Prerequisites

The `take-screenshots` skill requires at least one browser automation MCP server:

- **Chrome DevTools MCP** (preferred) — `npx chrome-devtools-mcp@latest`
- **Playwright MCP** (fallback) — `npx @anthropic/mcp-playwright`

If neither is available, the skill will report the error and stop.

## Skills

### take-screenshots

Capture full-page screenshots of web applications using MCP browser tools. The skill enforces read-only, single-threaded screenshot capture with no data modification.

**What it does:**
- Detects available MCP tools (Chrome DevTools MCP preferred, Playwright MCP fallback)
- Captures full-page screenshots (not just the viewport) for all specified pages
- Produces a structured screenshot report listing successes and failures
- Rejects non-web projects (CLI apps, desktop apps) with a clear error

**Key constraints:**
- MCP tools only — no Playwright/Puppeteer/Selenium scripts
- Read-only — never clicks delete, submit, save, or modifies data
- Single-thread — processes pages sequentially, never dispatches parallel sub-agents
- Full-page — every screenshot uses `fullPage: true` to capture all content

**Usage:** Just ask Claude to take screenshots:

> "Take screenshots of the login page, dashboard, and settings page from http://localhost:3000"

> "Screenshot all pages in the admin section and save to ./docs/screenshots/"

## License

MIT

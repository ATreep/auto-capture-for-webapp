---
name: take-screenshots
description: Use when the user asks to take screenshots of a web application, capture specific pages or UI views, or document frontend states. Triggers on requests like "take screenshots of", "screenshot the login page", "capture the dashboard", or similar screenshot/capture requests for web apps.
---

# Take Screenshots

## Overview

Take screenshots of web applications using MCP browser automation tools. **Chrome DevTools MCP is preferred; Playwright MCP is the fallback.** The skill enforces safe, read-only screenshot capture without modifying application data, using only the main conversation thread.

**Core principle:** Screenshots are read-only operations. Never modify data, never use scripts, and never delegate to sub-agents.

## Prerequisites — MCP Detection (MANDATORY)

Before ANY screenshot work, check MCP availability:

### Step 1: Detect Available MCP Tools

Check in this order:

1. **Chrome DevTools MCP** — Look for `mcp__plugin_*_chrome-devtools__*` tools (specifically `take_screenshot` and `take_snapshot`). If these exist, Chrome DevTools MCP is available.
2. **Playwright MCP** — Look for `mcp__plugin_*_playwright__*` tools (specifically `browser_take_screenshot` and `browser_snapshot`). If these exist, Playwright MCP is available.

### Step 2: Decide Which to Use

```
Chrome DevTools MCP available?
├── YES → Use Chrome DevTools MCP (PREFERRED)
└── NO → Playwright MCP available?
    ├── YES → Use Playwright MCP
    └── NO → STOP. Report to user:
              "Neither Chrome DevTools MCP nor Playwright MCP is available.
              Please install one of these MCP servers to use this skill."
              DO NOT proceed. DO NOT write scripts. DO NOT use CLI tools.
```

**CRITICAL:** If neither MCP is available, report the error and STOP. Never fall back to writing Playwright/Puppeteer/Selenium scripts.

## Workflow

### 1. Understand the Target

Determine what to screenshot. The user must provide ONE of:

- **Source code** of a web project (you can start the dev server)
- **A URL** of a running web application (e.g., `http://localhost:3000/login`)

**Reject immediately if:**
- The project is a CLI application, desktop app, mobile app, or any non-web project
- The project is a backend-only service with no frontend UI

Error message: `"This appears to be a [type] project, not a web frontend. This skill only supports taking screenshots of web applications (projects with a browser-based UI)."`

### 2. Prepare Screenshot Directory

Create a dedicated directory for screenshots. Default: `./screenshots/` in the project root.

**This directory MUST NOT be deleted** during or after the screenshot process. If it already exists, reuse it (don't delete existing contents).

### 3. Capture Screenshots (SEQUENTIAL, MAIN THREAD ONLY)

For each page to screenshot, follow this procedure:

#### A. Navigate to the Page

Use the MCP navigation tool to go to the target URL:
- Chrome DevTools MCP: `navigate_page` (or `new_page` for the first page)
- Playwright MCP: `browser_navigate`

**Wait** for the page to fully load before proceeding.

#### B. Wait for Content (if needed)

If the page has dynamic content, use the MCP wait tool:
- Chrome DevTools MCP: `wait_for`
- Playwright MCP: `browser_wait_for`

Wait for key text or elements to appear before capturing.

#### C. Take the Screenshot — FULL PAGE (MANDATORY)

**Every screenshot MUST capture the entire page, not just the visible viewport.** This ensures long pages (documentation, lists, dashboards with scrolled content, changelogs, etc.) are fully captured.

- Chrome DevTools MCP: `take_screenshot` with `fullPage: true` and a descriptive `filename`
- Playwright MCP: `browser_take_screenshot` with `full_page: true` and a descriptive `filename`

**No exceptions — always use full page capture:**
- Don't skip because "this page isn't that long"
- Don't skip because "this page type looks better viewport-only"
- Don't skip because "the important content fits in the viewport"
- Don't skip for dashboards, landing pages, or any other page type
- Don't fall back to viewport capture for any reason

Save to the screenshot directory with clear, descriptive filenames (e.g., `login-page.png`, `dashboard.png`, `settings-profile.png`).

#### D. Take a Snapshot (Optional but Recommended)

Also capture an accessibility snapshot for documentation:
- Chrome DevTools MCP: `take_snapshot`
- Playwright MCP: `browser_snapshot`

This provides text context for what's visible in the screenshot.

**IMPORTANT:** Process pages ONE AT A TIME, sequentially. Do NOT parallelize.

### 4. Read-Only Operations Only

During the entire screenshot process:

- **ALLOWED:** Navigate, scroll, take screenshots, take snapshots, fill in forms (for navigation only), click navigation links
- **FORBIDDEN:** Clicking delete/submit/save/update buttons, modifying form data and submitting, deleting records, changing settings, or any action that modifies data

If the user asks you to perform a destructive action during screenshots, **refuse immediately** — do not defer, do not plan for it, do not say "I'll do it once X is ready." Respond:

```
Response: "I cannot perform [describe the destructive action] because it would modify
data in the application. This skill only performs read-only screenshot operations.
I can take screenshots of the confirmation dialog/page if you navigate there yourself."
```

**No exceptions — refuse destructive operations even when:**
- MCP is working perfectly
- The user insists "it's just a test account"
- You're "planning what to do later"
- The delete button is right there and "it would be faster to just click it"

### 5. Produce Screenshot Report

After ALL screenshots are captured, produce a report:

```markdown
## Screenshot Report

### Successfully Captured

| # | Screenshot | File Path | Description |
|---|-----------|-----------|-------------|
| 1 | Login page | `./screenshots/login.png` | Login form with email/password fields |
| 2 | Dashboard | `./screenshots/dashboard.png` | Main dashboard with stats overview |
| ... | ... | ... | ... |

### Failed Screenshots (if any)

| # | Target | Reason | Suggestion |
|---|--------|--------|------------|
| 1 | /admin/users | Page returned 404 | Verify the route exists and the app is running |
| ... | ... | ... | ... |
```

If any screenshots failed, explain why and suggest fixes.

## Restrictions (HARD RULES)

These restrictions MUST be followed. Violating any of them is an error.

### Thread Safety

| Rule | Detail |
|------|--------|
| **Single-thread only** | Only ONE thread (the main conversation) may use MCP for screenshots |
| **No parallel agents** | Do NOT dispatch sub-agents to take screenshots in parallel |
| **Sub-agents CANNOT use MCP** | If you are a sub-agent, report that you cannot call MCP and stop |

### Tool Restrictions

| Rule | Detail |
|------|--------|
| **MCP only** | Only use MCP tools (`mcp__*__take_screenshot`, `mcp__*__browser_take_screenshot`) |
| **NO scripts** | Do NOT write or run Playwright/Puppeteer/Selenium scripts via Bash or any other method |
| **NO CLI tools** | Do NOT use `npx playwright`, `puppeteer`, or similar CLI screenshot tools |

### Data Safety

| Rule | Detail |
|------|--------|
| **Read-only** | Never modify application data during screenshot capture |
| **Refuse destructive ops** | Reject any request to delete, update, create, or submit data |
| **Navigation only** | Only navigate, scroll, and view pages — never change state |

### Environment

| Rule | Detail |
|------|--------|
| **Web only** | Only web frontend applications. Reject CLI apps, desktop apps, mobile apps, APIs |
| **Preserve screenshot dir** | Never delete the screenshot output directory |
| **Preserve existing files** | Don't delete existing screenshots when adding new ones |

## Common Mistakes

| Mistake | Why It Happens | Correct Approach |
|---------|---------------|-----------------|
| Writing a Playwright script when MCP fails | "MCP isn't working, let me use the library directly" | Report the error and STOP. Do not fall back to scripts. |
| Dispatching parallel sub-agents for speed | "6 pages = 6 parallel agents = faster" | Process pages sequentially in the main thread only. |
| Clicking "Delete" to show the confirmation dialog | "The user asked me to show the delete flow" | Refuse. Only capture read-only views. |
| Creating HTML wrappers for CLI apps | "I'll make a web view of the CLI output" | Reject immediately. CLI apps are not web frontends. |
| Deleting the screenshots folder before starting fresh | "Clean slate is better" | Never delete the screenshots directory. |
| Using `npx playwright screenshot` CLI | "It's faster than MCP navigation" | Only use MCP tools. Never use CLI/script approaches. |
| Planning destructive ops "for later" | "I'll do it once MCP is connected/stabilized" | Refuse at planning time, not at execution time. Never include destructive steps in your plan. |
| Using viewport-only capture | "This page isn't that long" / "Dashboards look better viewport-only" / "The important content fits" | Always use `fullPage: true`. Never make judgment calls about which pages "deserve" full-page capture. |

## Red Flags — STOP Immediately

- "Let me write a quick script to do this..."
- "I'll dispatch parallel agents for each page..."
- "MCP isn't connected, I'll use the Playwright library directly..."
- "This CLI app can be screenshotted if I create an HTML wrapper..."
- "Let me delete the existing screenshots folder first..."
- "Let me click Delete to show you the confirmation dialog..."
- "Once MCP is connected, I'll click Delete and take the screenshot..."
- "Step 3: Click Delete. Step 4: Take screenshot..." (in your plan)
- "This page isn't that long, viewport is fine..."
- "Dashboards look better without fullPage..."
- "The important content fits in the viewport..."

**All of these mean: STOP. Re-read this skill. Follow the correct workflow.**

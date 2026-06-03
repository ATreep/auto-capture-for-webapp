---
name: take-screenshots
description: Use when the user asks to take screenshots of a web application, capture specific pages or UI views, or document frontend states. Triggers on requests like "take screenshots of", "screenshot the login page", "capture the dashboard", or similar screenshot/capture requests for web apps.
---

# Take Screenshots

## Overview

Take screenshots of web applications using Playwright MCP browser automation tools. The skill enforces safe, read-only screenshot capture without modifying application data, using only the main conversation thread.

**Core principles:**
1. Screenshots are read-only operations. Never modify data, never use scripts, and never delegate to sub-agents.
2. **One screenshot per page.** Each user demand gets exactly one viewport screenshot. No scrolling captures, no multi-position captures, no full-page captures.
3. **Always use 1920×1200 viewport.** Set the browser to 1920×1200 before any screenshot to ensure consistent, readable output.
4. **Always verify results.** After every screenshot, double-check the MCP-returned content (via snapshot) to ensure the webpage actually shows what the user expected — not an error page, login gate, or empty state.

## Prerequisites — MCP Detection (MANDATORY)

Before ANY screenshot work, check MCP availability:

### Step 1: Detect Available MCP Tools

Check for Playwright MCP tools:

1. **Playwright MCP** — Look for `mcp__plugin_*_playwright__*` tools (specifically `browser_take_screenshot` and `browser_snapshot`). If these exist, Playwright MCP is available.

### Step 2: Decide Whether to Proceed

```
Playwright MCP available?
├── YES → Use Playwright MCP
└── NO → STOP. Report to user:
          "Playwright MCP is not available.
          Please install the Playwright MCP server to use this skill."
          DO NOT proceed. DO NOT write scripts. DO NOT use CLI tools.
```

**CRITICAL:** If Playwright MCP is not available, report the error and STOP. Never fall back to writing Playwright/Puppeteer/Selenium scripts or using other browser automation tools.

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

### 3. Set Viewport to 1920×1200 (MANDATORY)

Before capturing any screenshot, resize the browser viewport to **1920×1200**:

- **Playwright MCP:** Use `browser_resize` with `width: 1920, height: 1200`

This ensures consistent, readable screenshots at a standard desktop resolution.

**No exceptions:**
- Don't skip this step because "the default size is fine"
- Don't use a different resolution "to match the user's screen"
- Don't resize between pages — keep 1920×1200 for all captures in the session

### 4. Capture Screenshot (SEQUENTIAL, MAIN THREAD ONLY)

For each page to screenshot, follow this procedure:

#### A. Navigate to the Page

Use the Playwright MCP navigation tool to go to the target URL:
- `browser_navigate` to navigate to a URL
- `browser_tabs` with `action: "new"` to open a new tab for the first page

**Wait** for the page to fully load before proceeding.

#### B. Wait for Content (if needed)

If the page has dynamic content, use the Playwright MCP wait tool:
- `browser_wait_for`

Wait for key text or elements to appear before capturing.

#### C. Take the Screenshot

Take exactly ONE viewport screenshot per page:

- `browser_take_screenshot` (viewport only, do NOT use `full_page: true`)

Save as: `{num}-{description}.png` (e.g., `1-login-page.png`, `2-dashboard.png`, `3-settings-profile.png`)

**CRITICAL: One screenshot per page. Always viewport-only.** Do not use full_page. Do not take multiple screenshots at different scroll positions. The 1920×1200 viewport captures what matters.

#### D. Take a Snapshot (MANDATORY for Verification)

Capture an accessibility snapshot for content verification:
- `browser_snapshot`

This provides text content to verify the screenshot captured the correct page. **This step is mandatory** — you cannot verify content without it.

#### E. Verify Screenshot Content (MANDATORY)

**CRITICAL: After every screenshot, double-check the MCP results to ensure the webpage content satisfies the user's demand.** Do NOT blindly trust that the screenshot captured the right page. MCP operations can silently fail — the page may redirect to a login screen, show an error page, display empty data, or render partially.

**Verification checklist (run after every screenshot):**

1. **Check the page URL/title** — Does the snapshot text show the expected page name or heading?
2. **Check for error states** — Look for error messages, "404", "500", "Access Denied", "Loading failed", blank pages, or unexpected redirects
3. **Check for auth gates** — If expecting a logged-in page, verify the snapshot shows actual content (not "Please login", a login form, or an auth error)
4. **Check for empty/dummy data** — If expecting a list, table, or data view, verify the rows/cards actually contain data (not "No data", "Empty", or skeleton placeholders)
5. **Check for the expected UI elements** — Match against the user's described target: if the user asked for "dashboard with stats cards", verify stats cards appear in the snapshot text; if they asked for "settings page with profile form", verify profile form fields are present

**If verification FAILS (content does not match expectations):**

1. **Diagnose the problem** — What did the page actually show? Is it a login redirect, error, empty state, or wrong page?
2. **Fix the issue** — If it's a login gate, navigate to login first and authenticate. If it's a wrong URL, correct the route. If it's empty data, seed data or navigate to a populated view.
3. **Re-capture** — After fixing, re-take the screenshot AND re-verify.
4. **Report the issue** — If the problem cannot be fixed (e.g., requires backend data that doesn't exist), note it in the final report as a partial capture with explanation.

**Example verification failures and actions:**

| Snapshot Shows | Expected | Action |
|---------------|----------|--------|
| Login form / "请登录" | Dashboard with charts | Navigate to login → authenticate → re-navigate to target → re-capture |
| "404 Not Found" | User profile page | Check route correctness, fix URL → re-capture |
| "No data available" | Table with 20 rows | Seed test data or navigate to a populated view → re-capture |
| Blank white page | Full page content | Wait longer for JS to load, or check for JS errors → re-capture |
| Error "500 Internal Server" | Any page | Report as capture failure, note server error |
| Wrong page title/heading | Specific module page | Verify the navigation steps, correct route → re-capture |

**If verification PASSES:** Proceed to the next screenshot. Note in your response what you verified (e.g., "Confirmed: snapshot shows 'Dashboard' heading, 4 stat cards with data, and recent activity table with 10 rows").

**IMPORTANT:** Process pages ONE AT A TIME, sequentially. Do NOT parallelize.

### 5. Read-Only Operations Only

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

### 6. Produce Screenshot Report

After ALL screenshots are captured, produce a report:

```markdown
## Screenshot Report

### Successfully Captured

| # | Screenshot | File Path | Description | Verified |
|---|-----------|-----------|-------------|----------|
| 1 | Login page | `./screenshots/1-login-page.png` | Login form with username/password fields | ✅ Heading "登录", form fields present |
| 2 | Dashboard | `./screenshots/2-dashboard.png` | Main dashboard with stats overview | ✅ "仪表盘" title, 4 stat cards, data table |
| ... | ... | ... | ... | ... |

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
| **MCP only** | Only use Playwright MCP tools (`mcp__*__browser_take_screenshot`, `mcp__*__browser_snapshot`) |
| **NO scripts** | Do NOT write or run Playwright/Puppeteer/Selenium scripts via Bash or any other method |
| **NO CLI tools** | Do NOT use `npx playwright`, `puppeteer`, or similar CLI screenshot tools |

### Data Safety

| Rule | Detail |
|------|--------|
| **Read-only** | Never modify application data during screenshot capture |
| **Refuse destructive ops** | Reject any request to delete, update, create, or submit data |
| **Navigation only** | Only navigate, scroll, and view pages — never change state |

### Quality Assurance

| Rule | Detail |
|------|--------|
| **Verify every screenshot** | After every capture, check snapshot content against expectations. Never skip verification. |
| **Re-capture on failure** | If verification fails, diagnose and fix the issue, then re-capture. Don't proceed with wrong screenshots. |
| **Report unfixable failures** | If a page cannot be captured correctly (server error, missing data), report it explicitly. Never silently save a wrong screenshot. |

### Environment

| Rule | Detail |
|------|--------|
| **Web only** | Only web frontend applications. Reject CLI apps, desktop apps, mobile apps, APIs |
| **1920×1200 viewport** | Always resize browser to 1920×1200 before capturing. Never skip this step. |
| **One screenshot per page** | Each page gets exactly one viewport screenshot. No full_page, no multi-position, no scrolling captures. |
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
| Using full_page for screenshots | "full_page captures everything in one file, it's more efficient" | Always use viewport-only screenshots at 1920×1200. full_page produces unreadable, inconsistent output. |
| Using a non-standard viewport size | "The default size is fine" / "I'll match the user's screen" | Always set 1920×1200 before capturing. Consistent resolution matters. |
| Taking multiple screenshots per page at different scroll positions | "Different scroll positions show different content" | One screenshot per page. The 1920×1200 viewport captures the most important content at the top of the page. |
| Saving screenshots without verifying content | "MCP returned success, so the screenshot must be correct" | Always take a snapshot and verify: page title, no error messages, no auth gates, expected data present. MCP can succeed while the page shows a login redirect or error. |
| Proceeding after verification failure | "The login page screenshot is close enough" / "Empty state is fine, the layout is the same" | Fix the issue and re-capture. A screenshot of the wrong content is worse than no screenshot — it creates false confidence. |

## Red Flags — STOP Immediately

- "Let me write a quick script to do this..."
- "I'll dispatch parallel agents for each page..."
- "MCP isn't connected, I'll use the Playwright library directly..."
- "This CLI app can be screenshotted if I create an HTML wrapper..."
- "Let me delete the existing screenshots folder first..."
- "Let me click Delete to show you the confirmation dialog..."
- "Once MCP is connected, I'll click Delete and take the screenshot..."
- "Step 3: Click Delete. Step 4: Take screenshot..." (in your plan)
- "I'll use full_page to capture the whole page..."
- "This page is long, let me take screenshots at different scroll positions..."
- "I'll scroll down and take another screenshot of the lower content..."
- "Let me capture 0%, 50%, and 100% scroll positions..."
- "The default viewport size is fine, I don't need to resize..."
- "I'll use 1440x900 instead of 1920x1200..."
- "MCP said it worked, so the screenshot is fine..."
- "It's probably the right page, I don't need to check the snapshot..."
- "The login page showed up but the layout looks right anyway..."
- "Empty state screenshot is good enough, same UI structure..."
- "I'll just save this screenshot and move on to the next page quickly..."

**All of these mean: STOP. Re-read this skill. Follow the correct workflow.**

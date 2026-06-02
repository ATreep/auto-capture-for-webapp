---
name: take-screenshots
description: Use when the user asks to take screenshots of a web application, capture specific pages or UI views, or document frontend states. Triggers on requests like "take screenshots of", "screenshot the login page", "capture the dashboard", or similar screenshot/capture requests for web apps.
---

# Take Screenshots

## Overview

Take screenshots of web applications using MCP browser automation tools. **Chrome DevTools MCP is preferred; Playwright MCP is the fallback.** The skill enforces safe, read-only screenshot capture without modifying application data, using only the main conversation thread.

**Core principles:**
1. Screenshots are read-only operations. Never modify data, never use scripts, and never delegate to sub-agents.
2. **Always verify results.** After every screenshot, double-check the MCP-returned content (via snapshot) to ensure the webpage actually shows what the user expected — not an error page, login gate, or empty state.

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

#### C. Detect Page Scrollability (MANDATORY)

Before taking any screenshot, determine whether the page has a vertical scrollbar (i.e., content extends beyond one viewport height). Use `evaluate_script` (Chrome DevTools MCP) or `browser_evaluate` (Playwright MCP):

```
// Returns { scrollable: boolean, totalHeight: number, viewportHeight: number }
() => {
  return {
    scrollable: document.documentElement.scrollHeight > window.innerHeight + 10,
    totalHeight: document.documentElement.scrollHeight,
    viewportHeight: window.innerHeight
  }
}
```

- **If NOT scrollable** (totalHeight ≈ viewportHeight) → Take a single viewport screenshot (step C1).
- **If scrollable** (totalHeight > viewportHeight + 10px) → Evaluate the page content to decide between single or multi-position capture:

#### C0.5. Content Assessment (Scrollable Pages Only)

**Not every scrollable page needs multiple viewport screenshots.** After detecting that the page is scrollable, take a snapshot first to assess the page content, then decide:

**→ Use MULTI-POSITION capture (C2) when the page has DISTINCT content at different scroll positions:**

| Scenario | Why Multi-Position |
|----------|-------------------|
| Dashboard with multiple widget rows | Each row shows different charts/metrics — capture each section |
| Different functional modules stacked vertically | Each module is a separate logical unit worth its own screenshot |
| Important buttons/menus/actions hidden below the fold | Users need to see what's available after scrolling |
| Content organized into clearly separated sections | Section A (hero/overview) → Section B (features/details) → Section C (footer/CTAs) |
| Mixed content types at different scroll depths | Text → charts → tables → forms — each type benefits from viewport-level detail |

**→ Use SINGLE screenshot (C1) when the page has REPETITIVE or SINGLE-SECTION content:**

| Scenario | Why Single Screenshot |
|----------|----------------------|
| Long data list or table (uniform rows) | All rows look the same — one screenshot shows the pattern |
| Single main widget/content area dominates | The page is essentially one big widget with slight overflow |
| A single content section occupies >50% of scroll height | Most of the scrollbar represents one continuous section |
| Repeated widgets or text patterns | The same card/row pattern repeats — capturing once suffices |
| Long form with uniform field layout | All fields share the same visual pattern — one screenshot captures the form structure |
| Documentation/article page (continuous prose) | Text flows continuously — content is homogeneous throughout |

**Decision flowchart:**
1. Take a snapshot of the page at 0% scroll position
2. Scroll to 50% and compare — is the content at 50% meaningfully different from 0%?
3. Scroll to 100% and compare — is the bottom content distinct (not just continued list/table)?
4. If content is **distinctly different** at different positions → use C2 (multi-position)
5. If content is **same pattern repeated** or **one continuous section** → use C1 (single fullPage)

**Key question to ask: "Would a user lose important information if they only saw one screenshot of this page?"**
- YES → use C2 multi-position
- NO → use C1 single screenshot

#### C1. Single Screenshot (Non-Scrollable Page)

For pages whose entire content fits within one viewport:

- Chrome DevTools MCP: `take_screenshot` with `fullPage: true`
- Playwright MCP: `browser_take_screenshot` with `full_page: true`

Save as: `{num}-{description}.png` (e.g., `1-login-page.png`, `3-settings-profile.png`)

Then skip to step D.

#### C2. Multi-Position Screenshots (Scrollable Page with Distinct Content)

For scrollable pages where **different scroll positions show meaningfully different content** (determined in C0.5), capture viewport-level screenshots at multiple scroll positions. Full-page screenshots of very long pages lose detail; viewport-level captures at each distinct section preserve readability.

**Scroll position targets and naming convention:**

```
{num}-0%-{description}.png    → Top of page (scroll position 0%)
{num}-40%-{description}.png   → ~40% scroll position
{num}-70%-{description}.png   → ~70% scroll position
{num}-100%-{description}.png  → Bottom of page (scroll position 100%)
```

**How to capture each position:**

For each target percentage (0%, 40%, 70%, 100%), scroll to that position first, then take a **viewport-only** screenshot (NOT fullPage):

1. **Scroll to position** — Use `evaluate_script` to set `window.scrollTo(0, Math.floor(document.documentElement.scrollHeight * 0.X))` where `0.X` is the target ratio.

2. **Wait for content** — After scrolling, wait briefly for any lazy-loaded content or re-renders (500ms minimum).

3. **Take viewport screenshot** — Use `take_screenshot` WITHOUT `fullPage` flag, or `browser_take_screenshot` WITHOUT `full_page`. This captures exactly what the user sees at that scroll position.

```
# Example for a scrollable login page "【图1：登录页面】":
1-0%-login-page.png
1-40%-login-page.png
1-70%-login-page.png
1-100%-login-page.png
```

**Scroll position selection rules:**
- Always capture **0%** (top) and **100%** (bottom) — these are non-negotiable.
- For pages with 1.5–3× viewport height → 0%, 50%, 100% (3 screenshots, skip 40%/70%).
- For pages with 3–5× viewport height → 0%, 40%, 70%, 100% (4 screenshots — the standard).
- For pages with 5+× viewport height → 0%, 25%, 50%, 75%, 100% (5 screenshots).
- Adjust positions to align with logical content sections when possible (e.g., if a section boundary falls at 35%, use 35% instead of 40%).

**CRITICAL: Each scroll position screenshot must be a viewport capture, NOT fullPage.** The whole point is to capture readable viewport-level details at each position. Full-page at each scroll position would be redundant and miss the purpose.

**No exceptions:**
- Don't skip scroll detection because "this page looks short"
- Don't skip the content assessment (C0.5) — always compare positions before deciding
- Don't use fullPage instead of multiple viewport captures when content IS distinctly different
- Don't use multi-position when content IS repetitive/continuous (one fullPage is correct in that case)
- When using multi-position, don't skip the bottom (100%) position — verify it adds distinct content
- Don't guess — run the snapshot comparison at different scroll positions to decide

#### D. Take a Snapshot (MANDATORY for Verification)

Capture an accessibility snapshot for content verification:
- Chrome DevTools MCP: `take_snapshot`
- Playwright MCP: `browser_snapshot`

This provides text content to verify the screenshot captured the correct page. **This step is now mandatory** — you cannot verify content without it.

#### E. Verify Screenshot Content (MANDATORY)

**CRITICAL: After every screenshot, double-check the MCP results to ensure the webpage content satisfies the user's demand.** Do NOT blindly trust that the screenshot captured the right page. MCP operations can silently fail — the page may redirect to a login screen, show an error page, display empty data, or render partially.

**Verification checklist (run after every screenshot):**

1. **Check the page URL/title** — Does the snapshot text show the expected page name or heading?
2. **Check for error states** — Look for error messages, "404", "500", "Access Denied", "Loading failed", blank pages, or unexpected redirects
3. **Check for auth gates** — If expecting a logged-in page, verify the snapshot shows actual content (not "Please login", a login form, or an auth error)
4. **Check for empty/dummy data** — If expecting a list, table, or data view, verify the rows/cards actually contain data (not "No data", "Empty", or skeleton placeholders)
5. **Check for the expected UI elements** — Match against the user's described target: if the user asked for "dashboard with stats cards", verify stats cards appear in the snapshot text; if they asked for "settings page with profile form", verify profile form fields are present
6. **Check scroll position coverage** (for multi-position captures) — Each scroll position screenshot MUST show different content. If 0%, 40%, 70%, and 100% all show the same viewport, scrolling failed. Verify the snapshot text changes between positions.

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

| # | Screenshot | File Path | Description | Verified |
|---|-----------|-----------|-------------|----------|
| 1a | Login page (top) | `./screenshots/1-0%-login-page.png` | Logo + username/password fields | ✅ Heading "登录", form fields present |
| 1b | Login page (40%) | `./screenshots/1-40%-login-page.png` | Additional login options, SSO buttons | ✅ SSO buttons visible, "忘记密码" link |
| 1c | Login page (70%) | `./screenshots/1-70%-login-page.png` | Footer area, help links | ✅ Footer links, copyright |
| 1d | Login page (bottom) | `./screenshots/1-100%-login-page.png` | Full footer, legal, contact info | ✅ 备案号, contact email |
| 2 | Dashboard | `./screenshots/2-dashboard.png` | Main dashboard with stats overview (non-scrollable) | ✅ "仪表盘" title, 4 stat cards, data table |
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
| **MCP only** | Only use MCP tools (`mcp__*__take_screenshot`, `mcp__*__browser_take_screenshot`) |
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
| Using viewport-only capture for non-scrollable pages | "Full page might be better quality" | For non-scrollable pages, fullPage is fine. The distinction matters: fullPage for short pages, viewport-at-positions for scrollable pages. |
| Using fullPage for scrollable pages with distinct content | "FullPage captures everything in one file, it's more efficient" | FullPage on a long page produces unreadable text. When content differs at different scroll positions, use multi-position viewport capture. |
| Using multi-position for repetitive content | "The page has a scrollbar, so it needs multiple screenshots" | A scrollbar alone doesn't justify multi-position. Check if content actually differs between scroll positions. A long data list or single-section page only needs one screenshot. |
| Skipping the C0.5 content assessment | "I can decide without checking" / "It's obvious this page is distinct/repetitive" | Always run the 0%→50%→100% snapshot comparison before deciding. Visual guesses are unreliable. |
| Skipping scroll detection | "This page doesn't look that long" / "I can tell visually" | Always run the scroll detection script. Visual estimates are unreliable — pages with dynamic content often expand. |
| Missing the bottom scroll position (when multi-position applies) | "The footer isn't important" / "I already captured enough" | When using multi-position, always capture 100%. But verify it adds distinct content — if bottom is just more of the same, it's a sign you should have used single screenshot. |
| Using wrong screenshot count for page height | "3 screenshots is enough for any page" | Match count to page height AND content distinctness: fewer positions for borderline cases, more for clearly distinct sections. |
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
- "This page isn't that long, viewport is fine..."
- "Dashboards look better without fullPage..."
- "The important content fits in the viewport..."
- "I don't need to check scroll height, I can see it's short..."
- "It has a scrollbar, so I MUST take multiple screenshots..."
- "Just one fullPage screenshot will cover the whole page..."
- "4 screenshots is too many, I'll take 2 and cover the rest with fullPage..."
- "The footer isn't important, I'll skip the 100% position..."
- "I don't need to scroll, fullPage already captures everything..."
- "This long page is fine with just fullPage, the detail is still readable..."
- "I don't need to compare scroll positions, I already know this page is repetitive..."
- "It's a dashboard, so it definitely needs multi-position..."
- "It's a data list, so it definitely only needs one screenshot..."
- "I'll just guess whether content differs without actually checking..."
- "MCP said it worked, so the screenshot is fine..."
- "It's probably the right page, I don't need to check the snapshot..."
- "The login page showed up but the layout looks right anyway..."
- "Empty state screenshot is good enough, same UI structure..."
- "I'll just save this screenshot and move on to the next page quickly..."

**All of these mean: STOP. Re-read this skill. Follow the correct workflow.**

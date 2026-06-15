# Tooling & Conventions

This skill runs **in-conversation**: the live Claude session orchestrates the pipeline directly — spawning Task sub-agents, driving the browser, and running the deterministic helper scripts. **`00-contract.md` is the single source of truth.** Where this file and the contract disagree about paths, artifact names, fan-out keys, the gate, or the loop — the contract wins. Read it first.

---

## 1. Browser capture — detect your environment first

**Detect mode at startup:**
- `mcp__claude-in-chrome__navigate` available → **Chrome Extension mode** (Claude Code)
- `browser_navigate` available → **OpenCode Browser mode**

**One view/tab at a time.** Browser stages (recon, extraction, QA capture, polish) are **sequential**. Code stages fan out in parallel via Task sub-agents.

Page/viewport keys: `desktop` (1920×1080), `tablet` (768×1024), `mobile` (375×667). Artifacts use `{page}--{viewport}` so recon and QA reads line up 1:1.

---

### 1a. Chrome Extension (`mcp__claude-in-chrome__*`) — Claude Code

You're already authed in Chrome, so the real target loads behind its login. The ground truth is **computed styles via `mcp__claude-in-chrome__javascript_tool`**; a screenshot is a visual reference only (cannot be reliably saved to disk).

| What you're capturing | Tool | Why |
|---|---|---|
| **Real, logged-in site** | Chrome extension | Already authed in Chrome — real page loads behind login |
| **Localhost clone** (no auth) + QA loop | Chrome extension | Clone loads directly; compare `getComputedStyle` reads vs real page |

**Tool mapping (Chrome extension):**

| Need | Tool |
|---|---|
| open route | `mcp__claude-in-chrome__navigate` |
| run JS / computed styles (ground truth) | `mcp__claude-in-chrome__javascript_tool` |
| DOM snapshot / find element | `mcp__claude-in-chrome__read_page` / `find` |
| click / hover / focus | `mcp__claude-in-chrome__computer` (left_click / hover) |
| screenshot (visual reference — may not save to disk) | `mcp__claude-in-chrome__computer` (screenshot) |
| set viewport | `mcp__claude-in-chrome__resize_window <w> <h>` |
| press key | `mcp__claude-in-chrome__computer` (key) |

---

### 1b. OpenCode Browser (`browser_*`) — OpenCode

Uses Playwright/Chromium managed by OpenCode. **Unauthenticated by default** — ideal for public target sites; for authed sites handle login via `browser_evaluate` / `browser_run_code` after navigation.

**Prerequisites (one-time setup):**
```bash
npx playwright install chromium
```
**Enable in `opencode.json`:**
```json
{ "browser": true }
```
Or set env: `OPENCODE_ENABLE_BROWSER=true`

**Tool mapping (OpenCode browser):**

| Need | Tool |
|---|---|
| open route | `browser_navigate` |
| run JS / computed styles (ground truth) | `browser_evaluate` |
| DOM snapshot | `browser_snapshot` or `browser_content` |
| find element | `browser_search` |
| click / hover / focus | `browser_click` / `browser_hover` |
| screenshot **(saves to disk!)** | `browser_screenshot` |
| set viewport | `browser_run_code` → `await page.setViewportSize({width:1920,height:1080})` |
| press key | `browser_press_key` |
| scroll | `browser_scroll` |
| arbitrary Playwright code | `browser_run_code` |

**Auth for logged-in sites (OpenCode):** After `browser_navigate` to the target URL, use `browser_evaluate` or `browser_run_code` to inject session cookies, or perform a programmatic login flow before running recon. Alternatively, use `browser_run_code` with:
```js
await page.context().addCookies([{ name: 'session', value: '...', domain: '...', path: '/' }]);
```

---

## 2. Screenshots — visual reference (Chrome ext) vs disk artifact (OpenCode)

**Chrome Extension:** screenshots are a **visual reference only** — the file cannot be reliably saved to disk in this environment. Capture with `mcp__claude-in-chrome__computer` (screenshot) for a quick eyeball check, but never build the gate on a PNG.

**OpenCode Browser:** `browser_screenshot` **saves to disk** — you get real PNG files at workspace paths. This enables pixel-diff for public sites (no auth wall). Even so, the objective gate remains **computed styles**: a measured `getComputedStyle` value beats a screenshot impression every time.

**In both modes — the verdict is the computed style, not the image.** The gate (`assert-styles.mjs`) reads `getComputedStyle` values, never a pixel diff. Screenshots are for human eyeballing and catching gross structural issues; they are never the source of a gate decision.

---

## 3. Evidence & anti-hallucination

These rules apply to ALL QA, Fix, and Polish work, and complement the contract's EVIDENCE RULES and ANTI-HALLUCINATION sections (§7).

1. **Every visual claim needs a measured `evidence_method`** — a `getComputedStyle` value or a diff %. The pixel-diff gate (`metrics.json`) and `getComputedStyle` are the only verdicts that count. "Looks good" is not a grade.
2. **Authoritative-source order** when sources conflict (§3): **CSSOM authored rules > CDP forced-state computed > deduped computed archetypes > screenshot estimate.** A measured value always beats a screenshot impression — visual impression cannot distinguish `#f0f0f0` from `#f5f5f5`.
3. **Never invent a value.** If a value can't be read, record `null` + the reason.
4. **Never fabricate a match.** Fix the clone to match the original exactly, or record the bug honestly — the gate is objective, so gaming it is impossible. If the page can't be reached (auth wall / 302 to login), do NOT generate UI from memory — emit `<promise>BLOCKED: reason</promise>` (HARD-BLOCKER, contract §6).
5. **Never skip a page × viewport.** The gate evaluates every `{page}--{viewport}`; an unmeasured cell is a failure, not a pass.

### Bug fragment shape (parallel-safe)

QA agents write per-task fragments to `06-qa/cycle-N/fragments/{page}--{viewport}.bugs.json`; the orchestrator concatenates + renumbers into `bugs.json`. Every entry carries measured evidence:

```json
{
  "id": "BUG-001",
  "severity": "C",
  "category": "color",
  "element": "TopNav background",
  "page": "home",
  "viewport": "desktop",
  "expected": "rgb(255,255,255) (DESIGN.md token --surface)",
  "actual": "rgb(245,245,245) (getComputedStyle)",
  "evidence_method": "getComputedStyle",
  "diff_region": "cycle-1/diff/home--desktop.diff.png",
  "fix_hint": "Change bg-gray-50 → bg-white in TopNav.tsx"
}
```

`evidence_method` is required. Valid values: `getComputedStyle` (computed CSS via JS) · `CSSOM` (authored rule) · `forced-state` (CDP `:hover/:focus/:active` delta) · `pixelmatch diff` (mismatch region from `metrics.json` / a diff PNG) · `assert-styles.mjs` (a recorded style-assertion failure).

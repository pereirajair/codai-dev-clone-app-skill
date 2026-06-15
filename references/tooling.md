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

### 1b. BrowserMCP (`browser_*`) — OpenCode

BrowserMCP uses your **real Chrome** via a Chrome extension + Native Messaging — the same approach as the Claude Chrome extension. Your existing Chrome profile is used, so **authed sites load normally** without extra login setup.

**Setup (one-time):**
1. Install the Chrome extension from [browsermcp.io](https://browsermcp.io)
2. Add to `opencode.json`:

```json
{
  "mcp": {
    "browsermcp": {
      "type": "local",
      "command": ["npx", "@browsermcp/mcp@latest"],
      "enabled": true
    }
  }
}
```

**Full tool list (BrowserMCP `@browsermcp/mcp` v0.1.3):**

| Tool | What it does |
|---|---|
| `browser_navigate` | Open a URL in the current tab |
| `browser_go_back` | Navigate back |
| `browser_go_forward` | Navigate forward |
| `browser_snapshot` | ARIA accessibility tree + element selectors |
| `browser_click` | Click an element (by label/aria description) |
| `browser_hover` | Hover over an element |
| `browser_type` | Type text into an element |
| `browser_select_option` | Select a dropdown option |
| `browser_press_key` | Press a keyboard key |
| `browser_wait` | Wait N seconds |
| `browser_screenshot` | Capture screenshot (returns **base64 PNG** — visual reference, not a disk file) |
| `browser_get_console_logs` | Retrieve browser console output |

**❌ NOT available in BrowserMCP:**
- `browser_evaluate` / JS execution — **no JavaScript execution tool exists**
- Viewport resize — no equivalent to `resize_window`
- `browser_scroll`, `browser_search`, `browser_run_code`, `browser_drag` (drag is defined in source but not registered)

**⚠️ Critical limitation for this skill:** The clone pipeline's core mechanism is `getComputedStyle` read via JS execution. BrowserMCP has no JS execution tool, so **Stage 2 (Extraction), Stage 6 (QA), and Stage 9 (Polish)** cannot use BrowserMCP for computed-style reads. BrowserMCP is usable for Stage 1 (Recon) interaction sweeps — navigation, clicks, snapshots, screenshots. For JS-dependent stages, the Chrome Extension (Claude Code) or fallback Bash scripts are required.

---

## 2. Screenshots — visual reference in both modes

**Chrome Extension:** `mcp__claude-in-chrome__computer` (screenshot) — visual reference only; the file cannot be reliably saved to disk in this environment.

**BrowserMCP:** `browser_screenshot` — returns **base64 PNG** inline (not a disk file). Same status as the Chrome extension: a visual aid for eyeballing, never a gate input.

**In both modes — the gate is computed styles, not screenshots.** The gate (`assert-styles.mjs`) reads `getComputedStyle` values. Screenshots help catch gross layout issues; they never drive a pass/fail decision.

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

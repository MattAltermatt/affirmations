# TRMNL Affirmations Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a TRMNL Private Plugin that displays one of 13 personal affirmations daily on a 400×480 half-vertical e-ink canvas, hosted on GitHub Pages.

**Architecture:** Static JSON (data, served by GH Pages) + Liquid template (lives in TRMNL dashboard, rendered server-side) + HTML preview page (public landing). TRMNL backend polls the JSON URL, renders the Liquid template to BMP, and the device displays it.

**Tech Stack:** Plain JSON, Liquid (Shopify-style), vanilla HTML/CSS/JS, GitHub Pages, TRMNL Private Plugin dashboard, `gh` CLI for deploy.

**Spec:** `docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md`

**User context (important):** This is the user's first TRMNL plugin. Lead explanations should name buttons, paste exact URLs, and surface concrete framework classes — not assume familiarity with the platform.

---

## Execution Handoff (per CLAUDE.md per-task split)

```text
Task 1  — branch + data + template     → Lead-Inline   (foundational; locks contract for downstream)
Task 2  — index.html preview/landing   → Subagent-OK   (replicable after data+template contract)
Task 3  — README                       → Subagent-OK   (pure authoring)
Task 4  — local Chrome verify          → Lead-Inline   (Chrome DevTools MCP)
Task 5  — code review (pre-merge)      → Subagent      (fresh reviewer agent)
Task 6  — FF-merge + push + GH Pages   → Lead-Inline   (gh CLI, shell)
Task 7  — TRMNL dashboard + device     → Lead-Inline   (user-driven; lead supports)
```

---

## Phase 1 — Build (Tasks 1–3)

### Task 1: Feature branch + data file + Liquid template

**Files:**
- Create: `affirmations.json`
- Create: `template/half_vertical.html`

**Steps:**

- [ ] **Step 1: Create the feature branch off main**

```bash
git switch -c feature/initial-implementation
```

- [ ] **Step 2: Write `affirmations.json`**

```json
{
  "items": [
    "Moderation.",
    "Be present.",
    "Be positive.",
    "Assume good intentions.",
    "Feelings over efficiency.",
    "Stick with it.",
    "You won't remember, but someone will.",
    "Celebrate firsts, celebrate lasts.",
    "Say nothing until you are sure.",
    "Ask, don't assume.",
    "Something is usually better than nothing.",
    "Strength is impressive, kindness is remembered.",
    "Take the hint."
  ]
}
```

- [ ] **Step 3: Write `template/half_vertical.html`**

This file mirrors what will be pasted into the TRMNL dashboard's "Edit Markup" → `half_vertical` tab. Keeping it in git gives version control + diffability.

```liquid
{%- assign idx = 'now' | date: '%j' | minus: 0 | modulo: items.size -%}
{%- assign current = items[idx] -%}
{%- assign len = current | size -%}

{%- if len <= 12     -%}{%- assign size_class = 'value--xxxlarge' -%}
{%- elsif len <= 24  -%}{%- assign size_class = 'value--xxlarge'  -%}
{%- elsif len <= 40  -%}{%- assign size_class = 'value--xlarge'   -%}
{%- else             -%}{%- assign size_class = 'value--large'    -%}
{%- endif -%}

<div class="view view--half_vertical">
  <div class="layout layout--col layout--center gap--large">
    <span class="value {{ size_class }} text--center">{{ current }}</span>
  </div>
</div>
```

- [ ] **Step 4: Validate JSON parses**

Run: `python3 -m json.tool < affirmations.json > /dev/null && echo "JSON OK"`
Expected: `JSON OK`. If it errors, fix the JSON and retry.

- [ ] **Step 5: Commit**

```bash
git add affirmations.json template/half_vertical.html
git commit -m "feat: data file and Liquid half_vertical template"
```

---

### Task 2: Local preview / landing page (`index.html`)

**Files:**
- Create: `index.html`

This is the GitHub Pages root — doubles as a local dev preview (open in Chrome) and a public landing page (anyone visiting `mattaltermatt.github.io/affirmations/` sees today's affirmation).

The JS logic mirrors the Liquid template's picker + size-bucketer so what the browser shows matches what the device renders.

**Steps:**

- [ ] **Step 1: Write `index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Affirmations — TRMNL plugin preview</title>
  <link rel="stylesheet" href="https://usetrmnl.com/css/latest/plugins.css">
  <style>
    body {
      margin: 0;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 24px;
      padding: 40px 20px;
      background: #f3f4f6;
      font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
      color: #1f2937;
    }
    .device {
      width: 400px;
      height: 480px;
      background: white;
      border-radius: 12px;
      box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12), 0 2px 6px rgba(0, 0, 0, 0.08);
      overflow: hidden;
    }
    .controls {
      display: flex;
      align-items: center;
      gap: 16px;
      font-size: 14px;
    }
    button {
      padding: 8px 16px;
      border: 1px solid #d1d5db;
      background: white;
      border-radius: 6px;
      cursor: pointer;
      font-size: 14px;
      font-family: inherit;
    }
    button:hover { background: #f9fafb; }
    #counter { min-width: 60px; text-align: center; font-variant-numeric: tabular-nums; }
    .footer {
      font-size: 12px;
      color: #6b7280;
    }
    .footer a { color: #6b7280; }
  </style>
</head>
<body>
  <div class="device">
    <div class="view view--half_vertical">
      <div class="layout layout--col layout--center gap--large">
        <span class="value" id="hero">…</span>
      </div>
    </div>
  </div>

  <div class="controls">
    <button onclick="prev()">← prev</button>
    <span id="counter">0 / 0</span>
    <button onclick="next()">next →</button>
    <button onclick="goToday()">today</button>
  </div>

  <div class="footer">
    Live preview of today's affirmation. <a href="https://github.com/MattAltermatt/affirmations" target="_blank">View source ↗</a>
  </div>

  <script>
    let items = [];
    let cursor = 0;

    function sizeClass(text) {
      const len = text.length;
      if (len <= 12) return 'value--xxxlarge';
      if (len <= 24) return 'value--xxlarge';
      if (len <= 40) return 'value--xlarge';
      return 'value--large';
    }

    function dayOfYear() {
      const now = new Date();
      const start = new Date(now.getFullYear(), 0, 0);
      const diff = now - start;
      return Math.floor(diff / 86400000);
    }

    function todayIndex() {
      return dayOfYear() % items.length;
    }

    function render() {
      const text = items[cursor];
      const hero = document.getElementById('hero');
      hero.className = 'value text--center ' + sizeClass(text);
      hero.textContent = text;
      document.getElementById('counter').textContent = `${cursor + 1} / ${items.length}`;
    }

    function prev() {
      cursor = (cursor - 1 + items.length) % items.length;
      render();
    }

    function next() {
      cursor = (cursor + 1) % items.length;
      render();
    }

    function goToday() {
      cursor = todayIndex();
      render();
    }

    fetch('./affirmations.json')
      .then(r => r.json())
      .then(data => {
        items = data.items;
        cursor = todayIndex();
        render();
      })
      .catch(err => {
        document.getElementById('hero').textContent = 'Failed to load.';
        console.error(err);
      });
  </script>
</body>
</html>
```

- [ ] **Step 2: Smoke-test JS parses with no syntax errors**

Run: `node --check index.html 2>&1 || true`
(Node won't parse HTML, so this is best-effort. If a subagent runs this task, open the file with `head -5` to confirm it was written.)

Better local check — start a server and confirm 200 OK:

```bash
python3 -m http.server 8080 &
SERVER_PID=$!
sleep 1
curl -sI http://localhost:8080/index.html | head -1
curl -sI http://localhost:8080/affirmations.json | head -1
kill $SERVER_PID
```

Expected: both return `HTTP/1.0 200 OK`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: local preview / public landing page"
```

**Note for subagent execution:** The Chrome eyeball verify happens in Task 4 (lead-inline, needs Chrome DevTools MCP). This task only verifies the file is well-formed and served.

---

### Task 3: README with setup instructions

**Files:**
- Create: `README.md`

**Steps:**

- [ ] **Step 1: Write `README.md`**

```markdown
# Affirmations — TRMNL Plugin

A personal TRMNL e-ink display plugin: one of 13 affirmations, rotated daily.

🌐 **Preview / landing:** https://mattaltermatt.github.io/affirmations/
📡 **Polling URL:** https://mattaltermatt.github.io/affirmations/affirmations.json

## How it works

- `affirmations.json` is a static list of 13 mantras.
- The TRMNL backend polls the JSON URL on the configured refresh interval.
- A Liquid template (pasted into the TRMNL dashboard) picks today's mantra by day-of-year and renders it as a 400×480 half-vertical card.
- The device receives a BMP image from the TRMNL backend and displays it.

See [`docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md`](docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md) for the full design.

## Editing the affirmations

1. Edit `affirmations.json`.
2. Commit and push to `main`.
3. GitHub Pages redeploys within ~1 minute.
4. The TRMNL backend pulls the new JSON on its next refresh cycle.

The Liquid template's fit-to-box logic auto-sizes based on string length (4 buckets — `value--xxxlarge` / `value--xxlarge` / `value--xlarge` / `value--large`).

## TRMNL dashboard setup (one-time)

1. In TRMNL dashboard, click **+ New Plugin**.
2. Choose **Screen Templating** type.
3. **Name:** `Affirmations`.
4. **Strategy:** Polling.
5. **Polling URL:** `https://mattaltermatt.github.io/affirmations/affirmations.json`
6. **Refresh interval:** 15 minutes (or any — doesn't affect rotation cadence since the picker is day-based).
7. Save.
8. Click **Edit Markup**.
9. Switch to the **half_vertical** tab.
10. Paste the contents of [`template/half_vertical.html`](template/half_vertical.html).
11. Save. Confirm the live preview pane shows today's affirmation rendered correctly.
12. Add the plugin to a Playlist assigned to your device.

## Local development

```bash
# Serve the preview locally
python3 -m http.server 8080

# Open in Chrome:
# http://localhost:8080/
```

Use the prev/next buttons to cycle through all 13 entries and verify rendering for each.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: README with setup instructions and TRMNL dashboard steps"
```

---

## Phase 2 — Verify locally (Task 4)

### Task 4: Local Chrome verification (lead-inline)

**Files:** none modified. Eyeball verification only.

This uses Chrome DevTools MCP (per CLAUDE.md, NEVER the built-in preview). Lead must run this task.

**Steps:**

- [ ] **Step 1: Start the dev server in the background**

```bash
python3 -m http.server 8080
```

Run with `run_in_background: true` so the server stays up.

- [ ] **Step 2: Open `http://localhost:8080/` in Chrome via Chrome DevTools MCP**

Use the `mcp__plugin_chrome-devtools-mcp_chrome-devtools__new_page` (or `navigate_page`) tool.

- [ ] **Step 3: Take a screenshot of today's affirmation**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__take_screenshot`. Save the path; hand to user.

- [ ] **Step 4: Cycle through all 13 entries via the next button**

For each entry, take a screenshot. Pay attention to:
- ✅ Hero text fits within the 400×480 device frame (no overflow, no cropping)
- ✅ Size class scales appropriately (short = big, long = small)
- ✅ Multi-line wrapping looks balanced (not orphaned single words)
- ✅ TRMNL framework classes (`view--half_vertical`, `value--*`) are actually applying — if `plugins.css` failed to load (CORS, 404), the styling will look unstyled

- [ ] **Step 5: Verify console has no errors**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__list_console_messages`. Expected: empty or only INFO. If `plugins.css` 404s, capture the actual URL and fix in Step 6.

- [ ] **Step 6: If `plugins.css` URL fails, find the right one**

Likely candidates:
- `https://usetrmnl.com/css/latest/plugins.css`
- `https://trmnl.com/css/latest/plugins.css`
- A pinned version like `https://trmnl.com/framework/css/v3.1/plugins.css`

If none of these work, check `trmnl.com/framework/docs/3.1` for the current path. Update `index.html` accordingly and commit:

```bash
git add index.html
git commit -m "fix: correct TRMNL framework CSS URL"
```

- [ ] **Step 7: Hand the live URL + screenshots to user for confirmation**

Surface the localhost URL on its own line. List the screenshot file paths. Ask user to confirm visual fit before proceeding to code review.

- [ ] **Step 8: Stop the dev server**

Kill the background server process. (`KillShell` on the background shell ID.)

---

## Phase 3 — Review (Task 5)

### Task 5: Code review (subagent-driven)

**Files:** none modified directly. Review of all changes on `feature/initial-implementation`.

Per CLAUDE.md, code review is a required phase second-to-last before final verification. Dispatch a fresh reviewer subagent with no implementation bias.

**Steps:**

- [ ] **Step 1: Run the reviewer agent**

Use the `Agent` tool with `subagent_type: feature-dev:code-reviewer`. Prompt template:

```
Review the changes on branch feature/initial-implementation in /Users/matt/dev/MattAltermatt/affirmations.

Context: This is a TRMNL e-ink Private Plugin that displays one of 13 personal affirmations rotated daily on a 400×480 half-vertical canvas. User is new to TRMNL plugin development.

Spec: docs/superpowers/specs/2026-06-03-trmnl-affirmations-design.md

Files in scope:
  - affirmations.json
  - template/half_vertical.html (Liquid template, will be pasted into TRMNL dashboard)
  - index.html (local preview + GitHub Pages landing)
  - README.md

Focus on:
  - Does the Liquid template's day-of-year picker work correctly for all 13 entries across the year?
  - Are the fit-to-box size buckets reasonable for the actual string lengths in the list?
  - Does the JS in index.html truly mirror the Liquid logic? (Pickers and bucketers should produce identical output for the same data.)
  - Does the README accurately describe the dashboard setup steps?
  - Are there any TRMNL framework class names (view--half_vertical, value--*, layout--col, gap--large, text--center, layout--center) that look wrong against what days_left_until/half_vertical.html.erb uses as reference?
  - Any security issues with the static JSON (e.g., XSS risk via the Liquid output)?
  - Any "deliberately omits X" comments worth flagging?

Report findings by confidence; high-priority issues only.
```

- [ ] **Step 2: Address findings**

For each high-priority finding:
- If load-bearing: fix on the feature branch, commit with a descriptive message.
- If cosmetic/nit: capture in BACKLOG section of README or skip.

If multiple commits result, that's fine — each addresses one finding.

- [ ] **Step 3: Confirm cleanup commit lands cleanly**

```bash
git log --oneline feature/initial-implementation ^main
```

Expected: a clean commit log, no half-finished WIP.

---

## Phase 4 — Deploy (Task 6)

### Task 6: FF-merge + push to GitHub + enable GitHub Pages (lead-inline)

**Files:** none modified. Repo deployment.

This task requires `gh` CLI and shell-level git ops — lead-inline.

**Steps:**

- [ ] **Step 1: Switch to main and FF-merge the feature branch**

```bash
git switch main
git merge --ff-only feature/initial-implementation
git log --oneline -10
```

Expected: feature commits now on main. If FF-only fails, the feature branch has diverged from main; rebase first.

- [ ] **Step 2: Optionally squash the feature commits before pushing**

Per CLAUDE.md "Squash feature-branch commits before FF-merge when safe." Since this is the very first deploy and the commits are small build-out steps not independently revertable, squashing makes `git log` read as "what shipped":

```bash
git reset --soft 71d6e77  # the spec root commit; verify with: git log --oneline
git commit -m "feat: initial TRMNL affirmations plugin (data, template, preview, README)"
git log --oneline -5
```

If unsure, skip the squash — multiple small commits on main are fine for a first deploy.

- [ ] **Step 3: Confirm gh CLI is authenticated as MattAltermatt**

```bash
gh auth status
```

Expected: account `MattAltermatt` is active. If not, run `gh auth switch --user MattAltermatt && gh auth setup-git`.

- [ ] **Step 4: Create the GitHub repo and push main**

```bash
gh repo create MattAltermatt/affirmations \
  --public \
  --description "TRMNL e-ink plugin: one of 13 affirmations, rotated daily" \
  --source=. \
  --remote=origin \
  --push
```

Expected: new repo at https://github.com/MattAltermatt/affirmations, main pushed.

- [ ] **Step 5: Enable GitHub Pages from main / root**

```bash
gh api -X POST repos/MattAltermatt/affirmations/pages \
  -f "source[branch]=main" \
  -f "source[path]=/"
```

Expected: HTTP 201. If the API call shape is wrong on the current `gh` version, fall back to web UI: GitHub → repo Settings → Pages → Source = main / `/ (root)` → Save.

- [ ] **Step 6: Wait for the first GH Pages build**

```bash
sleep 30
gh api repos/MattAltermatt/affirmations/pages | python3 -m json.tool | grep -E '"status"|"html_url"'
```

Expected: `"status": "built"` and the public URL. GH Pages first build can take 30-90 seconds.

- [ ] **Step 7: Verify the live URLs respond**

```bash
curl -sI https://mattaltermatt.github.io/affirmations/ | head -1
curl -sI https://mattaltermatt.github.io/affirmations/affirmations.json | head -1
curl -s  https://mattaltermatt.github.io/affirmations/affirmations.json | python3 -m json.tool | head -5
```

Expected: both URLs return `HTTP/2 200`. JSON parses with the items array.

- [ ] **Step 8: Open the live preview in Chrome and confirm visible render**

Use Chrome DevTools MCP. Navigate to `https://mattaltermatt.github.io/affirmations/`. Confirm:
- ✅ Today's affirmation is displayed
- ✅ The mock device frame renders
- ✅ Prev/next buttons work
- ✅ No console errors

Per CLAUDE.md: "After deploy, validate live. Open the live URL, watch console, confirm visible change. Green build ≠ working deploy."

- [ ] **Step 9: Delete the merged feature branch**

Per CLAUDE.md post-ship cleanup standing authorization (since this is on main, tree clean, branch FF-merged this session):

```bash
git branch -d feature/initial-implementation
```

(No remote delete needed — feature branch was never pushed.)

---

## Phase 5 — Dashboard + device (Task 7)

### Task 7: TRMNL dashboard setup + device verification (lead-inline; user-driven)

**Files:** none modified. User performs dashboard steps; lead supports.

**Steps:**

- [ ] **Step 1: User opens TRMNL dashboard**

Surface the URL: `https://usetrmnl.com/dashboard` (or `https://trmnl.com/dashboard` — the dashboard subdomain may have changed). User logs in.

- [ ] **Step 2: User creates Private Plugin per README dashboard setup section**

Walk through the 12 steps in `README.md` → "TRMNL dashboard setup (one-time)". Lead names buttons and pastes URLs as needed.

Key items to surface:
- Polling URL (copy-paste exact): `https://mattaltermatt.github.io/affirmations/affirmations.json`
- Liquid template (paste from `template/half_vertical.html`)
- Layout tab to use: `half_vertical`

- [ ] **Step 3: User verifies dashboard live preview**

In the "Edit Markup" page, the live preview pane should render today's affirmation in a half_vertical canvas. If it doesn't:

- Check the **Open verification points** in the spec:
  - Is `items` the correct Liquid variable? If TRMNL wraps the polling response under `data` or another key, rename in the template.
  - Are framework classes correct? Cross-reference with `trmnl.com/framework/docs/3.1`.
- If a class is wrong, fix in `template/half_vertical.html` AND in the dashboard paste. Commit + push the corrected file.

- [ ] **Step 4: User adds plugin to a Playlist on their device**

The plugin only renders when it's part of a Playlist assigned to a device. User navigates to Playlists, adds the new "Affirmations" plugin.

- [ ] **Step 5: Device refresh + visual verification**

User waits for the next refresh cycle (or triggers a manual refresh from the device, if supported). Confirms:
- ✅ The affirmation shows on the e-ink display
- ✅ Text is sharp, no rasterization artifacts
- ✅ Sizing looks right at viewing distance
- ✅ Half-vertical placement is as expected (one side of the screen)

- [ ] **Step 6: If issues, iterate**

Common issues:
- Text cut off → adjust size buckets in `template/half_vertical.html` (e.g., move the `≤ 40` threshold down)
- Wrong class → look up correct framework class name
- Variable not found → check polling response shape vs Liquid scoping

Each fix = commit + push + paste the updated template into dashboard + observe next device refresh.

- [ ] **Step 7: Mark plan complete**

Once the user confirms the device shows the expected affirmation cleanly, the plan is done. Update ROADMAP (if one exists) and close out the implementation.

---

## Done state

- ✅ Repo at `MattAltermatt/affirmations` (public)
- ✅ GH Pages live at `https://mattaltermatt.github.io/affirmations/`
- ✅ Polling URL serving `affirmations.json`
- ✅ TRMNL Private Plugin configured with Liquid template
- ✅ Device showing today's affirmation, rotating daily by day-of-year

---

## Self-review (run before handing off to execution)

**Spec coverage:**
- Architecture → Tasks 1, 6 ✓
- Repo layout → Tasks 1, 2, 3 ✓
- Data shape → Task 1 ✓
- Rotation logic → Task 1 ✓
- Fit-to-box sizing → Task 1, mirrored in Task 2 ✓
- Liquid template → Task 1 ✓
- Local dev preview → Tasks 2, 4 ✓
- TRMNL dashboard setup → Tasks 3 (docs), 7 (execution) ✓
- Error handling → covered in Task 4 (console check), Task 7 (iterate) ✓
- Verification approach → Tasks 4, 6 (live URL), 7 (device) ✓
- Open verification points → Task 4 (class names, plugins.css URL), Task 7 (Liquid scoping) ✓

**Placeholder scan:** Searched for "TBD", "TODO", "implement later" → none. Every step has concrete code or commands.

**Type consistency:** Liquid variable name `items` used consistently in JSON, template, and JS. Function names (`sizeClass`, `dayOfYear`, `todayIndex`, `render`, `prev`, `next`, `goToday`) consistent in Task 2 vs anywhere else they're referenced.

**Ambiguity:** The `plugins.css` URL is the one unverified-at-author-time item. Task 4 Step 6 explicitly handles the case where the guessed URL is wrong with concrete fallbacks.

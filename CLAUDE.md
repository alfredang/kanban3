# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT project-management Kanban board for an internal IT PMO demo/training tool. The entire app is one file: `index.html` (markup + one `<style>` block + one `<script>` block).

## Hard constraints

These are requirements of the deliverable, not style preferences. Breaking any of them breaks the product:

- **Vanilla HTML/CSS/JS only.** No framework, no bundler, no npm, no build step.
- **One file.** Everything stays in `index.html`. Do not split out `.css`/`.js`.
- **No external resources.** No CDN scripts, no web fonts, no image files — system font stack and inline SVG / Unicode glyphs only. The single permitted URL in the file is `FORMSUBMIT_ENDPOINT`.
- **No persistence of any kind.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies. A refresh resetting the board to seed data is intended behaviour and is announced in the header.
- **Must run from `file://`** by double-clicking. Never introduce anything that needs a server (ES modules, `fetch` of local files, CORS-sensitive APIs).
- **No `alert()` or native `confirm()`.** Errors render inline under each field; delete confirmation is an inline Yes/No row inside the card.
- **No `!important`** in CSS.

## Architecture

`state` is the single source of truth:

```js
const state = { tasks: [], filters: { project, assignee, priority } };
```

The data flow is strictly one-directional — **mutate `state`, then call `renderBoard()`**. Never patch card contents in the DOM directly. `renderCard()` is the only function that produces card HTML; `renderBoard()` rebuilds all four dropzones from `applyFilters()` on every change. The sole permitted direct DOM touch is the `.drag-over` highlight class during a drag (styling only, no content).

Key structural points, all in the `<script>` block of [index.html](index.html), which is divided into 12 numbered comment banners (CONFIG, STATE, HELPERS, SEED DATA, FILTERING, RENDERING, ACTIONS, TOASTS, FORM VALIDATION, FORMSUBMIT, EVENT WIRING, INIT):

- **Option lists are declared once** as `STATUSES` / `PROJECTS` / `CATEGORIES` / `PRIORITIES` constants. Every `<select>` — form, filters, and each card's Move control — is populated from them via `optionsHtml()`. Add a project or category there, not in the markup.
- **`STATUSES` drives the columns.** The four `<section class="column" data-status="...">` elements in the markup must stay in sync with that array; `renderBoard()` looks up `[data-dropzone="<status>"]` and `[data-count="<status>"]` per status.
- **Events are delegated** on `#board` (click, change, and all five drag events), so re-renders never need re-binding.
- **Every user-supplied string passes through `escapeHtml()`** before reaching `innerHTML`. Card HTML is built by string concatenation, so a new field added to `renderCard()` without `escapeHtml()` is an injection hole.
- **Column count badges reflect the *filtered* view; the header summary strip counts *all* tasks.** This asymmetry is deliberate.
- **`idCounter` is module-level** and consumed by both `seedTasks()` and `addTask()` via `nextId()`, which is what keeps new IDs continuing past the seeded `UOB-ITPM-0008`.
- **Dates are compared as `YYYY-MM-DD` strings** against `todayISO()` (local time, not UTC) — never `new Date()` arithmetic. `isOverdue()` additionally requires `status !== "Done"`.

## FormSubmit

`notifyNewTask()` is the only network call in the app. The submit flow is optimistic: validate, add the card to `state`, re-render, reset the form and toast success *before* the fetch is fired. The fetch failing must only ever produce a warning toast — the card stays. Any change here must preserve that isolation.

`FORMSUBMIT_ENDPOINT` ([index.html:739](index.html#L739)) ships as the placeholder `YOUR_EMAIL@example.com`. FormSubmit requires a one-time activation: the first submission emails a confirmation link to that address, and nothing delivers until it is clicked. With the placeholder in place, the "email notification failed" warning toast on every submit is expected, not a bug.

## Verification

There is no test framework. Two checks cover most regressions.

**1. Constraint grep** — the only match in the second command should be the FormSubmit URL:

```bash
grep -inE "localStorage|sessionStorage|indexedDB|document\.cookie|!important" index.html
grep -inE "https?://|<script src|<link |@import" index.html
```

**2. Headless logic test.** Extract the script block and run it under a minimal DOM shim. Note the path quirk: Git Bash's `/tmp` maps to the Windows temp dir, but Node resolves `/tmp` as `C:\tmp` — pass Node absolute Windows-style paths or use the session scratchpad directory.

```bash
sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' > "$SCRATCH/app.js"
node --check "$SCRATCH/app.js"
```

A shim stubbing `document.getElementById`/`querySelector`/`querySelectorAll`/`createElement`, `fetch` (rejecting), and `setTimeout` is enough to `eval` the script and assert on `state`, `applyFilters()`, `moveTask()`, `deleteTask()`, `isOverdue()`, `escapeHtml()` and the HTML returned by `renderCard()` — including escaping assertions against `<script>`/`<img onerror>` payloads.

**3. Browser.** `start index.html` can report success without actually launching anything on this machine; launch the browser binary directly instead:

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" "file:///C:/Users/microsoft/Desktop/projects/kanbanboard/index.html"
```

Drag-and-drop behaviour and the sub-768px stacking cannot be verified headlessly — they need a real browser window.

## Branding

Internal demo only. Use the neutral "UOB IT PMO" text wordmark and the corporate blue palette in `:root`. Do not add UOB's real logo or trademarks, or make the UI imitate an official UOB system.

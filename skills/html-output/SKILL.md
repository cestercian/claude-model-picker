---
name: html-output
description: Write plans, specs, explorations, code-review and PR explainers, reports, design prototypes and throwaway editors as one self-contained HTML file instead of Markdown. Use when output would be long Markdown (~100+ lines), needs a diagram, table, color, diff or side-by-side view, will be shared or read once, or when picking, sorting or tuning in a UI beats describing it. Also for "HTML file", "HTML artifact", "explainer", "visual plan", "report".
---

# HTML output

Markdown's main benefit is hand-editing. Once the person asks the agent to change the file, or uses it only as a spec or reference, its limits cost more than its simplicity saves. One HTML file can carry almost anything the agent can read, be laid out for one careful read, open from a link, and take input back.

## Why HTML over Markdown

- **Density:** tables, real CSS swatches, inline SVG diagrams, highlighted code, form controls, canvas and images all have a native HTML form.
- **Readability past ~100 lines:** people rarely read Markdown past ~100 lines; HTML adds a summary up top, tabs, collapsible detail, diagrams beside text, a TOC, links between files, and phone layouts.
- **Sharing:** browsers don't render Markdown, so it travels as an unopened attachment; HTML opens anywhere from a link.
- **Two-way interaction:** sliders, toggles and drag-and-drop, plus a button that turns the reader's choices into text to paste back.
- **Data ingestion:** a coding agent can first read the codebase, MCP tools (Slack, Linear/Jira, docs), a browser and git history, then present it. Never invent data; mark unverified claims.

**Markdown failure signs (switch to HTML):** ASCII box diagrams, colors as hex codes plus Unicode blocks, tables too wide to read, a plan past ~100 lines with the key decision buried, a document meant for someone other than the person in the session.

Before writing, decide how the page will be used (read once, compare options, share with a reviewer, feed an implementing agent, edit and export) and let that shape the layout. Start fresh per kind of task; capture recurring patterns as a template or skill.

## When to use HTML vs keep Markdown

**HTML:** plans, specs and explorations; code review, PR descriptions, explanations of unfamiliar code; design work and interaction prototypes (even if the product is Swift, React, etc.); reports, research, explainers, status and incident reports; throwaway editors that end in an export.

**Markdown:** repo docs (README, CLAUDE.md, SKILL.md, changelogs, ADRs the tooling expects); files maintained by hand; short chat answers; whenever the person asks for Markdown or plain text. Some people want HTML for nearly everything: follow their lead and the repo's conventions.

## Use cases

**1. Specs, planning, exploration.** Produce a small web of linked files, not one plan: **explore** distinct approaches side by side, each tagged with its trade-off (complexity, performance, UX, risk, effort) → **expand** the chosen one (mockups, type/API sketches, data flow) → **plan** (steps, files touched, key snippets, risks, open questions) → **implement** in a fresh session given these files → **verify** against the same files, not just passing tests. Page: summary and recommendation first, grid or tabs for alternatives, trade-off table, diagrams beside text, open questions last; link siblings (`exploration.html`, `ui-options.html`, `plan.html`).

**2. Code review and understanding.** For reviewing code the agent just wrote, someone's PR (focus where the reviewer knows least), explaining your own PR, or mapping an unfamiliar area. Page: summary and top findings; the real diff grouped by file with margin notes; severity colors (blocking, should fix, nit, question, praise) with legend and filter; a flow or module diagram where logic is non-obvious; a list of what was not reviewed or verified.

**3. Design and prototypes.** HTML as a sketch surface even when shipping in React, SwiftUI or Kotlin; prototype behavior too (animations, transitions, hover/press, loading). Page: live component front and center, sliders/knobs/toggles beside it, current values shown live as code (CSS, Swift, JSON) with a copy-params button; use the project's real token values when they exist.

**4. Reports, research, learning.** Pull from Slack, code, git, trackers and the web into a long doc, interactive explainer or one-idea-per-screen deck; feature summaries, weekly status, incident reports (timeline, impact, root cause, follow-ups). Page: sticky TOC or tabs, collapsible steps, tabbed snippets for variants, SVG diagrams, gotchas/open questions last, sources cited (file paths, commit hashes, thread titles).

**5. Throwaway editors.** One file for one job when something is hard to describe in a text box. **Export rule:** every editor ends with a "copy as JSON / prompt / Markdown / diff" button, or the work is lost. Kinds: reorder/triage/bucket cards, structured config forms with dependency hints and validation, prompt/template tuning with live preview and counter, dataset curation, annotation with locations, and picking awkward values (colors, easing curves, crop regions, cron, regex with live matches). Page: real data inline (JS object or `<script type="application/json">`), initial state is the agent's best guess, validate as they go, export button always visible with a preview of what will be copied.

## FAQ

- **Tokens?** HTML costs more than Markdown, but gets read and carries more, so decisions improve; keep pages lean (no frameworks, no decoration).
- **When Markdown?** Repo docs, hand-edited files, short chat replies, anything a tool or person expects as Markdown.
- **Multiple files instead of one plan?** Yes, usually: e.g. an implementation plan, a separate UI-options exploration, a component page. Keep them for later work and verification.

The goal behind all of it: keep the person engaged with what the agent decides. Make decisions visible, disagreement cheap, and sending changes back easy.

## Building rules

- **One self-contained file.** Inline CSS and JS; no build step, npm or frameworks. If a dependency is truly unavoidable, load one pinned version from a major CDN and say so on the page.
- **Diagrams in inline SVG** with a `viewBox`: boxes, labels, arrows; `currentColor`/variables for fills; highlight the key path with `var(--accent)`, dim the rest.
- **Real data only.** Embed what you read; label estimates and unverified claims.
- **Organize for reading once.** Summary first; tabs, anchors or `<details>` for depth; gotchas and open questions last.
- **Readable everywhere.** Light and dark mode via color variables only, phone width, keyboard-usable controls, enough contrast, never color alone (pair with label or icon). Wrap wide tables in `.table-wrap`; option grids use `repeat(auto-fit, minmax(260px, 1fr))`.
- **Interactive pages end with an export** (snippet below), kept visible (`position: sticky; bottom: 0`) with a live preview.
- **Save sensibly.** The repo's docs/plans folder or a scratch folder, never `src/` or anywhere it could be bundled. Descriptive names (`2026-11-rate-limiter-explainer.html`).
- **Open it and give the path.** `open` (macOS), `xdg-open` (Linux), `start` (Windows); always state the absolute path. Offer a shareable link if the environment can publish pages.
- **Keep it as a reference** and pass it to the implementing and verifying agents.
- **Before handing over:** look at it (or re-read for unclosed tags/broken script), check phone width and dark mode, click every control, paste the export to confirm it is complete, confirm facts trace to sources you read. Reply with the path, a one-line summary and how to use any export.

Other patterns, no snippet needed: diffs as grid rows (line no. / code / note column that drops below on phones; escape `<`, `>`, `&`); tabs via `role=tab` buttons toggling `hidden` panels with page-unique ids; sliders driving CSS variables plus replay and reset buttons; native drag-and-drop between columns with move buttons as a touch fallback, export reading order from the DOM.

### Page skeleton

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Rate limiter explainer</title>
<style>
  :root {
    --bg: #ffffff; --fg: #1a1a1a; --muted: #666; --line: #e3e3e3;
    --panel: #f6f6f4; --accent: #3b5bdb;
    --red: #c92a2a; --amber: #b7791f; --green: #2b8a3e; --blue: #1c7ed6;
    color-scheme: light dark;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #161616; --fg: #e8e8e8; --muted: #9a9a9a; --line: #2e2e2e;
      --panel: #1f1f1f; --accent: #7c9cff;
      --red: #ff6b6b; --amber: #f0b44c; --green: #69db7c; --blue: #74c0fc;
    }
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--fg);
         font: 16px/1.55 system-ui, -apple-system, "Segoe UI", sans-serif; }
  main { max-width: 860px; margin: 0 auto; padding: 24px 16px 80px; }
  pre { background: var(--panel); padding: 12px; border-radius: 8px; overflow-x: auto;
        font: 13px/1.5 ui-monospace, Menlo, monospace; }
  table { border-collapse: collapse; width: 100%; }
  th, td { border-bottom: 1px solid var(--line); padding: 6px 8px; text-align: left; }
  .table-wrap { overflow-x: auto; }
  svg { max-width: 100%; height: auto; }
  svg text { fill: var(--fg); }
</style>
</head>
<body>
<main>
  <h1>How the rate limiter works</h1>
  <p>One-paragraph summary: what this page answers and the key takeaway.</p>
  <nav><a href="#flow">Flow</a> · <a href="#gotchas">Gotchas</a></nav>
  <section id="flow"><h2>Flow</h2><!-- inline SVG --></section>
  <section id="gotchas"><h2>Gotchas</h2></section>
</main>
</body>
</html>
```

### Export button (with `file://` fallback)

```html
<button id="copy">Copy as JSON</button>
<pre id="preview"></pre>
<script>
function exportText() { return JSON.stringify(state, null, 2); } // build from current state
function copy(text, btn) {
  const done = () => { const t = btn.textContent; btn.textContent = 'Copied'; setTimeout(() => btn.textContent = t, 1200); };
  if (navigator.clipboard && window.isSecureContext) {
    navigator.clipboard.writeText(text).then(done, () => fallback(text, done));
  } else fallback(text, done);
}
function fallback(text, done) {
  const ta = Object.assign(document.createElement('textarea'), { value: text });
  ta.style.position = 'fixed'; ta.style.opacity = '0';
  document.body.appendChild(ta); ta.select();
  try { document.execCommand('copy'); done(); } catch { alert('Select the preview text and copy it manually.'); }
  ta.remove();
}
document.getElementById('copy').onclick = e => copy(exportText(), e.currentTarget);
</script>
```

Export formats: **prompt** (instruction plus data), **JSON/YAML** (full structure), **diff** (only changes, `key: old -> new`), **Markdown** (lists or tables).

---

Based on: "Using Claude Code: The unreasonable effectiveness of HTML" by Thariq Shihipar (claude.dev Blog, Anthropic, May 20, 2026), https://claude.dev/blog/using-claude-code-the-unreasonable-effectiveness-of-html/

---
name: html-output
description: Write plans, specs, explorations, code-review and PR explainers, reports, design prototypes and throwaway editors as one self-contained HTML file instead of Markdown. Use when output would be long Markdown (~100+ lines), needs a diagram, table, color, diff or side-by-side view, will be shared or read once, or when picking, sorting or tuning in a UI beats describing it. Also for "HTML file", "HTML artifact", "explainer", "visual plan", "report".
---

# HTML output

Markdown is the default way agents talk to people: simple, portable, easy to hand-edit. That last benefit matters less once the person stops editing the file themselves and instead asks the agent to change it, or uses it only as a spec or reference. At that point Markdown's limits start to cost more than its simplicity saves: long files go unread, diagrams turn into ASCII art, colors get approximated with Unicode blocks, and the file is awkward to send to anyone.

A single HTML file fixes most of this. It can carry almost anything the agent can read, it can be laid out for one careful read-through, it opens in any browser from a link, and it can take input back from the reader. This skill explains when to reach for it, what each kind of page is for, and how to build one well.

## Why HTML beats Markdown for this work

### Information density

HTML can do everything Markdown does (headings, lists, emphasis, code) and much more. Practically any information the agent can gather has a natural HTML form:

| Kind of information | HTML form |
|---|---|
| Tabular data | `<table>`, sortable if useful |
| Design data (colors, spacing, type, radii, shadows) | real CSS, rendered as swatches and live components |
| Illustrations and diagrams | inline `<svg>` |
| Code snippets | `<pre><code>`, with highlighting done in CSS or a small inline script |
| Interactions | form controls and buttons wired with plain JavaScript and CSS |
| Workflows and processes | SVG flowcharts, or HTML boxes and arrows |
| Spatial data | absolutely positioned elements or a `<canvas>` |
| Images | `<img>` (inline as `data:` URIs or reference local files) |

Because so little is out of reach, HTML is an efficient channel for the agent to explain something in depth and for the person to review it.

### What it replaces

When Markdown is the only option, the agent falls back on workarounds that are worse for everyone:
- ASCII box diagrams that break at the first wrap or proportional font.
- Color palettes "shown" as hex codes next to Unicode block characters that only roughly suggest the shade.
- Tables too wide to read in a terminal or editor.
- Long linear plans where the important decision is buried on line 140.

If you notice yourself doing any of these, switch to HTML.

### Visual clarity and the ~100-line limit

As agents take on bigger tasks they produce bigger specs and plans. In practice, people rarely read a Markdown file past about a hundred lines, and almost never get colleagues to read one. HTML lets you shape the page around how it will be read: a summary up top, tabs for alternatives, collapsible detail, diagrams next to the text they explain, an in-page table of contents, and links between related files. It can also be responsive, so the same page reads properly on a phone.

### Sharing

Browsers do not render Markdown well, so a Markdown file usually travels as an email or chat attachment that nobody opens. An HTML file can be uploaded once and shared as a link that opens anywhere. A spec, report or PR write-up has a much better chance of being read when it is a page.

### Two-way interaction

A page can take input, not just display output. Add sliders and knobs to tune a design, toggles to try variants of an algorithm, drag-and-drop to reorder items. Then give the reader a button that turns their choices into text (a prompt, JSON, Markdown, a diff) they can paste back into the agent. This creates a small editing environment built for exactly the problem at hand.

### Data ingestion

The real advantage of having a coding agent produce these pages, rather than a chat tool, is everything it can read first:
- **The file system and codebase**: source, configs, tests, previously generated HTML files.
- **Connected tools via MCP servers**: Slack threads, Linear or Jira tickets, docs, dashboards.
- **A web browser** (for example a browser-control extension) for live pages and authenticated tools.
- **Git history**: who changed what, when and why; blame; PR history.

Gather from these sources, then present the result. Do not invent data. Mark anything unverified as such on the page.

## Getting started

Very little setup is needed. A request such as "make an HTML file" or "make this an HTML artifact" is enough. What matters is being clear about what the page must do and how the person will use it: read it once, compare options, share it with a reviewer, feed it to an implementing agent, or edit data in it and export the result. Decide that before writing, and let it shape the layout.

Start from scratch for each new kind of task. Once a pattern recurs (the same review layout, the same status report), it is worth capturing as a template or its own skill.

## When to use HTML, and when not

Use HTML for:
- Plans, specs and explorations, especially with alternatives, mockups, data flow or code to review.
- Code review, PR descriptions and explanations of unfamiliar code.
- Design work and interaction prototypes, even when the shipped code will be Swift, React or something else.
- Reports, research summaries, explainers, status and incident reports.
- Throwaway editors for one piece of data, always ending in an export.

Keep Markdown for:
- Files that live in the repo as documentation: README, CLAUDE.md, SKILL.md, changelogs, ADRs that the repo's tooling expects as Markdown.
- Anything the person maintains by hand in a text editor.
- Short answers in chat. A three-paragraph answer does not need a page.
- Situations where the person has asked for Markdown or a plain-text format.

Signs you should have used HTML: an ASCII diagram, colors described in words or Unicode blocks, a table too wide to read, a plan past roughly 100 lines, or a document meant for someone other than the person in the session.

Some people end up using HTML for nearly everything; that is a valid preference, but follow the person's lead and the repo's conventions.

## Use cases

### 1. Specs, planning and exploration

HTML gives the agent room to work through a problem visibly. Instead of one Markdown plan, expect to produce a small web of linked HTML files, each covering one stage.

**The workflow:**
1. **Explore.** Brainstorm several distinctly different approaches and put them on one page side by side, each labeled with the trade-off it makes (complexity, performance, UX, risk, effort).
2. **Expand.** Take the chosen direction further: mockups of the screens, sketches of the type interfaces or API shapes, data-flow diagrams.
3. **Plan.** Write the implementation plan: steps, files touched, data flow, key code snippets for review, risks and open questions.
4. **Implement.** Start a fresh session and hand it all of these files as input. They carry far more context than a one-line task.
5. **Verify.** Give the verification agent (or reviewer) the same files, so it checks the work against what was actually intended, not just against whether the tests pass.

Keep the files afterward as references. They explain why the code looks the way it does.

**Typical requests:**
- "I can't decide on a direction for the onboarding screen. Give me six genuinely different approaches, varying layout, tone and density, in one HTML grid so I can compare them, each tagged with its trade-off."
- "Write a thorough implementation plan as an HTML file with mockups, a data-flow diagram and the code snippets I should review. Make it easy to digest."
- "Show me three ways we could restructure this module, with the pros and cons of each and a sketch of the resulting file layout."

**Good for:** exploring alternative implementations in code; trying several visual designs at once; laying out a plan someone will actually read.

**Page shape:** a summary and recommendation at the top; a grid or tabs for alternatives; a comparison table of trade-offs; diagrams beside the text; open questions at the end. Link sibling files (`exploration.html`, `ui-options.html`, `plan.html`) to each other.

### 2. Code review and understanding

Code is hard to review in Markdown. In HTML you can render the actual diff, put annotations in the margin beside the lines they refer to, draw how modules connect, and step through a flow.

**Use it to:**
- Understand code the agent itself just wrote before accepting it.
- Review someone else's PR, focusing on the parts the reviewer knows least.
- Explain your own PR to the people reviewing it.
- Understand an unfamiliar area of a codebase.

**Typical requests:**
- "Help me review this PR as an HTML page. I don't know the streaming and backpressure logic well, so concentrate there. Render the real diff with notes in the margin, color findings by severity, and add whatever else helps explain it."
- "Write up my branch as a PR explainer a reviewer can read in five minutes: what changed, why, a diagram of the new flow, and the risky lines."
- "Map how a request moves through the auth middleware, with the relevant snippets annotated."

**Page shape:** a summary of what the change does and the top findings; the diff grouped by file with margin notes; severity colors (for example: blocking, should fix, nit, question, praise) with a legend and a filter; a flowchart or module diagram where the logic is non-obvious; a list of things not reviewed or not verified.

### 3. Design and prototypes

HTML is extremely expressive for visual design, which is why design tools are often built on it. It works as a sketch surface even when the product is not a web app: design in HTML, settle the details, then write the final version in React, SwiftUI, Kotlin or whatever the project uses.

You can also prototype behavior, not just appearance: animations, transitions, hover and press states, loading sequences. Give the reader sliders, knobs and toggles to dial in exactly what they want, plus a button to copy the resulting parameters.

**Typical requests:**
- "Prototype a new checkout button: on click it plays an animation and then quickly turns purple. Give me sliders and options to try variations, and a button that copies the parameters I like."
- "Render our design tokens (colors, type scale, spacing, radii) as they will actually look, with each component in all its states."
- "Show this card component at five density settings side by side so we can pick one."

**Good for:** design-system reference pages; adjusting existing components; visualizing a component library; prototyping animations and interactions.

**Page shape:** the live component front and center; controls grouped beside it; the current values shown as code (CSS, Swift, JSON) that updates live; a copy button for those values. Use the project's real token values when they exist in the codebase.

### 4. Reports, research and learning

A coding agent is good at pulling information together from many sources (Slack, the codebase, git history, ticket trackers, the web) and turning it into something easy to read. The output can be a long document, an interactive explainer, or a slide-style deck with one idea per screen. Use SVG for diagrams.

**Typical requests:**
- "I don't really understand how our rate limiter works. Read the code and make one HTML explainer: a diagram of the token-bucket flow, the three or four key snippets annotated, and a gotchas section at the end. Optimize it for reading once."
- "Summarize what the team shipped this week from merged PRs and the release channel, as a status report."
- "Write an incident report for yesterday's outage from the alert thread, the deploy log and the fix commit, with a timeline."

**Good for:** feature summaries; explainers; weekly status reports; incident reports (timeline, impact, root cause, follow-ups); SVG illustrations, flowcharts and technical diagrams.

**Page shape:** navigation (sticky table of contents or tabs); collapsible step-by-step sections; tabbed code snippets when showing variants or languages; a gotchas or open-questions section at the end; sources listed with file paths, commit hashes or thread titles.

### 5. Custom editing interfaces

Some things are hard to describe in a text box. For those, build a throwaway editor for the exact data at hand: not a product and not a reusable tool, just one HTML file made for this one job.

**The export rule:** every editor must end with an export. A button such as "copy as JSON", "copy as prompt", "copy as Markdown" or "copy diff" turns what the person did in the UI back into text they can paste into the agent or commit to a file. Without it the work done in the page is lost. The person stays in the loop, and the loop gets much shorter.

**Kinds of editors:**
- **Reordering, triaging or bucketing** anything: tickets, test cases, user feedback, bugs. Draggable cards across columns, pre-sorted by the agent's best guess.
- **Structured config editing**: feature flags, environment variables, JSON or YAML with constraints. Forms grouped by area, dependency hints, validation warnings, an export of only what changed.
- **Prompt, template or copy tuning with live preview**: an editable template with variable slots highlighted, sample inputs re-rendered as you type, a character or token counter.
- **Dataset curation**: approve or reject rows, tag examples, then export the selection.
- **Annotation**: mark up a document, transcript or diff and export the annotations with their locations.
- **Picking values that are awkward to describe in words**: colors, easing curves, crop regions, cron schedules, regular expressions (with live match highlighting).

**Typical requests:**
- "I need to reprioritize these 30 tickets. Give me draggable cards in Now / Next / Later / Cut columns, pre-sorted by your best guess, with a 'copy as Markdown' button that outputs the final order and a one-line rationale per column."
- "Build a form editor for this feature-flag config. Group flags by area, show which depend on which, warn me if I turn on a flag whose prerequisite is off, and give me a 'copy diff' containing only the keys that changed."
- "I'm tuning this system prompt. Make a side-by-side editor: the prompt on the left with variable slots highlighted, three sample inputs on the right that re-render live, a length counter and a copy button."

**Page shape:** load the real data inline (as a JS object or JSON in a `<script type="application/json">` tag); make the initial state the agent's best guess so the person only corrects it; validate as they go; keep the export button always visible; show what will be exported before copying.

## FAQ

**Isn't HTML less efficient?** It usually costs more tokens than the equivalent Markdown. The trade is worth it: the page carries more, and the person is far more likely to actually read it, which leads to better decisions and better final output. With large context windows the extra tokens are rarely noticeable. Still, keep pages lean: no frameworks, no decorative bloat.

**When is Markdown still the right choice?** Repo documentation, files people edit by hand, short chat replies, and anything a tool or person specifically expects as Markdown. Some heavy users barely use Markdown at all anymore; match the person's preference.

**Does this replace a single plan?** Usually it replaces one plan with a few files, each for a different stage or part: an implementation plan, a separate exploration of UI options, perhaps a component page listing every design. Keep them as references for later work and for verification.

## Staying in the loop

The deeper reason for all of this: as the agent takes on more work, people tend to skim its plans and hand decisions off without really engaging. A well-built HTML page (one that is pleasant to read, shows the trade-offs visually, and lets the person push choices back) keeps them engaged with what the agent is deciding. Build pages with that goal in mind: make the decisions visible, make disagreement cheap, and make it easy to send changes back.

## Building the file

### Rules

- **One self-contained file.** Inline all CSS and JS. Draw diagrams in inline SVG. No build step, no npm, no frameworks. If a dependency is truly unavoidable (for example a charting library for a large dataset), load one pinned version from a major CDN and say so on the page.
- **Real data only.** Embed what you read from the codebase and tools. Label estimates and unverified claims.
- **Organize for reading once.** Summary first; tabs, anchors or collapsible sections for depth; gotchas and open questions last.
- **Readable everywhere.** Light and dark mode, phone width, keyboard-usable controls, sufficient contrast. Do not rely on color alone; pair it with a label or icon.
- **Interactive pages end with an export** (see the export rule above).
- **Save it in a sensible place.** The repo's docs or plans folder if the project has one, or a scratch folder outside source directories. Never inside `src/` or similar where it could get bundled. Use descriptive names (`2026-11-rate-limiter-explainer.html`, `checkout-plan.html`).
- **Open it and give the path.** `open file.html` on macOS, `xdg-open file.html` on Linux, `start file.html` on Windows. Always state the absolute path in your reply. If the environment can publish pages as shareable links, offer that for anything meant for other people.
- **Keep it as a reference.** Pass plans and specs to the session that implements them and to the agent that verifies the work.

### Page skeleton

Start from this and add only what the page needs. It handles light and dark mode, phone widths and readable line length.

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
  h1 { font-size: 1.8rem; line-height: 1.2; margin: 0 0 8px; }
  h2 { margin-top: 2.2em; padding-top: .4em; border-top: 1px solid var(--line); }
  .lede { color: var(--muted); font-size: 1.1rem; }
  pre { background: var(--panel); padding: 12px; border-radius: 8px;
        overflow-x: auto; font: 13px/1.5 ui-monospace, Menlo, monospace; }
  table { border-collapse: collapse; width: 100%; }
  th, td { border-bottom: 1px solid var(--line); padding: 6px 8px; text-align: left; vertical-align: top; }
  .table-wrap { overflow-x: auto; }
  .callout { border-left: 4px solid var(--accent); background: var(--panel); padding: 10px 14px; border-radius: 4px; }
  svg { max-width: 100%; height: auto; }
  svg text { fill: var(--fg); }
  @media (max-width: 600px) { body { font-size: 15px; } h1 { font-size: 1.45rem; } }
</style>
</head>
<body>
<main>
  <h1>How the rate limiter works</h1>
  <p class="lede">One-paragraph summary: what this page answers and the key takeaway.</p>
  <nav><a href="#flow">Flow</a> · <a href="#code">Key code</a> · <a href="#gotchas">Gotchas</a></nav>
  <section id="flow"><h2>Flow</h2><!-- inline SVG diagram --></section>
  <section id="code"><h2>Key code</h2></section>
  <section id="gotchas"><h2>Gotchas</h2></section>
</main>
</body>
</html>
```

Notes:
- Define every color as a variable and use only variables, so dark mode stays consistent. In SVG, use `fill="currentColor"` or `style="fill: var(--fg)"` rather than hard-coded black.
- Wrap wide tables in `<div class="table-wrap">` so they scroll inside themselves instead of making the whole page scroll sideways.
- Grids of options: `display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 16px;` collapses to one column on phones without media queries.

### Diagrams in inline SVG

Keep them simple: boxes, labels, arrows. Use a `viewBox` so they scale.

```html
<svg viewBox="0 0 520 90" role="img" aria-label="Request passes through bucket check to handler">
  <defs><marker id="arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
    <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/></marker></defs>
  <g fill="none" stroke="currentColor" stroke-width="1.5">
    <rect x="10" y="25" width="120" height="40" rx="6"/>
    <rect x="200" y="25" width="120" height="40" rx="6"/>
    <rect x="390" y="25" width="120" height="40" rx="6"/>
    <path d="M130 45 H198" marker-end="url(#arr)"/>
    <path d="M320 45 H388" marker-end="url(#arr)"/>
  </g>
  <g font-size="13" text-anchor="middle" fill="currentColor">
    <text x="70" y="50">Request</text><text x="260" y="50">Bucket check</text><text x="450" y="50">Handler</text>
  </g>
</svg>
```

Highlight the path that matters with `stroke: var(--accent)` and dim the rest with lower opacity.

### Diff with margin notes

Render the real diff (from `git diff`) as rows, with notes in a right-hand column on wide screens that drop below the line on phones. Escape `<`, `>` and `&` in code before inserting it.

```html
<style>
  .diff { font: 13px/1.5 ui-monospace, Menlo, monospace; border: 1px solid var(--line); border-radius: 8px; overflow: hidden; }
  .diff .file { background: var(--panel); padding: 6px 10px; font-weight: 600; }
  .row { display: grid; grid-template-columns: 3.5em 1fr minmax(220px, 34%); border-top: 1px solid var(--line); }
  .ln { color: var(--muted); text-align: right; padding: 0 8px; user-select: none; }
  .code { white-space: pre; overflow-x: auto; padding: 0 8px; }
  .add { background: color-mix(in srgb, var(--green) 14%, transparent); }
  .del { background: color-mix(in srgb, var(--red) 14%, transparent); }
  .note { padding: 4px 10px; font-family: system-ui, sans-serif; font-size: 13px; border-left: 3px solid transparent; }
  .note.blocking { border-color: var(--red); }
  .note.should   { border-color: var(--amber); }
  .note.nit      { border-color: var(--blue); }
  .note .tag { font-weight: 700; text-transform: uppercase; font-size: 11px; margin-right: 6px; }
  @media (max-width: 700px) {
    .row { grid-template-columns: 3em 1fr; }
    .note { grid-column: 1 / -1; }
    .note:empty { display: none; }
  }
</style>
<div class="diff">
  <div class="file">src/stream/buffer.ts</div>
  <div class="row del"><span class="ln">41</span><span class="code">-  if (queue.length > MAX) drop();</span><span class="note"></span></div>
  <div class="row add"><span class="ln">41</span><span class="code">+  if (queue.length >= MAX) await drain();</span>
    <span class="note blocking"><span class="tag">Blocking</span>drain() can wait forever if the consumer has closed; add a timeout.</span></div>
</div>
```

Add a legend for the severities, and a small filter (checkboxes that toggle a class on the container) when there are many notes. Put a findings summary above the diff that links to each note by `id`.

### Tabs and collapsible sections without libraries

Collapsible sections are built in: use `<details>`.

```html
<details open><summary>Step 1: request arrives</summary><p>...</p></details>
<details><summary>Step 2: bucket refills</summary><p>...</p></details>
```

Tabs with a few lines of JavaScript, keyboard-accessible:

```html
<style>
  .tabs [role=tablist] { display: flex; gap: 4px; border-bottom: 1px solid var(--line); overflow-x: auto; }
  .tabs [role=tab] { background: none; border: 0; padding: 8px 12px; color: var(--muted); cursor: pointer; font: inherit; }
  .tabs [role=tab][aria-selected=true] { color: var(--fg); border-bottom: 2px solid var(--accent); }
</style>
<div class="tabs">
  <div role="tablist">
    <button role="tab" aria-selected="true" data-tab="a">Option A</button>
    <button role="tab" aria-selected="false" data-tab="b">Option B</button>
  </div>
  <div role="tabpanel" id="a">...</div>
  <div role="tabpanel" id="b" hidden>...</div>
</div>
<script>
document.querySelectorAll('.tabs').forEach(t => t.addEventListener('click', e => {
  const b = e.target.closest('[role=tab]'); if (!b) return;
  t.querySelectorAll('[role=tab]').forEach(x => x.setAttribute('aria-selected', x === b));
  t.querySelectorAll('[role=tabpanel]').forEach(p => p.hidden = p.id !== b.dataset.tab);
}));
</script>
```

Panel `id`s must be unique across the whole page; prefix them when there are several tab groups.

### Export button (copy to clipboard)

Every interactive page needs one. Build the export text from the page's state, show it in a preview, and copy it. The fallback covers pages opened from `file://` where the Clipboard API may be blocked.

```html
<button id="copy">Copy as JSON</button>
<pre id="preview"></pre>
<script>
function exportText() {
  // Build from current state; keep it paste-ready for the agent.
  return JSON.stringify(state, null, 2);
}
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

Export formats that work well:
- **Copy as prompt:** a short instruction plus the data, for example "Apply these changes to flags.yaml:" followed by the changed keys.
- **Copy as JSON / YAML:** the full edited structure, ready to save over the original.
- **Copy diff:** only what changed, as `key: old -> new` lines or a unified diff.
- **Copy as Markdown:** lists or tables for tickets, decisions and rationales.

Keep the export button visible (for example `position: sticky; bottom: 0`) and update the preview live.

### Controls for tuning

Sliders that update a live preview and the exported values:

```html
<label>Duration <input type="range" id="dur" min="100" max="1500" step="50" value="400"> <output id="durv">400</output> ms</label>
<script>
const dur = document.getElementById('dur');
dur.oninput = () => {
  document.getElementById('durv').textContent = dur.value;
  document.documentElement.style.setProperty('--dur', dur.value + 'ms');
};
</script>
```

Drive the prototype through CSS variables so one input can affect many elements. Add a "replay" button for animations and a "reset to defaults" button.

### Drag and drop between columns

Native HTML drag and drop is enough for bucketing:

```html
<div class="col" data-col="now"><div class="card" draggable="true" data-id="T-12">Fix login retry</div></div>
<div class="col" data-col="later"></div>
<script>
let dragged;
document.addEventListener('dragstart', e => { dragged = e.target.closest('.card'); });
document.querySelectorAll('.col').forEach(c => {
  c.addEventListener('dragover', e => e.preventDefault());
  c.addEventListener('drop', e => { e.preventDefault(); if (dragged) c.appendChild(dragged); });
});
</script>
```

Native drag does not work on most touch screens, so also give each card a small select or move-buttons to change its column, and make sure the export reads the order from the DOM.

### Final checks before handing it over

- Open the page and look at it, or at least re-read the HTML for unclosed tags and broken script.
- Check it at phone width and in dark mode (no hard-coded colors left behind).
- Click every control and the export button; paste the export and confirm it is complete and well-formed.
- Confirm the facts on the page trace back to files, commits or threads you actually read.
- Reply with the absolute path, a one-line summary of what the page contains, and how to use any export.

---

Based on: "Using Claude Code: The unreasonable effectiveness of HTML" by Thariq Shihipar (claude.dev Blog, Anthropic, May 20, 2026), https://claude.dev/blog/using-claude-code-the-unreasonable-effectiveness-of-html/

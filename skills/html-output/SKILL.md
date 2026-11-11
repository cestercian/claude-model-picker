---
name: html-output
description: Write plans, specs, explorations, PR and code explainers, reports, design prototypes and throwaway editing tools as a single self-contained HTML file instead of Markdown. Use when output would be a long Markdown document (over ~100 lines), would need a diagram, table, colors, diff or side-by-side comparison, is meant to be shared or read once by someone else, or when the user would find it easier to pick, sort, tune or annotate something in a UI than to describe it in text. Also use when asked for an "HTML file", "HTML artifact", "explainer", "visual plan" or "report".
---

# HTML output

Long Markdown gets skimmed; HTML can carry diagrams, tables, color, diffs and controls, and can be opened and shared as a link. Use it when the reader needs to actually take the content in, or act on it.

## When to use HTML, and when not

Use HTML for:
- **Plans, specs and explorations.** Several approaches side by side with trade-offs; mockups; then an implementation plan. Prefer a few linked files (exploration, UI options, plan) over one long file.
- **Code review and understanding.** Render the real diff with margin notes colored by severity; flowcharts of the module; annotated snippets. For creating a PR, reviewing one, or explaining an unfamiliar area.
- **Design and prototypes.** Tokens and components as they will look; animations with sliders to tune them, even when the final code is Swift or React.
- **Reports, research, learning.** Explainers with navigation, collapsible steps and tabbed snippets; status and incident reports; SVG diagrams.
- **Throwaway editors** for one piece of data: drag tickets into buckets, edit a config with validation, tune a prompt with live preview, approve or reject rows, pick colors, easing curves, crop regions, cron schedules or regexes.

Keep Markdown for: files that live in the repo as docs (README, CLAUDE.md, SKILL.md, changelogs), anything the user edits by hand, and short answers in chat.

Signs you should have used HTML: an ASCII diagram, colors described in words or unicode blocks, a table too wide to read, or a plan past ~100 lines.

## Building the file

- **One self-contained file.** Inline CSS and JS. Draw diagrams in inline SVG. Avoid external dependencies; if one is unavoidable, load a pinned version from a major CDN.
- **Use what Claude Code can reach.** Read the codebase, git history, connected tools (Slack, Linear…) and the web, then present what you found. Don't invent data; mark anything unverified.
- **Organize for reading once.** Summary first; tabs, anchors or collapsible sections for depth; a gotchas or open-questions section at the end. Make it readable on a phone and in light and dark mode.
- **Interactive tools end with an export.** Every editor gets a "copy as prompt", "copy as JSON", "copy as Markdown" or "copy diff" button that turns what the user did into text they can paste back into Claude Code or commit. Without it, the work in the UI is lost.
- **Show it.** Save it where the user can find it (the repo's docs or a scratch folder, not inside source directories), open it (`open file.html` on macOS, `xdg-open` on Linux), and give the path.
- **Keep it as a reference.** Plans and specs in HTML are good input for the session that implements them and for the agent that verifies the work; pass the files in.

## Example prompts this skill should handle

- "Help me review this PR. I don't know the streaming/backpressure logic, so focus there."
- "I don't understand how our rate limiter works. Explain it with a diagram and the key snippets."
- "Prototype a checkout button animation with sliders to try options, and a button to copy the parameters."
- "Reprioritize these 30 tickets into Now / Next / Later / Cut and give me the result as Markdown."

## Sources

- Thariq Shihipar, ["Using Claude Code: The unreasonable effectiveness of HTML"](https://claude.dev/blog/using-claude-code-the-unreasonable-effectiveness-of-html/), claude.dev Blog (Anthropic), May 20, 2026.

# Review guide

<!-- Owned by code owners. Keep it short. Replace the examples with this repo's own. -->

## What failure looks like here

<!-- e.g. for docs: "it lies"; for config: "it misroutes"; for billing: "it charges wrong". -->

## Focus on

- Correctness bugs and regressions
- Security issues
- Changes that break a convention in CLAUDE.md, or make a line in it out of date

## Skip

- Style and formatting (the linter handles these)
- Generated files: <!-- paths -->

## Rules for findings

- Cite a file and line as evidence. Don't guess from a name.
- At most 3 nits per review.
- Tag each finding Important, Nit, or Pre-existing.
- Start the summary with a count of findings. If there is nothing to fix, say so plainly.

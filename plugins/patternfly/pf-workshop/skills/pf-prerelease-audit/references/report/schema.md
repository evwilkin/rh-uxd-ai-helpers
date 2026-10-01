# PatternFly Prerelease Report — Data Model

The `pf-prerelease-audit` skill fills in this data model at the end of an audit, then renders it into **both** [`report-template.md`](./report-template.md) and [`report-template.html`](./report-template.html). One data model, two outputs — Markdown for PR/commit/Slack sharing, HTML for stakeholder-facing sharing.

Not every consumer project will have data for every field. Omit sections with no data rather than filling them with placeholder text — a template section with nothing to show should be dropped, not left as "N/A" filler.

## Fields

```text
repo            — consuming repository name or URL
branch          — testing branch name
date            — YYYY-MM-DD
tester          — name, if known

verdict         — one of: "compatible" | "regressions-found" | "blocked"
verdictNote     — one-sentence explanation, e.g. "no breaking changes found" or
                  "2 TypeScript API breaks require source fixes before shipping"

versions[]      — { package, before, after, consumed: bool, note? }
                  `consumed: false` = present on the prerelease manifest but not
                  actually a dependency of this repo — list separately, don't drop silently.
                  `note` — optional, e.g. "unchanged, no prerelease target differs from current"

checks[]        — { name, baselineResult, prereleaseResult, detail }
                  name is one of the repo's actual validation steps (build, tsc, lint,
                  unit tests, cypress, visual). baselineResult/prereleaseResult are one of:
                  "pass" | "fail" | "skip", plus a short detail string
                  (e.g. "3 failed, 4767 passed" or "906/906 passing").
                  Skip baselineResult/prereleaseResult distinction entirely for skills
                  that don't run a baseline pass — just report the single result.

findings[]      — { category, file, description, verdict }
                  category is one of: "typescript-api-break" | "import-path-break" |
                  "css-scss-break" | "runtime-failure" | "bundle-size-change" |
                  "peer-dep-warning" | "build-tooling-artifact"
                  ("build-tooling-artifact" covers things that look like a PF break but
                   aren't — e.g. package-manager resolution or bundler configuration artifacts.)
                  verdict is a short human judgment, e.g. "safe to accept" or
                  "needs investigation" or "fixed — see Fixes Applied".
                  Only include a category header in the rendered report if findings exist
                  for it, OR explicitly state "None observed" — don't imply a category was
                  tested if the skill has no step that would have caught it (e.g. don't
                  list "Bundle size change: None" if nothing measures bundle size).

installNotes[]  — { workaround, cause, outcome }
                  outcome is one of: "worked" | "did-not-work-fallback-used" | "not-needed"
                   Always state what was TRIED even if it failed. Don't only document
                   the happy path.

fixesApplied[]  — { file, description } — source changes made to unblock the bump.
                  Omit section entirely if none were needed.

preExisting[]   — warnings/failures present in both baseline and prerelease — call
                  these out explicitly so the PF team doesn't mistake them for new issues.

recommendations — { forPfTeam: [...], forConsumingTeam: [...] }

env             — { node, packageManager, packageManagerVersion, os, bundler? }
```

## Verdict rules

- `compatible` — no `findings[]` entries beyond `peer-dep-warning`/`build-tooling-artifact`, and no `checks[]` regressions (prerelease result worse than baseline, or a fresh failure with no baseline to compare against).
- `regressions-found` — at least one real finding or check regression, but nothing that blocks a merge outright (e.g. fixable with a documented workaround).
- `blocked` — a regression with no known fix, or a check that fails and can't be attributed to a pre-existing/tooling cause.

## Category vs. skill capability

Before writing `findings[]`, check which categories the audit actually covered. If the project has no bundle-analysis phase, CSS diffing phase, etc., leave that category out of the rendered report rather than asserting "None found" — the latter implies coverage that doesn't exist. Note the gap in Recommendations instead, e.g. "bundle size impact not measured in this run."

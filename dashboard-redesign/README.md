# Dashboard redesign docs automation: handoff package

Everything an implementing agent needs to automate the Dashboard link rewrite and screenshot regeneration in `auth0/docs-v2`.

| File | What it is |
| --- | --- |
| `../dashboard-redesign-docs-automation-plan.md` | The plan: ground truth, decisions, design of each track, PR strategy, milestones, risks |
| `IMPLEMENTATION.md` | Step-by-step instructions for the agent, including the hard rules and the handoff checklist |
| `files/` | Finished, tested code laid out in its target repo paths; copy the tree over the repo root |

Quick start for the agent, from the docs-v2 repo root:

```bash
cp -r /path/to/notes/dashboard-redesign/files/. .
node --test scripts/lib/dashboard-links.test.js scripts/update-dashboard-links.test.js scripts/check-dashboard-links.test.js scripts/screenshots/update-screenshot-refs.test.js
```

Then follow `IMPLEMENTATION.md` from Step 2.

Test status when this package was built (2026-10-07, against upstream `main` at `ae3b25e93`): 21 unit tests passing; the rewriter dry-runs the full tree (9,045 MDX files) in about 3 seconds; the guardrail scans the full tree and reports every legacy link.

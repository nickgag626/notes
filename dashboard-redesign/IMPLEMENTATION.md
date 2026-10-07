# Implementation guide: Dashboard redesign docs automation

Step-by-step instructions for the implementing agent. The design, decisions and risks are in `../dashboard-redesign-docs-automation-plan.md`; read it first. Everything under `files/` is finished, tested code laid out in its target path inside `auth0/docs-v2`. Your job is to apply it, feed it the real data, run it, and prepare PRs. You do not need to write the scripts.

## Hard rules

- Never push, never open or edit a PR, never create a fork. Prepare branches and commits locally, then stop and hand off. The human pushes.
- No AI attribution trailers in commit messages for this repo.
- Every script is a dry run by default. `--fix` is opt-in and is only run after the human has seen the dry-run report.
- Do not run Track 1 `--fix` until the blessed dashboard URL form is confirmed (plan, "Decisions to lock before coding").
- Do not add dependencies to the root `package.json`. The only package with dependencies is `scripts/screenshots/`.

## Prerequisites

- Node 22 or newer (`node --version`). The root `npm test` uses glob patterns that Node 20 does not support.
- A fresh clone of the docs-v2 fork, on a branch off `main`, with a clean working tree.
- Two CSV exports from the audit sheet, provided by the human: the `New Dashboard links` tab and the `Dashboard Screenshots` tab.
- For Track 2 only: a demo tenant, its Management API M2M credentials, and Auth0 Deploy CLI (`npm i -g auth0-deploy-cli`).

## Step 1: apply the files and run the tests

```bash
# from the docs-v2 repo root, with this handoff folder checked out next to it
cp -r ../notes/dashboard-redesign/files/. .
git status --short     # expect only additions under scripts/ and .github/workflows/
node --test scripts/lib/dashboard-links.test.js scripts/update-dashboard-links.test.js scripts/check-dashboard-links.test.js scripts/screenshots/update-screenshot-refs.test.js
npm test               # root runner; must still pass and must not pick up scripts/screenshots/node_modules later
```

Expected: 21 tests pass (17 link tests, 4 screenshot-ref tests). If `npm test` fails on Node 22 because of the glob, report it; do not change `package.json`.

Append to `.gitignore`:

```
scripts/screenshots/storageState.json
scripts/screenshots/test-results/
scripts/screenshots/review.csv
scripts/screenshots/.env.local
```

What was added:

| Path | Purpose |
| --- | --- |
| `scripts/lib/args.js`, `csv.js`, `mdx.js` | argv parsing, CSV read/write, MDX walking and fence-aware line mapping |
| `scripts/lib/dashboard-links.js` | URL extraction, normalization, classification; shared by both link scripts |
| `scripts/update-dashboard-links.js` | Track 1 rewriter |
| `scripts/check-dashboard-links.js` | Track 1b guardrail |
| `scripts/data/dashboard-link-patterns.json` | shape rules for URLs with an id segment; replacements still say `REPLACE_WITH_BLESSED_FORM` |
| `scripts/data/dashboard-link-policy.json` | guardrail rules: `reject` fails CI, `pendingDecision` warns |
| `scripts/data/import-link-map.js` | sheet CSV → `dashboard-link-map.json` |
| `scripts/__fixtures__/dashboard-links/` | test fixtures |
| `.github/workflows/dashboard-link-check.yml` | guardrail workflow |
| `scripts/screenshots/` | Track 2 Playwright package, manifest tooling, tenant YAML; see its README |

## Step 2: verify ground truth

Run the verification commands from the plan's appendix and compare with the plan's Ground truth table. Report what moved. These numbers go in the first PR description.

## Step 3: build the link map and dry-run

```bash
node scripts/data/import-link-map.js /path/to/new-dashboard-links.csv --out=scripts/data/dashboard-link-map.json
node scripts/update-dashboard-links.js --report=reports/links-dry-run.csv
```

The import prints counts by change type and warns about entries with no new URL. Fix column detection with `--old= --new= --type=` if the headers differ.

The dry run prints a locale × action table. Deliver to the human:

1. The table.
2. From the CSV, the top 30 `unmatched` old URLs by occurrence count (group the `oldUrl` column after normalizing with the same rules the script uses; simplest is `node -e` over the CSV using `scripts/lib/csv.js`).
3. The full list of `needs-human` rows.
4. The three competing new forms the map's `new` column uses, if more than one. The map must use exactly one.

Then stop. The next step needs decisions from the human.

## Step 4: lock the URL form, finish the patterns, extend the map

Once the blessed form is known (call it `BLESSED`, for example `https://manage.auth0.com/dashboard/*`):

1. In `scripts/data/dashboard-link-patterns.json`, replace every `REPLACE_WITH_BLESSED_FORM` with the blessed prefix. Keep `{id}` in both pattern and replacement.
2. In `scripts/data/dashboard-link-policy.json`, move the two losing forms from `pendingDecision` into `reject` and delete the winning one. If the human decided tenant-specific links are rewritten, add them to `reject` as well.
3. Extend `dashboard-link-map.json` with the unmatched shapes the human resolved. Keep the sheet as the source and re-import, or edit the JSON directly and tell the human to back-fill the sheet.
4. Re-run the dry run until `unmatched` and `needs-human` are zero or every remaining row is accepted as a hand fix.
5. Re-run the tests: the fixture map is separate from the real one, so they keep passing; add a fixture case if you add a new kind of pattern.

## Step 5: apply links per section and prepare PRs

Branch per section. Sections are the top-level folders under `main/docs` (`get-started`, `authenticate`, `customize`, `manage-users`, `secure`, `deploy-monitor`, `troubleshoot`, and so on) plus `main/ai`. Locale twins live under `main/docs/ja-jp/<section>` and `main/docs/fr-ca/<section>`, so one section run needs three `--path` invocations or one run with `--path=main/docs` filtered by section afterwards. The simplest reliable approach:

```bash
SECTION=get-started
git checkout -b dashboard-links/$SECTION main
for p in main/docs/$SECTION main/docs/ja-jp/$SECTION main/docs/fr-ca/$SECTION; do
  node scripts/update-dashboard-links.js --fix --path=$p --report=reports/links-$SECTION-$(basename $(dirname $p)).csv
done
node scripts/check-dashboard-links.js $(git diff --name-only -- '*.mdx')   # must report 0 errors
git add -A main && git commit -m "Update Dashboard links in $SECTION for the redesigned Dashboard"
```

If a section exceeds about 150 changed files, split it by subfolder. Write a PR description file per branch at `reports/pr-$SECTION.md` with: the command run, the action counts, a link to the CSV, and the CODEOWNERS note (writers team). Do not push.

The first branch is the tooling branch and contains only `scripts/`, `.github/workflows/dashboard-link-check.yml`, the `.gitignore` lines and the data files: `git checkout -b dashboard-links/tooling main`, commit, and note in its PR description that `.github` needs both codeowner teams.

## Step 6: guardrail verification

On the tooling branch:

```bash
node scripts/check-dashboard-links.js                 # whole tree: errors = legacy links still present (expected before section PRs merge)
node scripts/check-dashboard-links.js --ci main/docs/get-started/index.mdx   # annotation format
```

Tell the human to open a deliberate test PR adding `https://manage.auth0.com/#/applications` to any page after the tooling PR merges, and confirm the workflow fails it. The workflow runs only on changed files, so the backlog of legacy links in untouched files does not block unrelated PRs.

## Step 7: screenshot pipeline, setup

```bash
cd scripts/screenshots
npm install
npx playwright install chromium
cd ../..
npm test     # confirm the root runner still runs only the four test files
```

Tenant:

1. Human creates the tenant and an M2M app authorized for the Management API (scopes: read/create/update on clients, client_grants, resource_servers, connections, organizations, organization_connections, organization_member, roles, actions, tenant_settings, branding, email_templates, users).
2. Put `AUTH0_DOMAIN` and `AUTH0_CLIENT_ID` into `scripts/screenshots/tenant/config.json`; `AUTH0_CLIENT_SECRET` stays in the environment.
3. Generate a self-signed SAML cert, base64 it into `AUTH0_KEYWORD_REPLACE_MAPPINGS.SAML_SIGNING_CERT`, or delete the SAML block. Same for the Okta block (needs a real dev Okta org).
4. `a0deploy import -c scripts/screenshots/tenant/config.json -i scripts/screenshots/tenant/tenant.yaml`. Run it twice; the second run must report no changes.
5. Human creates one admin user without MFA for login, plus three or four sample users.
6. Write `scripts/screenshots/.env.local` with the variables listed in the README and source it.

## Step 8: build the manifest

```bash
node scripts/screenshots/build-manifest.js /path/to/dashboard-screenshots.csv
```

Report: entries, multi-ref groups, twins added, refs not found, images missing. Then the human (or you, where obvious) fills per entry: `newPath` (and clears `proposed`), `altText` in the locked format, `dashboardUrl`, `steps`, `crop`, `mask`. Re-run with `--merge` after any sheet update to keep those fields.

Dump images are proposed into `dashboard/unsorted/`; every one of those needs a real folder before capture. Use the naming convention in the plan.

## Step 9: rehearsal on ten entries

Pick ten entries across sections, fill them completely against the current Dashboard, then:

```bash
cd scripts/screenshots
npm run auth                                   # headed login, saves storageState.json
npm run diff -- --grep "@dashboard-applications"
npm run report
```

Exit criteria: auth reuse works on a second run the next day, placeholders resolve, crops are tight, masks hide secrets, diff images are produced for changed screens. Then `npm run capture` for those ten, review, set `status: approved`, and:

```bash
npm run update-refs -- --section=dashboard-applications --report=refs.csv
npm run update-refs -- --section=dashboard-applications --report=refs.csv --fix
```

Check `refs.csv` for `leftover` and `alt-fallback` rows. Commit images and MDX on a `dashboard-screenshots/rehearsal` branch; do not push.

Do not fill `steps` and `crop` for the other entries against the current Dashboard. They will be re-authored against the redesign.

## Step 10: redesign capture, by section

When the redesigned Dashboard is capturable (set `DASHBOARD_BASE_URL` if it is on a staging host), repeat per section: fill entries, `npm run diff`, review, approve, `npm run capture`, `npm run update-refs --fix`, commit on `dashboard-screenshots/<section>`. Keep under about 40 images per branch. Run `oxipng -o 4` on new PNGs if it is available. After each section, run the guardrail and the root tests.

## Step 11: handoff checklist per branch

- Branch name follows `dashboard-links/<section>` or `dashboard-screenshots/<section>`.
- One commit, or a few logical commits, no attribution trailers.
- `reports/pr-<branch>.md` written with command, counts, CSV link, reviewers.
- `node scripts/check-dashboard-links.js $(git diff --name-only main -- '*.mdx')` is clean.
- `npm test` passes.
- Nothing pushed.

## Known limits

- `update-dashboard-links.js` rewrites URLs in backticked inline code on purpose, and skips fenced blocks. If a page documents the old URL as an example, it will be rewritten; the report shows every change for review.
- The guardrail treats `manage.auth0.com/#/` with nothing after it as a root link and allows it.
- `build-manifest.js` trusts the sheet's image path column; rows whose image is not on disk are kept with `imageExists: false` so they can be triaged.
- `lib/tenant.js` lists up to 100 of each resource type. The demo tenant is small; raise `per_page` or add paging if it grows.
- Deploy CLI does not manage users. Enterprise connections (SAML, OIDC, Okta) may be rejected without real metadata; the YAML says what to replace or remove.

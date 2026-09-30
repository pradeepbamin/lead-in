# Multi-Tenant Playwright Framework

A **self-contained** Playwright E2E framework that runs the same UI test suite
against **7 tenants** (Aviation, Connect, SLI, Demo Interactions, Demo Topics,
Investor, Tahaluf), each with its own URL/credentials, across **2
environments** (QA, PROD), where **Connect** additionally has **5
independent licenses** (each with its own URL + creds).

Tests can be sliced three ways — and combined in a single command:

| Slice          | How                                            | Example                |
|----------------|--------------------------------------------------|-------------------------|
| **Tenant**     | Playwright `--project`                          | `--project=aviation`    |
| **Suite**      | `@smoke` / `@regression` tag                     | `--grep @smoke`         |
| **Module/tag** | Functionality tag (`@Lead`, `@Accounts`, ...)    | `--grep @Accounts`      |

---

## 1. Tenant → module coverage

| Tenant             | Licenses | Modules (tags)                                        |
|--------------------|----------|---------------------------------------------------------|
| Aviation           | 1        | `Lead`, `Dashboard`, `Users`, `Accounts`, `faqs`          |
| Connect            | 5        | `Lead`, `Dashboard`, `Users`, `faqs`                      |
| SLI                | 1        | `Lead`, `Dashboard`, `Users`, `Accounts`, `SLIAccount`, `faqs` |
| Demo Interactions  | 1        | `Lead`, `Dashboard`, `Users`                              |
| Demo Topics        | 1        | `Lead`, `Dashboard`, `Users`                              |
| Investor           | 1        | `Lead`, `Dashboard`, `Users`                              |
| Tahaluf            | 1        | `Lead`, `Dashboard`, `Users`                              |

`Lead`, `Dashboard`, `Users` are common to every tenant and live in
`src/tests/common/` — they run automatically for every tenant project instead
of being duplicated per folder. Add a new tenant/module in
`src/config/tenants.ts` only.

---

## 2. Folder structure

```
playwright-framework/
├── package.json              # npm scripts (install/run/report)
├── playwright.config.ts      # One Playwright PROJECT per tenant
├── tsconfig.json             # Path aliases: @config, @pages, @fixtures, @utils
├── .env.example                # Copy to .env for local credential overrides
├── .gitignore
├── README.md
│
├── fixtures/                    # Per-environment TEST DATA (credentials)
│   ├── fixture.types.ts         # Fixture/FixtureMap/Credentials types
│   ├── fixtures-global.ts       # Shared across every environment
│   ├── fixtures-qa.ts           # QA creds — one entry per tenant + li_standard..technomic
│   └── fixtures-prod.ts         # PROD creds — same shape
│
├── .github/workflows/
│   └── playwright.yml           # CI: matrix over tenants, ENV + smoke/regression inputs
│
├── reports/                     # Generated HTML/JUnit report (git-ignored)
├── test-results/                 # Generated traces/screenshots/videos (git-ignored)
│
└── src/
    ├── config/
    │   ├── tenants.ts            # Tenant → modules/licenses + URL builders
    │   ├── environment.ts        # QA/PROD resolution via `ENV`
    │   └── credentials.ts        # Selects fixtures-<env>.ts + resolves per tenant/license
    │
    ├── pages/                    # Page Object Model
    │   ├── base.po.ts            # BasePage: byId/goTo/waitForPageReady helpers
    │   └── login.po.ts           # LoginPage: login()
    │
    ├── fixtures/
    │   └── test-fixtures.ts      # `tenantContext`, `appPage`, `licensePage`, `loginPage`
    │
    ├── utils/
    │   ├── logger.ts             # Simple console logger
    │   └── custom-reporter.ts    # Color-coded reporter + per-tenant summary dashboard
    │
    └── tests/
        ├── common/                # Shared specs — run by every tenant project
        │   └── insights/
        ├── aviation/
        ├── connect/                  # li-standard/, li-plus/, li-delegate/, pre-event/, technomic/
        ├── sli/
        ├── demo-interactions/
        ├── demo-topics/
        ├── investor/
        └── tahaluf/
```

---

## 3. Setup

```bash
cd playwright-framework
npm install
npx playwright install --with-deps chrome   # projects use channel: 'chrome'
cp .env.example .env                        # fill in real creds for local runs
```

---

## 4. Run commands

### By environment
```bash
ENV=QA   npx playwright test      # default if ENV is omitted
ENV=PROD npx playwright test

npm run test:qa
npm run test:prod
```

### By tenant (folder-based, via Playwright project)
```bash
npx playwright test --project=aviation
npx playwright test --project=connect
npx playwright test --project=sli
npx playwright test --project=demo-interactions
npx playwright test --project=demo-topics
npx playwright test --project=investor
npx playwright test --project=tahaluf

npm run test:aviation
npm run test:connect
npm run test:sli
npm run test:demo-interactions
npm run test:demo-topics
npm run test:investor
npm run test:tahaluf
```

### By suite (tag-based: smoke / regression)
```bash
npx playwright test --grep @smoke
npx playwright test --grep @regression

npm run test:smoke
npm run test:regression
npm run test:smoke:qa
npm run test:smoke:prod
npm run test:regression:qa
npm run test:regression:prod
```

### By module/functionality tag
```bash
npx playwright test --grep @Lead
npx playwright test --grep @Dashboard
npx playwright test --grep @Users
npx playwright test --grep @Accounts
npx playwright test --grep @SLIAccount

npm run test:tag:lead
npm run test:tag:dashboard
npm run test:tag:users
npm run test:tag:accounts
```

### Combined (tenant + suite + module — all in one command)
```bash
ENV=PROD npx playwright test --project=aviation --grep @smoke
ENV=QA   npx playwright test --project=sli      --grep @regression
ENV=QA   npx playwright test --project=investor --grep @smoke
```
> To AND two tag conditions in one `--grep`, use a lookahead regex, e.g.
> `--grep "(?=.*@smoke)(?=.*@Accounts)"`.

### Connect license-specific run
Every Connect spec defaults to license 1 unless run via one of the 5
per-license projects (`li-standard`, `li-plus`, `li-delegate`, `pre-event`,
`technomic`), each of which pins its own `licenseIndex` in
`playwright.config.ts`. To target one license in a custom spec instead,
override it explicitly:
```ts
test.use({ licenseIndex: 3 });
```

### Other useful commands
```bash
npx playwright test --headed        # see the browser
npx playwright test --ui            # Playwright UI mode
npx playwright test --debug         # step through with the inspector
npx playwright test --list          # list matched tests without running them

npm run report                      # open the last HTML report
npm run typecheck                   # tsc --noEmit
```

---

## 5. CI (GitHub Actions)

`.github/workflows/playwright.yml` has two triggers:

- **`push`, on every branch** — fully automatic, no inputs to set (push
  events can't carry inputs). Runs every project, QA, `@smoke`.
- **`workflow_dispatch`** — manual, runnable from **any branch**, with:
  - `env` — QA | PROD
  - `project` — `all`, or one specific project (aviation, connect, sli,
    demo-interactions, demo-topics, investor, tahaluf, li-standard, li-plus,
    li-delegate, pre-event, technomic)
  - `tag` — smoke | regression | Lead | Dashboard | Accounts | faqs | all (maps to
    `--grep`)
  - `language` — en | es | pt (only read by Connect's license specs)

### One-time setup
Add every tenant/license credential as a **repo secret** (Settings → Secrets
and variables → Actions), named exactly as in `.env.example` (e.g.
`QA_AVIATION_USERNAME`, `PROD_LI_DELEGATE_PASSWORD`). Nothing runs
correctly in CI until these exist — a run without them will still execute
but every login will fail with empty credentials.

### Day-to-day flow
1. **Push anything, to any branch** (including merging a PR into `main`) →
   the `push` trigger fires by itself, every time. No action needed — it
   always runs every project on QA with `@smoke`, since a push event
   can't carry custom inputs.
2. **Want different inputs for that branch** (a different `env`, a single
   `project`, `@regression`, a specific `language`) → go to
   **Actions → Playwright Tests → Run workflow**, pick the branch, choose
   `env` / `project` / `tag` / `language`, and click **Run workflow**. This
   runs in addition to the automatic push run, not instead of it.
3. **Read the result** — open the run in the Actions tab:
   - Each project's job has a **Job Summary** with the same Pass/Fail/Flaky
     dashboard the console reporter prints locally (see
     `src/utils/custom-reporter.ts`) — no log-scrolling needed to see if it
     went green.
   - If something failed, download that job's `playwright-report-<project>-<env>`
     artifact and open it with `npx playwright show-report <unzipped-folder>`
     for the full trace/screenshot/video.

Trigger manually from the Actions tab UI, or via `gh`:
```bash
gh workflow run playwright.yml -f env=PROD -f project=investor -f tag=regression
```

---

## 6. Adding new coverage

- **New tenant** → add an entry to `TENANTS` in `src/config/tenants.ts`, a
  fixture map entry in `fixtures/fixtures-qa.ts` / `fixtures-prod.ts`, a
  `src/tests/<tenant>/` folder, and a matching project block in
  `playwright.config.ts`.
- **New module for an existing tenant** → add it to that tenant's `modules`
  array in `tenants.ts`, then write a spec tagged `@<ModuleName>` using the
  native `{ tag: [...] }` syntax — see
  `src/tests/aviation/sample-advanced-reference.spec.ts` for the full pattern
  (tags, `test.use`, `test.step`, data-driven `forEach`).
- **New shared (cross-tenant) spec** → drop it in `src/tests/common/`; it
  runs for every tenant project automatically.

  =====

  ┌───────────────────────────────────────────────┬────────────────────────────┬────────────────────────────────┐
│ Command                                       │ Resolved tenantContext.env │ Resolved URL                   │
├───────────────────────────────────────────────┼────────────────────────────┼────────────────────────────────┤
│ ENV=PROD npx playwright test                  │ prod                       │ http://lead.prod.aviation.com  │
├───────────────────────────────────────────────┼────────────────────────────┼────────────────────────────────┤
│ ENV=QA npx playwright test                    │ qa                         │ http://lead.qa.aviation.com    │
├───────────────────────────────────────────────┼────────────────────────────┼────────────────────────────────┤
│ npm run test:prod (uses cross-env ENV=PROD)   │ prod                       │ http://lead.prod.aviation.com  │
├───────────────────────────────────────────────┼────────────────────────────┼────────────────────────────────┤
│ npm run test:qa (uses cross-env ENV=QA)       │ qa                         │ http://lead.qa.aviation.com    │
└───────────────────────────────────────────────┴────────────────────────────┴────────────────────────────────┘

Why it works:  ENV  is read by  src/config/environment.ts  the moment the config/fixtures load, and since it's set on the shell before the  playwright test / npm run  process even starts, it's already in  process.env  — no extra wiring needed. The  npm run test:*  scripts just wrap the same thing via  cross-env  so it also works identically on Windows.

One caveat: if you omit  ENV  entirely, it defaults to  QA  (not an error) — so always set it explicitly for PROD runs.

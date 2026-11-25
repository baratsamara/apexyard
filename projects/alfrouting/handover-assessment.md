# alfrouting — Handover Assessment

**Date**: 2026-11-24
**Assessor**: Bara (operator) + agent
**Status**: handover

## Origin

- **Where it came from**: inherited internal codebase (Alfanar Low Voltage Factory job-card system, LV-AE03), built outside ApexYard governance
- **Repo location**: https://github.com/baratsamara/alfrouting (private)
- **First commit date**: pre-2026 (122 commits in last 90 days; full history in `workspace/alfrouting`)
- **Last commit date**: 2026-11-17 (`fff19ba Daily S.O report: explode rows per production order`)

## Current State

### Tech stack

- Language: TypeScript (strict)
- Runtime: Node.js 20+ (`README.md` prerequisites; `vite.config.ts` + `tsx`)
- Framework: React 18 + Vite (frontend) + Express 5 (backend API) + wouter + TanStack Query
- Database: PostgreSQL with Drizzle ORM (`drizzle.config.ts` dialect `postgresql`, `shared/schema.ts` ~1515 lines, `migrations/`)
- Auth: Passport.js local strategy + express-session + connect-pg-simple
- Test framework: guarded custom runner `scripts/run-tests.mjs` (requires disposable `TEST_DATABASE_URL`; refuses prod DB) — 25 `server/*.test.ts` + ~10 `client/src/*.test.ts`
- CI: none (`workspace/alfrouting/.github/` absent)

### Build status

- `npm install`: not attempted in this handover (node_modules present from prior dev)
- `npm run build`: not attempted (`tsx script/build.ts` → `dist/`)
- `npm run test`: not attempted (requires `TEST_DATABASE_URL`; runner refuses without it — correct guard)
- `npm run check`: not attempted (`tsc`, excludes `*.test.ts` per `tsconfig.json`)

### Test coverage

- Unknown (no coverage threshold in `package.json` / `vite.config.ts`, no coverage step in CI — CI absent)

### Repo activity

- Commits in last 90 days: 122
- Open issues: 0
- Open PRs: 0
- Top contributors: Bara (single-operator history; `git shortlog` shows one author stream)

## Harnessability assessment

**Overall verdict**: `low`

> ⚠ Harnessability: LOW
>
> Rex's architecture handbooks will fire advisory-only on this codebase. The blocking gate (`ENFORCEMENT: blocking`) will generate false positives. Recommended: adopt as advisory-only, plan a follow-up to add the missing scaffolding (typescript strict, lint baseline, etc.)

| Dimension | Score | Evidence |
|-----------|-------|----------|
| Type safety | `strong` | `tsconfig.json`: `"strict": true` with `noEmit`, `moduleResolution: bundler` |
| Module boundaries | `flat` | only `client/`, `server/`, `shared/` — no `src/domain/` + `src/application/` + `src/infrastructure/` split |
| Framework opinionation | `moderate` | `package.json` has `express` + `react` (HTTP + UI only, no DI/persistence opinionation like NestJS/Next.js) |
| Test coverage signal | `absent` | no `coverageThreshold` in `package.json`/`vitest.config`, no coverage step (no CI workflows at all) |
| Lint baseline | `absent` | no `.eslintrc.*` / `eslint.config.*` at repo root (`AGENTS.md` explicitly: "There is no linter configured") |

See AgDR-0042 for the scoring rationale and v1 thresholds. Re-score via `/handover` once lint + coverage land.

## Quality Risks

### Security

- Session secret fallback: `AGENTS.md` notes server generates a temporary per-boot secret if `SESSION_SECRET` absent (sessions reset on restart) — safe default but masks misconfiguration in prod.
- No secrets scan baseline; `.env.example` only carries `DATABASE_URL`/`TEST_DATABASE_URL` (good) — verify no real secrets in shell history or `saas-rollout-plan.local.md` / `staging-test-logins.md` (both untracked in workspace root, must stay untracked).

### Dependencies

- Express 5 (`^5.0.1`) + large Radix/shadcn surface — no `/audit-deps` baseline yet. No lockfile audit in this handover.

### Technical debt

- No lint baseline — style drift likely across 122 recent commits.
- No coverage gate — 35+ test files exist but no threshold; regressions can slip.
- In-repo test exclusion: `tsconfig.json` excludes `**/*.test.ts` from `tsc` — type errors in tests are invisible to `npm run check`.
- Untracked local docs (`docs/`, `ibnNazer-maturity-scale.md`, `saas-rollout-plan.local.md`, `scripts/render-demo-video.py`, `staging-test-logins.md`) — decide what belongs in git vs stays local.

### Operational

- No CI (`golden-paths/pipelines/ci.yml` never copied) — `main` auto-deploys per `AGENTS.md` ("Every push to `main` deploys it") with no gate.
- Schema/code coupling risk is known (3 Nov outage: code shipped ahead of `order_details` columns) — mitigated by convention (`migrations/` + `npm run db:verify`) but not mechanically enforced.
- Branch model: `main` (prod) + `staging` (temp) both exist remotely; local clone sits on `main` with 5 untracked paths.

## Integration Plan

### Roles that apply

- tech-lead, backend-engineer, frontend-engineer, qa-engineer (from registry; matches Express + React + test surface)
- Consider adding `platform-engineer` once CI lands, `security-auditor` for the session-secret + SAP-export surface on first high-risk PR

### Workflows that kick in

- [ ] PR workflow (`.claude/rules/pr-workflow.md`) — every change through a PR (repo currently ships via direct pushes to `main`)
- [ ] AgDR for technical decisions (first: CI + lint + coverage baselines)
- [ ] Code Reviewer agent on every PR
- [ ] Security Reviewer agent on first pass and auth/session PRs
- [ ] `/audit-deps` on adoption and monthly thereafter

### Hooks to enable

- [ ] `block-git-add-all`
- [ ] `block-main-push` (high value here — `main` deploys on push)
- [ ] `validate-branch-name` (tracker currently unused — 0 open issues; decide tracker before enforcing)
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets`

### CI templates to copy in

- [ ] `golden-paths/pipelines/ci.yml` (typecheck + build + guarded tests with `TEST_DATABASE_URL`)
- [ ] `golden-paths/pipelines/pr-title-check.yml` (once tracker is adopted)

### Registry entry

```yaml
- name: alfrouting
  repo: baratsamara/alfrouting
  workspace: workspace/alfrouting
  docs: projects/alfrouting
  status: handover
  tier: P1
  roles:
    - tech-lead
    - backend-engineer
    - frontend-engineer
    - qa-engineer
```

Note: registry previously had typo `alfrrouting` — fixed 2026-11-24. `AGENTS.md` already present in target repo (preserved, not generated).

## Next Steps

1. ~~Copy in `golden-paths/pipelines/ci.yml` with `TEST_DATABASE_URL` wired as a CI secret (repo pushes to `main` deploy with no gate today)~~ → Filed as [#1](https://github.com/baratsamara/alfrouting/issues/1)
2. ~~Decide tracker discipline: 0 open issues + 0 open PRs with 122 commits in 90 days means work is invisible — adopt GitHub Issues or document why not~~ → Filed as [#2](https://github.com/baratsamara/alfrouting/issues/2)
3. ~~Add lint baseline (ESLint) and run before pushing — `AGENTS.md` currently says "no linter, run `npm run check` instead"~~ → Filed as [#3](https://github.com/baratsamara/alfrouting/issues/3)
4. Add coverage reporting (vitest/node-test coverage config + threshold) before the first feature under governance
5. Triage the 5 untracked workspace paths (`docs/`, `ibnNazer-maturity-scale.md`, `saas-rollout-plan.local.md`, `scripts/render-demo-video.py`, `staging-test-logins.md`) — commit what belongs, gitignore the rest (especially `staging-test-logins.md`)
6. `/code-review` the most-recent `main` commits on this repo as Rex to calibrate review standards
7. Stakeholder sync with the factory operator to cover what the static read couldn't surface (shift model, SAP export contract, badge-scan failure modes)

## Post-Handover Checklist

- [ ] Review this assessment with the previous owner (self-review: Bara is both owner and operator — record explicit sign-off)
- [ ] No-CI-with-auto-deploy — close before the first feature PR
- [ ] Untracked login/rollout docs — scheduled in the first 2 weeks
- [ ] Add `alfrouting` to the weekly `/stakeholder-update` rollup
- [ ] Onboard the roles above into the review rotation
- [ ] Set up a test coverage baseline (run tests with coverage and commit the threshold)
- [ ] Run `/audit-deps alfrouting` monthly for the next 3 months

## Open Questions

- Why 0 issues / 0 PRs despite active development — is GitHub Issues intentionally off, or just unused?
- Which of the untracked local files are canonical (should be committed) vs scratch?
- Staging flow: is `staging` branch still the deploy gate for `alfrouting-temp`, and who promotes `staging` → `main`?

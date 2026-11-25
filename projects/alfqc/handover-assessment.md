# alfqc — Handover Assessment

**Date**: 2026-11-24
**Assessor**: Bara (operator) + agent
**Status**: handover

## Origin

- **Where it came from**: inherited internal QC codebase (quality-control inspections, defects, control tower), built outside ApexYard governance
- **Repo location**: https://github.com/baratsamara/alfQc (private, note capital `Qc`)
- **First commit date**: pre-2026 (280 commits in last 90 days)
- **Last commit date**: 2026-11-22 (`b3af229 UI: shift notification bell clear of page header actions`)

## Current State

### Tech stack

- Language: TypeScript (strict)
- Runtime: Node.js (`tsx`, `vite`; `package.json` name `rest-express`)
- Framework: React 18 + Vite (frontend) + Express 4 (backend API) + wouter + TanStack Query
- Database: PostgreSQL with Drizzle ORM (`drizzle.config.ts` dialect `postgresql`, `shared/schema.ts` 836 lines, `migrations/`, Neon serverless driver present)
- Auth: Passport.js local strategy + express-session (+ `connect-pg-simple`, `memorystore`); `bcryptjs`
- Storage: local `uploads/` + `attached_assets/` + `exports/`; S3 (`@aws-sdk/client-s3`) + Replit object storage (`@replit/object-storage`, `server/objectStorage.ts`); `multer` uploads
- Test framework: `node --test` via `tsx` — `npm test` runs `tests/*.test.tsx` (29 files: app-navigation, qc-inbox, defect-routing, control-tower-*, photo-annotation, etc.)
- CI: none (no `.github/` in `workspace/alfqc/alfQc`)

### Build status

- `npm install`: not attempted in this handover (node_modules present at both container and repo levels)
- `npm run build`: not attempted (`vite build && esbuild server/index.ts --bundle --format=esm --outdir=dist`)
- `npm run test`: not attempted
- `npm run check`: not attempted (`tsc`, excludes `**/*.test.ts`)

### Test coverage

- Unknown (no coverage threshold in `package.json`, no coverage step — no CI)

### Repo activity

- Commits in last 90 days: 280
- Open issues: 0
- Open PRs: 0 (PR #1 "Session persistence, PWA installability, mobile performance + phone UI fixes" merged 2026-11-25)
- Top contributors: Bara (single-operator history)

## Harnessability assessment

**Overall verdict**: `low`

> ⚠ Harnessability: LOW
>
> Rex's architecture handbooks will fire advisory-only on this codebase. The blocking gate (`ENFORCEMENT: blocking`) will generate false positives. Recommended: adopt as advisory-only, plan a follow-up to add the missing scaffolding (typescript strict, lint baseline, etc.)

| Dimension | Score | Evidence |
|-----------|-------|----------|
| Type safety | `strong` | `tsconfig.json`: `"strict": true`, `noEmit`, `moduleResolution: bundler` |
| Module boundaries | `flat` | only `client/`, `server/`, `shared/` (+ `tests/`, `scripts/`) — no `src/domain/` + `src/application/` + `src/infrastructure/` split |
| Framework opinionation | `moderate` | `package.json` has `express` + `react` (HTTP + UI only, no DI/persistence opinionation) |
| Test coverage signal | `absent` | no `coverageThreshold`/coverage config in `package.json`, no coverage step (no CI workflows at all) |
| Lint baseline | `absent` | no `.eslintrc.*` / `eslint.config.*` at repo root |

See AgDR-0042 for the scoring rationale and v1 thresholds. Re-score via `/handover` once lint + coverage land.

## Quality Risks

### Security

- Auth surface (passport-local, sessions, PWA) with no security-reviewer baseline; open PR #1 touches session persistence — review before merge.
- `.env` lives at container root (`workspace/alfqc/.env`, gitignored) plus `MIGRATION_PLAN.md` — verify no production secrets in dumps (`backup-*.sql`, `dump_*.sql` ~15–58 MB each, kept local only, must never be committed).
- S3 + object-storage paths (`server/objectStorage.ts`) need a STRIDE pass once under governance.

### Dependencies

- Large UI + export surface (`exceljs`, `jspdf`, `papaparse`, Radix) — no `/audit-deps` baseline yet.

### Technical debt

- No lint baseline across 280 recent commits.
- No coverage gate despite 29 test files.
- `tsconfig.json` excludes `**/*.test.ts` from `tsc` — type errors in tests invisible to `npm run check`.
- Dual lockfiles (`package-lock.json` + `pnpm-lock.yaml`) — pick one package manager.
- Container layout: repo lives at `workspace/alfqc/alfQc` (nested), with DB dumps (`*.sql*`, `*.dump`, `*.gz`), photo/export `out/` (~10G with manifests), `cranl_bucket/` tooling and `MIGRATION_PLAN.md` alongside it at `workspace/alfqc/` — intentional local-data quarantine, but registry previously pointed at the container instead of the git root (fixed 2026-11-24 → `workspace/alfqc/alfQc`).

### Operational

- No CI — `staging` branch is active (local HEAD) with PR #1 open, but no automated gate.
- Heavy local-only data (`out/`, dumps) — needs explicit backup/retention policy, not just "kept local".
- `replit.md`, `design_guidelines.md`, `screenshots/`, `artifacts/` suggest a Replit origin — confirm the canonical dev loop is now local `npm run dev`, not Replit.

## Integration Plan

### Roles that apply

- tech-lead, backend-engineer, frontend-engineer, qa-engineer (from registry; matches Express + React + 29-test surface)
- Consider adding `platform-engineer` once CI lands, `security-auditor` for session/PWA + S3/object-storage PRs

### Workflows that kick in

- [ ] PR workflow (`.claude/rules/pr-workflow.md`) — PR #1 is already the right shape; keep it
- [ ] AgDR for technical decisions (first: CI + lint + coverage + lockfile choice)
- [ ] Code Reviewer agent on every PR
- [ ] Security Reviewer agent on first pass and auth/storage PRs
- [ ] `/audit-deps` on adoption and monthly thereafter

### Hooks to enable

- [ ] `block-git-add-all` (critical here — `out/` + dumps must never be staged)
- [ ] `block-main-push`
- [ ] `validate-branch-name` (tracker currently unused — 0 open issues; decide tracker before enforcing)
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets`

### CI templates to copy in

- [ ] `golden-paths/pipelines/ci.yml` (typecheck + build + `node --test`)
- [ ] `golden-paths/pipelines/pr-title-check.yml` (once tracker is adopted)

### Registry entry

```yaml
- name: alfqc
  repo: baratsamara/alfQc
  workspace: workspace/alfqc/alfQc
  docs: projects/alfqc
  status: handover
  tier: P1
  roles:
    - tech-lead
    - backend-engineer
    - frontend-engineer
    - qa-engineer
```

Note: `workspace` previously pointed at the container `workspace/alfqc` — fixed 2026-11-24 to the git root `workspace/alfqc/alfQc`. Container-level data (dumps, `out/`, `cranl_bucket/`, `MIGRATION_PLAN.md`, `.env`) stays outside git by design.

## Next Steps

1. ~~Review and merge-or-close PR #1 (session persistence + PWA + mobile Save-button fixes)~~ → Done 2026-11-25 (merged `staging` → `main`)
2. Copy in `golden-paths/pipelines/ci.yml` (typecheck + build + tests) — no gate exists today
3. Decide tracker discipline: 0 open issues with 280 commits in 90 days means work is invisible — adopt GitHub Issues or document why not
4. Add lint baseline (ESLint) and run before pushing
5. Add coverage reporting + threshold before the first feature under governance
6. Resolve dual lockfiles (`package-lock.json` vs `pnpm-lock.yaml`) — one package manager only
7. `/code-review` the most-recent `staging` commits on this repo as Rex to calibrate review standards
8. Stakeholder sync with the QC operator to cover what the static read couldn't surface (inspection SLAs, defect-routing rules, control-tower export contract)

## Post-Handover Checklist

- [ ] Review this assessment with the previous owner (self-review: Bara is both owner and operator — record explicit sign-off)
- [ ] PR #1 session/PWA change — reviewed before merge
- [ ] `out/` + dumps quarantine (`gitignore` + never-stage rule) — confirmed in the first 2 weeks
- [ ] Add `alfqc` to the weekly `/stakeholder-update` rollup
- [ ] Onboard the roles above into the review rotation
- [ ] Set up a test coverage baseline (run tests with coverage and commit the threshold)
- [ ] Run `/audit-deps alfqc` monthly for the next 3 months

## Open Questions

- Is `staging` the integration branch with `main` as prod (mirroring alfrouting's `main`/`staging` split), and who promotes?
- What is the retention/backup policy for `workspace/alfqc/out/` and the `*.sql` dumps — how long are they kept, and where is the off-machine copy?
- Is Replit still in the loop, or is local `npm run dev` canonical now (`replit.md` vs reality)?
- `cranl_bucket` + `bucket-size.mjs`: what bucket is QC storage targeting, and who owns its credentials?

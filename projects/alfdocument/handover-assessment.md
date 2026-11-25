# alfdocument — Handover Assessment

**Date**: 2026-11-25
**Assessor**: Bara (operator) + agent
**Status**: handover — re-assessment complete; baseline quality work remains

## Origin

- **Where it came from**: greenfield skeleton created by the operator on 2026-11-15, then built into an application under ApexYard governance
- **Repo location**: https://github.com/baratsamara/alfdocument (private)
- **First commit date**: 2026-11-15 (`46bca43`)
- **Last commit date**: 2026-11-19 (`489960f`, CSRF/header/login-throttle hardening batch)

## Current State

### What it does

Staff create sales orders, add numeric pallet/product suffixes, upload versioned documents, and print QR labels. Each suffix has a public bearer-link page that exposes the latest documents for pickup. Rotating a suffix token invalidates earlier printed labels.

Evidence: repo `README.md:3`, `server/routes.ts:349-371,498-606`.

### Tech stack

- Language: TypeScript with `strict: true` (`tsconfig.json:5-16`)
- Runtime: Node.js 22+ (`package.json:48-50`)
- Framework: React 18 + Vite 6 SPA; Express 5 API; Tailwind
- Database: PostgreSQL with Drizzle ORM; three tables for sales orders, suffixes, and documents (`shared/schema.ts:3-48`)
- Auth: one environment-configured staff account; scrypt password hash; express-session with MemoryStore
- Storage: local disk or S3-compatible object storage through an `ObjectStore` abstraction
- Test framework: none yet
- Lint: none yet
- CI: PR/main workflow runs `npm ci`, typecheck, and build (`.github/workflows/ci.yml:15-36`)

### Build status

- `npm install`: not attempted; used the existing local dependencies
- `npm run check`: **passed** (`tsc --noEmit`, 2026-11-25)
- `npm run build`: **passed** (Vite client + server TypeScript build, 2026-11-25)
- `npm test`: unavailable — no test script or test files
- `npm run lint`: unavailable — no lint script or configuration

### Test coverage

- Unmeasured. No test runner, coverage report, or threshold exists. This is not evidence of 0% coverage; it means coverage cannot currently be reported.

### Repo activity

- Reachable commits: 35, all since 2026-11-15
- Commits in last 90 days: 35
- Open issues: 10 (#31, #32, #33, #34, #38, #39, #40, #42, #43, #44)
- Open PRs: 0
- Top contributor: Bara
- Local branch: clean `staging`; the local checkout had not been fetched immediately before this assessment

## Harnessability assessment

**Overall verdict**: `low`

> ⚠ Harnessability: LOW
>
> Rex's architecture handbooks will fire advisory-only on this codebase. The blocking gate (`ENFORCEMENT: blocking`) will generate false positives. Recommended: adopt as advisory-only, plan a follow-up to add the missing scaffolding (typescript strict, lint baseline, etc.)

| Dimension | Score | Evidence |
|-----------|-------|----------|
| Type safety | `strong` | `tsconfig.json:5-16` sets `strict: true`, NodeNext resolution, and isolated modules |
| Module boundaries | `flat` | `client/`, `server/`, and `shared/` exist, but there is no domain/application/infrastructure split; `server/routes.ts` and `client/src/App.tsx` concentrate the main flows |
| Framework opinionation | `moderate` | `package.json` includes Express and React, but no opinionated application/DI framework |
| Test coverage signal | `absent` | no test script, test files, coverage configuration, or coverage CI step |
| Lint baseline | `absent` | no ESLint/Prettier dependency, configuration, or lint script |

The score improved from `none` type safety and `weak` framework opinionation on 2026-11-15. It remains `low` because tests, coverage, and linting are absent. See AgDR-0042 for the scoring rationale and v1 thresholds.

## Quality Risks

The findings below come from a static read. No exploit was run. Existing security and threat-model reports remain useful, but they predate the latest hardening batch and need a delta review.

### Security

- **Document replacement is not atomic.** The previous latest row is marked non-latest before object storage and insertion succeed. There is no transaction or uniqueness constraint for `(suffix_id, filename, version_no)` or one latest row per filename (`server/routes.ts:429-460`, `shared/schema.ts:33-47`). A failed write can orphan a blob or leave no latest version; concurrent same-name uploads can create duplicate versions.
- **Prior document IDs are not filtered by `isLatest`.** The public file route can serve an older row while its blob remains (`server/routes.ts:534-596`). Current best-effort deletion reduces exposure but does not enforce the rule.
- **Public bearer links have no expiry, hash-at-rest, access audit, or public-route rate limit.** Tokens are strong 24-byte random values, but anyone holding a link can retrieve order/customer/project metadata and files until rotation or deletion (`shared/schema.ts:25-28`, `server/routes.ts:519-606`).
- **Resource use is not bounded safely.** Multer buffers up to 10 files of 50 MB each in memory; downloads buffer whole objects; scrypt runs synchronously on the main thread (`server/routes.ts:10-13,404-425,592-596`, `server/storage.ts:72-76`, `server/auth.ts:14-15,85-92`).
- Upload validation trusts filename extension and client-supplied MIME rather than inspecting content (`server/storage.ts:101-107`).
- `trust proxy: 1` is correct only when exactly one trusted proxy is the sole ingress (`server/index.ts:28-30`).

### Dependencies

- `/audit-deps` has not been recorded. Issue #40 already covers dependency auditing and pinning CI Actions.
- Runtime Express is 5.x while `@types/express` is 4.x (`package.json:20,31`). The current typecheck passes, but the mismatch weakens type assurance.

### Technical debt

- No automated regression tests, lint baseline, or coverage gate.
- `server/routes.ts` is 607 lines and `client/src/App.tsx` is 543 lines; API DTOs are manually duplicated in the client.
- CI uses moving Action major tags rather than commit SHAs; tracked in #40.
- `AgDR-0002` describes an alphanumeric/hyphen suffix pattern, while the implementation accepts digits only (`docs/agdr/AgDR-0002-so-suffix-qr.md:32,51`, `server/routes.ts:105-111`).

### Operational

- Sessions always use MemoryStore, so restarts lose sessions and multiple instances do not share state (`server/auth.ts:69-76`). Issue #39 covers persistent sessions and expiry.
- Missing or partial S3 configuration silently falls back to local disk (`server/storage.ts:85-99`). Hosted, ephemeral, or multi-instance deployments need explicit storage selection; #31 covers CranL blob storage.
- No committed deploy definition, readiness/health endpoint, monitoring, backup/restore runbook, or rollback procedure was found.
- Migrations are manual. The application does not run them at startup.
- Staging credential rotation remains open as #32.

## Integration Plan

### Roles that apply

- tech-lead
- backend-engineer
- frontend-engineer
- qa-engineer — establish the missing test and coverage baseline
- platform-engineer — CI, deployment, and operations
- security-auditor — authentication, public bearer links, uploads, and storage

### Workflows that kick in

- [x] PR workflow — the repo has active GitHub Issues and protected `main`
- [x] AgDR for material technical decisions — stack and suffix-model decisions already exist
- [ ] Code Reviewer agent on every PR — continue the existing Rex + human-approval gate
- [ ] Security Reviewer delta pass over the latest auth/public-fetch/storage changes
- [ ] `/audit-deps` on adoption and monthly thereafter — already tracked in #40

### Hooks to enable

- [x] `block-git-add-all`
- [x] `block-main-push` — local agent gate plus server-side `main` protection
- [x] `validate-branch-name` — GitHub issue IDs are in use
- [x] `validate-pr-create`
- [x] `pre-push-gate`
- [x] `check-secrets`
- [x] Fleet OpenCode gate adapter — sessions under this workspace inherit the ops-fork hooks

### CI templates to copy in

- [x] CI exists — typecheck + build on PR and `main`
- [ ] Add test, lint, coverage, dependency, and migration checks as the baseline lands
- [ ] Pin Actions by commit SHA — #40

### Registry entry

```yaml
- name: alfdocument
  repo: baratsamara/alfdocument
  workspace: workspace/alfdocument
  docs: projects/alfdocument
  status: handover
  tier: P1
  roles:
    - tech-lead
    - backend-engineer
    - frontend-engineer
    - qa-engineer
    - platform-engineer
    - security-auditor
  tags:
    - sandbox
  ticket_prefix: ""
```

The expanded roles replace the skeleton-era `tech-lead`-only entry. `status: handover` stays until the minimum test/lint/coverage and production-readiness decisions are closed; ticket-first feature work can proceed in the meantime.

## Next Steps

1. ~~Pin CI Actions and audit dependencies~~ → Filed as [#40](https://github.com/baratsamara/alfdocument/issues/40)
2. ~~Add an automated test runner, regression tests for order/suffix/document flows, coverage reporting, and a committed threshold before the next feature PR merges~~ → Filed as [#42](https://github.com/baratsamara/alfdocument/issues/42)
3. ~~Make document-version replacement atomic: transaction, uniqueness constraints, one-latest enforcement, and stale-download rejection~~ → Filed as [#44](https://github.com/baratsamara/alfdocument/issues/44)
4. ~~Bound public-link and upload/download resource use; review token expiry/audit and move CPU-heavy password work off the main thread~~ → Filed as [#43](https://github.com/baratsamara/alfdocument/issues/43)
5. ~~Add persistent sessions and expiry~~ → Filed as [#39](https://github.com/baratsamara/alfdocument/issues/39)
6. ~~Move hosted blob storage to CranL~~ → Filed as [#31](https://github.com/baratsamara/alfdocument/issues/31)
7. ~~Rotate staging credentials~~ → Filed as [#32](https://github.com/baratsamara/alfdocument/issues/32)
8. Record the hosting/ingress/readiness/backup/rollback decision through `/decide` + AgDR before production deployment
9. Run a security-review delta over the 2026-11-19 hardening batch and the public fetch/upload/storage paths
10. Write `projects/alfdocument/roadmap.md` from the launch chores and the owner's feature priorities

## Post-Handover Checklist

- [x] Registry entry and core docs restored to `main` through a reviewed PR
- [x] Local typecheck and production build verified
- [ ] Test, lint, and coverage baseline landed
- [ ] Document-version integrity fix reviewed and merged
- [ ] Hosting and operations decision recorded
- [ ] Security-review delta completed
- [ ] Add alfdocument to the weekly `/stakeholder-update` rollup
- [ ] Re-score harnessability after the baseline lands

## Open Questions

- Which hosted ingress and process topology will run the single Express container?
- Should prior document versions remain downloadable, or is metadata-only history sufficient?
- What are the expiry, revocation, audit, and retention rules for public QR links?
- What backup/restore and blob-recovery procedure is required before production?
- Which feature is next after the launch-hardening chores (#31–#40) are triaged?

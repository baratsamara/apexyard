# alfdocument

**Repo**: https://github.com/baratsamara/alfdocument
**Workspace**: `workspace/alfdocument`
**Status**: handover — live re-assessment complete; quality baseline remains
**Tier**: P1

## What it is

Document management with QR pickup against sales orders. Staff create orders and numeric pallet/product suffixes, upload versioned documents, and print QR labels. Public suffix links expose the latest documents for pickup.

## Who owns it

- **Tech Lead**: Bara
- **Product**: Bara
- **Stakeholders**: sales-order and document-pickup operators

## Tech stack

- React 18 + Vite 6 + TypeScript strict
- Express 5 API serving the SPA and JSON endpoints
- PostgreSQL + Drizzle ORM
- Environment-configured staff auth with express-session
- Local disk or S3-compatible object storage
- CI: typecheck + build on PR and `main`

## Key links

- [Handover assessment](./handover-assessment.md)
- [Container diagram](./architecture/container.md)
- [Data-flow diagram](./architecture/dfd.md)
- [Security review](./audits/security-review/2026-11-17T13-38-07Z.md)
- [Threat model](./audits/threat-model/2026-11-19T12-28-18Z.md)
- Roadmap: not yet tracked

## Recent activity

- Last local commit: 2026-11-19 — CSRF origin guard, hardened headers, login throttle, and blob-miss logging (#41)
- Local health check: typecheck and production build passed on 2026-11-25
- Open issues: 10; open PRs: 0
- Harnessability: `low` — strict types are present, but tests, coverage, and linting are absent

## Governance state

The registry entry and this documentation were restored after the v5.10.1 sync merge dropped an unmerged registration commit. Project `main` is protected: PR required, no force-push or deletion. Feature work remains ticket-first; status stays `handover` until the minimum quality and operations baseline is closed.

<!-- Source: templates/prd.md -->

# PRD: The Aunties Website MVP

**Status**: Draft
**Author**: @baratsamara
**Created**: 2026-09-28
**Last Updated**: 2026-09-28

---

## Summary

Build and launch a Next.js website for The Aunties community. The site will serve as a simple hub introducing the community, hosting a directory of aunties, and providing a contact method. Deploys on Vercel with a Next.js App Router stack. First release is a static-first site with no backend auth or database — future phases add member features.

---

## Overview

### Problem Statement

The Aunties community needs a web presence. Currently there is no central place for aunties to find each other, learn about the community, or get in touch.

### Target User

**Primary**: Aunties (women in the community) looking for connection and information.
**Secondary**: Prospective members and community partners.

### Goals

1. Launch a public website at theaunties.com within 4 weeks
2. Achieve a Lighthouse performance score of 90+ on mobile
3. Get 50 unique visitors in the first week after launch

### Non-Goals (Out of Scope)

- User authentication / member login
- Auntie profiles with private messaging
- Event booking system
- Payment processing

### Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Site launched | 1.0 | Manual |
| Performance (Lighthouse) | >= 90 | Lighthouse CI |
| Visitors (week 1) | >= 50 | Vercel Analytics |

---

## User Stories

### US-1: Visit the homepage
> As a visitor, I want to land on a homepage that explains what The Aunties is, so that I understand the community.

**Acceptance Criteria**:
- [ ] Hero section with community tagline
- [ ] Brief mission statement
- [ ] Link to the directory page

### US-2: Browse the auntie directory
> As a community member, I want to see a list of aunties, so that I can find who's in the community.

**Acceptance Criteria**:
- [ ] Grid or list of auntie cards
- [ ] Each card shows name and a short bio
- [ ] Mobile-responsive layout

### US-3: Contact the community
> As a visitor, I want to find a contact form, so that I can ask questions.

**Acceptance Criteria**:
- [ ] Contact form with name, email, message fields
- [ ] Form submission shows a success message
- [ ] Form submissions are sent to a Slack channel or email

### Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| Form submitted with empty fields | Validation error shown inline |
| Visitor on slow mobile connection | Site remains usable, images lazy-loaded |

---

## Requirements

### Functional Requirements

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-1 | Homepage with hero, mission, CTA | Must | Simple static page |
| FR-2 | Auntie directory page | Must | Read from a JSON seed file |
| FR-3 | Contact form | Must | Formspree or similar |
| FR-4 | Mobile-responsive layout | Must | Tailwind CSS |
| FR-5 | SEO meta tags | Should | Basic Open Graph |

**Priority Key**: Must (required for launch) | Should (important) | Could (nice to have)

### Non-Functional Requirements

| Category | Requirement | Target |
|----------|-------------|--------|
| Performance | Page load time | < 2 seconds |
| Accessibility | WCAG compliance | Level AA |
| Hosting | Deploy target | Vercel |

---

## Technical Notes

### Dependencies

| Dependency | Type | Status | Owner |
|------------|------|--------|-------|
| Next.js 15 | Internal/framework | Ready | @baratsamara |
| Tailwind CSS | Internal/styling | Ready | @baratsamara |
| Vercel | External/hosting | Ready | @baratsamara |

### Technical Constraints

- Static export where possible (ISR for directory data)
- No server-side authentication in MVP

---

## Launch Plan

### Rollout Strategy

- [ ] All users at once

---

## Open Questions

| Question | Owner | Status | Resolution |
|----------|-------|--------|------------|
| Domain registration status | @baratsamara | Open | |

---

## Timeline

| Milestone | Target Date | Status |
|-----------|-------------|--------|
| PRD Approved | 2026-09-30 | |
| Design Complete | 2026-10-05 | |
| Dev Complete | 2026-10-18 | |
| QA Complete | 2026-10-22 | |
| Launch | 2026-10-25 | |

---

## Approvals

| Role | Name | Date | Status |
|------|------|------|--------|
| Product Manager | @baratsamara | 2026-09-28 | Author |
| Tech Lead | @baratsamara | | Pending

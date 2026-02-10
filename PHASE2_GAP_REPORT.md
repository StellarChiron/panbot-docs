# PanBot Content & Logic Audit - Gap Report

**Auditor:** Stellar (Quality Gatekeeper)  
**Date:** 2026-02-10  
**Scope:** panbot-docs vs panbot codebase (main + feature/sprint-21/phase1-multi-tenant)  
**Status:** Phase 2 Complete - Ready for Chiron alignment fixes

---

## Executive Summary

**Critical Finding:** Documentation is 2-4 weeks behind code reality in several key areas. The Infisical migration is functionally complete in code but docs still present legacy `.env` workflows as primary. Multi-tenant auth is in a hybrid state that isn't clearly documented.

---

## Gap 1: Infisical Pipeline Documentation (HIGH PRIORITY)

### Current Code Reality
- `src/common/config.py:auto_load_environment_file()` implements centralized env loading
- Priority order: `{TENANT}.env` → `.env` → `dev.env` → `prod.env`
- `README_ci_cd.md` (in feature branch) is the authoritative CI/CD reference
- `make sync-env` is the canonical developer workflow

### Documentation Gap
- `guides/development.md` describes tenant env files as primary workflow
- `guides/makefiles.md` shows `cp env.example dev.env` as step 1
- `guides/ci-cd.md` correctly describes Infisical but doesn't emphasize it's the canonical path
- README in panbot repo mentions Infisical but still shows legacy `.env` setup prominently

### Fix Required
- [ ] Add "Infisical First" banner to all environment setup docs
- [ ] Rename legacy sections to "Legacy: Local .env Files (Not Recommended)"
- [ ] Add explicit `make sync-env` callout as primary workflow
- [ ] Document exact precedence order from `auto_load_environment_file()`

---

## Gap 2: Logto Integration Status Confusion (HIGH PRIORITY)

### Current Code Reality
- `src/auth/ARCHITECTURE.md` marks Logto as "Planned (Issue #30)"
- No `src/auth/oidc.py` file exists in main branch
- No `apps/web/lib/logto.ts` exists
- No `/auth/oidc/callback` endpoint implemented

### Documentation Gap
- `guides/logto-auth.md` presents complete Logto setup as current
- Files like `src/auth/oidc.py`, `apps/web/lib/logto.ts` documented but don't exist
- No warning/banner that Logto is target state, not current implementation

### Fix Required
- [ ] Add prominent "🚧 PLANNED / TARGET STATE" banner to `guides/logto-auth.md`
- [ ] Clarify current auth uses fastapi-users JWT
- [ ] Add checklist: "How to verify Logto implementation is live"

---

## Gap 3: Multi-Tenant Auth Hybrid State (MEDIUM PRIORITY)

### Current Code Reality
- `require_business_access()` in `feature/sprint-21/phase1-multi-tenant` expects `business_ids` list
- `UserBusinessDB` table does NOT exist (no migration for it)
- Architecture docs in `src/` correctly mark UserBusinessDB as "NOT IMPLEMENTED"

### Documentation Gap
- Docs present UserBusinessDB as planned
- No mention of the hybrid state where JWTs already expect `business_ids` but DB uses single `business_id`
- `reference/auth-system.md` doesn't reflect this gap

### Fix Required
- [ ] Add "Current State: Hybrid" section to auth docs
- [ ] Document: JWTs use `business_ids` list for forward compatibility
- [ ] Document: DB still has single `business_id` (migration pending Issue #29)

---

## Gap 4: Missing README_ci_cd.md Reference (LOW PRIORITY)

### Current State
- `guides/ci-cd.md` links to `README_ci_cd.md` as "Full CI/CD reference"
- `README_ci_cd.md` doesn't exist in main branch of panbot repo
- Content exists in feature branch commits but may not be merged

### Fix Required
- [ ] Verify if `README_ci_cd.md` should be in main or if link should point to dev_docs
- [ ] Update link or ensure file is merged to main

---

## Recommended Doc Updates Priority

| Priority | File | Action |
|----------|------|--------|
| P0 | `guides/development.md` | Add Infisical First workflow |
| P0 | `guides/logto-auth.md` | Add "Planned" banner |
| P1 | `guides/makefiles.md` | Reorder: Infisical first, legacy second |
| P1 | `reference/auth-system.md` | Document hybrid multi-tenant state |
| P2 | `guides/ci-cd.md` | Verify README_ci_cd.md link |

---

## Test Plan for Nemo (Post-Fix Validation)

1. **Infisical Flow:**
   - Verify `make sync-env` works as documented
   - Verify `config.py` auto-load precedence matches docs
   - Verify legacy `.env` fallback still works

2. **Logto Endpoint Checks:**
   - 404 check on `/auth/oidc/callback`
   - File existence check on `src/auth/oidc.py`
   - Verify docs correctly mark as "planned"

3. **Multi-Tenant Auth:**
   - Verify JWT contains `business_ids` claim
   - Verify DB has `business_id` (single) field
   - Verify no `UserBusinessDB` table exists

---

## Handoff Notes

- **To Chiron:** All gaps identified with specific fix actions. Ready for doc update PRs.
- **To Nemo:** Test plan above ready for implementation once Chiron's fixes are live.
- **To Aegis:** DocOps strategy should account for "hybrid state" documentation—where code is ahead of reality by design during migrations.

---

*Report compiled by Stellar. Phase 2 audit complete.*

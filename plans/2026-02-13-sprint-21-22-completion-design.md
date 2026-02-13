# Sprint 21/22 Completion — Design Document

**Date**: 2026-02-13
**Branch**: `feature/sprint-21-22/completion`
**Related Issues**: #28 (Sprint 21), #51 (Sprint 22), #66 (Logout fix)
**Related PRs**: #43 (Sprint 21 impl), #69 (Sprint 23 voice)

## Goal

Complete all remaining Sprint 21 cleanup work and implement the full Sprint 22 Desktop POS Authentication system (device provisioning + operator PIN).

## Sprint 21 Remaining Work

### 1. E2E Test Logout Flow (Issue #66)
The three-layer logout fix (sign-out route cookie deletion, api-client 401 redirect, proxy.ts cookie validation) has been implemented but needs manual browser testing. Test with Chrome browser at localhost:3001.

### 2. Remove `UserDB.business_id` Column
Backward-compat column from pre-multi-tenant era. Create Alembic migration to drop it. Ensure no code references remain.

### 3. Add Auth to Onboarding Endpoints
9 endpoints in the onboarding router are publicly accessible. Add `require_business_access()` or appropriate auth dependency.

### 4. Restrict CORS Origins
Currently allowing all origins. Lock down to known frontends (localhost:3000/3001 for dev, production domains).

### 5. Merge Web Feature Branch
`apps/web` submodule feature branch `feature/sprint-21/phase2-phase3-frontend` needs to be merged to `main` after E2E verification.

## Sprint 22 Design

### Architecture: Two-Tier Device Auth

```
Device Provisioning (one-time, by owner/manager)
  → Option A: Owner Logto login on device
  → Option B: Pairing code from web dashboard
  → Result: Device bound to business, long-lived device token

Operator Session (daily use, by staff)
  → 4-6 digit PIN per staff member
  → Quick-switch between operators (<3s)
  → "Open register" mode (no PIN)

Privileged Operations (occasional, manager PIN)
  → Required for voids, refunds, discounts, settings
```

### Data Model

**New table: `DeviceDB`**
- `id` (UUID PK), `business_id` (FK), `device_name`, `device_token_hash`
- `provisioned_by` (FK to users), `provisioned_at`, `last_seen_at`
- `is_active`, `device_metadata` (JSONB)

**Extended: `UserBusinessDB`**
- Add `staff_pin_hash` (bcrypt), `pin_failed_attempts`, `pin_locked_until`

### API Endpoints

| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `/api/v1/devices/provision` | Owner JWT | Provision via owner login |
| POST | `/api/v1/devices/pair` | Pairing code | Pair via dashboard code |
| POST | `/api/v1/devices/heartbeat` | Device token | Update last_seen_at |
| GET | `/api/v1/devices` | Owner JWT | List business devices |
| DELETE | `/api/v1/devices/{id}` | Owner JWT | Deactivate device |
| POST | `/api/v1/auth/pin-login` | Device token | Staff PIN → session |
| POST | `/api/v1/auth/pin-verify` | Device + session | Manager PIN for privileged ops |
| POST | `/api/v1/staff/{id}/set-pin` | Owner JWT | Set staff PIN |
| POST | `/api/v1/staff/{id}/reset-pin` | Owner JWT | Reset PIN + clear lockout |

### Desktop App Changes

- Device provisioning flow (Logto login OR pairing code)
- PINPad component (touch-friendly, 4-6 digits)
- Operator quick-switch UI
- Manager PIN prompt modal
- Device heartbeat service
- Tauri secure storage for tokens

### Web Dashboard Changes

- Staff list with PIN management (set/reset)
- Device management page (list, view, deactivate)
- Pairing code generation UI (6-char code, 15-min expiry)

## Team Structure

| Role | Sprint 21 | Sprint 22 |
|------|-----------|-----------|
| Lead | E2E browser testing | E2E testing, coordination |
| backend-dev | DB migration, auth, CORS | Phase 1: DeviceDB, PIN auth, APIs |
| desktop-dev | — | Phase 2: PINPad, provisioning, secure storage |
| web-dev | Merge web branch | Phase 3: Staff mgmt, device mgmt |
| tester | — | Unit + integration tests |

## Acceptance Criteria

- [ ] Logout flow works end-to-end (browser verified)
- [ ] `UserDB.business_id` column removed
- [ ] Onboarding endpoints require auth
- [ ] CORS restricted to known origins
- [ ] Web feature branch merged to main
- [ ] Device provisioning via owner login works
- [ ] Device provisioning via pairing code works
- [ ] Staff PIN login creates operator session
- [ ] Quick-switch between operators < 3 seconds
- [ ] Manager PIN required for privileged ops
- [ ] PIN lockout after 5 failed attempts (15 min)
- [ ] All unit and integration tests pass

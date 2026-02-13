---
title: Authentication System
description: Current and planned authentication architecture.
---

## Current State

### Backend (FastAPI)

| Feature | Implementation |
|---------|---------------|
| Auth library | fastapi-users |
| Token type | JWT (access + refresh) |
| Claims | `user_id`, `business_id`, `role`, `permissions` |
| Validation | Claim-first (no DB lookup per request) |
| Token revocation | `users.token_version` field |
| DB validation | Optional via `AUTH_VALIDATE_DB_ON_REQUEST` flag |

### Multi-Tenant Auth (UserBusinessDB)

- `UserBusinessDB` is the source of truth for user-to-business mappings.
- `UserDB.business_id` column has been **removed** — all business access is resolved via `UserBusinessDB`.
- `get_current_user()` returns both `business_id` (primary) and `business_ids` (all accessible).
- Realtime/Centrifugo token requests check against the full `business_ids` list.

### Desktop App (JWT Login)

| Feature | Implementation |
|---------|---------------|
| Login | Email/phone → POST /api/v1/auth/login |
| Token storage | localStorage |
| Auto-refresh | `AutoRefreshTokenManager` with JWT config from backend |
| Business context | From JWT claims (`business_id`) |
| Realtime token | Separate JWT via POST /api/v1/realtime/token |

### Desktop App (Device + PIN Login)

| Feature | Implementation |
|---------|---------------|
| Device identity | One-time pairing code → permanent device token |
| Staff identity | 4-6 digit PIN per staff member per business |
| Token storage | `device_token` + `operator_session_token` in localStorage |
| Session lifetime | 8 hours (operator session JWT) |
| Brute-force protection | 5 attempts then 15-minute lockout |

See [Device Authentication guide](/guides/device-authentication/) for the complete flow.

### Web Dashboard

| Feature | Implementation |
|---------|---------------|
| Auth library | NextAuth v5 beta |
| Providers | Google, Microsoft Entra ID, Apple |
| Strategy | JWT sessions (no DB sessions) |
| Token exchange | Not yet wired to backend JWT |

## Target State: Logto Migration (Sprint 21 Phase 2)

| Feature | Planned |
|---------|---------|
| IAM provider | Logto (replaces NextAuth + fastapi-users) |
| Social login | Google, Apple via Logto |
| Token format | OIDC tokens validated by backend |
| Multi-business | UserBusinessDB with per-business roles |
| Desktop | Logto SDK (replaces direct JWT login) |
| Web | Logto SDK (replaces NextAuth) |

### Migration Checklist

- [ ] Set up Logto instance
- [ ] Replace NextAuth with Logto SDK in web app
- [ ] Update backend for OIDC token validation
- [ ] Create UserBusinessDB sync endpoint
- [ ] Handle Logto webhooks for user lifecycle
- [ ] Migrate existing users
- [ ] Update desktop app auth flow

## JWT Token Configuration

All token lifetimes are configured in backend settings:

| Setting | Purpose |
|---------|---------|
| `JWT_ACCESS_TOKEN_MINUTES` | Access token lifetime |
| `JWT_REFRESH_TOKEN_DAYS` | Refresh token lifetime |
| `JWT_ALGORITHM` | Signing algorithm (HS256) |
| `JWT_SECRET` | Token signing secret |

Desktop app fetches these settings from `/api/v1/realtime/status` for consistent token management.

## Roles & Permissions

| Role | Login Method | Access |
|------|-------------|--------|
| STAFF | PIN on paired device | Single business, desktop app |
| OWNER | Email/OAuth or PIN | Multiple businesses, web + desktop |
| ADMIN | Email/OAuth | All businesses, system settings |

Roles are stored per-business in `UserBusinessDB.role`. Owners manage staff access and PIN assignment.

---
title: Device Authentication
description: Device pairing and PIN-based staff login for POS terminals and kiosks.
---

PanBot supports device-based authentication for POS terminals, kiosks, and shared restaurant devices. This system separates **device identity** (which restaurant) from **operator identity** (which staff member), allowing multiple staff to share a single device.

## Overview

| Concept | Purpose |
|---------|---------|
| **Device pairing** | Associates a physical device with a business via a one-time pairing code |
| **Device token** | Permanent credential stored on the device, identifies the business |
| **Staff PIN** | 4-6 digit code that identifies a staff member within the business |
| **Operator session** | Short-lived JWT issued after PIN login, scoped to user + business + device |

## Architecture

```
Admin (Web Dashboard)           Backend                    Device (Desktop App)
        |                          |                              |
  Generate pairing code ---------> Store in Redis (15 min) -----> Enter pairing code
        |                          |                              |
        |                   Validate code, create device -------> Store device_token
        |                   Return device_token (once)            in localStorage
        |                          |                              |
        |                          |                       Staff enters PIN
        |                   Validate device_token <-------------- Send token + PIN
        |                   Find business, match PIN              |
        |                   Issue operator session JWT ---------> Store session
        |                          |                              App ready
```

## Device Pairing Flow

### Step 1: Admin generates a pairing code

From the web dashboard, a business owner/admin generates a 6-character pairing code:

```
POST /api/v1/devices/pairing-code
Authorization: Bearer <admin-jwt>
Content-Type: application/json

{
  "business_id": "uuid",
  "device_name": "Front Counter iPad"
}
```

Response:
```json
{
  "pairing_code": "ABC123",
  "expires_in_seconds": 900
}
```

The code is stored in Redis with a 15-minute TTL. It can only be used once.

### Step 2: Device enters the pairing code

On the desktop app, the `DeviceSetup` component presents a pairing code entry screen. The device sends the code to the backend:

```
POST /api/v1/devices/pair
Content-Type: application/json

{
  "pairing_code": "ABC123"
}
```

No authentication is required — the pairing code itself is the authorization.

Response:
```json
{
  "device": {
    "id": "device-uuid",
    "business_id": "business-uuid",
    "device_name": "Front Counter iPad",
    "is_active": true
  },
  "device_token": "dvc_<random-token>"
}
```

The raw `device_token` is returned **once** and must be stored by the device. The backend only stores a SHA-256 hash.

### Step 3: Device stores credentials

The desktop app stores these in localStorage:

| Key | Value | Persists across sessions |
|-----|-------|:---:|
| `device_token` | Raw device token | Yes |
| `device_id` | Device UUID | Yes |
| `device_business_id` | Business UUID | Yes |
| `device_name` | Human-readable name | Yes |

## Staff PIN Login

After a device is paired, staff members authenticate with a PIN.

### Setting a PIN

Admins set PINs for staff members via the web dashboard:

```
POST /api/v1/staff/{user_business_id}/set-pin
Authorization: Bearer <admin-jwt>
Content-Type: application/json

{
  "pin": "1234"
}
```

PINs are 4-6 digits, stored as bcrypt hashes in `UserBusinessDB.staff_pin_hash`.

### PIN Login

The device sends the device token + PIN:

```
POST /api/v1/auth/pin-login
Content-Type: application/json

{
  "device_token": "dvc_...",
  "pin": "1234"
}
```

The backend:
1. Validates the device token (SHA-256 hash lookup in `devices` table)
2. Finds all staff with PINs for that business
3. Compares the entered PIN against each staff member's bcrypt hash
4. Issues an operator session JWT (8-hour lifetime)

Response:
```json
{
  "operator_session_token": "eyJ...",
  "user_id": "staff-uuid",
  "business_id": "business-uuid",
  "role": "staff",
  "full_name": "John Doe",
  "expires_in_seconds": 28800
}
```

### Operator Session Token Claims

```json
{
  "sub": "user-id",
  "user_id": "user-id",
  "business_id": "business-id",
  "device_id": "device-id",
  "role": "staff",
  "permissions": ["order:create", "order:view"],
  "token_type": "operator_session",
  "aud": "panbot:operator"
}
```

The token is bound to the specific device (`device_id` claim).

## Brute-Force Protection

| Setting | Value |
|---------|-------|
| Max failed attempts | 5 per staff member |
| Lockout duration | 15 minutes |
| Lockout scope | Per user-business (not per device) |

After 5 consecutive failed attempts, the staff member's account is locked for 15 minutes. Successful login resets the counter.

## Security Model

| Aspect | Implementation |
|--------|---------------|
| Device token storage | Raw token in localStorage, SHA-256 hash in DB |
| PIN storage | Bcrypt hash in `UserBusinessDB.staff_pin_hash` |
| Pairing code | One-time use, deleted from Redis immediately after redemption |
| Pairing code lifetime | 15 minutes |
| Operator session lifetime | 8 hours |
| Session scope | Bound to user + business + device |

## API Endpoints

| Endpoint | Auth | Purpose |
|----------|------|---------|
| `POST /api/v1/devices/pairing-code` | JWT (admin) | Generate pairing code |
| `POST /api/v1/devices/pair` | None (code is auth) | Pair device with code |
| `POST /api/v1/devices/heartbeat` | `X-Device-Token` header | Device heartbeat |
| `GET /api/v1/devices/?business_id=` | JWT (admin) | List business devices |
| `DELETE /api/v1/devices/{id}` | JWT (admin) | Deactivate device |
| `POST /api/v1/staff/{ub_id}/set-pin` | JWT (admin) | Set staff PIN |
| `POST /api/v1/staff/{ub_id}/reset-pin` | JWT (admin) | Clear staff PIN + unlock |
| `POST /api/v1/auth/pin-login` | None (device token in body) | PIN login |
| `POST /api/v1/auth/pin-verify` | None (device token in body) | Verify PIN without session |

## Desktop App Components

| Component | File | Purpose |
|-----------|------|---------|
| `DeviceSetup` | `apps/desktop/src/components/DeviceSetup.tsx` | Pairing code entry + success screen |
| `PinPad` | `apps/desktop/src/components/PinPad.tsx` | PIN entry with auto-submit at 6 digits |

The app state machine in `App.tsx` transitions between states:
1. `device_setup` — no device token, show `DeviceSetup`
2. `pin_login` — device paired, show `PinPad`
3. `initializing` — PIN login succeeded, loading business data
4. `ready` — app fully loaded

## Key Files

| File | Layer | Purpose |
|------|-------|---------|
| `src/services/device_service.py` | Backend | Device provisioning, pairing, token validation |
| `src/services/pin_service.py` | Backend | PIN hashing, verification, login, lockout |
| `src/api/v1/devices.py` | Backend | Device REST endpoints |
| `src/api/v1/staff.py` | Backend | PIN management + auth endpoints |
| `src/models/database.py` | Backend | `DeviceDB` and `UserBusinessDB` models |
| `apps/desktop/src/components/DeviceSetup.tsx` | Frontend | Pairing UI |
| `apps/desktop/src/components/PinPad.tsx` | Frontend | PIN entry UI |
| `apps/desktop/src/App.tsx` | Frontend | App state machine |

# Doctor Invitation & Sign-Up Flow - Implementation Guide

**Feature:** Complete user management system for Owner registration and Doctor invitation/activation
**Version:** 1.0
**Last Updated:** 2025-11-09

---

## Table of Contents
- [Overview](#overview)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Data Flow Architecture](#data-flow-architecture)
- [Frontend Components](#frontend-components)
- [Backend Lambda Functions](#backend-lambda-functions)
- [Email Templates](#email-templates)
- [Security Implementation](#security-implementation)
- [Testing Checklist](#testing-checklist)

---

## Overview

This implementation creates a complete multi-tenant user management system where:
1. **Owner** registers and creates a clinic/IPRESS
2. **Owner** invites doctors via email
3. **Doctor** activates account and sets password
4. **Doctor** completes clinical profile
5. **Owner** validates and activates doctor

### User Flow States
```
Owner:  Draft → Incomplete → Active → Suspended → Closed
Doctor: Invited (🟠) → Incomplete (🟡) → Pending (🟣) → Active (🟢) → Inactive (🔴)
```

---

## Database Schema

### Table 1: Users_Table

**Purpose:** Central user management across all roles (owner, doctor, patient)

**Primary Key:** `user_id` (String, UUID)

**Global Secondary Indexes:**
- `email-index`: `email` (PK) - Unique constraint
- `tenant_id-index`: `tenant_id` (PK), `created_at` (SK) - List users by tenant
- `status-index`: `status` (PK), `created_at` (SK) - Filter by status

**Attributes:**
```typescript
{
  user_id: string;              // PK: UUID
  email: string;                // GSI: email-index (UK)
  password_hash: string;        // Argon2id hashed password
  full_name: string;            // Full name
  role: string;                 // "owner" | "doctor" | "patient"
  status: string;               // "draft" | "invited" | "incomplete" | "pending" | "active" | "inactive" | "suspended" | "closed"
  tenant_id: string;            // FK to Tenants_Table, GSI: tenant_id-index
  invite_token?: string;        // Single-use JWT token (only for invited users)
  invite_token_expires?: number; // Epoch timestamp (24h from creation)
  mfa_enabled: boolean;         // Multi-factor authentication enabled
  mfa_secret?: string;          // TOTP secret (encrypted)
  created_at: string;           // ISO timestamp
  updated_at: string;           // ISO timestamp
  last_login?: string;          // ISO timestamp
}
```

**State Transitions:**
```
Owner:
  draft → (email verified) → incomplete → (terms accepted) → active

Doctor:
  invited → (password set) → incomplete → (profile completed) → pending → (admin validated) → active
```

---

### Table 2: Tenants_Table

**Purpose:** Multi-tenant clinic/IPRESS management

**Primary Key:** `tenant_id` (String, UUID)

**Global Secondary Indexes:**
- `owner_user_id-index`: `owner_user_id` (PK) - Find tenant by owner

**Attributes:**
```typescript
{
  tenant_id: string;            // PK: UUID (clinic/IPRESS ID)
  owner_user_id: string;        // FK to Users_Table, GSI
  clinic_name: string;          // Clinic/IPRESS name
  country: string;              // ISO country code (locked after setup)
  state_region: string;         // State/Region (locked after setup)
  city: string;                 // City/District (editable)
  timezone: string;             // IANA timezone (auto-set, locked)
  language: string;             // "es" | "en" | "pt" (default: "es", editable)
  status: string;               // "draft" | "incomplete" | "active" | "suspended" | "closed"
  created_at: string;           // ISO timestamp
  updated_at: string;           // ISO timestamp
}
```

---

### Table 3: Audit_Logs_Table

**Purpose:** WORM (Write-Once-Read-Many) audit trail for compliance

**Primary Key:** `log_id` (String, UUID)
**Sort Key:** `timestamp` (String, ISO timestamp)

**Global Secondary Indexes:**
- `user_id-timestamp-index`: `user_id` (PK), `timestamp` (SK) - User activity
- `tenant_id-timestamp-index`: `tenant_id` (PK), `timestamp` (SK) - Tenant activity
- `action-timestamp-index`: `action` (PK), `timestamp` (SK) - Filter by action type

**Attributes:**
```typescript
{
  log_id: string;               // PK: UUID
  timestamp: string;            // SK: ISO timestamp
  user_id: string;              // GSI: user_id-timestamp-index
  tenant_id: string;            // GSI: tenant_id-timestamp-index
  action: string;               // "user.created" | "user.invited" | "user.activated" | "user.validated" | etc.
  entity_type: string;          // "user" | "tenant" | "doctor" | "patient"
  entity_id: string;            // ID of affected entity
  ip_address: string;           // IPv4/IPv6
  user_agent: string;           // Browser/client info
  metadata: object;             // JSON with additional context
  created_at: string;           // ISO timestamp (same as timestamp, for consistency)
}
```

**Audit Actions:**
```typescript
// Owner actions
"owner.registered"
"owner.email_verified"
"owner.terms_accepted"
"owner.signin"

// Doctor actions
"doctor.invited"
"doctor.activated"
"doctor.terms_accepted"
"doctor.profile_completed"
"doctor.validated"
"doctor.signin"

// Admin actions
"user.status_changed"
"user.role_changed"
```

---

### Table 4: Link Users_Table ↔ Doctors_Table_V2

**Modification to existing Doctors_Table_V2:**

Add new attribute:
```typescript
{
  doctor_id: string;            // PK (existing)
  user_id: string;              // NEW: FK to Users_Table
  // ... rest of existing fields
}
```

**New GSI on Doctors_Table_V2:**
- `user_id-index`: `user_id` (PK) - Find doctor by user_id

---

## API Endpoints

### Owner Registration & Authentication

#### 1. POST /auth/register-owner
**Purpose:** Create owner user + tenant

**Request:**
```json
{
  "email": "owner@clinic.com",
  "password": "SecurePass123!",
  "fullName": "Dr. Juan Pérez",
  "clinicName": "Clínica San Juan",
  "country": "PE",
  "stateRegion": "Lima",
  "city": "San Isidro"
}
```

**Response 201:**
```json
{
  "userId": "uuid-123",
  "tenantId": "uuid-456",
  "message": "Registration successful. Check your email to verify your account."
}
```

**Response 400:**
```json
{
  "error": "INVALID_EMAIL_FORMAT",
  "message": "Email format is invalid"
}
```

**Response 409:**
```json
{
  "error": "EMAIL_ALREADY_EXISTS",
  "message": "An account with this email already exists"
}
```

---

#### 2. GET /auth/verify-email?token={JWT}
**Purpose:** Verify email with 24h token

**Response 200:**
```json
{
  "success": true,
  "message": "Email verified successfully",
  "redirectUrl": "/onboarding"
}
```

**Response 400:**
```json
{
  "error": "INVALID_TOKEN",
  "message": "Verification token is invalid or expired"
}
```

---

#### 3. POST /auth/signin-owner
**Purpose:** Owner login (email + password)

**Request:**
```json
{
  "email": "owner@clinic.com",
  "password": "SecurePass123!"
}
```

**Response 200:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "userId": "uuid-123",
    "email": "owner@clinic.com",
    "fullName": "Dr. Juan Pérez",
    "role": "owner",
    "status": "active",
    "tenantId": "uuid-456"
  }
}
```

**Response 401:**
```json
{
  "error": "INVALID_CREDENTIALS",
  "message": "Email or password is incorrect"
}
```

---

#### 4. POST /legal/accept-terms
**Purpose:** Accept Terms & Conditions with IP stamp

**Request:**
```json
{
  "userId": "uuid-123",
  "termsVersion": "v1.0",
  "ipAddress": "192.168.1.1"
}
```

**Response 200:**
```json
{
  "success": true,
  "message": "Terms accepted",
  "userStatus": "active"
}
```

---

### Doctor Invitation Flow

#### 5. POST /users/invite-doctor
**Purpose:** Owner sends invitation to doctor

**Headers:**
```
Authorization: Bearer {ownerJWT}
```

**Request:**
```json
{
  "email": "doctor@example.com",
  "fullName": "Dr. María García",
  "role": "doctor"
}
```

**Response 201:**
```json
{
  "userId": "uuid-789",
  "message": "Invitation sent successfully",
  "inviteExpiresAt": "2025-11-10T15:30:00Z"
}
```

**Response 400:**
```json
{
  "error": "MISSING_REQUIRED_FIELD",
  "message": "Email and fullName are required"
}
```

**Response 409:**
```json
{
  "error": "USER_ALREADY_EXISTS",
  "message": "A user with this email already exists"
}
```

---

#### 6. GET /users/validate-invite-token?token={JWT}
**Purpose:** Validate invite token (frontend check before showing form)

**Response 200:**
```json
{
  "valid": true,
  "email": "doctor@example.com",
  "fullName": "Dr. María García",
  "clinicName": "Clínica San Juan"
}
```

**Response 400:**
```json
{
  "valid": false,
  "error": "TOKEN_EXPIRED",
  "message": "This invitation link has expired. Please contact the clinic administrator."
}
```

**Response 401:**
```json
{
  "valid": false,
  "error": "INVALID_TOKEN",
  "message": "This invitation link is invalid"
}
```

**Response 409:**
```json
{
  "valid": false,
  "error": "ALREADY_ACTIVATED",
  "message": "This account has already been activated"
}
```

---

#### 7. POST /users/accept-invite
**Purpose:** Doctor activates account and sets password

**Request:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "password": "SecurePass123!",
  "confirmPassword": "SecurePass123!",
  "acceptedTerms": true
}
```

**Response 200:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "userId": "uuid-789",
    "email": "doctor@example.com",
    "fullName": "Dr. María García",
    "role": "doctor",
    "status": "incomplete",
    "tenantId": "uuid-456"
  },
  "redirectUrl": "/clinical-profile"
}
```

**Response 400:**
```json
{
  "error": "WEAK_PASSWORD",
  "message": "Password must be at least 8 characters with 1 uppercase, 1 lowercase, and 1 number"
}
```

---

#### 8. POST /users/resend-invite
**Purpose:** Resend expired invitation

**Headers:**
```
Authorization: Bearer {ownerJWT}
```

**Request:**
```json
{
  "userId": "uuid-789"
}
```

**Response 200:**
```json
{
  "success": true,
  "message": "Invitation resent successfully",
  "inviteExpiresAt": "2025-11-10T15:30:00Z"
}
```

---

### User Management

#### 9. GET /users?tenantId={tenantId}&status={status}&role={role}
**Purpose:** List all users by tenant with filters

**Headers:**
```
Authorization: Bearer {ownerJWT}
```

**Query Parameters:**
- `tenantId` (required): Tenant ID
- `status` (optional): Filter by status (invited, incomplete, pending, active, inactive)
- `role` (optional): Filter by role (owner, doctor)

**Response 200:**
```json
{
  "users": [
    {
      "userId": "uuid-789",
      "email": "doctor@example.com",
      "fullName": "Dr. María García",
      "role": "doctor",
      "status": "pending",
      "lastLogin": "2025-11-08T10:30:00Z",
      "createdAt": "2025-11-01T08:00:00Z"
    }
  ],
  "count": 1
}
```

---

#### 10. PATCH /users/:userId/status
**Purpose:** Change user status (admin action)

**Headers:**
```
Authorization: Bearer {ownerJWT}
```

**Request:**
```json
{
  "status": "active",
  "reason": "Profile validated"
}
```

**Response 200:**
```json
{
  "success": true,
  "userId": "uuid-789",
  "previousStatus": "pending",
  "newStatus": "active"
}
```

---

#### 11. GET /users/:userId
**Purpose:** Get user details

**Headers:**
```
Authorization: Bearer {ownerJWT or doctorJWT}
```

**Response 200:**
```json
{
  "userId": "uuid-789",
  "email": "doctor@example.com",
  "fullName": "Dr. María García",
  "role": "doctor",
  "status": "active",
  "tenantId": "uuid-456",
  "mfaEnabled": false,
  "createdAt": "2025-11-01T08:00:00Z",
  "updatedAt": "2025-11-05T12:00:00Z",
  "lastLogin": "2025-11-08T10:30:00Z"
}
```

---

#### 12. POST /users/clinical-profile
**Purpose:** Update clinical profile (doctor onboarding)

**Headers:**
```
Authorization: Bearer {doctorJWT}
```

**Request:**
```json
{
  "userId": "uuid-789",
  "personalData": {
    "documentType": "DNI",
    "documentNumber": "12345678",
    "dateOfBirth": "1985-05-15",
    "gender": "female",
    "address": "Av. Principal 123",
    "personalPhone": "+51999888777",
    "emergencyContactPhone": "+51999888666",
    "signatureUrl": "s3://bucket/signatures/uuid-789.png"
  },
  "professionalData": {
    "profession": "Doctor",
    "cmp": "123456",
    "rne": "78901",
    "primarySpecialty": "Cardiología",
    "subspecialty": "Electrofisiología",
    "primaryLocation": "location-uuid-1"
  }
}
```

**Response 200:**
```json
{
  "success": true,
  "userId": "uuid-789",
  "doctorId": "doctor-uuid-123",
  "status": "pending",
  "message": "Clinical profile completed. Awaiting admin validation."
}
```

---

#### 13. POST /users/:userId/validate
**Purpose:** Owner validates doctor profile

**Headers:**
```
Authorization: Bearer {ownerJWT}
```

**Request:**
```json
{
  "validated": true,
  "notes": "Profile reviewed and approved"
}
```

**Response 200:**
```json
{
  "success": true,
  "userId": "uuid-789",
  "doctorId": "doctor-uuid-123",
  "previousStatus": "pending",
  "newStatus": "active",
  "activeForBooking": true
}
```

---

## Data Flow Architecture

### Flow 1: Owner Registration

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Owner Registers                                          │
└─────────────────────────────────────────────────────────────────┘

[Owner Frontend: /register]
    ↓ User enters: email, password, fullName, clinicName, country
    ↓ POST /api/auth/register-owner
    ↓ { email, password, fullName, clinicName, country, stateRegion, city }

[Next.js API Route: /api/auth/register-owner]
    ↓ Extract cookie (none expected)
    ↓ Proxy to AWS API Gateway
    ↓ POST {AWS_API_URL}/auth/register-owner
    ↓ Body: forward request body

[API Gateway → Lambda: RegisterOwnerFunction]
    ↓ Validate email format (RFC 5322)
    ↓ Validate password strength (min 8 chars, 1 upper, 1 lower, 1 number)
    ↓ Check if email exists (query Users_Table via email-index)
    ↓ If exists → Return 409 Conflict
    ↓
    ↓ Hash password with Argon2id:
    ↓   const passwordHash = await argon2.hash(password, {
    ↓     type: argon2.argon2id,
    ↓     memoryCost: 65536,
    ↓     timeCost: 3,
    ↓     parallelism: 4
    ↓   });
    ↓
    ↓ Generate UUID for user and tenant
    ↓ const userId = uuidv4();
    ↓ const tenantId = uuidv4();
    ↓
    ↓ Generate email verification token (24h JWT):
    ↓   const verificationToken = await new jose.SignJWT({
    ↓     userId,
    ↓     email,
    ↓     type: 'email_verification'
    ↓   })
    ↓     .setProtectedHeader({ alg: 'HS256' })
    ↓     .setExpirationTime('24h')
    ↓     .sign(secret);
    ↓
    ↓ Create Users_Table record:
    ↓   {
    ↓     user_id: userId,
    ↓     email,
    ↓     password_hash: passwordHash,
    ↓     full_name: fullName,
    ↓     role: "owner",
    ↓     status: "draft",
    ↓     tenant_id: tenantId,
    ↓     mfa_enabled: false,
    ↓     created_at: new Date().toISOString(),
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Create Tenants_Table record:
    ↓   {
    ↓     tenant_id: tenantId,
    ↓     owner_user_id: userId,
    ↓     clinic_name: clinicName,
    ↓     country,
    ↓     state_region: stateRegion,
    ↓     city,
    ↓     timezone: getTimezoneByCountry(country), // e.g., "America/Lima"
    ↓     language: "es",
    ↓     status: "draft",
    ↓     created_at: new Date().toISOString(),
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Send verification email via Mailgun:
    ↓   await mailgun.messages.create({
    ↓     from: "VITAS Clinic <noreply@vitas.com>",
    ↓     to: email,
    ↓     subject: "Verifica tu correo - VITAS Clinic",
    ↓     html: renderEmailTemplate('owner-verification', {
    ↓       fullName,
    ↓       verificationLink: `${FRONTEND_URL}/verify-email?token=${verificationToken}`
    ↓     })
    ↓   });
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId,
    ↓     tenantId,
    ↓     action: "owner.registered",
    ↓     entityType: "user",
    ↓     entityId: userId,
    ↓     ipAddress: event.requestContext.identity.sourceIp,
    ↓     metadata: { email, clinicName }
    ↓   });
    ↓
    ← Return 201 { userId, tenantId, message: "Check email" }

[Next.js API Route]
    ← Forward response to frontend

[Frontend]
    ↓ Show success message:
    ↓   "Registration successful! Check your email to verify your account."
    ↓ Redirect to /check-email page
```

---

### Flow 2: Email Verification

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: Owner Verifies Email                                    │
└─────────────────────────────────────────────────────────────────┘

[Owner clicks email verification link]
    ↓ Navigate to /verify-email?token={verificationToken}

[Frontend: /verify-email page]
    ↓ On mount, extract token from URL
    ↓ GET /api/auth/verify-email?token={token}

[Next.js API Route: /api/auth/verify-email]
    ↓ Proxy to AWS API Gateway
    ↓ GET {AWS_API_URL}/auth/verify-email?token={token}

[API Gateway → Lambda: VerifyEmailFunction]
    ↓ Verify JWT signature + expiry:
    ↓   const { payload } = await jose.jwtVerify(token, secret);
    ↓   // payload: { userId, email, type: 'email_verification' }
    ↓
    ↓ Check token type:
    ↓   if (payload.type !== 'email_verification') → Return 400 Invalid Token
    ↓
    ↓ Get user from Users_Table:
    ↓   const user = await getUserById(payload.userId);
    ↓
    ↓ Check user status:
    ↓   if (user.status !== 'draft') → Return 409 Already Verified
    ↓
    ↓ Update Users_Table:
    ↓   {
    ↓     status: "incomplete",
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Update Tenants_Table:
    ↓   {
    ↓     status: "incomplete",
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId: user.user_id,
    ↓     tenantId: user.tenant_id,
    ↓     action: "owner.email_verified",
    ↓     entityType: "user",
    ↓     entityId: user.user_id,
    ↓     ipAddress: event.requestContext.identity.sourceIp
    ↓   });
    ↓
    ← Return 200 { success: true, redirectUrl: "/onboarding" }

[Frontend]
    ↓ Show success message
    ↓ Redirect to /onboarding (complete profile + accept T&C)
```

---

### Flow 3: Accept Terms & Conditions

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: Owner Accepts Terms & Completes Onboarding              │
└─────────────────────────────────────────────────────────────────┘

[Owner completes onboarding form]
    ↓ Checkbox: Accept Terms & Privacy Policy (required)
    ↓ POST /api/legal/accept-terms
    ↓ { userId, termsVersion: "v1.0", ipAddress: clientIP }

[Next.js API Route: /api/legal/accept-terms]
    ↓ Extract client IP from headers
    ↓ Proxy to AWS API Gateway

[API Gateway → Lambda: AcceptTermsFunction]
    ↓ Get user from Users_Table
    ↓ Verify user status is "incomplete"
    ↓
    ↓ Record consent in legal system (future: legal_consents table)
    ↓ For now, log in audit trail
    ↓
    ↓ Update Users_Table:
    ↓   {
    ↓     status: "active",
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Update Tenants_Table:
    ↓   {
    ↓     status: "active",
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId,
    ↓     tenantId: user.tenant_id,
    ↓     action: "owner.terms_accepted",
    ↓     entityType: "user",
    ↓     entityId: userId,
    ↓     ipAddress,
    ↓     metadata: { termsVersion: "v1.0" }
    ↓   });
    ↓
    ← Return 200 { success: true, userStatus: "active" }

[Frontend]
    ↓ Show success message
    ↓ Redirect to /signin (owner logs in manually)
```

---

### Flow 4: Owner Sign In

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: Owner Signs In                                          │
└─────────────────────────────────────────────────────────────────┘

[Owner enters email + password]
    ↓ POST /api/auth/signin-owner
    ↓ { email, password }

[Next.js API Route: /api/auth/signin-owner]
    ↓ Proxy to AWS API Gateway
    ↓ POST {AWS_API_URL}/auth/signin-owner

[API Gateway → Lambda: SigninOwnerFunction]
    ↓ Query Users_Table via email-index:
    ↓   const user = await getUserByEmail(email);
    ↓
    ↓ If not found → Return 401 Invalid Credentials
    ↓
    ↓ Verify role is "owner":
    ↓   if (user.role !== 'owner') → Return 401 Invalid Credentials
    ↓
    ↓ Verify password with Argon2id:
    ↓   const isValid = await argon2.verify(user.password_hash, password);
    ↓   if (!isValid) → Return 401 Invalid Credentials
    ↓
    ↓ Check user status:
    ↓   if (user.status === 'inactive' || user.status === 'suspended')
    ↓     → Return 403 Account Disabled
    ↓
    ↓ Generate auth JWT (24h):
    ↓   const token = await new jose.SignJWT({
    ↓     userId: user.user_id,
    ↓     email: user.email,
    ↓     role: "owner",
    ↓     tenantId: user.tenant_id
    ↓   })
    ↓     .setProtectedHeader({ alg: 'HS256' })
    ↓     .setExpirationTime('24h')
    ↓     .sign(secret);
    ↓
    ↓ Update last_login:
    ↓   await updateUser(user.user_id, {
    ↓     last_login: new Date().toISOString()
    ↓   });
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId: user.user_id,
    ↓     tenantId: user.tenant_id,
    ↓     action: "owner.signin",
    ↓     entityType: "user",
    ↓     entityId: user.user_id,
    ↓     ipAddress: event.requestContext.identity.sourceIp
    ↓   });
    ↓
    ← Return 200 { token, user: { userId, email, fullName, role, status, tenantId } }

[Next.js API Route]
    ↓ Set HTTP-only cookie:
    ↓   cookies().set('authToken', token, {
    ↓     httpOnly: true,
    ↓     secure: process.env.NODE_ENV === 'production',
    ↓     sameSite: 'lax',
    ↓     maxAge: 60 * 60 * 24 // 24 hours
    ↓   });
    ↓
    ← Return user data (without token)

[Frontend]
    ↓ Store user in UserContext
    ↓ Redirect to /dashboard
```

---

### Flow 5: Owner Invites Doctor

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 5: Owner Invites Doctor                                    │
└─────────────────────────────────────────────────────────────────┘

[Owner Dashboard: /users → Click "Invite User"]
    ↓ Modal opens with form:
    ↓   - Full Name (required)
    ↓   - Email (required)
    ↓   - Role (default: Clinical)
    ↓
    ↓ POST /api/users/invite-doctor
    ↓ { email, fullName, role: "doctor" }
    ↓ Authorization: Bearer {ownerJWT} (from cookie)

[Next.js API Route: /api/users/invite-doctor]
    ↓ Extract JWT from cookie
    ↓ Proxy to AWS API Gateway
    ↓ POST {AWS_API_URL}/users/invite-doctor
    ↓ Headers: Authorization: Bearer {ownerJWT}

[API Gateway → Lambda: InviteDoctorFunction]
    ↓ Verify owner JWT:
    ↓   const { payload } = await jose.jwtVerify(token, secret);
    ↓   if (payload.role !== 'owner') → Return 403 Forbidden
    ↓
    ↓ Validate request:
    ↓   - Email format (RFC 5322)
    ↓   - Full name (1-100 chars)
    ↓
    ↓ Check if email exists (query Users_Table via email-index):
    ↓   const existingUser = await getUserByEmail(email);
    ↓   if (existingUser) → Return 409 User Already Exists
    ↓
    ↓ Generate UUID for new user:
    ↓   const userId = uuidv4();
    ↓
    ↓ Generate invite token (24h JWT):
    ↓   const inviteToken = await new jose.SignJWT({
    ↓     userId,
    ↓     email,
    ↓     role: "doctor",
    ↓     tenantId: payload.tenantId,
    ↓     type: 'invite'
    ↓   })
    ↓     .setProtectedHeader({ alg: 'HS256' })
    ↓     .setExpirationTime('24h')
    ↓     .setIssuedAt()
    ↓     .sign(secret);
    ↓
    ↓ Create Users_Table record:
    ↓   {
    ↓     user_id: userId,
    ↓     email,
    ↓     full_name: fullName,
    ↓     role: "doctor",
    ↓     status: "invited",
    ↓     tenant_id: payload.tenantId,
    ↓     invite_token: inviteToken,
    ↓     invite_token_expires: Date.now() + (24 * 60 * 60 * 1000),
    ↓     mfa_enabled: false,
    ↓     created_at: new Date().toISOString(),
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Get tenant info:
    ↓   const tenant = await getTenantById(payload.tenantId);
    ↓
    ↓ Send invitation email via Mailgun:
    ↓   await mailgun.messages.create({
    ↓     from: "VITAS Clinic <noreply@vitas.com>",
    ↓     to: email,
    ↓     subject: `Invitación a VITAS - ${tenant.clinic_name}`,
    ↓     html: renderEmailTemplate('doctor-invitation', {
    ↓       fullName,
    ↓       clinicName: tenant.clinic_name,
    ↓       activationLink: `${FRONTEND_URL}/activate?token=${inviteToken}`,
    ↓       expiresIn: "24 horas"
    ↓     })
    ↓   });
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId: payload.userId, // Owner who invited
    ↓     tenantId: payload.tenantId,
    ↓     action: "doctor.invited",
    ↓     entityType: "user",
    ↓     entityId: userId, // New doctor user
    ↓     ipAddress: event.requestContext.identity.sourceIp,
    ↓     metadata: { email, fullName, invitedBy: payload.email }
    ↓   });
    ↓
    ← Return 201 {
    ←   userId,
    ←   message: "Invitation sent successfully",
    ←   inviteExpiresAt: ISO timestamp
    ← }

[Frontend]
    ↓ Close modal
    ↓ Show success snackbar: "Invitation sent to {email}"
    ↓ Refresh user list
    ↓ New doctor appears with status: 🟠 Invited
```

---

### Flow 6: Doctor Activation

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 6: Doctor Activates Account                                │
└─────────────────────────────────────────────────────────────────┘

[Doctor receives email → clicks activation link]
    ↓ Navigate to /activate?token={inviteToken}

[Frontend: /activate page]
    ↓ On mount, extract token from URL
    ↓ GET /api/users/validate-invite-token?token={token}

[Next.js API Route → API Gateway → Lambda: ValidateInviteTokenFunction]
    ↓ Verify JWT signature + expiry:
    ↓   const { payload } = await jose.jwtVerify(token, secret);
    ↓
    ↓ Check token type:
    ↓   if (payload.type !== 'invite') → Return 400 Invalid Token
    ↓
    ↓ Get user from Users_Table:
    ↓   const user = await getUserById(payload.userId);
    ↓
    ↓ Validate user status:
    ↓   if (user.status !== 'invited')
    ↓     → Return 409 Already Activated
    ↓
    ↓ Validate invite_token matches:
    ↓   if (user.invite_token !== token)
    ↓     → Return 401 Invalid Token
    ↓
    ↓ Check token expiry:
    ↓   if (user.invite_token_expires < Date.now())
    ↓     → Return 400 Token Expired
    ↓
    ↓ Get tenant info:
    ↓   const tenant = await getTenantById(user.tenant_id);
    ↓
    ← Return 200 {
    ←   valid: true,
    ←   email: user.email,
    ←   fullName: user.full_name,
    ←   clinicName: tenant.clinic_name
    ← }

[Frontend]
    ↓ Token valid → Show activation form:
    ↓   - Welcome message: "Hola, Dr. {fullName}"
    ↓   - Clinic: {clinicName}
    ↓   - Password field (with strength indicator)
    ↓   - Confirm password field
    ↓   - Checkbox: Accept Terms & Privacy Policy (link)
    ↓   - Button: "Activate Account"

[Doctor fills form and submits]
    ↓ POST /api/users/accept-invite
    ↓ { token, password, confirmPassword, acceptedTerms: true }

[Next.js API Route → API Gateway → Lambda: AcceptInviteFunction]
    ↓ Verify JWT + invite_token (same validation as above)
    ↓
    ↓ Validate password:
    ↓   - Min 8 chars
    ↓   - 1 uppercase
    ↓   - 1 lowercase
    ↓   - 1 number
    ↓   if invalid → Return 400 Weak Password
    ↓
    ↓ Verify password === confirmPassword:
    ↓   if not match → Return 400 Passwords Don't Match
    ↓
    ↓ Verify acceptedTerms === true:
    ↓   if false → Return 400 Must Accept Terms
    ↓
    ↓ Hash password with Argon2id:
    ↓   const passwordHash = await argon2.hash(password, {
    ↓     type: argon2.argon2id,
    ↓     memoryCost: 65536,
    ↓     timeCost: 3,
    ↓     parallelism: 4
    ↓   });
    ↓
    ↓ Update Users_Table:
    ↓   {
    ↓     password_hash: passwordHash,
    ↓     status: "incomplete", // From "invited"
    ↓     invite_token: null,    // Consumed
    ↓     invite_token_expires: null,
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Log audit (terms accepted):
    ↓   await logAudit({
    ↓     userId: user.user_id,
    ↓     tenantId: user.tenant_id,
    ↓     action: "doctor.terms_accepted",
    ↓     entityType: "user",
    ↓     entityId: user.user_id,
    ↓     ipAddress: event.requestContext.identity.sourceIp,
    ↓     metadata: { termsVersion: "v1.0" }
    ↓   });
    ↓
    ↓ Log audit (activated):
    ↓   await logAudit({
    ↓     userId: user.user_id,
    ↓     tenantId: user.tenant_id,
    ↓     action: "doctor.activated",
    ↓     entityType: "user",
    ↓     entityId: user.user_id,
    ↓     ipAddress: event.requestContext.identity.sourceIp
    ↓   });
    ↓
    ↓ Generate auth JWT (24h):
    ↓   const authToken = await new jose.SignJWT({
    ↓     userId: user.user_id,
    ↓     email: user.email,
    ↓     role: "doctor",
    ↓     tenantId: user.tenant_id
    ↓   })
    ↓     .setProtectedHeader({ alg: 'HS256' })
    ↓     .setExpirationTime('24h')
    ↓     .sign(secret);
    ↓
    ← Return 200 {
    ←   token: authToken,
    ←   user: { userId, email, fullName, role: "doctor", status: "incomplete", tenantId },
    ←   redirectUrl: "/clinical-profile"
    ← }

[Next.js API Route]
    ↓ Set HTTP-only cookie:
    ↓   cookies().set('authToken', authToken, {
    ↓     httpOnly: true,
    ↓     secure: true,
    ↓     sameSite: 'lax',
    ↓     maxAge: 60 * 60 * 24
    ↓   });
    ↓
    ← Return user data (without token)

[Frontend]
    ↓ Store user in UserContext
    ↓ Auto-login (no manual credentials needed)
    ↓ Show success message
    ↓ Redirect to /clinical-profile (onboarding mode)
    ↓ Status changes from 🟠 Invited → 🟡 Incomplete
```

---

### Flow 7: Clinical Profile Completion

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 7: Doctor Completes Clinical Profile                       │
└─────────────────────────────────────────────────────────────────┘

[Doctor lands on /clinical-profile (onboarding mode)]
    ↓ Status: 🟡 Incomplete
    ↓ Sidebar hidden (focused onboarding UI)
    ↓ Toast: "Account activated. Complete your professional profile."
    ↓ Profile opens in edit mode by default

[Doctor completes form]
    ↓ Personal Data Section:
    ↓   - Document type (DNI/CE/Passport)
    ↓   - Document number
    ↓   - Date of birth (age auto-calculated)
    ↓   - Gender
    ↓   - Address
    ↓   - Personal phone
    ↓   - Emergency contact phone
    ↓   - Digital signature (PNG upload → S3 presigned URL)
    ↓
    ↓ Professional Data Section:
    ↓   - Profession (Doctor, Nurse, Obstetrician, etc.)
    ↓   - CMP (required if profession = Doctor)
    ↓   - RNE (optional)
    ↓   - Primary specialty (dropdown)
    ↓   - Subspecialty (optional)
    ↓   - Primary work location (required)
    ↓   - Secondary work location (optional)

[Doctor clicks "Save"]
    ↓ POST /api/users/clinical-profile
    ↓ { userId, personalData: {...}, professionalData: {...} }
    ↓ Authorization: Bearer {doctorJWT} (from cookie)

[Next.js API Route → API Gateway → Lambda: UpdateClinicalProfileFunction]
    ↓ Verify doctor JWT:
    ↓   const { payload } = await jose.jwtVerify(token, secret);
    ↓   if (payload.role !== 'doctor') → Return 403 Forbidden
    ↓   if (payload.userId !== request.userId) → Return 403 Forbidden
    ↓
    ↓ Validate required fields:
    ↓   - If profession = "Doctor" → CMP required
    ↓   - Primary specialty required
    ↓   - Primary location required
    ↓
    ↓ Generate doctor_id:
    ↓   const doctorId = uuidv4();
    ↓
    ↓ Create Doctors_Table_V2 record:
    ↓   {
    ↓     doctor_id: doctorId,
    ↓     user_id: userId,          // NEW: FK to Users_Table
    ↓     email: payload.email,
    ↓     password: null,            // Not stored here anymore (in Users_Table)
    ↓     full_name: user.full_name,
    ↓     speciality: professionalData.primarySpecialty,
    ↓     id_number: personalData.documentNumber,
    ↓     phone_number: personalData.personalPhone,
    ↓     profile_picture_url: personalData.signatureUrl, // S3 URL
    ↓     active_for_ai_booking: false, // Not yet validated
    ↓     created_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Update Users_Table:
    ↓   {
    ↓     status: "pending",  // From "incomplete"
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId,
    ↓     tenantId: payload.tenantId,
    ↓     action: "doctor.profile_completed",
    ↓     entityType: "doctor",
    ↓     entityId: doctorId,
    ↓     ipAddress: event.requestContext.identity.sourceIp,
    ↓     metadata: { specialty: professionalData.primarySpecialty, cmp: professionalData.cmp }
    ↓   });
    ↓
    ← Return 200 {
    ←   success: true,
    ←   userId,
    ←   doctorId,
    ←   status: "pending",
    ←   message: "Clinical profile completed. Awaiting admin validation."
    ← }

[Frontend]
    ↓ Show success message
    ↓ Status changes from 🟡 Incomplete → 🟣 Pending
    ↓ Show banner: "Your profile is pending validation by the clinic administrator."
    ↓ Redirect to limited dashboard (cannot schedule appointments yet)
```

---

### Flow 8: Owner Validates Doctor

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 8: Owner Validates Doctor Profile                          │
└─────────────────────────────────────────────────────────────────┘

[Owner Dashboard: /users → View doctor with 🟣 Pending status]
    ↓ Click on doctor card
    ↓ Navigate to /users/{userId}

[Doctor Profile Page (Owner view)]
    ↓ Display all professional information:
    ↓   - Personal data (document, DOB, phone, etc.)
    ↓   - Professional data (CMP, specialty, location)
    ↓   - Digital signature preview
    ↓ Button: "Verify Profile" (enabled only for pending status)

[Owner clicks "Verify Profile"]
    ↓ Confirmation modal:
    ↓   "You are about to validate this professional's profile."
    ↓   "They will be able to receive appointments after validation."
    ↓   [Cancel] [Confirm Validation]
    ↓
    ↓ Owner clicks "Confirm Validation"
    ↓ POST /api/users/{userId}/validate
    ↓ { validated: true, notes: "Profile reviewed and approved" }
    ↓ Authorization: Bearer {ownerJWT}

[Next.js API Route → API Gateway → Lambda: ValidateDoctorFunction]
    ↓ Verify owner JWT:
    ↓   const { payload } = await jose.jwtVerify(token, secret);
    ↓   if (payload.role !== 'owner') → Return 403 Forbidden
    ↓
    ↓ Get user from Users_Table:
    ↓   const user = await getUserById(userId);
    ↓
    ↓ Verify user belongs to same tenant:
    ↓   if (user.tenant_id !== payload.tenantId) → Return 403 Forbidden
    ↓
    ↓ Verify user status is "pending":
    ↓   if (user.status !== 'pending') → Return 400 Invalid Status
    ↓
    ↓ Get doctor from Doctors_Table_V2 via user_id-index:
    ↓   const doctor = await getDoctorByUserId(userId);
    ↓
    ↓ Update Users_Table:
    ↓   {
    ↓     status: "active",  // From "pending"
    ↓     updated_at: new Date().toISOString()
    ↓   }
    ↓
    ↓ Update Doctors_Table_V2:
    ↓   {
    ↓     active_for_ai_booking: true  // From false
    ↓   }
    ↓
    ↓ Create PractitionerRole (FHIR) - Link: User ↔ Specialty ↔ Location:
    ↓   (This would be in a separate table in future, for now just note it)
    ↓
    ↓ Log audit:
    ↓   await logAudit({
    ↓     userId: payload.userId,      // Owner who validated
    ↓     tenantId: payload.tenantId,
    ↓     action: "doctor.validated",
    ↓     entityType: "doctor",
    ↓     entityId: doctor.doctor_id,
    ↓     ipAddress: event.requestContext.identity.sourceIp,
    ↓     metadata: {
    ↓       validatedDoctorUserId: userId,
    ↓       validatedBy: payload.email,
    ↓       notes: request.notes
    ↓     }
    ↓   });
    ↓
    ← Return 200 {
    ←   success: true,
    ←   userId,
    ←   doctorId: doctor.doctor_id,
    ←   previousStatus: "pending",
    ←   newStatus: "active",
    ←   activeForBooking: true
    ← }

[Frontend]
    ↓ Show success snackbar: "Dr. {fullName} has been validated and activated"
    ↓ Status changes from 🟣 Pending → 🟢 Active
    ↓ Doctor can now receive appointments
    ↓ Doctor appears in "Available Doctors" list
```

---

## Frontend Components

### File Structure: vitas-client

```
src/app/
├── (auth)/
│   ├── register/
│   │   └── page.tsx                      # Owner registration form
│   ├── verify-email/
│   │   └── page.tsx                      # Email verification page
│   ├── activate/
│   │   └── page.tsx                      # Doctor activation page
│   ├── signin/
│   │   └── page.tsx                      # Existing login (updated for owners)
│   └── check-email/
│       └── page.tsx                      # "Check your email" info page
│
├── (dashboard)/
│   ├── onboarding/
│   │   └── page.tsx                      # Owner onboarding (accept T&C)
│   ├── users/
│   │   ├── page.tsx                      # User management list
│   │   ├── components/
│   │   │   ├── UserListTable.tsx         # Table with filters
│   │   │   ├── InviteUserModal.tsx       # Invite doctor modal
│   │   │   └── UserStatusBadge.tsx       # Status indicator (🟠🟡🟣🟢🔴)
│   │   └── [userId]/
│   │       └── page.tsx                  # User detail + validate button
│   └── clinical-profile/
│       ├── page.tsx                      # Clinical profile (onboarding mode)
│       └── components/
│           ├── PersonalDataForm.tsx      # Personal data section
│           └── ProfessionalDataForm.tsx  # Professional data section
│
├── api/
│   ├── auth/
│   │   ├── register-owner/
│   │   │   └── route.ts                  # POST /api/auth/register-owner
│   │   ├── verify-email/
│   │   │   └── route.ts                  # GET /api/auth/verify-email
│   │   └── signin-owner/
│   │       └── route.ts                  # POST /api/auth/signin-owner
│   ├── legal/
│   │   └── accept-terms/
│   │       └── route.ts                  # POST /api/legal/accept-terms
│   └── users/
│       ├── invite-doctor/
│       │   └── route.ts                  # POST /api/users/invite-doctor
│       ├── validate-invite-token/
│       │   └── route.ts                  # GET /api/users/validate-invite-token
│       ├── accept-invite/
│       │   └── route.ts                  # POST /api/users/accept-invite
│       ├── clinical-profile/
│       │   └── route.ts                  # POST /api/users/clinical-profile
│       ├── route.ts                      # GET /api/users (list with filters)
│       └── [userId]/
│           ├── route.ts                  # GET /api/users/:userId
│           └── validate/
│               └── route.ts              # POST /api/users/:userId/validate
│
└── lib/
    ├── api-connectors/
    │   ├── auth.ts                       # Auth API calls
    │   └── users.ts                      # User management API calls
    └── utils/
        ├── password-strength.ts          # Password validation
        └── validators.ts                 # Email, phone validators
```

---

## Backend Lambda Functions

### File Structure: vitas-main-stack

```
vitas-main-stack/
├── lib/
│   └── vitas-main-stack.ts               # CDK stack definition
│       ├── DynamoDB tables (Users, Tenants, Audit_Logs)
│       ├── Lambda functions
│       ├── API Gateway routes
│       └── IAM permissions
│
└── lambda/
    └── vitas-auth/
        ├── package.json                  # Dependencies: argon2, jose, uuid, mailgun-js
        └── src/
            ├── auth/
            │   ├── register-owner.ts     # POST /auth/register-owner
            │   ├── verify-email.ts       # GET /auth/verify-email
            │   ├── signin-owner.ts       # POST /auth/signin-owner
            │   └── accept-terms.ts       # POST /legal/accept-terms
            │
            ├── users/
            │   ├── invite-doctor.ts      # POST /users/invite-doctor
            │   ├── validate-invite-token.ts  # GET /users/validate-invite-token
            │   ├── accept-invite.ts      # POST /users/accept-invite
            │   ├── resend-invite.ts      # POST /users/resend-invite
            │   ├── list-users.ts         # GET /users
            │   ├── get-user.ts           # GET /users/:userId
            │   ├── update-user-status.ts # PATCH /users/:userId/status
            │   ├── update-clinical-profile.ts  # POST /users/clinical-profile
            │   └── validate-doctor.ts    # POST /users/:userId/validate
            │
            ├── services/
            │   ├── mailgun.service.ts    # Email sending via Mailgun
            │   ├── jwt.service.ts        # JWT generation/validation
            │   ├── password.service.ts   # Argon2id hashing
            │   ├── audit.service.ts      # Audit logging
            │   └── dynamodb.service.ts   # DynamoDB helpers
            │
            └── utils/
                ├── validators.ts         # Input validation
                ├── errors.ts             # Custom error classes
                └── constants.ts          # Constants (statuses, roles, etc.)
```

---

## Email Templates

### Mailgun Configuration

**Environment Variables:**
```bash
MAILGUN_API_KEY=your-mailgun-api-key
MAILGUN_DOMAIN=mg.yourdomain.com
MAILGUN_FROM_EMAIL=noreply@vitas.com
MAILGUN_FROM_NAME=VITAS Clinic
```

### Template 1: Owner Email Verification

**File:** `lambda/vitas-auth/src/templates/owner-verification.html`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
    .header { background: #4A90E2; color: white; padding: 20px; text-align: center; }
    .content { padding: 30px; background: #f9f9f9; }
    .button { display: inline-block; padding: 12px 24px; background: #4A90E2; color: white; text-decoration: none; border-radius: 4px; margin: 20px 0; }
    .footer { text-align: center; padding: 20px; font-size: 12px; color: #666; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>VITAS Clinic</h1>
    </div>
    <div class="content">
      <h2>Hola {{fullName}},</h2>
      <p>Gracias por registrarte en VITAS.</p>
      <p>Por favor, verifica tu correo electrónico haciendo clic en el siguiente enlace:</p>
      <p style="text-align: center;">
        <a href="{{verificationLink}}" class="button">Verificar Email</a>
      </p>
      <p><strong>Este enlace expira en 24 horas.</strong></p>
      <p>Si no creaste esta cuenta, puedes ignorar este correo.</p>
    </div>
    <div class="footer">
      <p>© 2025 VITAS Clinic. Todos los derechos reservados.</p>
      <p>Este correo cumple con la Ley 29733 de Protección de Datos Personales.</p>
    </div>
  </div>
</body>
</html>
```

---

### Template 2: Doctor Invitation

**File:** `lambda/vitas-auth/src/templates/doctor-invitation.html`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
    .header { background: #50C878; color: white; padding: 20px; text-align: center; }
    .content { padding: 30px; background: #f9f9f9; }
    .button { display: inline-block; padding: 12px 24px; background: #50C878; color: white; text-decoration: none; border-radius: 4px; margin: 20px 0; }
    .warning { background: #FFF3CD; border-left: 4px solid #FFC107; padding: 10px; margin: 20px 0; }
    .footer { text-align: center; padding: 20px; font-size: 12px; color: #666; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>Invitación a VITAS</h1>
    </div>
    <div class="content">
      <h2>Hola, Dr. {{fullName}},</h2>
      <p>Has sido invitado por <strong>{{clinicName}}</strong> para unirte a VITAS como profesional médico.</p>
      <p>Para activar tu cuenta, haz clic en el siguiente enlace:</p>
      <p style="text-align: center;">
        <a href="{{activationLink}}" class="button">Activar Cuenta</a>
      </p>
      <div class="warning">
        <strong>Importante:</strong>
        <ul>
          <li>Este enlace expira en {{expiresIn}}</li>
          <li>No compartas este correo o enlace con nadie</li>
          <li>Al activar, deberás crear una contraseña segura</li>
        </ul>
      </div>
      <p>Si no esperabas esta invitación, puedes ignorar este correo.</p>
    </div>
    <div class="footer">
      <p>© 2025 VITAS Clinic. Todos los derechos reservados.</p>
      <p>Por seguridad, este mensaje cumple con la Ley 29733 de Protección de Datos Personales.</p>
    </div>
  </div>
</body>
</html>
```

---

### Mailgun Service Implementation

**File:** `lambda/vitas-auth/src/services/mailgun.service.ts`

```typescript
import formData from 'form-data';
import Mailgun from 'mailgun.js';

const mailgun = new Mailgun(formData);
const mg = mailgun.client({
  username: 'api',
  key: process.env.MAILGUN_API_KEY!,
  url: 'https://api.mailgun.net'
});

interface EmailTemplate {
  subject: string;
  html: string;
}

function renderTemplate(templateName: string, data: any): EmailTemplate {
  // Load HTML template and replace variables
  const templates: Record<string, (data: any) => EmailTemplate> = {
    'owner-verification': (data) => ({
      subject: 'Verifica tu correo - VITAS Clinic',
      html: ownerVerificationTemplate(data)
    }),
    'doctor-invitation': (data) => ({
      subject: `Invitación a VITAS - ${data.clinicName}`,
      html: doctorInvitationTemplate(data)
    })
  };

  return templates[templateName](data);
}

export async function sendEmail(
  to: string,
  templateName: string,
  templateData: any
): Promise<void> {
  const { subject, html } = renderTemplate(templateName, templateData);

  await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
    from: `${process.env.MAILGUN_FROM_NAME} <${process.env.MAILGUN_FROM_EMAIL}>`,
    to: [to],
    subject,
    html
  });
}
```

---

## Security Implementation

### 1. Password Hashing (Argon2id)

**File:** `lambda/vitas-auth/src/services/password.service.ts`

```typescript
import argon2 from 'argon2';

export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536,      // 64 MB
    timeCost: 3,            // 3 iterations
    parallelism: 4          // 4 parallel threads
  });
}

export async function verifyPassword(
  hash: string,
  password: string
): Promise<boolean> {
  try {
    return await argon2.verify(hash, password);
  } catch (error) {
    return false;
  }
}

export function validatePasswordStrength(password: string): {
  valid: boolean;
  errors: string[];
} {
  const errors: string[] = [];

  if (password.length < 8) {
    errors.push('Password must be at least 8 characters');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least 1 uppercase letter');
  }

  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain at least 1 lowercase letter');
  }

  if (!/[0-9]/.test(password)) {
    errors.push('Password must contain at least 1 number');
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

---

### 2. JWT Service

**File:** `lambda/vitas-auth/src/services/jwt.service.ts`

```typescript
import * as jose from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET);

export interface AuthTokenPayload {
  userId: string;
  email: string;
  role: 'owner' | 'doctor' | 'patient';
  tenantId: string;
}

export interface InviteTokenPayload extends AuthTokenPayload {
  type: 'invite';
}

export interface VerificationTokenPayload {
  userId: string;
  email: string;
  type: 'email_verification';
}

export async function generateAuthToken(
  payload: AuthTokenPayload
): Promise<string> {
  return await new jose.SignJWT(payload as any)
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .setIssuedAt()
    .sign(JWT_SECRET);
}

export async function generateInviteToken(
  payload: InviteTokenPayload
): Promise<string> {
  return await new jose.SignJWT(payload as any)
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .setIssuedAt()
    .sign(JWT_SECRET);
}

export async function generateVerificationToken(
  payload: VerificationTokenPayload
): Promise<string> {
  return await new jose.SignJWT(payload as any)
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .setIssuedAt()
    .sign(JWT_SECRET);
}

export async function verifyToken<T = any>(token: string): Promise<T> {
  try {
    const { payload } = await jose.jwtVerify(token, JWT_SECRET);
    return payload as T;
  } catch (error) {
    throw new Error('Invalid or expired token');
  }
}
```

---

### 3. Audit Logging Service

**File:** `lambda/vitas-auth/src/services/audit.service.ts`

```typescript
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { marshall } from '@aws-sdk/util-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const dynamodb = new DynamoDBClient({ region: process.env.AWS_REGION });

export interface AuditLogEntry {
  userId: string;
  tenantId: string;
  action: string;
  entityType: string;
  entityId: string;
  ipAddress: string;
  userAgent?: string;
  metadata?: any;
}

export async function logAudit(entry: AuditLogEntry): Promise<void> {
  const timestamp = new Date().toISOString();

  await dynamodb.send(
    new PutItemCommand({
      TableName: process.env.AUDIT_LOGS_TABLE_NAME,
      Item: marshall({
        log_id: uuidv4(),
        timestamp,
        ...entry,
        created_at: timestamp
      })
    })
  );
}
```

---

### 4. Input Validators

**File:** `lambda/vitas-auth/src/utils/validators.ts`

```typescript
export function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

export function isValidPassword(password: string): boolean {
  return (
    password.length >= 8 &&
    /[A-Z]/.test(password) &&
    /[a-z]/.test(password) &&
    /[0-9]/.test(password)
  );
}

export function isValidFullName(name: string): boolean {
  return name.length >= 1 && name.length <= 100;
}

export function sanitizeInput(input: string): string {
  return input.trim().replace(/[<>]/g, '');
}
```

---

## Testing Checklist

### Owner Registration Flow
- [ ] Register with valid email/password
- [ ] Error: Email already exists (409)
- [ ] Error: Invalid email format (400)
- [ ] Error: Weak password (400)
- [ ] Receive verification email (Mailgun)
- [ ] Click verification link (valid token)
- [ ] Error: Expired verification link (400)
- [ ] Error: Already verified (409)
- [ ] Complete onboarding + accept T&C
- [ ] User status: draft → incomplete → active
- [ ] Tenant status: draft → incomplete → active
- [ ] Audit logs created for each step

### Doctor Invitation Flow
- [ ] Owner invites doctor (valid email)
- [ ] Error: Email already exists (409)
- [ ] Error: Unauthorized (non-owner) (403)
- [ ] Receive invitation email (Mailgun)
- [ ] Invitation shows 🟠 Invited status in user list
- [ ] Validate invite token (GET request)
- [ ] Error: Invalid token (401)
- [ ] Error: Expired token (400)
- [ ] Activate account with valid password
- [ ] Error: Weak password (400)
- [ ] Error: Passwords don't match (400)
- [ ] Error: Terms not accepted (400)
- [ ] Auto-login after activation
- [ ] User status: invited → incomplete
- [ ] Audit logs for invitation + activation

### Clinical Profile Completion
- [ ] Complete personal data section
- [ ] Complete professional data section
- [ ] Error: Missing CMP (if profession = Doctor) (400)
- [ ] Upload digital signature (S3 presigned URL)
- [ ] Save profile successfully
- [ ] User status: incomplete → pending
- [ ] Doctor record created in Doctors_Table_V2
- [ ] Link: user_id → doctor_id
- [ ] Audit log for profile completion

### Owner Validation
- [ ] Owner views pending doctor profile
- [ ] Click "Verify Profile" button
- [ ] Confirmation modal displayed
- [ ] Validate doctor successfully
- [ ] User status: pending → active
- [ ] active_for_ai_booking: false → true
- [ ] Doctor appears in "Available Doctors" list
- [ ] Audit log with owner IP + timestamp
- [ ] Error: Unauthorized (non-owner) (403)

### Security Tests
- [ ] Password hashed with Argon2id
- [ ] JWT tokens expire after 24h
- [ ] HTTP-only cookies set correctly
- [ ] Invite tokens single-use (consumed after activation)
- [ ] CORS headers correct
- [ ] Rate limiting (if implemented)
- [ ] SQL injection prevention (DynamoDB safe)
- [ ] XSS prevention (input sanitization)

### Email Tests (Mailgun)
- [ ] Owner verification email sent
- [ ] Doctor invitation email sent
- [ ] Email templates render correctly
- [ ] Links in emails are correct
- [ ] Expiration times displayed correctly
- [ ] Legal footer present (Law 29733)

### Audit Trail Tests
- [ ] All actions logged (owner.*, doctor.*)
- [ ] IP addresses captured
- [ ] Timestamps in ISO format
- [ ] Metadata includes relevant context
- [ ] WORM storage (records immutable)
- [ ] Query by userId
- [ ] Query by tenantId
- [ ] Query by action type

---

## Implementation Phases

### Phase 1: Database Setup (1 day)
- Create DynamoDB tables (Users, Tenants, Audit_Logs)
- Create GSI indexes
- Add user_id to Doctors_Table_V2
- Test table creation with CDK deploy

### Phase 2: Backend Lambda Functions (3-4 days)
- Implement all 13 Lambda functions
- Set up JWT service (jose)
- Set up password service (argon2)
- Set up Mailgun service
- Set up audit logging service
- Test each Lambda locally

### Phase 3: API Gateway Routes (1 day)
- Configure all 13 API endpoints
- Set up CORS
- Test with Postman/curl

### Phase 4: Frontend Components (3-4 days)
- Owner registration page
- Email verification page
- Doctor activation page
- User management page
- Clinical profile page
- Invite user modal
- Test UI flows end-to-end

### Phase 5: Integration Testing (2 days)
- Test complete owner registration flow
- Test complete doctor invitation flow
- Test clinical profile completion
- Test owner validation
- Test error scenarios
- Test email delivery

### Phase 6: Security Audit (1 day)
- Review password hashing
- Review JWT implementation
- Review input validation
- Review audit logging
- Test rate limiting (if implemented)

**Total Estimated Time:** 11-13 days

---

## Environment Variables Required

### Backend (Lambda)
```bash
# AWS
AWS_REGION=sa-east-1

# DynamoDB Tables
USERS_TABLE_NAME=Users_Table
TENANTS_TABLE_NAME=Tenants_Table
AUDIT_LOGS_TABLE_NAME=Audit_Logs_Table
DOCTORS_TABLE_NAME=Doctors_Table_V2

# JWT
JWT_SECRET=your-super-secret-key-min-32-chars

# Mailgun
MAILGUN_API_KEY=your-mailgun-api-key
MAILGUN_DOMAIN=mg.yourdomain.com
MAILGUN_FROM_EMAIL=noreply@vitas.com
MAILGUN_FROM_NAME=VITAS Clinic

# Frontend URL (for email links)
FRONTEND_URL=https://app.vitas.com
```

### Frontend (Next.js)
```bash
# API Gateway URL
NEXT_PUBLIC_API_BASE_URL=https://api.vitas.com

# Environment
NODE_ENV=production
```

---

## Next Steps After Implementation

1. **Phase 2 Features:**
   - MFA setup (TOTP)
   - Audit history export (CSV)
   - Session management (view/revoke)
   - Password reset flow
   - Resend invitation email

2. **Multi-Location Support:**
   - Create Locations table
   - Link PractitionerRole to Location
   - Update UI for location selection

3. **Advanced User Roles:**
   - Secretary role
   - Location Admin role
   - Auditor role
   - Role-based permissions matrix

4. **Legal Documents Module:**
   - Create legal_documents table
   - Document versioning
   - Consent tracking
   - WORM storage for signatures

---

**Last Updated:** 2025-11-09
**Version:** 1.0
**Status:** Ready for Implementation

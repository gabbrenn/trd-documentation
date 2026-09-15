# Frontend API Guide

This is the frontend contract for the TRD LF API. The frontend communicates **only** with the API Gateway. Never call internal service ports directly.

---

## Base Request

All operations use a single endpoint:

```
POST http://localhost:3000/api
```

### Required headers

| Header | Required | Description |
| --- | --- | --- |
| `X-Server-Key` | Yes | Gateway application key for your environment. |
| `X-Command` | Yes | One command ID from the catalog below. |
| `Content-Type` | JSON commands | `application/json` |
| `Content-Type` | File upload commands | Set automatically when using `FormData`; **do not set manually**. |
| `Cookie: sessionId=...` | Protected commands | Set automatically by the browser after login. |

> **Never** send `AUTH_INTERNAL_API_KEY`, internal service URLs, or internal service ports from frontend code.

### JSON request example

```ts
// Standard JSON command
const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',          // required — sends the session cookie
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'LOGIN_AUTH_3B4',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ email, password }),
});
```

### Multipart / file upload request example

For any command with `requestType: multipart`, send a `FormData` body. **Do NOT set `Content-Type` manually** — the browser must set it (including the boundary string).

```ts
// File upload command
const form = new FormData();
form.append('file', selectedFile);           // the File object
form.append('documentType', 'LICENSE');      // any additional text fields

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'DOC_IMG_D1B',
    // ⚠️  Do NOT add Content-Type here — let the browser do it
  },
  body: form,
});
```

---

## Registration

Command: `REGISTER_AUTH_1A2` — no session required.

Registration is always **JSON only**. Profile images and documents are uploaded **after** registration using dedicated upload commands.

`accountType` defaults to `PATIENT` when omitted.

**Allowed public account types:**

- `PATIENT`
- `DOCTOR`
- `PHARMACY_ADMIN`
- `CLINIC_ADMIN`

`ADMIN` and `INSPECTOR` are controlled roles and are rejected by public registration.

### Common fields

| Field | Required | Description |
| --- | --- | --- |
| `accountType` | No | Public role. Defaults to `PATIENT`. |
| `name` | Yes | Display name, max 150 characters. |
| `email` | Yes | Valid email address. |
| `phone` | Yes | Phone number, max 30 characters. |
| `password` | Yes | At least 8 characters. |
| `doctor` | Doctor only | Doctor profile object (see below). |
| `facility` | Pharmacy / Clinic only | Facility object (see below). |

### Patient registration

```json
{
  "accountType": "PATIENT",
  "name": "Example Patient",
  "email": "patient@example.com",
  "phone": "+15550000001",
  "password": "ExamplePassword123!"
}
```

Patient accounts are immediately active.

```json
{
  "success": true,
  "statusCode": 201,
  "message": "Registration successful",
  "data": {
    "user": {
      "id": "00000000-0000-4000-8000-000000000080",
      "name": "Example Patient",
      "email": "patient@example.com",
      "phone": "+15550000001",
      "roleId": "00000000-0000-4000-8000-000000000081",
      "role": "PATIENT",
      "status": "active",
      "accountStatus": "ACTIVE"
    }
  }
}
```

### Doctor registration

Doctor fields inside `doctor`:

| Field | Required for completion | Description |
| --- | --- | --- |
| `licenseNumber` | Yes | Professional license number. |
| `specialty` | Yes | Professional specialty. |
| `facilityId` | No | Existing facility UUID, when applicable. |

```json
{
  "accountType": "DOCTOR",
  "name": "Dr. Example",
  "email": "doctor@example.com",
  "phone": "+15550000002",
  "password": "ExamplePassword123!",
  "doctor": {
    "licenseNumber": "MD-12345",
    "specialty": "Cardiology",
    "facilityId": "00000000-0000-4000-8000-000000000010"
  }
}
```

The user is created with `status: "pending_verification"`. To add a profile photo or upload license documents, use the dedicated upload commands **after** login (see [Doctor File Uploads](#doctor-file-uploads)).

If `doctor` is omitted, the user is created with `status: "incomplete"` and no profile is created.

### Pharmacy admin registration

Facility fields inside `facility`:

| Field | Required | Description |
| --- | --- | --- |
| `name` | Yes | Pharmacy name. |
| `licenseNumber` | Yes | Pharmacy license number. |
| `address` | Yes | Pharmacy address. |
| `pharmacyEnabled` | No | Defaults to `true` for pharmacy admins. |

```json
{
  "accountType": "PHARMACY_ADMIN",
  "name": "Example Pharmacy Admin",
  "email": "pharmacy-admin@example.com",
  "phone": "+15550000003",
  "password": "ExamplePassword123!",
  "facility": {
    "name": "Downtown Pharmacy",
    "licenseNumber": "PH-987654",
    "address": "145 Main Street",
    "pharmacyEnabled": true
  }
}
```

Response (role-specific portion):

```json
{
  "user": {
    "id": "00000000-0000-4000-8000-000000000080",
    "role": "PHARMACY_ADMIN",
    "status": "pending_verification",
    "accountStatus": "PENDING_VERIFICATION"
  },
  "facility": {
    "id": "00000000-0000-4000-8000-000000000010",
    "ownerUserId": "00000000-0000-4000-8000-000000000080",
    "name": "Downtown Pharmacy",
    "facilityType": "PHARMACY",
    "pharmacyEnabled": true,
    "licenseNumber": "PH-987654",
    "address": "145 Main Street",
    "verificationStatus": "pending"
  }
}
```

If `facility` is omitted, the user is created with `status: "incomplete"` and no facility is created.

To upload facility license documents after registration, use `FAC_DOC_D3E` (see [Facility File Uploads](#facility-file-uploads)).

### Clinic admin registration

```json
{
  "accountType": "CLINIC_ADMIN",
  "name": "Example Clinic Admin",
  "email": "clinic-admin@example.com",
  "phone": "+15550000004",
  "password": "ExamplePassword123!",
  "facility": {
    "name": "Downtown Medical Clinic",
    "licenseNumber": "CL-123456",
    "address": "20 Health Avenue",
    "pharmacyEnabled": false
  }
}
```

Regulatory creates the facility with `facilityType: "CLINIC"`. `pharmacyEnabled` defaults to `false` when omitted.

---

## Login

Command: `LOGIN_AUTH_3B4` — no session required.

```json
{
  "email": "pharmacy-admin@example.com",
  "password": "ExamplePassword123!"
}
```

A successful login sets an HTTP-only `sessionId` cookie. Never store the session ID in `localStorage`.

**Patient / Doctor response:**

```json
{
  "authenticated": true,
  "accountStatus": "ACTIVE",
  "role": "PATIENT",
  "requiresRegistrationCompletion": false,
  "message": "Login successful",
  "data": {
    "user": {
      "id": "....",
      "name": "Example Patient",
      "email": "patient@example.com",
      "phone": "+15550000001",
      "roleId": "......",
      "status": "active",
      "accountStatus": "ACTIVE",
      "role": "PATIENT"
    }
  }
}
```

**Pharmacy / Clinic admin response** (includes `facility`):

```json
{
  "authenticated": true,
  "accountStatus": "PENDING_VERIFICATION",
  "role": "PHARMACY_ADMIN",
  "requiresRegistrationCompletion": false,
  "message": "Login successful",
  "data": {
    "user": {
      "id": "....",
      "name": "Example Pharmacy Admin",
      "email": "pharmacy-admin@example.com",
      "phone": "+15550000003",
      "roleId": "......",
      "status": "pending_verification",
      "accountStatus": "PENDING_VERIFICATION",
      "role": "PHARMACY_ADMIN",
      "facility": {
        "id": ".....",
        "ownerUserId": "......",
        "name": "Downtown Pharmacy",
        "facilityType": "PHARMACY",
        "pharmacyEnabled": true,
        "licenseNumber": "PH-987654",
        "address": "145 Main Street",
        "verificationStatus": "pending",
        "createdAt": "2026-09-10T15:31:05.483Z",
        "updatedAt": "2026-09-10T15:31:05.483Z"
      }
    }
  }
}
```

- `accountStatus: "INCOMPLETE"` → `requiresRegistrationCompletion: true`. Show completion screen.
- `accountStatus: "PENDING_VERIFICATION"` → Show a verification-pending screen. The account can authenticate but must not access active-only business features.

---

## Complete Registration

Command: `COMPLETE_REGISTRATION_AUTH_7M3` — session required.

The authenticated user comes from the session cookie. Never send `userId` or role in the body.

### Complete a doctor account

```json
{
  "doctor": {
    "licenseNumber": "MD-12345",
    "specialty": "Cardiology",
    "facilityId": "00000000-0000-4000-8000-000000000010"
  }
}
```

Response:

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Doctor registration completed successfully",
  "data": { "accountStatus": "pending_verification", "role": "DOCTOR" }
}
```

### Complete a pharmacy or clinic account

```json
{
  "facility": {
    "name": "Downtown Pharmacy",
    "licenseNumber": "PH-987654",
    "address": "145 Main Street",
    "pharmacyEnabled": true
  }
}
```

Use `pharmacyEnabled: false` for a clinic. The operation is idempotent — an existing facility owned by the authenticated user is updated instead of duplicated.

### Complete a patient account

Send only genuinely missing fields:

```json
{
  "name": "Updated Patient Name",
  "phone": "+15550000005",
  "password": "NewPassword123!"
}
```

---

## Logout

Command: `LOGOUT_AUTH_5C6` — session required.

```json
{}
```

Response:

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Logout successful",
  "data": null
}
```

---

## Doctor File Uploads

All doctor upload commands require a **valid session** and use `multipart/form-data`. Do not set `Content-Type` manually.

---

### Upload Doctor Profile Image

Command: `DOC_IMG_D1B`

Uploads or replaces the doctor's profile photo. Accepted types: `image/jpeg`, `image/png`, `image/webp`.

**FormData fields:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | File | Yes | Image file. Max 10 MB. |

**JavaScript example:**

```ts
const form = new FormData();
form.append('file', imageFile);

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'DOC_IMG_D1B',
  },
  body: form,
});
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Doctor profile image uploaded successfully",
  "data": {
    "profileImageUrl": "https://res.cloudinary.com/your-cloud/image/upload/v1234567890/doctors/abc123.jpg"
  }
}
```

---

### Upload Doctor Document

Command: `DCT_DOC_D4F`

Upload a professional document (e.g., medical license, certificate). Accepted types: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`.

**FormData fields:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | File | Yes | Document file. Max 10 MB. |
| `documentType` | string | Yes | Type of document. See allowed values below. |

**Allowed `documentType` values:**

| Value | Meaning |
| --- | --- |
| `LICENSE` | Medical / professional license |
| `CERTIFICATE` | Specialty certificate |
| `IDENTIFICATION` | National ID or passport |
| `OTHER` | Any other supporting document |

**JavaScript example:**

```ts
const form = new FormData();
form.append('file', pdfFile);
form.append('documentType', 'LICENSE');

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'DCT_DOC_D4F',
  },
  body: form,
});
```

**Success response (201):**

```json
{
  "success": true,
  "statusCode": 201,
  "message": "Document uploaded successfully",
  "data": {
    "document": {
      "id": "00000000-0000-4000-8000-000000000030",
      "doctorProfileId": "00000000-0000-4000-8000-000000000031",
      "documentType": "LICENSE",
      "documentUrl": "https://res.cloudinary.com/your-cloud/raw/upload/v1234567890/doctor-documents/abc123.pdf",
      "status": "pending",
      "uploadedAt": "2026-09-15T10:00:00.000Z"
    }
  }
}
```

The document starts in `status: "pending"`. An inspector must review it before the account is verified.

---

### Get Pending Doctor Documents *(Inspector only)*

Command: `DOC_LST_D5G` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

```json
{}
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Pending doctor documents retrieved",
  "data": {
    "documents": [
      {
        "id": "00000000-0000-4000-8000-000000000030",
        "doctorProfileId": "00000000-0000-4000-8000-000000000031",
        "documentType": "LICENSE",
        "documentUrl": "https://res.cloudinary.com/...",
        "status": "pending",
        "uploadedAt": "2026-09-15T10:00:00.000Z"
      }
    ]
  }
}
```

---

### Review Doctor Document *(Inspector only)*

Command: `DOC_REV_D6H` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

```json
{
  "documentId": "00000000-0000-4000-8000-000000000030",
  "decision": "approved",
  "notes": "License verified."
}
```

**`decision` values:** `approved` | `rejected`

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Document reviewed successfully",
  "data": {
    "document": {
      "id": "00000000-0000-4000-8000-000000000030",
      "status": "approved",
      "reviewedAt": "2026-09-15T11:00:00.000Z",
      "reviewNotes": "License verified."
    }
  }
}
```

---

### Get Pending Doctor Accounts *(Inspector only)*

Command: `DOC_ACT_LST_D7J` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

Retrieves all doctor accounts currently in `pending_verification` status or awaiting verification review, along with their doctor profile and uploaded documents.

```json
{}
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Pending doctor accounts retrieved successfully",
  "data": {
    "doctors": [
      {
        "id": "00000000-0000-4000-8000-000000000031",
        "userId": "00000000-0000-4000-8000-000000000030",
        "licenseNumber": "DOC-12345",
        "specialty": "General Medicine",
        "facilityId": null,
        "imageUrl": "https://res.cloudinary.com/...",
        "verificationStatus": "pending",
        "createdAt": "2026-09-15T10:00:00.000Z",
        "user": {
          "id": "00000000-0000-4000-8000-000000000030",
          "name": "Dr. Jane Doe",
          "email": "doctor@test.com",
          "phone": "+254700000002",
          "status": "pending_verification",
          "createdAt": "2026-09-15T10:00:00.000Z"
        },
        "documents": [
          {
            "id": "00000000-0000-4000-8000-000000000032",
            "documentType": "LICENSE",
            "fileUrl": "https://res.cloudinary.com/...",
            "originalName": "medical-license.pdf",
            "mimeType": "application/pdf",
            "fileSize": 102400,
            "verificationStatus": "pending",
            "inspectorNotes": null,
            "uploadedAt": "2026-09-15T10:05:00.000Z",
            "reviewedAt": null
          }
        ]
      }
    ],
    "total": 1
  }
}
```

---

### Get Doctor Profile

Command: `DOC_PRF_G9M` — requires a valid session.

Retrieves a doctor's full profile, user details, and uploaded documents.
If called without parameters by a logged-in doctor, returns their own profile. Inspectors/Admins can pass `doctorId` or `userId`.

```json
{
  "doctorId": "00000000-0000-4000-8000-000000000031"
}
```

Or by `userId`:

```json
{
  "userId": "00000000-0000-4000-8000-000000000030"
}
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Doctor profile retrieved successfully",
  "data": {
    "doctor": {
      "id": "00000000-0000-4000-8000-000000000031",
      "userId": "00000000-0000-4000-8000-000000000030",
      "licenseNumber": "DOC-12345",
      "specialty": "General Medicine",
      "imageUrl": "https://res.cloudinary.com/...",
      "verificationStatus": "pending",
      "user": {
        "id": "00000000-0000-4000-8000-000000000030",
        "name": "Dr. Jane Doe",
        "email": "doctor@test.com",
        "phone": "+254700000002",
        "status": "pending_verification"
      },
      "documents": []
    }
  }
}
```

---

### Review Doctor Account *(Inspector only)*

Command: `DOC_ACT_REV_D8K` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

Approve or reject a doctor's account. When approved, the doctor's user status becomes `active` and their profile verification status becomes `approved`. When rejected, their account status becomes `rejected` and their profile status becomes `rejected`.

```json
{
  "doctorId": "00000000-0000-4000-8000-000000000031",
  "decision": "approved",
  "reason": "Credentials and medical board license verified."
}
```

Or specify `userId`:

```json
{
  "userId": "00000000-0000-4000-8000-000000000030",
  "decision": "approved"
}
```

**`decision` values:** `approved` | `rejected`

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Doctor account approved successfully",
  "data": {
    "doctor": {
      "id": "00000000-0000-4000-8000-000000000031",
      "userId": "00000000-0000-4000-8000-000000000030",
      "verificationStatus": "approved"
    },
    "user": {
      "id": "00000000-0000-4000-8000-000000000030",
      "name": "Dr. Jane Doe",
      "email": "doctor@test.com",
      "status": "active",
      "accountStatus": "Active"
    }
  }
}
```

---

## Facility File Uploads

All facility upload commands require a **valid session** and use `multipart/form-data`.

---

### Upload Facility Profile Image

Command: `FAC_IMG_C3A`

> **Important:** Registration (`REGISTER_AUTH_1A2`) is JSON-only and cannot include an image. This separate command is used **after login** to attach a profile photo to an existing facility.
>
> This command is **idempotent** — if a facility already exists for the authenticated owner and `facilityType`, it is **updated** with the new image rather than creating a duplicate.

Accepted image types: `image/jpeg`, `image/png`, `image/webp`.

**FormData fields:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | File | Yes | Profile photo. Max 10 MB. |
| `name` | string | Yes | Facility name (same value used at registration). |
| `facilityType` | string | Yes | `PHARMACY` or `CLINIC`. |
| `licenseNumber` | string | Yes | License number (same value used at registration). |
| `address` | string | No | Facility address. |
| `pharmacyEnabled` | string | No | `true` or `false`. |

**JavaScript example:**

```ts
const form = new FormData();
form.append('file', imageFile);
form.append('name', 'Downtown Pharmacy');
form.append('facilityType', 'PHARMACY');
form.append('licenseNumber', 'PH-987654');
form.append('address', '145 Main Street');
form.append('pharmacyEnabled', 'true');

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'FAC_IMG_C3A',
    // ⚠️  Do NOT add Content-Type — let the browser set it
  },
  body: form,
});
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Facility updated successfully",
  "data": {
    "facility": {
      "id": "00000000-0000-4000-8000-000000000010",
      "ownerUserId": "00000000-0000-4000-8000-000000000080",
      "name": "Downtown Pharmacy",
      "facilityType": "PHARMACY",
      "pharmacyEnabled": true,
      "licenseNumber": "PH-987654",
      "address": "145 Main Street",
      "imageUrl": "https://res.cloudinary.com/your-cloud/image/upload/v1234567890/facilities/abc123.jpg",
      "verificationStatus": "pending"
    }
  }
}
```

---

### Upload Facility Document


Command: `FAC_DOC_D3E`

Upload a facility license or regulatory document. Accepted types: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`.

**FormData fields:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | File | Yes | Document file. Max 10 MB. |
| `documentType` | string | Yes | Type of document. See allowed values below. |
| `facilityId` | string | Yes | UUID of the facility this document belongs to. |

**Allowed `documentType` values:**

| Value | Meaning |
| --- | --- |
| `LICENSE` | Facility operating license |
| `REGISTRATION` | Government registration certificate |
| `INSPECTION_REPORT` | Latest inspection report |
| `OTHER` | Any other supporting document |

**JavaScript example:**

```ts
const form = new FormData();
form.append('file', pdfFile);
form.append('documentType', 'LICENSE');
form.append('facilityId', '00000000-0000-4000-8000-000000000010');

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'FAC_DOC_D3E',
  },
  body: form,
});
```

**Success response (201):**

```json
{
  "success": true,
  "statusCode": 201,
  "message": "Facility document uploaded successfully",
  "data": {
    "document": {
      "id": "00000000-0000-4000-8000-000000000060",
      "facilityId": "00000000-0000-4000-8000-000000000010",
      "documentType": "LICENSE",
      "documentUrl": "https://res.cloudinary.com/your-cloud/raw/upload/v1234567890/facility-documents/xyz789.pdf",
      "status": "pending",
      "uploadedAt": "2026-09-15T10:00:00.000Z"
    }
  }
}
```

---

### Get Pending Facility Documents *(Inspector only)*

Command: `FAC_DOC_LST_F8H` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

```json
{}
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Pending facility documents retrieved",
  "data": {
    "documents": [
      {
        "id": "00000000-0000-4000-8000-000000000060",
        "facilityId": "00000000-0000-4000-8000-000000000010",
        "documentType": "LICENSE",
        "documentUrl": "https://res.cloudinary.com/...",
        "status": "pending",
        "uploadedAt": "2026-09-15T10:00:00.000Z"
      }
    ]
  }
}
```

---

### Review Facility Document *(Inspector only)*

Command: `FAC_DOC_REV_F9I` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

```json
{
  "documentId": "00000000-0000-4000-8000-000000000060",
  "decision": "approved",
  "notes": "Facility license is current and valid."
}
```

**`decision` values:** `approved` | `rejected`

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Facility document reviewed successfully",
  "data": {
    "document": {
      "id": "00000000-0000-4000-8000-000000000060",
      "status": "approved",
      "reviewedAt": "2026-09-15T11:00:00.000Z",
      "reviewNotes": "Facility license is current and valid."
    }
  }
}
```

---

### Get Pending Facilities *(Inspector only)*

Command: `FAC_LST_PND_F1M` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

Retrieves all facilities awaiting review (`verificationStatus: "pending"`), along with their uploaded documents.

```json
{}
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Pending facilities retrieved successfully",
  "data": {
    "facilities": [
      {
        "id": "00000000-0000-4000-8000-000000000010",
        "ownerUserId": "00000000-0000-4000-8000-000000000002",
        "name": "Nairobi Central Pharmacy",
        "facilityType": "PHARMACY",
        "pharmacyEnabled": true,
        "licenseNumber": "PH-001-2026",
        "address": "123 Kenyatta Avenue, Nairobi",
        "imageUrl": "https://res.cloudinary.com/...",
        "verificationStatus": "pending",
        "createdAt": "2026-09-15T10:00:00.000Z",
        "documents": [
          {
            "id": "00000000-0000-4000-8000-000000000060",
            "documentType": "PHARMACY_BOARD_LICENSE",
            "fileUrl": "https://res.cloudinary.com/...",
            "verificationStatus": "pending",
            "uploadedAt": "2026-09-15T10:05:00.000Z"
          }
        ]
      }
    ],
    "total": 1
  }
}
```

---

### Review Facility *(Inspector only)*

Command: `FAC_REV_F2N` — requires `INSPECTOR` or `SYSTEM_ADMIN` role.

Approve or reject a facility. When approved, the facility's `verificationStatus` is updated to `approved` and the facility owner's user account in Auth Service is automatically activated (`status: "active"`). When rejected, `verificationStatus` is set to `rejected` and the owner's status is set to `rejected`.

```json
{
  "facilityId": "00000000-0000-4000-8000-000000000010",
  "decision": "approved",
  "inspectorNotes": "Facility premises inspected and license confirmed."
}
```

**`decision` values:** `approved` | `rejected`

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Facility approved successfully",
  "data": {
    "facility": {
      "id": "00000000-0000-4000-8000-000000000010",
      "name": "Nairobi Central Pharmacy",
      "verificationStatus": "approved",
      "documents": []
    }
  }
}
```

---

## Drug File Uploads

---

### Upload Drug Image

Command: `DRG_IMG_D2C` — session required. Admin / Inspector only.

Upload a product photo for an approved drug. Accepted types: `image/jpeg`, `image/png`, `image/webp`.

**FormData fields:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | File | Yes | Image file. Max 10 MB. |
| `drugId` | string | Yes | UUID of the drug. |

**JavaScript example:**

```ts
const form = new FormData();
form.append('file', imageFile);
form.append('drugId', '00000000-0000-4000-8000-000000000001');

const res = await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'DRG_IMG_D2C',
  },
  body: form,
});
```

**Success response (200):**

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Drug image uploaded successfully",
  "data": {
    "imageUrl": "https://res.cloudinary.com/your-cloud/image/upload/v1234567890/drugs/abc123.jpg"
  }
}
```

---

## Regulatory

### Search Drugs

Command: `DRG_LST_X7K` — session required.

All fields are optional filters:

```json
{
  "query": "amoxicillin",
  "category": "antibiotic",
  "approvalStatus": "approved"
}
```

### Register Drug *(Admin / Inspector only)*

Command: `DRG_CRT_R7A`

```json
{
  "name": "Amoxicillin 500 mg",
  "activeIngredient": "amoxicillin",
  "dosageForm": "capsule",
  "category": "antibiotic",
  "requiresPrescription": true
}
```

After creating a drug, upload its product image with `DRG_IMG_D2C`.

### Get Drug

Command: `DRG_GET_R8B`

```json
{ "id": "00000000-0000-4000-8000-000000000001" }
```

### Update Drug *(Admin / Inspector only)*

Command: `DRG_UPD_R9C` — approved drugs cannot be changed.

```json
{
  "id": "00000000-0000-4000-8000-000000000001",
  "category": "antibiotic"
}
```

### Submit Drug

Command: `SUB_CRT_S1D` — facility owners / admins.

Either link an existing registered drug or propose a new one:

```json
{
  "facilityId": "00000000-0000-4000-8000-000000000020",
  "proposedDrugData": {
    "name": "Paracetamol 500mg",
    "activeIngredient": "Paracetamol",
    "dosageForm": "Tablet",
    "category": "Analgesic",
    "requiresPrescription": false
  }
}
```

Response:

```json
{
  "success": true,
  "statusCode": 201,
  "message": "Drug submission created successfully",
  "data": {
    "submission": {
      "id": "00000000-f3e1-40bc-90d5-e823498b55be",
      "facilityId": "daf30657-2781-4fc2-b352-4f68712838a1",
      "drugId": null,
      "proposedDrugData": {
        "name": "Paracetamol 500mg",
        "category": "Analgesic",
        "dosageForm": "Tablet",
        "activeIngredient": "Paracetamol",
        "requiresPrescription": false
      },
      "status": "pending",
      "inspectorNotes": null,
      "submittedAt": "2026-09-10T18:05:58.991Z",
      "reviewedAt": null,
      "facility": {
        "id": "daf30657-2781-4fc2-b352-4f68712838a1",
        "name": "Downtown Pharmacy",
        "facilityType": "PHARMACY",
        "verificationStatus": "pending"
      },
      "drug": null
    }
  }
}
```

### List Pending Submissions *(Admin / Inspector only)*

Command: `SUB_LST_S2E`

```json
{}
```

### Review Submission *(Admin / Inspector only)*

Command: `SUB_REV_S3F`

```json
{
  "submissionId": "00000000-0000-4000-8000-000000000020",
  "decision": "approved",
  "reason": "License and product details verified."
}
```

Response:

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Drug submission approved successfully",
  "data": {
    "review": {
      "id": "111111-c8f1-4333-881b-a888dedd5674",
      "inspectorId": "1873e067-3796-455c-8fec-71d2190e4ba6",
      "submissionId": "2bc6c6a8-f3e1-40bc-90d5-e823498b55be",
      "decision": "approved",
      "reason": "License and product details verified.",
      "createdAt": "2026-09-10T18:12:54.983Z"
    },
    "submission": {
      "id": "00000000-0000-4000-8000-000000000020",
      "status": "approved",
      "drug": {
        "id": "999999-03b1-4984-8440-a8f5e49248d1",
        "name": "Paracetamol 500mg",
        "approvalStatus": "approved",
        "approvedAt": "2026-09-10T18:12:54.980Z"
      }
    }
  }
}
```

---

## Commerce

### Add Drug to Inventory

Command: `INV_ADD_C5G` — pharmacy facility owners / admins.

```json
{
  "facilityId": "00000000-0000-4000-8000-000000000010",
  "drugId": "00000000-0000-4000-8000-000000000001",
  "stockCount": 100,
  "price": 1500
}
```

Response:

```json
{
  "success": true,
  "statusCode": 201,
  "message": "Drug added to inventory successfully",
  "data": {
    "inventory": {
      "id": "00000000-0000-4000-8000-000000000090",
      "facilityId": "00000000-0000-4000-8000-000000000010",
      "drugId": "00000000-0000-4000-8000-000000000001",
      "stockCount": 100,
      "price": "1500.00",
      "createdAt": "2026-09-11T17:13:27.346Z",
      "updatedAt": "2026-09-11T17:13:27.346Z"
    },
    "drug": {
      "id": "00000000-0000-4000-8000-000000000001",
      "name": "Amoxicillin 500 mg",
      "activeIngredient": "amoxicillin",
      "dosageForm": "capsule",
      "category": "antibiotic",
      "requiresPrescription": true,
      "approvalStatus": "approved",
      "approvedAt": "2026-09-10T18:12:54.980Z"
    }
  }
}
```

---

## Command Catalog

Every command uses `POST /api` with `X-Server-Key` and `X-Command`. The **Type** column tells you how to format the body.

| Command | Session | Type | Description |
| --- | --- | --- | --- |
| `REGISTER_AUTH_1A2` | No | JSON | Register a new user account. |
| `LOGIN_AUTH_3B4` | No | JSON | Login; sets session cookie. |
| `COMPLETE_REGISTRATION_AUTH_7M3` | Yes | JSON | Complete an incomplete account. |
| `LOGOUT_AUTH_5C6` | Yes | JSON | Destroy session. |
| `DOC_IMG_D1B` | Yes | **multipart** | Upload doctor profile image. |
| `DCT_DOC_D4F` | Yes | **multipart** | Upload doctor professional document. |
| `DOC_LST_D5G` | Yes (Inspector) | JSON | List pending doctor documents for review. |
| `DOC_REV_D6H` | Yes (Inspector) | JSON | Approve or reject a doctor document. |
| `DOC_ACT_LST_D7J` | Yes (Inspector) | JSON | List pending doctor accounts & profiles. |
| `DOC_ACT_REV_D8K` | Yes (Inspector) | JSON | Approve or reject a doctor account. |
| `DOC_PRF_G9M` | Yes | JSON | Get doctor profile and uploaded documents. |
| `FAC_IMG_C3A` | Yes | **multipart** | Upload facility profile image (after registration). |
| `FAC_DOC_D3E` | Yes | **multipart** | Upload facility license / regulatory document. |
| `FAC_DOC_LST_F8H` | Yes (Inspector) | JSON | List pending facility documents for review. |
| `FAC_DOC_REV_F9I` | Yes (Inspector) | JSON | Approve or reject a facility document. |
| `FAC_LST_PND_F1M` | Yes (Inspector) | JSON | List pending facilities for review. |
| `FAC_REV_F2N` | Yes (Inspector) | JSON | Approve or reject a facility account. |
| `DRG_IMG_D2C` | Yes (Admin/Inspector) | **multipart** | Upload drug product image. |
| `DRG_LST_X7K` | Yes | JSON | Search / list drug registry. |
| `DRG_CRT_R7A` | Yes (Admin/Inspector) | JSON | Register a new drug. |
| `DRG_GET_R8B` | Yes | JSON | Get drug by ID. |
| `DRG_UPD_R9C` | Yes (Admin/Inspector) | JSON | Update drug fields. |
| `SUB_CRT_S1D` | Yes | JSON | Submit a drug for regulatory review. |
| `SUB_LST_S2E` | Yes (Inspector) | JSON | List pending drug submissions. |
| `SUB_REV_S3F` | Yes (Inspector) | JSON | Review a drug submission. |
| `INV_ADD_C5G` | Yes | JSON | Add drug to pharmacy inventory. |
| `ORD_CHK_9QZ` | Yes | JSON | Checkout an order. |
| `PAY_CHG_M3P` | Yes | JSON | Charge payment for an order. |
| `RX_ISS_L4T` | Yes | JSON | Issue a prescription. |
| `NTF_CRT_N8P` | Yes | JSON | Create a notification. |
| `AUD_CRT_Q5R` | Yes | JSON | Create an audit event. |
| `APT_CRT_A2B` | Yes | JSON | Create a telehealth appointment. |
| `FCLTY_LST_F7L` | No | JSON | List all facilities (public). |
| `PUB_DRG_C1H` | No | JSON | List public drug products (commerce). |

---

## File Upload Rules

| Rule | Detail |
| --- | --- |
| Max file size | 10 MB per file |
| Accepted image types | `image/jpeg`, `image/png`, `image/webp` |
| Accepted document types | `application/pdf`, `image/jpeg`, `image/png`, `image/webp` |
| `Content-Type` header | **Do NOT set manually** for multipart requests. Let `FormData` set it. |
| File field name | Always `file` |
| Session cookie | Required for all upload commands |
| Upload timing | Register first (JSON), then upload files after login |

---

## Standard Errors

| HTTP status | Meaning |
| --- | --- |
| `400` | Invalid or incomplete request body / unsupported file type. |
| `401` | Missing / invalid session or invalid credentials. |
| `403` | Invalid Gateway key, unknown command, or insufficient role. |
| `409` | Duplicate email, duplicate drug, or conflicting operation. |
| `413` | File exceeds the 10 MB limit. |
| `503` | Target internal service is unavailable. |

Typical validation error shape:

```json
{
  "statusCode": 400,
  "message": ["email must be an email", "password must be longer than or equal to 8 characters"],
  "error": "Bad Request"
}
```

---

## Account Status Reference

| Database status | Frontend status | Login | Registration completion |
| --- | --- | --- | --- |
| `active` | `ACTIVE` | Yes | No |
| `incomplete` | `INCOMPLETE` | Yes | Yes |
| `pending_verification` | `PENDING_VERIFICATION` | Yes | Normally no |
| `suspended` | `SUSPENDED` | No | No |
| `rejected` | `REJECTED` | No | No |

> The frontend must **never** send account status, verification status, role, permissions, or another user's ID to authorize an operation.

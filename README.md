# trd-documentation
# Frontend API Guide

This is the frontend contract for the TRD LF API. The frontend communicates only with the API Gateway.

## Base Request

All operations use one endpoint:

```http
POST http://localhost:3000/api
```

Required headers:

| Header | Required | Description |
| --- | --- | --- |
| `X-Server-Key` | Yes | Gateway application key configured for the frontend environment. |
| `X-Command` | Yes | One command ID from the catalog below. |
| `Content-Type` | Yes | `application/json`. |
| `Cookie: sessionId=...` | Protected commands | Set automatically by the browser after login. |

Do not send `AUTH_INTERNAL_API_KEY`, internal service URLs, or internal service ports from frontend code.

Example:

```bash
curl -i http://localhost:3000/api \
  -X POST \
  -H 'X-Server-Key: development-gateway-key' \
  -H 'X-Command: REGISTER_AUTH_1A2' \
  -H 'Content-Type: application/json' \
  -d '{"accountType":"PATIENT","name":"Example Patient","email":"patient@example.com","phone":"+15550000001","password":"ExamplePassword123!"}'
```

For browser requests, use `credentials: 'include'` so the HTTP-only session cookie is stored and sent:

```ts
await fetch(`${API_BASE_URL}/api`, {
  method: 'POST',
  credentials: 'include',
  headers: {
    'X-Server-Key': gatewayKey,
    'X-Command': 'LOGIN_AUTH_3B4',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ email, password }),
});
```

## Registration

Command: `REGISTER_AUTH_1A2`

This command does not require a session. `accountType` is optional for backward compatibility; when omitted, it means `PATIENT`.

Allowed public account types:

- `PATIENT`
- `DOCTOR`
- `PHARMACY_ADMIN`
- `CLINIC_ADMIN`

`ADMIN` and `INSPECTOR` are controlled roles and must be rejected by public registration.

### Common fields

| Field | Required | Description |
| --- | --- | --- |
| `accountType` | No | Public role. Defaults to `PATIENT`. |
| `name` | Yes | Display name, maximum 150 characters. |
| `email` | Yes | Valid email address. |
| `phone` | Yes | Phone number, maximum 30 characters. |
| `password` | Yes | At least 8 characters. |
| `doctor` | Doctor only | Doctor profile object. |
| `facility` | Pharmacy/clinic only | Facility object. |

### Patient registration

Request:

```json
{
  "accountType": "PATIENT",
  "name": "Example Patient",
  "email": "patient@example.com",
  "phone": "+15550000001",
  "password": "ExamplePassword123!"
}
```

Response: patient accounts are immediately active.

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

Doctor fields:

| Field | Required for completion | Description |
| --- | --- | --- |
| `doctor.licenseNumber` | Yes | Professional license number. |
| `doctor.specialty` | Yes | Professional specialty. |
| `doctor.facilityId` | No | Existing Regulatory facility UUID, when applicable. |

Complete request:

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

The Auth database creates `DoctorProfile` and returns it as `data.doctorProfile`, alongside the user. The user has `status: "pending_verification"` and `accountStatus: "PENDING_VERIFICATION"`.

Basic-only request:

```json
{
  "accountType": "DOCTOR",
  "name": "Dr. Example",
  "email": "doctor@example.com",
  "phone": "+15550000002",
  "password": "ExamplePassword123!"
}
```

This creates the user with `status: "incomplete"`. No incomplete DoctorProfile is created.

### Pharmacy admin registration

Facility fields:

| Field | Required | Description |
| --- | --- | --- |
| `facility.name` | Yes for completion | Pharmacy name. |
| `facility.licenseNumber` | Yes for completion | Pharmacy license number. |
| `facility.address` | Yes for completion | Pharmacy address. |
| `facility.pharmacyEnabled` | No | Defaults to `true` for pharmacy admins. |

Request:

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

The Gateway forwards the request to Auth. Auth creates the user, then securely asks Regulatory to create the authoritative facility with `facilityType: "PHARMACY"`, `pharmacyEnabled: true`, and `verificationStatus: "pending"`. The response includes the created facility in `data.facility`, and the user status is `pending_verification`.

The role-specific response data looks like this:

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

If `facility` is omitted, Auth creates only the user with `status: "incomplete"`, returns `data.facility: null`, and does not create an empty facility.

### Clinic admin registration

Request:

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

Regulatory creates the authoritative facility with `facilityType: "CLINIC"`. `pharmacyEnabled` defaults to `false` when omitted. The response includes the created facility in `data.facility`, and the user status is `pending_verification`.

If `facility` is omitted, the user is created as `incomplete`, `data.facility` is `null`, and no facility is created.

## Login

Command: `LOGIN_AUTH_3B4`. No session is required.

Request:

```json
{
  "email": "pharmacy-admin@example.com",
  "password": "ExamplePassword123!"
}
```

The response sets an HTTP-only `sessionId` cookie. Store no session ID in local storage.

Doctor Or Patient response:

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
        "name": "PATIENT",
        "email": "patient@example.com",
        "phone": "+15550000003",
        "roleId": "......",
        "status": "pending_verification",
        "accountStatus": "PENDING_VERIFICATION",
        "role": "PATIENT"
    } 
    }
}
```

Facility response:

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
        "name": "Example Pharmacy Admin",
        "email": "PATIENT-admin@example.com",
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

Incomplete response has `accountStatus: "INCOMPLETE"` and `requiresRegistrationCompletion: true`. Pending verification has `accountStatus: "PENDING_VERIFICATION"` and should show a verification-pending screen. Both states can authenticate, but they must not receive active-only business privileges.

## Complete Registration

Command: `COMPLETE_REGISTRATION_AUTH_7M3`. A valid session is required.

The authenticated user is taken from the session cookie. Do not send `userId`, role, status, permissions, or verification status. The Gateway forwards the authenticated `X-User-Id` internally.

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

Result:

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

Use `pharmacyEnabled: false` for a normal clinic. The facility operation is idempotent for the authenticated owner and facility type: an existing facility is updated instead of duplicated.

### Complete a patient account

Only genuinely missing profile fields should be sent:

```json
{
  "name": "Updated Patient Name",
  "phone": "+15550000005",
  "password": "NewPassword123!"
}
```

The resulting account status is `active`.

## Logout

Command: `LOGOUT_AUTH_5C6`. Requires a session.

Request body:

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

## Command Catalog

Every command below uses `POST /api`, `X-Server-Key`, and `X-Command`. Every command marked “Yes” also requires the session cookie.

| Command | Session | Body summary |
| --- | --- | --- |
| `REGISTER_AUTH_1A2` | No | Registration body described above. |
| `LOGIN_AUTH_3B4` | No | `email`, `password`. |
| `COMPLETE_REGISTRATION_AUTH_7M3` | Yes | Role-specific completion body described above. |
| `LOGOUT_AUTH_5C6` | Yes | `{}`. |
| `APT_CRT_A2B` | Yes | `{ "appointmentId": "<uuid>" }`; telehealth fields are pending their milestone. |
| `DRG_LST_X7K` | Yes | Optional `query`, `category`, `approvalStatus`. |
| `DRG_CRT_R7A` | Yes | `name`; optional `activeIngredient`, `dosageForm`, `category`, `requiresPrescription`. Admin/inspector only. |
| `DRG_GET_R8B` | Yes | `{ "id": "<drug uuid>" }`. |
| `DRG_UPD_R9C` | Yes | `id` plus optional drug fields. Admin/inspector only; approved drugs cannot be changed. |
| `SUB_CRT_S1D` | Yes | `facilityId` plus `drugId` or `proposedDrugData`. Facility owners/admins. |
| `SUB_LST_S2E` | Yes | `{}`. Admin/inspector only. |
| `SUB_REV_S3F` | Yes | `submissionId`, `decision` (`approved`/`rejected`), `reason`. Admin/inspector only. |
| `INV_ADD_C5G` | Yes | `facilityId`, `drugId`, `stockCount`, `price`. Pharmacy facility owners/admins. |
| `ORD_CHK_9QZ` | Yes | `facilityId`, `items[]`, optional `deliveryAddress`. |
| `PAY_CHG_M3P` | Yes | `orderId`, `method`, `amount`. |
| `RX_ISS_L4T` | Yes | `patientId`, `drugId`, `digitalSignature`, optional `dosage`. |
| `NTF_CRT_N8P` | Yes | `userId`, `type`, `title`, `message`. |
| `AUD_CRT_Q5R` | Yes | `action`, `service`, `entityType`, `entityId`, optional `metadata`. |

Add a drug to pharmacy inventory:

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
      "approvedBy": "00000000-0000-4000-8000-000000000081",
      "approvedAt": "2026-09-10T18:12:54.980Z",
      "createdAt": "2026-09-10T18:12:54.981Z"
    }
  }
}
```

The inventory response contains the created inventory record and the approved drug registry record. It does not include the drug's submission history.

### Regulatory request examples

Register drug:

```json
{
  "name": "Amoxicillin 500 mg",
  "activeIngredient": "amoxicillin",
  "dosageForm": "capsule",
  "category": "antibiotic",
  "requiresPrescription": true
}
```

Search drugs:

```json
{
  "query": "amoxicillin",
  "category": "antibiotic",
  "approvalStatus": "approved"
}
```

Submit a drug:

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

response
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
                "ownerUserId": "41a2df6e-4495-473c-874c-eb36f682edcb",
                "name": "Downtown Pharmacy",
                "facilityType": "PHARMACY",
                "pharmacyEnabled": true,
                "licenseNumber": "PH-987654",
                "address": "145 Main Street",
                "verificationStatus": "pending",
                "createdAt": "2026-09-10T15:31:05.483Z",
                "updatedAt": "2026-09-10T15:31:05.483Z"
            },
            "drug": null
        }
    }
}
```

Review a submission:

```json
{
  "submissionId": "00000000-0000-4000-8000-000000000020",
  "decision": "approved",
  "reason": "License and product details verified."
}
```

Response
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
            "facilityId": "daf30657-2781-4fc2-b352-4f68712838a1",
            "drugId": "03eddeea-03b1-4984-8440-a8f5e49248d1",
            "proposedDrugData": {
                "name": "Paracetamol 500mg",
                "category": "Analgesic",
                "dosageForm": "Tablet",
                "activeIngredient": "Paracetamol",
                "requiresPrescription": false
            },
            "status": "approved",
            "inspectorNotes": "License and product details verified.",
            "submittedAt": "2026-09-10T18:05:58.991Z",
            "reviewedAt": "2026-09-10T18:12:54.984Z",
            "facility": {
                "id": "222222-2781-4fc2-b352-4f68712838a1",
                "ownerUserId": "41a2df6e-4495-473c-874c-eb36f682edcb",
                "name": "Downtown Pharmacy",
                "facilityType": "PHARMACY",
                "pharmacyEnabled": true,
                "licenseNumber": "PH-987654",
                "address": "145 Main Street",
                "verificationStatus": "pending",
                "createdAt": "2026-09-10T15:31:05.483Z",
                "updatedAt": "2026-09-10T15:31:05.483Z"
            },
            "drug": {
                "id": "999999-03b1-4984-8440-a8f5e49248d1",
                "name": "Paracetamol 500mg",
                "activeIngredient": "Paracetamol",
                "dosageForm": "Tablet",
                "category": "Analgesic",
                "requiresPrescription": false,
                "approvalStatus": "approved",
                "approvedBy": "1873e067-3796-455c-8fec-71d2190e4ba6",
                "approvedAt": "2026-09-10T18:12:54.980Z",
                "createdAt": "2026-09-10T18:12:54.981Z"
            },
            "inspectorReviews": [
                {
                    "id": "000000-c8f1-4333-881b-a888dedd5674",
                    "inspectorId": "1873e067-3796-455c-8fec-71d2190e4ba6",
                    "submissionId": "2bc6c6a8-f3e1-40bc-90d5-e823498b55be",
                    "decision": "approved",
                    "reason": "License and product details verified.",
                    "createdAt": "2026-09-10T18:12:54.983Z"
                }
            ]
        }
    }
}
```

## Standard Errors

Errors are returned through the common API exception envelope. Frontend code should use the HTTP status and message, not infer authorization from `X-Command`.

| HTTP status | Meaning |
| --- | --- |
| `400` | Invalid or incomplete request body. |
| `401` | Missing/invalid session or invalid credentials. |
| `403` | Invalid Gateway key, unknown command, or insufficient role permission. |
| `409` | Duplicate email, duplicate drug, or conflicting operation. |
| `503` | Target internal service is unavailable. |

Typical validation response:

```json
{
  "statusCode": 400,
  "message": ["email must be an email", "password must be longer than or equal to 8 characters"],
  "error": "Bad Request"
}
```

## Account Status Rules

| Database status | Frontend status | Login | Registration completion |
| --- | --- | --- | --- |
| `active` | `ACTIVE` | Yes | No |
| `incomplete` | `INCOMPLETE` | Yes | Yes |
| `pending_verification` | `PENDING_VERIFICATION` | Yes | Normally no, unless the account still has missing data |
| `suspended` | `SUSPENDED` | No | No |
| `rejected` | `REJECTED` | No | No |

The frontend must never send account status, verification status, role, permissions, or another user’s ID to authorize an operation.

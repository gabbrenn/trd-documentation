# TRD LF API — Testing Flow & Test Data

All requests go through the API Gateway at `POST http://localhost:3000/api`.
Every request needs the header `X-Server-Key: development-gateway-key`.

---

## Prerequisites

Make sure all three services are running:

```bash
npm run start:auth        # Auth Service     → port 3001
npm run start:regulatory  # Regulatory       → port 3002
npm run start:gateway     # API Gateway      → port 3000
```

---

## Test Accounts (use these exact values)

### Patient
| Field    | Value                      |
|----------|----------------------------|
| email    | `patient@test.com`         |
| password | `Test1234!`                |
| role     | `PATIENT`                  |

### Doctor
| Field    | Value                      |
|----------|----------------------------|
| email    | `doctor@test.com`          |
| password | `Test1234!`                |
| role     | `DOCTOR`                   |
| license  | `MD-99001`                 |
| specialty| `Cardiology`               |

### Pharmacy Admin
| Field    | Value                        |
|----------|------------------------------|
| email    | `pharmacy@test.com`          |
| password | `Test1234!`                  |
| role     | `PHARMACY_ADMIN`             |
| facility name     | `Test Pharmacy`     |
| facility license  | `PH-77001`          |
| address           | `10 Test Avenue`    |

### Clinic Admin
| Field    | Value                         |
|----------|-------------------------------|
| email    | `clinic@test.com`             |
| password | `Test1234!`                   |
| role     | `CLINIC_ADMIN`                |
| facility name     | `Test Clinic`        |
| facility license  | `CL-88001`           |
| address           | `20 Health Street`   |

### Inspector *(must be seeded or created by SYSTEM_ADMIN)*
| Field    | Value                      |
|----------|----------------------------|
| email    | `inspector@test.com`       |
| password | `Test1234!`                |
| role     | `INSPECTOR`                |

---

## Flow 1 — Patient Registration & Login

**Step 1 — Register**

```http
POST /api
X-Command: REGISTER_AUTH_1A2
Content-Type: application/json

{
  "accountType": "PATIENT",
  "name": "Test Patient",
  "email": "patient@test.com",
  "phone": "+254700000001",
  "password": "Test1234!"
}
```

Expected: `201`, `accountStatus: "ACTIVE"`.

---

**Step 2 — Login**

```http
POST /api
X-Command: LOGIN_AUTH_3B4
Content-Type: application/json

{
  "email": "patient@test.com",
  "password": "Test1234!"
}
```

Expected: `200`, `authenticated: true`. Browser/Postman stores `sessionId` cookie automatically.

---

**Step 3 — Logout**

```http
POST /api
X-Command: LOGOUT_AUTH_5C6
Content-Type: application/json

{}
```

Expected: `200`, `"Logout successful"`.

---

## Flow 2 — Doctor Registration + Image + Document Upload

**Step 1 — Register Doctor**

```http
POST /api
X-Command: REGISTER_AUTH_1A2
Content-Type: application/json

{
  "accountType": "DOCTOR",
  "name": "Dr. Test",
  "email": "doctor@test.com",
  "phone": "+254700000002",
  "password": "Test1234!",
  "doctor": {
    "licenseNumber": "MD-99001",
    "specialty": "Cardiology"
  }
}
```

Expected: `201`, `accountStatus: "PENDING_VERIFICATION"`.

---

**Step 2 — Login as Doctor**

```http
POST /api
X-Command: LOGIN_AUTH_3B4
Content-Type: application/json

{
  "email": "doctor@test.com",
  "password": "Test1234!"
}
```

Expected: `200`, session cookie set.

---

**Step 3 — Upload Doctor Profile Image** *(multipart — no Content-Type header)*

```
POST /api
X-Command: DOC_IMG_D1B
[session cookie required]

FormData:
  file  →  any .jpg / .png / .webp image (max 10 MB)
```

Curl example:
```bash
curl -X POST http://localhost:3000/api \
  -H "X-Server-Key: development-gateway-key" \
  -H "X-Command: DOC_IMG_D1B" \
  -b "sessionId=<your-session-id>" \
  -F "file=@/path/to/photo.jpg"
```

Expected: `200`, `data.profileImageUrl` is a Cloudinary URL.

---

**Step 4 — Upload Doctor License Document** *(multipart)*

```
POST /api
X-Command: DCT_DOC_D4F
[session cookie required]

FormData:
  file          →  any .pdf / .jpg / .png / .webp (max 10 MB)
  documentType  →  LICENSE
```

Curl example:
```bash
curl -X POST http://localhost:3000/api \
  -H "X-Server-Key: development-gateway-key" \
  -H "X-Command: DCT_DOC_D4F" \
  -b "sessionId=<your-session-id>" \
  -F "file=@/path/to/license.pdf" \
  -F "documentType=LICENSE"
```

Expected: `201`, `data.document.status: "pending"`.

---

**Step 5 — Upload Additional Certificate** *(same command, different type)*

```
FormData:
  file          →  certificate.pdf
  documentType  →  CERTIFICATE
```

Expected: `201`, another document record with `status: "pending"`.

---

**Step 6 — Login as Inspector & Review Documents**

```http
POST /api
X-Command: LOGIN_AUTH_3B4
Content-Type: application/json

{
  "email": "inspector@test.com",
  "password": "Test1234!"
}
```

---

**Step 7 — List Pending Doctor Documents**

```http
POST /api
X-Command: DOC_LST_D5G
Content-Type: application/json

{}
```

Expected: `200`, array of pending documents. Copy `id` from a document for the next step.

---

**Step 8 — Approve Doctor Document**

```http
POST /api
X-Command: DOC_REV_D6H
Content-Type: application/json

{
  "documentId": "<id from step 7>",
  "decision": "approved",
  "notes": "License verified and valid."
}
```

Expected: `200`, `data.document.status: "approved"`.

**Step 8b — Reject a document (alternative)**

```http
{
  "documentId": "<id from step 7>",
  "decision": "rejected",
  "notes": "Document is expired."
}
```

Expected: `200`, `data.document.status: "rejected"`.

---

**Step 9 — List Pending Doctor Accounts**

```http
POST /api
X-Command: DOC_ACT_LST_D7J
Content-Type: application/json

{}
```

Expected: `200`, array of doctor accounts in `pending_verification` status with their doctor profiles and uploaded documents. Copy `doctorId` or `userId`.

---

**Step 10 — View Doctor Profile**

```http
POST /api
X-Command: DOC_PRF_G9M
Content-Type: application/json

{
  "doctorId": "<doctorId from step 9>"
}
```

Expected: `200`, complete doctor profile including user contact info and verification status.

---

**Step 11 — Approve Doctor Account**

```http
POST /api
X-Command: DOC_ACT_REV_D8K
Content-Type: application/json

{
  "doctorId": "<doctorId from step 9>",
  "decision": "approved",
  "reason": "Doctor credentials verified and approved."
}
```

Expected: `200`, `user.status: "active"` and `doctor.verificationStatus: "approved"`. Doctor can now log in as an active verified doctor!

---

## Flow 3 — Pharmacy Registration + Facility Document Upload

**Step 1 — Register Pharmacy Admin**

```http
POST /api
X-Command: REGISTER_AUTH_1A2
Content-Type: application/json

{
  "accountType": "PHARMACY_ADMIN",
  "name": "Test Pharmacy Admin",
  "email": "pharmacy@test.com",
  "phone": "+254700000003",
  "password": "Test1234!",
  "facility": {
    "name": "Test Pharmacy",
    "licenseNumber": "PH-77001",
    "address": "10 Test Avenue",
    "pharmacyEnabled": true
  }
}
```

Expected: `201`, `data.facility.verificationStatus: "pending"`. Save `data.facility.id` as **facilityId**.

---

**Step 2 — Login as Pharmacy Admin**

```http
POST /api
X-Command: LOGIN_AUTH_3B4
Content-Type: application/json

{
  "email": "pharmacy@test.com",
  "password": "Test1234!"
}
```

---

**Step 3 — Upload Facility Profile Image** *(optional, multipart)*

> Registration is JSON-only and cannot include an image. Use this separate step **after login** to attach a photo to the facility.
>
> Command: `FAC_IMG_C3A`

```
POST /api
X-Command: FAC_IMG_C3A
[session cookie required]

FormData:
  file            →  any .jpg / .png / .webp (max 10 MB)
  name            →  Test Pharmacy                       (same values used at registration)
  facilityType    →  PHARMACY
  licenseNumber   →  PH-77001
  address         →  10 Test Avenue
  pharmacyEnabled →  true
```

Curl example:
```bash
curl -X POST http://localhost:3000/api \
  -H "X-Server-Key: development-gateway-key" \
  -H "X-Command: FAC_IMG_C3A" \
  -b "sessionId=<your-session-id>" \
  -F "file=@/path/to/pharmacy-photo.jpg" \
  -F "name=Test Pharmacy" \
  -F "facilityType=PHARMACY" \
  -F "licenseNumber=PH-77001" \
  -F "address=10 Test Avenue" \
  -F "pharmacyEnabled=true"
```

Expected: `200`, `data.facility.imageUrl` is a Cloudinary URL.

> **Note:** `FAC_IMG_C3A` calls `POST /api/facility/create` on regulatory-service. It is idempotent — if the facility already exists for that owner and `facilityType`, it is **updated** with the new image instead of duplicated.

---

**Step 4 — Upload Facility License Document** *(multipart)*

```
POST /api
X-Command: FAC_DOC_D3E
[session cookie required]

FormData:
  file          →  license.pdf (max 10 MB)
  documentType  →  LICENSE
  facilityId    →  <facilityId from step 1>
```

Curl example:
```bash
curl -X POST http://localhost:3000/api \
  -H "X-Server-Key: development-gateway-key" \
  -H "X-Command: FAC_DOC_D3E" \
  -b "sessionId=<your-session-id>" \
  -F "file=@/path/to/license.pdf" \
  -F "documentType=LICENSE" \
  -F "facilityId=<facilityId>"
```

Expected: `201`, `data.document.status: "pending"`.

---

**Step 4 — Login as Inspector & List Pending Facility Documents**

```http
POST /api
X-Command: FAC_DOC_LST_F8H
Content-Type: application/json

{}
```

Expected: `200`, array of pending facility documents. Copy `id`.

---

**Step 5 — Approve Facility Document**

```http
POST /api
X-Command: FAC_DOC_REV_F9I
Content-Type: application/json

{
  "documentId": "<id from step 4>",
  "decision": "approved",
  "notes": "Facility license is current and valid."
}
```

Expected: `200`, `data.document.status: "approved"`.

---

**Step 6 — List Pending Facilities**

```http
POST /api
X-Command: FAC_LST_PND_F1M
Content-Type: application/json

{}
```

Expected: `200`, array of pending facilities with their uploaded documents. Copy `id` of the facility.

---

**Step 7 — Review & Approve Facility Account**

```http
POST /api
X-Command: FAC_REV_F2N
Content-Type: application/json

{
  "facilityId": "<facilityId from step 6>",
  "decision": "approved",
  "inspectorNotes": "Facility premises and licenses verified."
}
```

Expected: `200`, `data.facility.verificationStatus: "approved"`. The owner's user account in Auth Service is automatically activated!

---

## Flow 4 — Drug Registration + Image Upload

*(Requires INSPECTOR or SYSTEM_ADMIN session)*

**Step 1 — Register a Drug**

```http
POST /api
X-Command: DRG_CRT_R7A
Content-Type: application/json

{
  "name": "Test Drug 500mg",
  "activeIngredient": "testamine",
  "dosageForm": "tablet",
  "category": "analgesic",
  "requiresPrescription": false
}
```

Expected: `201`, save `data.drug.id` as **drugId**.

---

**Step 2 — Upload Drug Product Image** *(multipart)*

```
POST /api
X-Command: DRG_IMG_D2C
[session cookie required — Inspector/Admin]

FormData:
  file    →  drug-photo.jpg (max 10 MB)
  drugId  →  <drugId from step 1>
```

Curl example:
```bash
curl -X POST http://localhost:3000/api \
  -H "X-Server-Key: development-gateway-key" \
  -H "X-Command: DRG_IMG_D2C" \
  -b "sessionId=<your-session-id>" \
  -F "file=@/path/to/drug-photo.jpg" \
  -F "drugId=<drugId>"
```

Expected: `200`, `data.imageUrl` is a Cloudinary URL.

---

**Step 3 — Search Drugs (verify image appears)**

```http
POST /api
X-Command: DRG_LST_X7K
Content-Type: application/json

{
  "query": "Test Drug"
}
```

Expected: drug appears in results with `imageUrl` populated.

---

## Flow 5 — Drug Submission & Approval

*(Pharmacy Admin submits, Inspector reviews)*

**Step 1 — Submit a Drug (as Pharmacy Admin)**

```http
POST /api
X-Command: SUB_CRT_S1D
Content-Type: application/json

{
  "facilityId": "<facilityId>",
  "proposedDrugData": {
    "name": "New Drug 250mg",
    "activeIngredient": "newamine",
    "dosageForm": "capsule",
    "category": "antibiotic",
    "requiresPrescription": true
  }
}
```

Expected: `201`, `data.submission.status: "pending"`. Save `data.submission.id` as **submissionId**.

---

**Step 2 — List Pending Submissions (as Inspector)**

```http
POST /api
X-Command: SUB_LST_S2E
Content-Type: application/json

{}
```

---

**Step 3 — Approve Submission**

```http
POST /api
X-Command: SUB_REV_S3F
Content-Type: application/json

{
  "submissionId": "<submissionId>",
  "decision": "approved",
  "reason": "Product and facility details verified."
}
```

Expected: `200`, `data.submission.status: "approved"`, `data.submission.drug` populated with new drug record.

---

## Flow 6 — Add Drug to Inventory & Order Checkout

**Step 1 — Add Drug to Inventory (as Pharmacy Admin)**

```http
POST /api
X-Command: INV_ADD_C5G
Content-Type: application/json

{
  "facilityId": "<facilityId>",
  "drugId": "<drugId>",
  "stockCount": 200,
  "price": 500
}
```

Expected: `201`, inventory record created.

---

**Step 2 — Checkout Order (as Patient)**

```http
POST /api
X-Command: ORD_CHK_9QZ
Content-Type: application/json

{
  "facilityId": "<facilityId>",
  "items": [
    { "drugId": "<drugId>", "quantity": 2, "price": 500 }
  ],
  "deliveryAddress": "5 Patient Street"
}
```

---

## Quick Error Test Cases

| Scenario | How to trigger | Expected |
|---|---|---|
| Missing session | Send any protected command without cookie | `401` |
| Wrong gateway key | Set `X-Server-Key: wrong-key` | `403` |
| Unknown command | Set `X-Command: FAKE_CMD` | `403` |
| Duplicate email | Register same email twice | `409` |
| File too large | Upload file > 10 MB | `413` |
| Wrong MIME type | Upload `.txt` as a file | `400` |
| Missing required field | Omit `email` from register | `400`, validation array |
| Inspector command as Doctor | Call `DOC_LST_D5G` with doctor session | `403` |

---

## Notes

- **Session cookie**: Postman → Settings → **"Send cookies"** must be ON. Cookies are automatic in browsers with `credentials: 'include'`.
- **Multipart requests in Postman**: Use **Body → form-data**. Select **File** type for the `file` key. Do NOT add `Content-Type` to the header tab.
- **IDs**: All IDs are UUIDs (e.g. `00000000-0000-4000-8000-000000000001`). Copy them from previous responses when chaining flows.
- **documentType allowed values**: `LICENSE`, `CERTIFICATE`, `IDENTIFICATION`, `INSPECTION_REPORT`, `REGISTRATION`, `OTHER`.
- **decision allowed values**: `approved`, `rejected`.

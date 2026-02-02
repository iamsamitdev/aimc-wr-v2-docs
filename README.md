# WR API v2 - Web Register API for Exam Center Integration

## Overview
API สำหรับเชื่อมต่อระหว่าง Web Register (AIMC Knowledge) กับ Exam Center (EC)

**Version:** 2.0  
**PHP Version:** 5.6.40+  
**Base URL (WR):** `https://aimc.or.th/center/aimc_wr_v2/api`  
**Base URL (EC):** `https://api.dev.sete.skooldio.dev/exg/api`

## Requirements
- PHP 5.6.40+
- MySQL 5.6+
- Apache with mod_rewrite
- phpseclib (included)
- cURL extension enabled

## Installation

### 1. Database Setup
```sql
-- สร้าง database และ run schema
mysql -u root -p < database/schema.sql
```

### 2. Configuration
แก้ไขไฟล์ `config/config.php`:
```php
// Database
define('DB_HOST', 'localhost');
define('DB_NAME', 'wr_api_v2');
define('DB_USER', 'your_username');
define('DB_PASS', 'your_password');

// Exam Center API URL
define('EC_API_BASE_URL', 'https://api.dev.sete.skooldio.dev/exg/api');

// Client IDs
define('WR_CLIENT_ID', 'AIMC_WR_001');
define('EC_CLIENT_ID', 'EC_AIMC_001');
```

### 3. Generate Keys (for testing)
```bash
php generate_keys.php
```

### 4. Key Exchange with Exam Center
1. ส่ง `keys/WRPublicKey.xml` ให้ Exam Center
2. รับ `ECPublicKey.xml` จาก Exam Center และวางใน `keys/`

### 5. สรุปไฟล์ Keys ที่ต้องมีในแต่ละฝ่าย

#### 🏢 ฝั่ง Web Register (WR) - `wr_v2/keys/`

| ไฟล์ | ประเภท | ใช้ทำอะไร |
|------|--------|----------|
| `WRPrivateKey.xml` | **Private Key** | Sign JWT เมื่อ WR ขอ token จาก EC |
| `WRPublicKey.xml` | **Public Key** | ส่งให้ EC เพื่อ verify JWT จาก WR |
| `ECPublicKey.xml` | **Public Key** | Verify JWT จาก EC เมื่อ EC ขอ token |

#### 🏛️ ฝั่ง Exam Center (EC)

| ไฟล์ | ประเภท | ใช้ทำอะไร |
|------|--------|----------|
| `ECPrivateKey.xml` | **Private Key** | Sign JWT เมื่อ EC ขอ token จาก WR |
| `ECPublicKey.xml` | **Public Key** | ส่งให้ WR เพื่อ verify JWT จาก EC |
| `WRPublicKey.xml` | **Public Key** | Verify JWT จาก WR เมื่อ WR ขอ token |

#### 🔄 Flow การแลกเปลี่ยน Keys

```
┌─────────────────────────────────────────────────────────────┐
│                    Initial Key Exchange                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   WR สร้าง Key Pair          EC สร้าง Key Pair              │
│   ├── WRPrivateKey.xml       ├── ECPrivateKey.xml           │
│   └── WRPublicKey.xml        └── ECPublicKey.xml            │
│                                                              │
│            WRPublicKey.xml ──────────────►                   │
│            ◄────────────── ECPublicKey.xml                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### ⚠️ สิ่งที่ต้องระวัง

| ❌ ไม่ต้องมีใน WR | ✅ ต้องมีใน WR |
|------------------|---------------|
| `ECPrivateKey.xml` | `ECPublicKey.xml` |

| ❌ ไม่ต้องมีใน EC | ✅ ต้องมีใน EC |
|------------------|---------------|
| `WRPrivateKey.xml` | `WRPublicKey.xml` |

> **หมายเหตุ:** ไฟล์ `ECPrivateKey.xml` ที่อยู่ใน `wr_v2/keys/` ใช้สำหรับ **ทดสอบเท่านั้น** - ใน Production ต้องลบไฟล์นี้ออก

---

## Authentication

### Authentication Header
ทุก API (ยกเว้น /auth/token) ต้องส่ง header:
```
Content-Type: application/json
x-app-token: <access_token>
```

> ⚠️ **สำคัญ:** ใช้ `x-app-token` header แทน `Authorization: Bearer`

### Authentication Flow
```
1. Client สร้าง JWT (client_assertion) signed ด้วย Private Key
2. Client ส่ง POST /auth/token พร้อม client_assertion
3. Server ตรวจสอบ JWT ด้วย Client's Public Key
4. Server ส่ง access_token กลับ
5. Client ใช้ access_token ใน x-app-token header สำหรับ requests ถัดไป
```

---

## API Endpoints Summary

### EC → WR (Exam Center เรียก Web Register)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /auth/token | ขอ access token |
| POST | /examEvents | สร้างรอบสอบใหม่ |
| PATCH | /examEvents/:id | อัพเดทข้อมูลรอบสอบ |
| GET | /examEvents/:id/examinees | ดึงรายชื่อผู้สมัครสอบ |
| POST | /examEvents/:id/examResults | ส่งผลสอบ |
| POST | /announcements | สร้างประกาศ |

### WR → EC (Web Register เรียก Exam Center)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /auth/token | ขอ access token |
| POST | /blocked-users/checks | ตรวจสอบ blocklist |
| GET | /examEvents/:id/examResults | ดึงผลสอบ |
| PATCH | /examEvents/:id/examinees | ส่งรายชื่อผู้สมัครสอบ |
| POST | /examEvents/:id/updateStatus | อัพเดทสถานะรอบสอบ |
| GET | /certificates/:id | ดาวน์โหลด certificate PDF |

---

# EC → WR Endpoints (Exam Center เรียก Web Register)

## 1. POST /auth/token - ขอ Access Token

### Request
```bash
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": "<JWT_TOKEN_SIGNED_WITH_EC_PRIVATE_KEY>",
    "client_id": "EC_AIMC_001"
  }'
```

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| grant_type | string | ✅ | Fix: `urn:ietf:params:oauth:grant-type:jwt-bearer` |
| client_assertion_type | string | ✅ | Fix: `urn:ietf:params:oauth:client-assertion-type:jwt-bearer` |
| client_assertion | string | ✅ | JWT signed ด้วย EC Private Key |
| client_id | string | ✅ | Client ID ของ EC (`EC_AIMC_001`) |

### JWT Payload (client_assertion)
```json
{
  "iss": "EC_AIMC_001",
  "sub": "EC_AIMC_001",
  "aud": "https://aimc.or.th/center/aimc_wr_v2/api/auth/token",
  "exp": 1738500000,
  "iat": 1738496400,
  "jti": "unique-request-id-123"
}
```

### Response Success (200)
```json
{
  "token_type": "custom",
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}
```

### Response Error (401)
```json
{
  "error": "unauthorized",
  "message": "Invalid client assertion"
}
```

---

## 2. POST /examEvents - สร้างรอบสอบใหม่

### Request
```bash
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/examEvents \
  -H "Content-Type: application/json" \
  -H "x-app-token: <access_token>" \
  -d '{
    "examEventId": "a1b2c3d4-e5f6-7890-abcd-1234567890ef",
    "name": "IC Plain - รอบสอบวันที่ 15 มี.ค. 2569",
    "examType": "IC_PLAIN",
    "registrationType": "INDIVIDUAL",
    "examCourseCodes": ["IC001", "IC002"],
    "examDate": "2026-03-15",
    "examSessions": [
      {
        "sessionNumber": 1,
        "sessionName": "รอบเช้า",
        "startTime": "09:00",
        "endTime": "12:00"
      },
      {
        "sessionNumber": 2,
        "sessionName": "รอบบ่าย",
        "startTime": "13:00",
        "endTime": "16:00"
      }
    ],
    "registrationStartDate": "2026-02-01",
    "registrationEndDate": "2026-03-10",
    "maxCapacity": 100,
    "status": "SCHEDULED",
    "examLocation": {
      "name": "ศูนย์สอบ AIMC",
      "address": "123 ถนนรัชดาภิเษก",
      "rooms": [
        {
          "roomId": "room-001",
          "roomName": "ห้องสอบ A",
          "capacity": 50
        },
        {
          "roomId": "room-002",
          "roomName": "ห้องสอบ B",
          "capacity": 50
        }
      ]
    }
  }'
```

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| examEventId | string (uuid) | ✅ | รหัสรอบสอบ (UUID) |
| name | string | ✅ | ชื่อรอบสอบ |
| examType | string | ✅ | ประเภทการสอบ: `IC_PLAIN`, `IC_COMPLEX_1`, `IC_COMPLEX_2`, etc. |
| registrationType | string | ✅ | ประเภทการสมัคร: `INDIVIDUAL`, `GROUP` |
| examCourseCodes | string[] | ✅ | รหัสวิชาสอบ |
| examDate | string | ✅ | วันสอบ (YYYY-MM-DD) |
| examSessions | object[] | ✅ | รอบเวลาสอบ |
| registrationStartDate | string | ✅ | วันเริ่มรับสมัคร |
| registrationEndDate | string | ✅ | วันสิ้นสุดรับสมัคร |
| maxCapacity | integer | ✅ | จำนวนที่นั่งสูงสุด |
| status | string | ✅ | สถานะ: `SCHEDULED`, `OPEN_FOR_REGISTRATION`, etc. |
| examLocation | object | ❌ | ข้อมูลสถานที่สอบ |

### Status Values
| Value | Description |
|-------|-------------|
| SCHEDULED | รอเปิดรับสมัคร |
| OPEN_FOR_REGISTRATION | เปิดรับสมัคร |
| TEMPORARILY_CLOSED | ปิดรับสมัครชั่วคราว |
| SESSION_FULL | รอบสอบเต็ม |
| CANCELED | ยกเลิกรอบสอบ |
| REGISTRATION_CLOSED | ปิดรับสมัคร |

### Response Success (200)
```json
{
  "result": true
}
```

### Response Error (400)
```json
{
  "error": "invalid_request",
  "message": "Missing required field: examEventId"
}
```

---

## 3. PATCH /examEvents/:id - อัพเดทข้อมูลรอบสอบ

### Request
```bash
curl -X PATCH https://aimc.or.th/center/aimc_wr_v2/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef \
  -H "Content-Type: application/json" \
  -H "x-app-token: <access_token>" \
  -d '{
    "name": "IC Plain - รอบสอบวันที่ 15 มี.ค. 2569 (อัพเดท)",
    "status": "OPEN_FOR_REGISTRATION",
    "maxCapacity": 150,
    "registrationEndDate": "2026-03-12"
  }'
```

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | ❌ | ชื่อรอบสอบ |
| examType | string | ❌ | ประเภทการสอบ |
| registrationType | string | ❌ | ประเภทการสมัคร |
| examCourseCodes | string[] | ❌ | รหัสวิชาสอบ |
| examDate | string | ❌ | วันสอบ |
| examSessions | object[] | ❌ | รอบเวลาสอบ |
| registrationStartDate | string | ❌ | วันเริ่มรับสมัคร |
| registrationEndDate | string | ❌ | วันสิ้นสุดรับสมัคร |
| maxCapacity | integer | ❌ | จำนวนที่นั่งสูงสุด |
| status | string | ❌ | สถานะรอบสอบ |
| examLocation | object | ❌ | ข้อมูลสถานที่สอบ |

### Response Success (200)
```json
{
  "result": true
}
```

### Response Error (404)
```json
{
  "error": "not_found",
  "message": "Exam event not found: a1b2c3d4-e5f6-7890-abcd-1234567890ef"
}
```

---

## 4. GET /examEvents/:id/examinees - ดึงรายชื่อผู้สมัครสอบ

### Request
```bash
curl -X GET "https://aimc.or.th/center/aimc_wr_v2/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examinees" \
  -H "Content-Type: application/json" \
  -H "x-app-token: <access_token>"
```

### Response Success (200)
```json
[
  {
    "id": "examinee-uuid-001",
    "title": "นาย",
    "firstName": "สมชาย",
    "lastName": "ใจดี",
    "idType": "CITIZEN_ID",
    "idNumber": "1234567890123",
    "email": "somchai@example.com",
    "phoneNumber": "0812345678",
    "registeredAt": "2026-02-05T10:30:00+07:00",
    "status": "REGISTERED",
    "examCourseCode": "IC001"
  },
  {
    "id": "examinee-uuid-002",
    "title": "นางสาว",
    "firstName": "สมหญิง",
    "lastName": "รักดี",
    "idType": "PASSPORT",
    "idNumber": "AB1234567",
    "email": "somying@example.com",
    "phoneNumber": "0898765432",
    "registeredAt": "2026-02-06T14:15:00+07:00",
    "status": "REGISTERED",
    "examCourseCode": "IC002"
  }
]
```

### idType Values
| Value | Description |
|-------|-------------|
| CITIZEN_ID | เลขบัตรประชาชน |
| PASSPORT | เลขหนังสือเดินทาง |

### status Values
| Value | Description |
|-------|-------------|
| REGISTERED | ลงทะเบียนแล้ว |
| CONFIRMED | ยืนยันแล้ว |
| CANCELLED | ยกเลิก |

### Response Error (404)
```json
{
  "error": "not_found",
  "message": "Exam event not found"
}
```

---

## 5. POST /examEvents/:id/examResults - ส่งผลสอบ

### Request
```bash
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examResults \
  -H "Content-Type: application/json" \
  -H "x-app-token: <access_token>" \
  -d '{
    "examineeResults": [
      {
        "examineeId": "examinee-uuid-001",
        "idType": "CITIZEN_ID",
        "idNumber": "1234567890123",
        "result": "PASSED"
      },
      {
        "examineeId": "examinee-uuid-002",
        "idType": "PASSPORT",
        "idNumber": "AB1234567",
        "result": "FAILED"
      },
      {
        "examineeId": "examinee-uuid-003",
        "idType": "CITIZEN_ID",
        "idNumber": "9876543210987",
        "result": "NONE"
      }
    ]
  }'
```

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| examineeResults | object[] | ✅ | รายการผลสอบ |
| examineeResults[].examineeId | string | ✅ | รหัสผู้สอบ (UUID) |
| examineeResults[].idType | string | ✅ | ประเภทเอกสาร: `CITIZEN_ID`, `PASSPORT` |
| examineeResults[].idNumber | string | ✅ | เลขเอกสาร |
| examineeResults[].result | string | ✅ | ผลสอบ: `PASSED`, `FAILED`, `NONE` |

### result Values
| Value | Description |
|-------|-------------|
| PASSED | สอบผ่าน |
| FAILED | สอบไม่ผ่าน |
| NONE | ไม่เข้าสอบ |

### Response Success (200)
```json
{
  "result": true
}
```

### Response Error (400)
```json
{
  "error": "invalid_request",
  "message": "Missing required field: examineeResults"
}
```

---

## 6. POST /announcements - สร้างประกาศ

### Request
```bash
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/announcements \
  -H "Content-Type: application/json" \
  -H "x-app-token: <access_token>" \
  -d '{
    "announcementId": "announcement-uuid-001",
    "topic": "แจ้งเลื่อนวันสอบ IC Plain รอบเดือนมีนาคม 2569",
    "message": "เนื่องจากมีการปรับปรุงระบบ จึงขอเลื่อนวันสอบจากวันที่ 15 มี.ค. เป็นวันที่ 20 มี.ค. 2569",
    "announcedAt": "2026-02-10T09:00:00+07:00"
  }'
```

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| announcementId | string (uuid) | ✅ | รหัสประกาศ |
| topic | string | ✅ | หัวข้อประกาศ |
| message | string | ✅ | เนื้อหาประกาศ |
| announcedAt | string | ✅ | วันที่ประกาศ (ISO 8601) |

### Response Success (200)
```json
{
  "result": true
}
```

### Response Error (400)
```json
{
  "error": "invalid_request",
  "message": "Missing required field: topic"
}
```

---

# WR → EC Endpoints (Web Register เรียก Exam Center)

ใช้ ExamCenterClient class ในการเรียก API ของ Exam Center

```php
require_once 'includes/bootstrap.php';

$client = new ExamCenterClient();
```

## 1. POST /auth/token - ขอ Access Token

### PHP Usage
```php
$client = new ExamCenterClient();
$result = $client->authenticate();

if ($result) {
    echo "Authenticated successfully!";
    echo "Token: " . $client->getAccessToken();
}
```

### Request (ที่ส่งไป EC)
```json
POST https://api.dev.sete.skooldio.dev/exg/api/auth/token
Content-Type: application/json

{
  "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "client_assertion": "<JWT_TOKEN_SIGNED_WITH_WR_PRIVATE_KEY>",
  "client_id": "AIMC_WR_001"
}
```

### Response Success (200)
```json
{
  "token_type": "custom",
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}
```

---

## 2. POST /blocked-users/checks - ตรวจสอบ Blocklist

### PHP Usage
```php
$client = new ExamCenterClient();

$users = array(
    array(
        'idType' => 'CITIZEN_ID',
        'idNumber' => '1234567890123'
    ),
    array(
        'idType' => 'PASSPORT',
        'idNumber' => 'AB1234567'
    )
);

$result = $client->checkBlocklistBatch($users, 'IC001');

if ($result['success']) {
    foreach ($result['data'] as $user) {
        if ($user['isBlocked']) {
            echo $user['idNumber'] . " is blocked: " . $user['reason'];
        }
    }
}
```

### Request (ที่ส่งไป EC)
```json
POST https://api.dev.sete.skooldio.dev/exg/api/blocked-users/checks
Content-Type: application/json
x-app-token: <access_token>

{
  "userIdentities": [
    {
      "idType": "CITIZEN_ID",
      "idNumber": "1234567890123"
    },
    {
      "idType": "PASSPORT",
      "idNumber": "AB1234567"
    }
  ],
  "examCourseCode": "IC001"
}
```

### Response Success (200)
```json
{
  "results": [
    {
      "idType": "CITIZEN_ID",
      "idNumber": "1234567890123",
      "isBlocked": false,
      "reason": null,
      "blockedUntil": null
    },
    {
      "idType": "PASSPORT",
      "idNumber": "AB1234567",
      "isBlocked": true,
      "reason": "Failed exam 3 times consecutively",
      "blockedUntil": "2026-06-15"
    }
  ]
}
```

---

## 3. GET /examEvents/:id/examResults - ดึงผลสอบ

### PHP Usage
```php
$client = new ExamCenterClient();

$result = $client->getExamResults('a1b2c3d4-e5f6-7890-abcd-1234567890ef');

if ($result['success']) {
    foreach ($result['data'] as $examinee) {
        echo $examinee['idNumber'] . ": " . $examinee['result'];
    }
}
```

### Request (ที่ส่งไป EC)
```
GET https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examResults
Content-Type: application/json
x-app-token: <access_token>
```

### Response Success (200)
```json
{
  "examineeResults": [
    {
      "examineeId": "examinee-uuid-001",
      "idType": "CITIZEN_ID",
      "idNumber": "1234567890123",
      "result": "PASSED"
    },
    {
      "examineeId": "examinee-uuid-002",
      "idType": "PASSPORT",
      "idNumber": "AB1234567",
      "result": "FAILED"
    }
  ]
}
```

---

## 4. PATCH /examEvents/:id/examinees - ส่งรายชื่อผู้สมัครสอบ

### PHP Usage
```php
$client = new ExamCenterClient();

$examinees = array(
    array(
        'id' => 'wr-examinee-001',
        'title' => 'นาย',
        'firstName' => 'สมชาย',
        'lastName' => 'ใจดี',
        'idType' => 'CITIZEN_ID',
        'idNumber' => '1234567890123',
        'email' => 'somchai@example.com',
        'phoneNumber' => '0812345678',
        'examCourseCode' => 'IC001'
    ),
    array(
        'id' => 'wr-examinee-002',
        'title' => 'นางสาว',
        'firstName' => 'สมหญิง',
        'lastName' => 'รักดี',
        'idType' => 'PASSPORT',
        'idNumber' => 'AB1234567',
        'email' => 'somying@example.com',
        'phoneNumber' => '0898765432',
        'examCourseCode' => 'IC002'
    )
);

$result = $client->enrollExaminees('a1b2c3d4-e5f6-7890-abcd-1234567890ef', $examinees);

if ($result['success']) {
    echo "Enrolled " . count($result['enrolled']) . " examinees";
}
```

### Request (ที่ส่งไป EC)
```json
PATCH https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examinees
Content-Type: application/json
x-app-token: <access_token>

[
  {
    "id": "wr-examinee-001",
    "title": "นาย",
    "firstName": "สมชาย",
    "lastName": "ใจดี",
    "idType": "CITIZEN_ID",
    "idNumber": "1234567890123",
    "email": "somchai@example.com",
    "phoneNumber": "0812345678",
    "examCourseCode": "IC001"
  },
  {
    "id": "wr-examinee-002",
    "title": "นางสาว",
    "firstName": "สมหญิง",
    "lastName": "รักดี",
    "idType": "PASSPORT",
    "idNumber": "AB1234567",
    "email": "somying@example.com",
    "phoneNumber": "0898765432",
    "examCourseCode": "IC002"
  }
]
```

### Response Success (200)
```json
{
  "result": true,
  "enrolled": [
    {
      "id": "wr-examinee-001",
      "ecExamineeId": "ec-examinee-uuid-001",
      "status": "REGISTERED"
    },
    {
      "id": "wr-examinee-002",
      "ecExamineeId": "ec-examinee-uuid-002",
      "status": "REGISTERED"
    }
  ]
}
```

---

## 5. POST /examEvents/:id/updateStatus - อัพเดทสถานะรอบสอบ

### PHP Usage
```php
$client = new ExamCenterClient();

// วิธีที่ 1: ใช้ method หลัก
$result = $client->updateExamEventStatus(
    'a1b2c3d4-e5f6-7890-abcd-1234567890ef',
    'OPEN_FOR_REGISTRATION'
);

// วิธีที่ 2: ใช้ convenience methods
$result = $client->openExamEventRegistration('a1b2c3d4-e5f6-7890-abcd-1234567890ef');
$result = $client->closeExamEventRegistration('a1b2c3d4-e5f6-7890-abcd-1234567890ef');
$result = $client->temporarilyCloseExamEvent('a1b2c3d4-e5f6-7890-abcd-1234567890ef');
$result = $client->markExamEventFull('a1b2c3d4-e5f6-7890-abcd-1234567890ef');
$result = $client->cancelExamEvent('a1b2c3d4-e5f6-7890-abcd-1234567890ef');

if ($result['success']) {
    echo "Status updated successfully";
}
```

### Request (ที่ส่งไป EC)
```json
POST https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/updateStatus
Content-Type: application/json
x-app-token: <access_token>

{
  "status": "OPEN_FOR_REGISTRATION"
}
```

### Status Values
| Value | Description |
|-------|-------------|
| SCHEDULED | รอเปิดรับสมัคร |
| OPEN_FOR_REGISTRATION | เปิดรับสมัคร |
| TEMPORARILY_CLOSED | ปิดรับสมัครชั่วคราว |
| SESSION_FULL | รอบสอบเต็ม |
| CANCELED | ยกเลิกรอบสอบ |
| REGISTRATION_CLOSED | ปิดรับสมัคร |

### Response Success (200)
```json
{
  "result": true
}
```

### Response Error (403)
```json
{
  "error": "forbidden",
  "message": "Status transition from CANCELED to OPEN_FOR_REGISTRATION is not allowed"
}
```

---

## 6. GET /certificates/:id - ดาวน์โหลด Certificate PDF

### PHP Usage
```php
$client = new ExamCenterClient();

// วิธีที่ 1: ดึง PDF content
$result = $client->getCertificate('cert-uuid-001');

if ($result['success']) {
    // Output PDF to browser
    header('Content-Type: ' . $result['content_type']);
    header('Content-Disposition: ' . $result['content_disposition']);
    echo $result['data'];
}

// วิธีที่ 2: บันทึกเป็นไฟล์
$result = $client->downloadCertificate('cert-uuid-001', '/path/to/save/certificate.pdf');

if ($result['success']) {
    echo "Saved to: " . $result['file_path'];
    echo "File size: " . $result['file_size'] . " bytes";
}
```

### Request (ที่ส่งไป EC)
```
GET https://api.dev.sete.skooldio.dev/exg/api/certificates/cert-uuid-001
Content-Type: application/json
x-app-token: <access_token>
Accept: application/pdf
```

### Response Success (200)
```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: inline; filename="certificate_1234567890123.pdf"

<PDF Binary Content>
```

### Response Error (404)
```json
{
  "error": "not_found",
  "message": "Not found certificate ID: cert-uuid-001"
}
```

---

## Error Responses (ทุก Endpoint)

### HTTP 400 - Bad Request
```json
{
  "error": "invalid_request",
  "message": "Request body is missing required fields or contains invalid values."
}
```

### HTTP 401 - Unauthorized
```json
{
  "error": "unauthorized",
  "message": "Access token is missing or invalid."
}
```

### HTTP 403 - Forbidden
```json
{
  "error": "forbidden",
  "message": "You do not have permission to access this resource."
}
```

### HTTP 404 - Not Found
```json
{
  "error": "not_found",
  "message": "The requested resource was not found."
}
```

### HTTP 500 - Internal Server Error
```json
{
  "error": "server_error",
  "message": "An unexpected error occurred. Please try again later."
}
```

---

## Directory Structure
```
wr_v2/
├── api/
│   ├── auth/
│   │   └── token.php           # EC→WR: Authentication
│   ├── examEvents/
│   │   ├── create.php          # EC→WR: Create exam event
│   │   ├── update.php          # EC→WR: Update exam event
│   │   ├── examinees.php       # EC→WR: Get examinees
│   │   └── examResults.php     # EC→WR: Receive results
│   ├── announcements/
│   │   └── create.php          # EC→WR: Create announcement
│   └── internal/
│       ├── register.php        # Internal: Register examinee
│       └── sync.php            # Internal: Sync to EC
├── classes/
│   ├── AuthMiddleware.php      # Token validation
│   ├── Cipher.php              # AES/RSA encryption
│   ├── Database.php            # PDO wrapper
│   ├── ExamCenterClient.php    # WR→EC API client
│   ├── ExamineeService.php     # Examinee business logic
│   ├── JwtToken.php            # JWT operations
│   ├── Logger.php              # Logging
│   ├── Response.php            # HTTP responses
│   └── Validator.php           # Input validation
├── config/
│   └── config.php              # Configuration
├── database/
│   └── schema.sql              # Database schema
├── includes/
│   └── bootstrap.php           # Autoloader
├── keys/
│   ├── WRPrivateKey.xml        # WR RSA Private Key
│   ├── WRPublicKey.xml         # WR RSA Public Key
│   └── ECPublicKey.xml         # EC RSA Public Key
├── logs/                       # Log files
├── certificates/               # Downloaded certificates
├── phpseclib/                  # Encryption library
├── .htaccess                   # URL rewriting
├── index.php                   # Router
├── generate_keys.php           # Key generator
└── README.md
```

---

## Testing with cURL

### ทดสอบ EC → WR ด้วย cURL

```bash
# 1. Get token
TOKEN=$(curl -s -X POST https://aimc.or.th/center/aimc_wr_v2/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": "<JWT>",
    "client_id": "EC_AIMC_001"
  }' | jq -r '.access_token')

# 2. Create exam event
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/examEvents \
  -H "Content-Type: application/json" \
  -H "x-app-token: $TOKEN" \
  -d '{"examEventId":"test-001","name":"Test Exam","examType":"IC_PLAIN",...}'

# 3. Get examinees
curl -X GET "https://aimc.or.th/center/aimc_wr_v2/api/examEvents/test-001/examinees" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $TOKEN"

# 4. Send results
curl -X POST https://aimc.or.th/center/aimc_wr_v2/api/examEvents/test-001/examResults \
  -H "Content-Type: application/json" \
  -H "x-app-token: $TOKEN" \
  -d '{"examineeResults":[{"examineeId":"ex-001","idType":"CITIZEN_ID","idNumber":"1234567890123","result":"PASSED"}]}'
```

### ทดสอบ WR → EC ด้วย cURL

```bash
# ============================================
# WR → EC API Testing with cURL
# Base URL: https://api.dev.sete.skooldio.dev/exg/api
# ============================================

# 1. Get Access Token from EC
# -------------------------------------------
# สร้าง JWT client_assertion ก่อน (ใช้ WR Private Key)
# JWT Payload:
# {
#   "iss": "AIMC_WR_001",
#   "sub": "AIMC_WR_001", 
#   "aud": "https://api.dev.sete.skooldio.dev/exg/api/auth/token",
#   "exp": 1738500000,
#   "iat": 1738496400,
#   "jti": "unique-request-id-123"
# }

EC_TOKEN=$(curl -s -X POST https://api.dev.sete.skooldio.dev/exg/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": "<JWT_SIGNED_WITH_WR_PRIVATE_KEY>",
    "client_id": "AIMC_WR_001"
  }' | jq -r '.access_token')

echo "EC Token: $EC_TOKEN"

# 2. Check Blocklist
# -------------------------------------------
curl -X POST https://api.dev.sete.skooldio.dev/exg/api/blocked-users/checks \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '{
    "userIdentities": [
      {
        "idType": "CITIZEN_ID",
        "idNumber": "1234567890123"
      },
      {
        "idType": "PASSPORT",
        "idNumber": "AB1234567"
      }
    ],
    "examCourseCode": "IC001"
  }'

# Expected Response:
# {
#   "results": [
#     {"idType": "CITIZEN_ID", "idNumber": "1234567890123", "isBlocked": false, "reason": null, "blockedUntil": null},
#     {"idType": "PASSPORT", "idNumber": "AB1234567", "isBlocked": true, "reason": "Failed exam 3 times", "blockedUntil": "2026-06-15"}
#   ]
# }

# 3. Enroll Examinees (PATCH)
# -------------------------------------------
curl -X PATCH "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examinees" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '[
    {
      "id": "wr-examinee-001",
      "title": "นาย",
      "firstName": "สมชาย",
      "lastName": "ใจดี",
      "idType": "CITIZEN_ID",
      "idNumber": "1234567890123",
      "email": "somchai@example.com",
      "phoneNumber": "0812345678",
      "examCourseCode": "IC001"
    },
    {
      "id": "wr-examinee-002",
      "title": "นางสาว",
      "firstName": "สมหญิง",
      "lastName": "รักดี",
      "idType": "PASSPORT",
      "idNumber": "AB1234567",
      "email": "somying@example.com",
      "phoneNumber": "0898765432",
      "examCourseCode": "IC002"
    }
  ]'

# Expected Response:
# {
#   "result": true,
#   "enrolled": [
#     {"id": "wr-examinee-001", "ecExamineeId": "ec-uuid-001", "status": "REGISTERED"},
#     {"id": "wr-examinee-002", "ecExamineeId": "ec-uuid-002", "status": "REGISTERED"}
#   ]
# }

# 4. Get Exam Results
# -------------------------------------------
curl -X GET "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/examResults" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN"

# Expected Response:
# {
#   "examineeResults": [
#     {"examineeId": "ex-001", "idType": "CITIZEN_ID", "idNumber": "1234567890123", "result": "PASSED"},
#     {"examineeId": "ex-002", "idType": "PASSPORT", "idNumber": "AB1234567", "result": "FAILED"}
#   ]
# }

# 5. Update Exam Event Status
# -------------------------------------------
# Open registration
curl -X POST "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/updateStatus" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '{"status": "OPEN_FOR_REGISTRATION"}'

# Close registration
curl -X POST "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/updateStatus" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '{"status": "REGISTRATION_CLOSED"}'

# Mark as full
curl -X POST "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/updateStatus" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '{"status": "SESSION_FULL"}'

# Cancel event
curl -X POST "https://api.dev.sete.skooldio.dev/exg/api/examEvents/a1b2c3d4-e5f6-7890-abcd-1234567890ef/updateStatus" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -d '{"status": "CANCELED"}'

# Expected Response:
# {"result": true}

# 6. Download Certificate PDF
# -------------------------------------------
# Download and save to file
curl -X GET "https://api.dev.sete.skooldio.dev/exg/api/certificates/cert-uuid-001" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -H "Accept: application/pdf" \
  -o certificate.pdf

# Check response headers
curl -I -X GET "https://api.dev.sete.skooldio.dev/exg/api/certificates/cert-uuid-001" \
  -H "Content-Type: application/json" \
  -H "x-app-token: $EC_TOKEN" \
  -H "Accept: application/pdf"

# Expected Headers:
# HTTP/1.1 200 OK
# Content-Type: application/pdf
# Content-Disposition: inline; filename="certificate_1234567890123.pdf"
```

---

### ทดสอบ WR → EC ด้วย PHP

```php
<?php
require_once 'includes/bootstrap.php';

// Initialize client
$client = new ExamCenterClient();

// Test 1: Authenticate
echo "=== Test Authentication ===\n";
if ($client->authenticate()) {
    echo "✓ Authenticated successfully\n";
} else {
    echo "✗ Authentication failed\n";
    exit;
}

// Test 2: Check blocklist
echo "\n=== Test Blocklist ===\n";
$users = array(
    array('idType' => 'CITIZEN_ID', 'idNumber' => '1234567890123')
);
$result = $client->checkBlocklistBatch($users, 'IC001');
print_r($result);

// Test 3: Update status
echo "\n=== Test Update Status ===\n";
$result = $client->updateExamEventStatus('exam-event-id', 'OPEN_FOR_REGISTRATION');
print_r($result);

// Test 4: Enroll examinees
echo "\n=== Test Enroll Examinees ===\n";
$examinees = array(
    array(
        'id' => 'wr-001',
        'title' => 'นาย',
        'firstName' => 'ทดสอบ',
        'lastName' => 'ระบบ',
        'idType' => 'CITIZEN_ID',
        'idNumber' => '1234567890123',
        'email' => 'test@example.com',
        'phoneNumber' => '0812345678',
        'examCourseCode' => 'IC001'
    )
);
$result = $client->enrollExaminees('exam-event-id', $examinees);
print_r($result);

// Test 5: Get results
echo "\n=== Test Get Results ===\n";
$result = $client->getExamResults('exam-event-id');
print_r($result);

// Test 6: Get certificate
echo "\n=== Test Get Certificate ===\n";
$result = $client->downloadCertificate('cert-id', 'test_cert.pdf');
print_r($result);
```

---

## Security Notes

1. **HTTPS Required** - ใช้ HTTPS เสมอใน production
2. **Private Key Security** - เก็บ Private Key ให้ปลอดภัย ห้ามเปิดเผย
3. **File Permissions** - ตั้ง permissions: 755 (dirs), 644 (files), 600 (keys)
4. **IP Whitelist** - ควรจำกัด IP ที่เรียก API ได้
5. **Token Expiry** - Token หมดอายุใน 1 ชั่วโมง ต้อง refresh ใหม่
6. **Key Rotation** - เปลี่ยน keys ทุก 6-12 เดือน
7. **Logging** - ตรวจสอบ logs เป็นประจำ

---

## Changelog

### v2.0.0 (2026-02-02)
- Initial release with full EC↔WR integration
- JWT-based authentication with x-app-token header
- All endpoints compliant with API specification
- Support for PDF certificate download
- Comprehensive error handling
- Docker support (PHP 5.6 + MySQL 5.5)

---

## Support
**Contact:** AIMC Knowledge Team  
**Email:** support@aimc.or.th

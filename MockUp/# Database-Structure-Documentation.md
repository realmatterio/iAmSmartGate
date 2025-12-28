# iAmSmartGate Database Structure and Content

## Overview

The system uses **SQLite** database (`iamsmartgate.db`) located in `backend/instance/` with **5 main tables** for managing visitor access control with cryptographic security.

---

## Database Tables

### 1. Users Table (`users`)

Stores visitor/user accounts with cryptographic key pairs.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `iamsmart_id` | String(100) | PRIMARY KEY | Unique iAmSmart user identifier |
| `public_key` | Text | NOT NULL | User's RSA public key for digital signatures |
| `private_key_ref` | String(200) | NOT NULL | HSM reference to user's private key |
| `device_id` | String(100) | | Device identifier for the user |
| `created_at` | DateTime | DEFAULT utcnow | Account creation timestamp |

**Relationships:** 
- One-to-Many with `passes` table

**Methods:**
- `to_dict()` - Serializes user data to JSON (excludes private_key_ref)

---

### 2. Gates Table (`gates`)

Represents physical access gates/tablets at various sites.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `tablet_id` | String(100) | PRIMARY KEY | Unique gate/tablet identifier |
| `gps_location` | String(200) | NOT NULL | GPS coordinates (latitude,longitude format) |
| `public_key` | Text | NOT NULL | Gate's RSA public key |
| `private_key_ref` | String(200) | NOT NULL | HSM reference to gate's private key |
| `site_id` | String(50) | NOT NULL | Associated site identifier |
| `created_at` | DateTime | DEFAULT utcnow | Gate registration timestamp |

**Demo Data** (automatically created on initialization):

| Tablet ID | Site ID | Site Name | GPS Location |
|-----------|---------|-----------|--------------|
| GATE001 | SITE001 | Main Campus | 22.3193,114.1694 |
| GATE002 | SITE002 | Student Halls | 22.3200,114.1700 |
| GATE003 | SITE003 | Research Center | 22.3210,114.1710 |
| GATE004 | SITE004 | Library | 22.3220,114.1720 |

**Methods:**
- `to_dict()` - Serializes gate data to JSON (excludes private_key_ref)

---

### 3. Passes Table (`passes`)

Core table managing the complete visit pass lifecycle.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `pass_id` | String(100) | PRIMARY KEY | Unique pass identifier (UUID format) |
| `iamsmart_id` | String(100) | FOREIGN KEY, NOT NULL | References users.iamsmart_id |
| `site_id` | String(50) | NOT NULL | Target site for visit |
| `purpose_id` | String(50) | NOT NULL | Purpose code for visit |
| `visit_date_time` | DateTime | NOT NULL | Scheduled visit date and time |
| `status` | String(20) | DEFAULT 'In Process' | Current pass status |
| `qr_signature` | Text | | Digitally signed QR code data |
| `created_timestamp` | DateTime | DEFAULT utcnow | Pass application time |
| `approved_timestamp` | DateTime | | Admin approval time |
| `used_timestamp` | DateTime | | When pass was scanned at gate |
| `expiry_timestamp` | DateTime | | Pass expiration time |
| `used_flag` | Boolean | DEFAULT False | Whether pass has been used |
| `revoked_flag` | Boolean | DEFAULT False | Whether pass has been revoked |
| `device_id` | String(100) | | User's device identifier |

**Pass Status Values:**
- `In Process` - Awaiting admin approval
- `Pass` - Approved and active (ready for use)
- `No Pass` - Rejected by administrator
- `Used` - Successfully scanned at gate
- `Revoked` - Cancelled by administrator

**Site Definitions:**
```
SITE001 → Main Campus
SITE002 → Student Halls
SITE003 → Research Center
SITE004 → Library
```

**Purpose Codes:**
```
PURP001 → Meeting
PURP002 → Tour
PURP003 → Delivery
PURP004 → Maintenance
```

**Relationships:**
- Many-to-One with `users` table (via iamsmart_id)

**Methods:**
- `to_dict()` - Serializes pass data to JSON

---

### 4. Audit Logs Table (`audit_logs`)

Comprehensive audit trail of all system events for security and compliance.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `log_id` | Integer | PRIMARY KEY, AUTO INCREMENT | Unique log entry identifier |
| `timestamp` | DateTime | NOT NULL, DEFAULT utcnow | Event occurrence time |
| `event_type` | String(50) | NOT NULL | Type of event |
| `user_id` | String(100) | | Related user ID (if applicable) |
| `gate_id` | String(100) | | Related gate ID (if applicable) |
| `pass_id` | String(100) | | Related pass ID (if applicable) |
| `result` | String(50) | | Event result (success/failure/denied) |
| `details` | Text | | Additional event details (JSON format) |

**Event Types:**
- `login` - User or gate authentication attempt
- `approval` - Pass approval or rejection by admin
- `scan` - QR code scan at gate
- `revoke` - Pass revocation by admin
- `pause` - System or site pause/resume action

**Example Log Entries:**
```json
{
  "event_type": "login",
  "user_id": "HKID123456A",
  "result": "success",
  "details": "{\"method\": \"iAmSmart\", \"device_id\": \"DEV001\"}"
}

{
  "event_type": "scan",
  "gate_id": "GATE001",
  "pass_id": "PASS-UUID-123",
  "result": "success",
  "details": "{\"gps_matched\": true, \"signature_valid\": true}"
}
```

**Retention Policy:**
- Logs retained for 30 days (configurable)
- Cleaned up by background job

**Methods:**
- `to_dict()` - Serializes log entry to JSON

---

### 5. System State Table (`system_state`)

Stores system-wide configuration and operational state.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `key` | String(50) | PRIMARY KEY | Configuration key name |
| `value` | Text | NOT NULL | Configuration value (often JSON) |
| `updated_at` | DateTime | DEFAULT utcnow, AUTO UPDATE | Last modification timestamp |

**Initial Configuration:**

| Key | Value | Description |
|-----|-------|-------------|
| `global_pause` | `"false"` | System-wide pause status (string boolean) |
| `site_pauses` | `"{}"` | JSON object of site-specific pause states |

**Example Site Pauses:**
```json
{
  "SITE001": false,
  "SITE002": true,
  "SITE003": false,
  "SITE004": false
}
```

**Methods:**
- `to_dict()` - Serializes state data to JSON

---

## Database Features

### Security Architecture

1. **Cryptographic Key Management**
   - RSA key pairs generated for all users and gates
   - Private keys stored via HSM references (dummy implementation in PoC)
   - Public keys used for signature verification

2. **Digital Signatures**
   - QR codes contain digitally signed pass data
   - Signed using user's private key
   - Verified using user's public key at gate

3. **Authentication**
   - JWT tokens for API authentication
   - Token expiration: 24 hours (configurable)
   - iAmSmart integration for user verification (dummy in PoC)

4. **QR Code Security**
   - Time-based expiration: 60 seconds
   - Includes timestamp in signature
   - Dynamically regenerated on each request
   - Prevents replay attacks

### Access Control Flow

```
1. USER REGISTRATION
   User → /api/login
   ├─ Authenticate via iAmSmart (dummy: password "demo123")
   ├─ Generate RSA key pair
   ├─ Create User record
   ├─ Return JWT token
   └─ Log: event_type="login"

2. PASS APPLICATION
   User → /api/apply-pass
   ├─ Validate JWT token
   ├─ Create Pass record (status="In Process")
   ├─ Set visit_date_time, site_id, purpose_id
   └─ Return pass_id

3. ADMIN APPROVAL
   Admin → /admin/approve-pass/{pass_id}
   ├─ Update status to "Pass"
   ├─ Set approved_timestamp
   ├─ Calculate expiry_timestamp (default: 24 hours)
   ├─ Log: event_type="approval", result="approved"
   └─ Return success

4. QR CODE GENERATION
   User → /api/get-qr/{pass_id}
   ├─ Validate pass status="Pass"
   ├─ Check not expired, used, or revoked
   ├─ Create payload: {pass_id, timestamp, user_id}
   ├─ Sign payload with user's private key
   ├─ Update qr_signature field
   └─ Return signed QR data (valid 60 seconds)

5. GATE SCANNING
   Gate → /api/scan-qr
   ├─ Verify QR signature with user's public key
   ├─ Check timestamp (< 60 seconds old)
   ├─ Validate pass status="Pass"
   ├─ Check expiry_timestamp
   ├─ Verify GPS location matches gate
   ├─ Check site/system pause status
   ├─ Update: status="Used", used_flag=true, used_timestamp
   ├─ Log: event_type="scan", result="success/denied"
   └─ Return access granted/denied

6. PASS REVOCATION
   Admin → /admin/revoke-pass/{pass_id}
   ├─ Update revoked_flag=true
   ├─ Update status to "Revoked"
   ├─ Log: event_type="revoke"
   └─ Prevent further use
```

### Business Logic Rules

1. **Pass Lifecycle**
   - Pass must be approved before QR generation
   - QR codes regenerate every 60 seconds
   - Single-use: pass marked as "Used" after scan
   - Cannot reuse or transfer passes
   - Admin can revoke at any time

2. **Expiration Handling**
   - Default expiry: 24 hours after approval
   - Configurable per approval (expiry_hours parameter)
   - Background job checks expiration every 5 minutes
   - Expired passes automatically denied at gate

3. **GPS Validation**
   - Gate must be at registered location
   - User GPS compared to gate GPS
   - Tolerance: ~100 meters (demo implementation)
   - Location spoofing warning (browser-based in PoC)

4. **Pause Controls**
   - Global pause: disables entire system
   - Site-specific pause: disables access to specific site
   - Pause status checked before granting access
   - Can pause/resume via admin console

5. **Audit Requirements**
   - All authentication attempts logged
   - All pass approvals/rejections logged
   - All gate scans logged (success and failure)
   - All revocations logged
   - Log retention: 30 days

### Background Jobs

**Implemented Jobs:**
1. **Pass Expiration Checker**
   - Interval: 5 minutes (300 seconds)
   - Updates expired passes
   - Cleans up old data

2. **Audit Log Cleanup**
   - Retention: 30 days
   - Removes old audit entries
   - Maintains compliance

---

## Database Configuration

**Location:** `backend/instance/iamsmartgate.db`

**Connection String:** `sqlite:///iamsmartgate.db` (relative to Flask app)

**Settings (config.py):**
```python
SQLALCHEMY_DATABASE_URI = 'sqlite:///iamsmartgate.db'
SQLALCHEMY_TRACK_MODIFICATIONS = False
JWT_EXPIRATION_HOURS = 24
QR_EXPIRATION_SECONDS = 60
PASS_EXPIRATION_CHECK_INTERVAL = 300  # 5 minutes
AUDIT_LOG_RETENTION_DAYS = 30
```

**Initialization:**
- Database created automatically on first run
- Tables created via SQLAlchemy models
- Demo gates auto-populated
- System state initialized with default values

---

## API Integration Points

### User Endpoints
- `POST /api/login` - Creates/retrieves User record
- `POST /api/apply-pass` - Creates Pass record
- `GET /api/my-passes` - Queries user's passes
- `GET /api/get-qr/{pass_id}` - Generates signed QR

### Gate Endpoints
- `POST /api/gate-login` - Validates gate credentials
- `POST /api/scan-qr` - Validates and consumes pass

### Admin Endpoints
- `GET /admin/pending-passes` - Queries passes with status="In Process"
- `GET /admin/all-passes` - Queries all passes
- `POST /admin/approve-pass/{pass_id}` - Updates pass to approved
- `POST /admin/reject-pass/{pass_id}` - Updates pass to rejected
- `POST /admin/revoke-pass/{pass_id}` - Revokes active pass
- `POST /admin/pause-system` - Updates system_state
- `POST /admin/pause-site` - Updates site_pauses
- `GET /admin/statistics` - Aggregates pass counts
- `GET /admin/audit-logs` - Queries audit_logs

---

## Schema Diagram

```
┌─────────────────┐
│     users       │
│─────────────────│
│ iamsmart_id (PK)│───┐
│ public_key      │   │
│ private_key_ref │   │
│ device_id       │   │
│ created_at      │   │
└─────────────────┘   │
                      │
                      │ 1:N
                      │
┌─────────────────┐   │
│     passes      │   │
│─────────────────│   │
│ pass_id (PK)    │   │
│ iamsmart_id (FK)│◄──┘
│ site_id         │
│ purpose_id      │
│ visit_date_time │
│ status          │
│ qr_signature    │
│ timestamps...   │
│ flags...        │
└─────────────────┘

┌─────────────────┐
│     gates       │
│─────────────────│
│ tablet_id (PK)  │
│ gps_location    │
│ public_key      │
│ private_key_ref │
│ site_id         │
│ created_at      │
└─────────────────┘

┌─────────────────┐
│  audit_logs     │
│─────────────────│
│ log_id (PK)     │
│ timestamp       │
│ event_type      │
│ user_id         │
│ gate_id         │
│ pass_id         │
│ result          │
│ details         │
└─────────────────┘

┌─────────────────┐
│  system_state   │
│─────────────────│
│ key (PK)        │
│ value           │
│ updated_at      │
└─────────────────┘
```

---

## Testing & Demo Data

**Test Credentials:**
- Any iAmSmart ID with password: `demo123`

**Pre-configured Gates:**
- GATE001 through GATE004 (see Gates table above)

**Sample Workflow:**
```bash
# 1. User login
curl -X POST http://localhost:5000/api/login \
  -H "Content-Type: application/json" \
  -d '{"iamsmart_id": "HKID123456A", "password": "demo123", "device_id": "TEST001"}'

# 2. Apply for pass
curl -X POST http://localhost:5000/api/apply-pass \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"site_id": "SITE001", "purpose_id": "PURP001", "visit_date_time": "2025-12-29T10:00:00"}'

# 3. Admin approve
curl -X POST http://localhost:5000/admin/approve-pass/{pass_id} \
  -H "Content-Type: application/json" \
  -d '{"expiry_hours": 24}'

# 4. Get QR code
curl -X GET http://localhost:5000/api/get-qr/{pass_id} \
  -H "Authorization: Bearer {token}"

# 5. Scan at gate
curl -X POST http://localhost:5000/api/scan-qr \
  -H "Content-Type: application/json" \
  -d '{"qr_data": "{signed_qr_data}", "tablet_id": "GATE001", "gps_location": "22.3193,114.1694"}'
```

---

## Future Enhancements

1. **Production Database**
   - Migrate from SQLite to PostgreSQL/MySQL
   - Add database indexes for performance
   - Implement connection pooling

2. **Real HSM Integration**
   - Replace dummy HSM with actual hardware security module
   - Secure key storage and operations
   - Key rotation policies

3. **Enhanced Security**
   - Multi-factor authentication
   - Biometric verification
   - Blockchain audit trail

4. **Scalability**
   - Database sharding
   - Read replicas
   - Caching layer (Redis)

5. **Analytics**
   - Real-time dashboard
   - Usage statistics
   - Anomaly detection

---

**Last Updated:** December 28, 2025  
**Version:** PoC 1.0  
**Database Schema Version:** 1.0

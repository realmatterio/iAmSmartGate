# iAmSmartGate System (PoC)
## Quantum-Secure Public Access Control with Hong Kong iAM Smart eID 

# Software Specification (Functional Mock-Up Demo)

## 1. System Overview

### 1.1 Project Description
The iAmSmart Public Access Gate App System is a functional mock-up demo designed to simulate secure access control for public sites using Hong Kong's iAmSmart eID system. It integrates user authentication, visit pass applications, dynamic QR code generation, and gate scanning for access verification. The system comprises three main components:
- **User Wallet Web App**: A client-side HTML-based application running on mobile devices for user login, pass application, and QR code presentation.
- **Access Gate QR Code Reader Web App**: A client-side HTML-based application on embedded tablets for scanning and verifying QR codes at access gates.
- **Backend Access Control Server**: A Python-based server on Google Cloud Platform (GCP) Ubuntu VM handling authentication, data management, and access logic.

The system emphasizes security through public/private key pairs stored server-side in a simulated cloud Hardware Security Module (HSM), digitally signed QR codes, and standard HTTPS channels. **Note**: Private keys remain on the backend (not true wallet architecture); clients only receive public keys. All communications use standard TLS 1.3. It uses dummy routines for external integrations (e.g., iAmSmart authentication and HSM).

### 1.2 Objectives
- Provide a secure, user-friendly interface for applying and using site visit passes.
- Simulate real-time access control with dynamic, time-limited QR codes.
- Demonstrate backend management of user/gate accounts, approvals, and access logging.
- Ensure single-use passes and exception handling for pauses.

### 1.3 Scope
- In-Scope: User login, pass application/approval, QR code generation/scanning, access verification, basic database management, console visualization.
- Out-of-Scope: Full production integration with real iAmSmart API or cloud HSM; advanced error handling; scalability optimizations; mobile app packaging (web-based only).

### 1.4 Assumptions and Dependencies
- Users have mobile devices with internet, camera, and browser support for HTML5.
- Gate devices are embedded tablets with front camera and GPS capabilities running standard web browsers.
- Backend runs on GCP Ubuntu VM with Python3.
- Dummy stubs for iAmSmart authentication and HSM interactions.
- Simple database (e.g., SQLite) for mock-up; no production database required.
- Internet connectivity for all components.
- **Demo Limitations**: GPS location can be spoofed via browser APIs (no device attestation). Post-quantum cryptography is not implemented; system uses standard TLS 1.3 which is vulnerable to future quantum attacks. These are acceptable for demonstration purposes only.

## 2. Functional Requirements

### 2.1 User Wallet Web App
- **Technology**: HTML/JavaScript web app, responsive for mobile devices.
- **Features**:
  - **Login**: Authenticate via iAmSmart ID and password over standard HTTPS (TLS 1.3). Call backend to verify credentials using dummy iAmSmart routine.
  - **Wallet Page**: Display user's public key associated with iAmSmart ID. **Note**: Private keys are stored and managed server-side only via backend dummy HSM (not a true client-side wallet).
  - **Apply Visit Pass**: Form to input iAmSmart ID, mobile device ID (auto-detected if possible), select site (e.g., dropdown from predefined list), purpose (e.g., dropdown: meeting, tour), date-time (calendar picker). Submit to backend for approval.
  - **Status Page**: Query backend for application status (In Process, Pass, No Pass). Display user details linked to iAmSmart ID.
  - **Pass Page**: List approved passes (latest first). Active passes in color; outdated/inactive/revoked in grey. On selection, request dynamic QR code from backend (valid for 1 minute). Display QR full-screen with a 1-minute count-down timer.
  - **QR Code Presentation**: QR data includes iAmSmart ID, mobile device ID, site ID, purpose ID, date-time, server timestamp. **Digitally signed** by backend using user's private key (stored server-side). QR payload is JSON with data and signature.
  - **Error Handling**: Alerts for invalid inputs, network issues, expired sessions, or revoked passes.

### 2.2 Access Gate QR Code Reader Web App
- **Technology**: HTML/JavaScript web app on embedded tablet, utilizing browser APIs for camera and GPS.
- **Features**:
  - **Login**: Fixed tablet ID and password over standard HTTPS (TLS 1.3). Verify GPS location matches preset (note: browser geolocation is spoofable; for demo only).
  - **Wallet Page**: Display gate's public key associated with tablet ID. **Note**: Private keys stored server-side only via backend dummy HSM.
  - **Scan Page**: Activate front camera to detect QR codes. On detection, display popped-up QR image with "Rescan" button.
  - **QR Processing**: Send QR data (JSON with payload and signature) to backend for signature verification and validity check. Backend verifies signature using user's public key and checks pass status. Display result (Pass/No Pass/Revoked) from backend response.
  - **Error Handling**: Alerts for scan failures, invalid QR, invalid signature, revoked passes, or connection issues.

### 2.3 Backend Access Control Server
- **Technology**: Python application on GCP Ubuntu VM. Use frameworks like Flask/Django for API, SQLite for database.
- **Features**:
  - **User Account Management**: API endpoints to create/manage user accounts by iAmSmart ID. Dummy call to iAmSmart for auth. Dummy HSM for key generation/distribution and signatures. Private keys stored server-side only.
  - **Gate Account Management**: Register/manage gates by tablet ID and GPS. Validate logins.
  - **Admin Interface**: Web-based console or API endpoints for:
    - Manual approval/rejection of pending pass requests
    - Revocation of approved/active passes
    - System-wide pause toggle (blocks all access verification)
    - Site-specific pause toggles (blocks verification for specific sites)
    - View logs and statistics
  - **Access Control Logic**:
    - Receive visit pass requests; store in database with status "In Process".
    - Approval workflow: Manual via admin interface or auto-approve for demo; update status to "Pass" or "No Pass".
    - Generate dynamic QR code on request (if approved, not used, and not revoked): **Sign** data (iAmSmart ID, device ID, site/purpose/date-time, timestamp) with user's private key via dummy HSM. Return JSON payload with signature. Expire after 1 minute.
    - On QR scan: **Verify signature** using user's public key; check against database for validity, single-use enforcement (mark as used using **atomic database transaction** to prevent race conditions). Return Pass/No Pass/Revoked.
  - **Database Schema** (Simple SQLite):
    - Users: iAmSmart_ID (PK), public_key, private_key_ref (HSM reference), device_ID.
    - Gates: tablet_ID (PK), GPS_location, public_key, private_key_ref (HSM reference).
    - Passes: pass_ID (PK), iAmSmart_ID (FK), site_ID, purpose_ID, date_time, status (In Process/Pass/No Pass/Used/Revoked), qr_signature, created_timestamp, approved_timestamp, used_timestamp, expiry_timestamp, used_flag, revoked_flag.
    - AuditLog: log_ID (PK), timestamp, event_type (login/approval/scan/revoke/pause), user_ID, gate_ID, pass_ID, result, details.
  - **Server Console**:
    - Scrolling log of approvals, access attempts, revocations, user details from AuditLog table.
    - Grouped views by site (e.g., Main Campus, Student Halls): Counts of approved, requested, used, revoked passes.
    - Controls: Buttons to pause all access or specific sites (sets pause flag in DB to block verifications).
    - Admin actions: Approve/reject pending requests, revoke active passes, view detailed pass history.
  - **Background Jobs**:
    - **Pass Expiration Job**: Periodic task (e.g., every 5 minutes) to mark passes as expired based on expiry_timestamp and update status.
    - **Audit Log Cleanup**: Optional periodic task to archive or purge old audit entries (configurable retention period).
  - **Concurrency Control**: Use database transactions with row-level locking for QR validation to ensure atomic single-use enforcement (e.g., SELECT FOR UPDATE before marking pass as used).
  - **Error Handling**: Log exceptions to AuditLog; return meaningful errors to clients (e.g., 401 Unauthorized, 403 Forbidden, 409 Conflict for already-used passes).
  - **Debug mode**: Debug Log for server processes, user flow, gate access procedure, all API requests and responses.

### 2.4 Integration Points
- **APIs**: RESTful endpoints over standard HTTPS (TLS 1.3) (e.g., /login, /apply-pass, /get-qr, /scan-qr, /admin/approve, /admin/revoke, /admin/pause). Use JSON for data exchange.
- **Security**: All communications encrypted via standard TLS 1.3. QR codes contain digitally signed payloads (not encrypted). Signature verification prevents tampering.
- **Dummy Integrations**:
  - iAmSmart: Stub function returning mock auth success.
  - Cloud HSM: Stub for server-side key storage/signing (e.g., in-memory simulation with persistent file storage).

## 3. Non-Functional Requirements

### 3.1 Performance
- Response time: <2 seconds for API calls; QR generation <5 second.
- Scalability: Mock-up for 10-50 concurrent users; no optimization needed.
- Uptime: 99% for demo; basic monitoring on GCP.

### 3.2 Security
- Authentication: Token-based (JWT) post-login.
- Digital Signatures: QR data digitally signed with user's private key (server-side); verified with public key. Prevents tampering.
- Transport Security: Standard HTTPS (TLS 1.3) for all communications.
- Data Privacy: Store minimal PII; comply with mock GDPR/HK data protection. Private keys stored server-side only.
- Vulnerabilities: Input validation, rate limiting to prevent abuse.
- **Demo Limitations**: No post-quantum cryptography (vulnerable to future quantum attacks); GPS location via browser API is spoofable (no device attestation). Acceptable for demonstration purposes only.

### 3.3 Usability
- Responsive design for mobile/tablet.
- Intuitive UI: Simple navigation, clear buttons (e.g., "Apply", "Scan", "Rescan").
- Accessibility: Basic WCAG compliance (e.g., alt text, keyboard nav).

### 3.4 Reliability
- Exception Handling: Graceful degradation (e.g., offline alerts).
- Logging: Console and file-based for debugging.

### 3.5 Maintainability
- Code Structure: Modular (e.g., separate files for API, DB, console).
- Documentation: Inline comments; API docs via Swagger/Postman.

## 4. Architecture

### 4.1 High-Level Diagram
- **Clients**: User Mobile (Browser) ↔ Gate Tablet (Browser)
- **Backend**: GCP VM (Python Server) ↔ SQLite DB
- **Externals**: Dummy iAmSmart/HSM stubs.
- Flow: Clients ↔ APIs (HTTPS) ↔ Server Logic ↔ DB.

### 4.2 Data Flow
1. User logs in → Backend verifies (dummy iAmSmart) → Returns token/user public key.
2. User applies pass → Backend stores request with status "In Process" → Admin approves/rejects via admin interface → Updates status → Notifies user.
3. User requests QR → Backend generates payload, signs with user's private key (server-side) → Returns JSON with data + signature to app → App displays QR with 1-min timer.
4. Gate scans QR → Sends JSON to backend → Backend verifies signature with user's public key → Checks validity/status in DB using atomic transaction → Marks as used if valid → Returns result (Pass/No Pass/Revoked) → Gate displays result.
5. Background job periodically expires old passes based on expiry_timestamp.
6. Admin can revoke passes or pause access via admin interface → Backend updates DB flags → Affects future verifications.

## 5. Implementation Guidelines
- **Frontend**: HTML5, CSS (Bootstrap for responsiveness), JavaScript (e.g., ZXing for QR scanning, qrcode.js for generation). Both user and gate apps remain web-based.
- **Backend**: Python 3, Flask for API, SQLAlchemy for DB with transaction support (BEGIN/COMMIT for atomic operations), APScheduler for background jobs, Tkinter/Streamlit for admin console UI.
- **Cryptography**: Use Python `cryptography` library for digital signatures (e.g., RSA or ECDSA). Dummy HSM implemented as secure file storage or in-memory dict.
- **Deployment**: GCP VM setup with Ubuntu, NGINX for serving, Let's Encrypt SSL certs for standard TLS 1.3.
- **Testing**: Unit tests for API/logic/signature verification; integration tests for concurrency (simulate multiple gate scans); manual end-to-end for demo.

## 6. Risks and Mitigations
- Risk: Dummy integrations fail realism → Mitigation: Use configurable stubs with realistic delays/responses.
- Risk: QR expiration timing issues → Mitigation: Server-side timestamp checks; reject if >1 minute old.
- Risk: GPS inaccuracies/spoofing → Mitigation: Loose validation for demo; document limitation that browser geolocation is not secure.
- Risk: Race conditions on QR validation → Mitigation: Use database transactions with row locking (SELECT FOR UPDATE).
- Risk: No post-quantum security → Mitigation: Document as demo limitation; standard TLS 1.3 used.
- Risk: Private keys on server create single point of failure → Mitigation: Document as demo architecture (not production-ready); consider future migration to client-side keys.
- Risk: Pass revocation not propagated in time → Mitigation: Backend checks revoked_flag on every scan; no caching of pass status.


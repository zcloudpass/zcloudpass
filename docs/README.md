# zCloudPass System Diagrams

Comprehensive descriptions for all four diagrams that show the complete system (frontend + backend) working together.

## 1. Combined Use Case Diagram

<div align="center">
    <img src="assets/Use Case Diagram.png" alt="Combined Use Case Diagram" width="720">
</div>

### System: zCloudPass Password Manager (Full Stack)

### Actors

| Actor | Type | Description |
|-------|------|-------------|
| User | Primary | End user interacting with the desktop application |
| Tauri Desktop Shell | System | Native OS window embedding the React app |
| Backend API Server | External System | Rust/Axum REST API server |
| PostgreSQL Database | External System | Data persistence layer |

### Use Cases

#### Authentication & Account Management

| # | Use Case | Actor | Pre-condition | Description |
|---|----------|-------|---------------|-------------|
| UC-1 | Register Account | User | None | User provides email + master password (≥8 chars) via frontend. Client validates, creates empty vault `{entries:[]}`, encrypts it with PBKDF2+AES-256-GCM (100k iterations), sends to backend. Backend hashes master password with Argon2, stores user + encrypted vault in PostgreSQL, returns `{id, email}`. Frontend auto-logs in. |
| UC-2 | Login | User | Must have registered | User enters email + master password. Frontend sends to backend `/api/v1/auth/login`. Backend verifies against Argon2 hash, generates UUID v4 session token (1-hour expiry), stores in sessions table, returns `{session_token, expires_at}`. Frontend stores token in localStorage. |
| UC-3 | Logout | User | Must be logged in | User clicks logout. Frontend clears session_token from localStorage, wipes all in-memory vault data and master password, redirects to landing page. |
| UC-4 | Change Master Password | User | Must be authenticated | User enters current + new password in Settings. Frontend sends to backend `/api/v1/auth/change-password` with Bearer token. Backend's auth middleware validates session, verifies current password against Argon2 hash, generates new Argon2 hash with fresh salt, updates `srp_verifier` and `srp_salt` columns. Returns 200 OK. Frontend logs user out after 2 seconds. |

#### Vault Operations

| # | Use Case | Actor | Pre-condition | Description |
|---|----------|-------|---------------|-------------|
| UC-5 | Unlock Vault | User | Logged in, vault locked | Frontend prompts for master password. Makes `GET /api/v1/vault` with Bearer token. Backend auth middleware validates token against sessions table (checks expiry, `account_status='active'`), updates `last_activity`, returns `{encrypted_vault}`. Frontend decodes base64, extracts `salt[0:16]`, `iv[16:28]`, `ciphertext[28:]`, derives AES key via PBKDF2, decrypts with AES-256-GCM, parses JSON `{entries:[]}` into memory. |
| UC-6 | Lock Vault | User | Vault unlocked | User clicks lock. Frontend wipes decrypted entries array, master password, and all UI state from memory. User must re-unlock to access vault. |
| UC-7 | Browse/Search Entries | User | Vault unlocked | User views list of decrypted entries in memory. Can filter by name/username/URL. Entries show favicons (fetched from `https://www.google.com/s2/favicons?domain=...`) or letter avatars as fallback. |
| UC-8 | View Entry Details | User | Vault unlocked | User selects entry. Frontend displays name, URL, username, password (masked, toggleable), notes. All data is from in-memory decrypted vault. |
| UC-9 | Create Entry | User | Vault unlocked | User fills form (name, username, password, URL, notes). Frontend generates ID via `Date.now().toString()`, adds to in-memory `entries[]` array. Includes UC-13 (Encrypt & Sync Vault). |
| UC-10 | Edit Entry | User | Vault unlocked | User modifies entry fields. Frontend updates in-memory entry object. Includes UC-13 (Encrypt & Sync Vault). |
| UC-11 | Delete Entry | User | Vault unlocked | User deletes entry (with confirmation). Frontend removes from in-memory `entries[]` array. Includes UC-13 (Encrypt & Sync Vault). |
| UC-12 | Copy Credentials | User | Vault unlocked | User clicks copy button for username/password/URL. Frontend copies to system clipboard. |

#### Shared Sub-Use Cases

| # | Use Case | Included By | Description |
|---|----------|-------------|-------------|
| UC-13 | Encrypt & Sync Vault | UC-9, UC-10, UC-11 | Frontend generates new random 16-byte salt + 12-byte IV, derives AES-256-GCM key from master password via PBKDF2 (100k iterations, SHA-256), encrypts JSON vault, encodes as base64 (format: salt + iv + ciphertext). Makes `PUT /api/v1/vault` with Bearer token and `{encrypted_vault}`. Backend auth middleware validates token, updates `users.encrypted_vault` and `users.last_login` columns. Returns 200 OK. |

#### Utility Features

| # | Use Case | Actor | Pre-condition | Description |
|---|----------|-------|---------------|-------------|
| UC-14 | Generate Password | User | Creating/editing entry | User opens Password Generator. Two modes: (1) Random (8–64 chars, toggle uppercase/lowercase/numbers/symbols/exclude-ambiguous) or (2) Passphrase (3–15 words from EFF large wordlist, configurable separator, capitalization, optional number suffix). User copies or inserts into entry form. All generation happens client-side in Web Crypto API. |
| UC-15 | Toggle Theme | User | None | User switches between light/dark mode. Frontend updates theme in localStorage and applies CSS classes. Available on all pages. |
| UC-16 | Check Backend Health | System | None | Frontend or monitoring tool calls `GET /health` or `GET /api/v1/auth/health` (no auth required). Backend returns 200 OK plain text. Used for service health monitoring. |

### Relationships & Dependencies

**Include relationships:**
- UC-9 (Create), UC-10 (Edit), UC-11 (Delete) all include → UC-13 (Encrypt & Sync Vault)
- UC-2 (Login) includes → Backend session validation against PostgreSQL

**Extend relationships:**
- UC-14 (Generate Password) extends → UC-9 (Create Entry), UC-10 (Edit Entry) — optional enhancement

**Precondition chains:**
- UC-2 (Login) is required before → UC-4, UC-5, UC-13
- UC-5 (Unlock Vault) is required before → UC-6 through UC-12, UC-14

**Actor interactions:**
- User → interacts with → Tauri Shell → which hosts → React Frontend
- React Frontend → communicates via REST/JSON → Backend API
- Backend API → validates sessions and persists data → PostgreSQL

## 2. Sequence Diagrams

### Sequence A: Complete Registration Flow

<div align="center">
    <img src="assets/SeqDiagram - Registration Flow.png" alt="Registration Flow Sequence Diagram" width="720">
</div>

**Participants:** User, React UI, Web Crypto API, Axum Backend, Argon2, PostgreSQL

1. User → React UI: Fills email, master password, confirm password, clicks "Create Master Vault"
2. React UI: Validates password match, length ≥ 8 chars
3. React UI → Web Crypto: `createEmptyVault()`
4. Web Crypto → React UI: Returns `{entries: []}`
5. React UI → Web Crypto: `encryptVault(emptyVault, masterPassword)`
6. Web Crypto: Generate random 16-byte salt via `crypto.getRandomValues()`
7. Web Crypto: Generate random 12-byte IV
8. Web Crypto: `PBKDF2(masterPassword, salt, 100k iterations, SHA-256)` → 256-bit AES key
9. Web Crypto: `AES-256-GCM.encrypt(JSON.stringify(vault), key, iv)` → ciphertext
10. Web Crypto → React UI: Returns `base64(salt + iv + ciphertext)`
11. React UI → Axum Backend: `POST /api/v1/auth/register {email, master_password, encrypted_vault}`
12. Axum Backend → Argon2: `SaltString::generate(&mut OsRng)`
13. Argon2 → Axum Backend: Random salt string
14. Axum Backend → Argon2: `Argon2::default().hash_password(master_password, salt)`
15. Argon2 → Axum Backend: PHC hash string (e.g., `$argon2id$v=19$m=19456$...`)
16. Axum Backend → PostgreSQL: `INSERT INTO users (email, srp_verifier, encrypted_vault, created_at) VALUES (...)`
17. PostgreSQL → Axum Backend: `RETURNING id, email` (or error if email exists)
18. Axum Backend → React UI: `200 OK {id, email}` (or `409 Conflict`)
19. React UI: Auto-triggers login flow (Sequence B)
20. React UI → User: Redirects to `/vault`

### Sequence B: Login + Vault Unlock Flow

<div align="center">
    <img src="assets/SeqDiagram - Login and Vault.png" alt="Login and Vault Unlock Sequence Diagram" width="720">
</div>

<div align="center">
    <img src="assets/Authentication Flow.png" alt="Authentication Flow Diagram" width="720">
</div>

**Participants:** User, React UI, Axum Backend, PostgreSQL, Argon2, Web Crypto API

#### Login Phase

1. User → React UI: Enters email + master password, clicks "Unlock Vault"
2. React UI → Axum Backend: `POST /api/v1/auth/login {email, master_password}`
3. Axum Backend → PostgreSQL: `SELECT id, srp_verifier FROM users WHERE email = $1`
4. PostgreSQL → Axum Backend: Returns `{id, srp_verifier}` (or empty → 404)
5. Axum Backend → Argon2: `PasswordHash::new(srp_verifier).verify(master_password)`
6. Argon2 → Axum Backend: Verification result (match → continue, mismatch → 401)
7. Axum Backend: Generate UUID v4 session token via `uuid::Uuid::new_v4()`
8. Axum Backend → PostgreSQL: `INSERT INTO sessions (user_id, session_token, created_at, expires_at, last_activity) VALUES ($1, $2, now(), now() + interval '1 hour', now())`
9. PostgreSQL → Axum Backend: `RETURNING session_token, expires_at`
10. Axum Backend → React UI: `200 OK {session_token, expires_at}`
11. React UI: Stores `session_token` in localStorage, navigates to `/vault`

#### Vault Phase Unlock

12. React UI → Axum Backend: `GET /api/v1/vault` (Header: `Authorization: Bearer <token>`)
13. Axum Backend → Auth Middleware: `extract_bearer_token(Authorization header)`
14. Auth Middleware: Parses "Bearer " prefix, extracts token string
15. Auth Middleware → PostgreSQL: `SELECT s.user_id, u.email, u.account_status FROM sessions s JOIN users u ON s.user_id = u.id WHERE s.session_token = $1 AND s.expires_at > now()`
16. PostgreSQL → Auth Middleware: Returns `{user_id, email}` (or empty → 401)
17. Auth Middleware: Checks `account_status = 'active'` (if not → 401)
18. Auth Middleware → PostgreSQL: `UPDATE sessions SET last_activity = now() WHERE session_token = $1`
19. Auth Middleware → Axum Backend: Injects `AuthUser {user_id}` into request
20. Axum Backend → PostgreSQL: `SELECT encrypted_vault FROM users WHERE id = $1`
21. PostgreSQL → Axum Backend: Returns `encrypted_vault` (String or NULL)
22. Axum Backend → React UI: `200 OK {encrypted_vault: "<base64 blob>"}`
23. React UI: Detects vault exists but locked, prompts for master password
24. User → React UI: Enters master password, clicks unlock
25. React UI → Web Crypto: `decryptVault(encrypted_vault, masterPassword)`
26. Web Crypto: Decode base64 → extract `salt[0:16]`, `iv[16:28]`, `ciphertext[28:]`
27. Web Crypto: `PBKDF2(masterPassword, salt, 100k iterations, SHA-256)` → AES key
28. Web Crypto: `AES-256-GCM.decrypt(ciphertext, key, iv)` → JSON string
29. Web Crypto: `JSON.parse(decrypted)` → `{entries: VaultEntry[]}`
30. Web Crypto → React UI: Returns decrypted Vault object
31. React UI: Stores `entries[]` in React state (in-memory only)
32. React UI → User: Displays vault entries list

### Sequence C: Create/Edit/Delete Entry with Sync

<div align="center">
    <img src="assets/SeqDiagram - Entries.png" alt="Entry CRUD and Sync Sequence Diagram" width="720">
</div>

**Participants:** User, React UI, Web Crypto API, Axum Backend, Auth Middleware, PostgreSQL

1. User → React UI: Creates/edits/deletes entry via UI (fills form or clicks delete with confirmation)
2. React UI: Updates in-memory `entries[]` array (add new entry, modify existing, or filter out deleted)
3. React UI: Retrieves master password from component state (user entered during unlock)
4. React UI → Web Crypto: `encryptVault({entries: updatedEntries}, masterPassword)`
5. Web Crypto: Generate NEW random 16-byte salt via `crypto.getRandomValues()`
6. Web Crypto: Generate NEW random 12-byte IV
7. Web Crypto: `PBKDF2(masterPassword, new_salt, 100k iterations, SHA-256)` → AES key
8. Web Crypto: `AES-256-GCM.encrypt(JSON.stringify(updatedVault), key, iv)` → ciphertext
9. Web Crypto → React UI: Returns `base64(new_salt + new_iv + ciphertext)`
10. React UI → Axum Backend: `PUT /api/v1/vault {encrypted_vault: "<new blob>"}` (Header: `Authorization: Bearer <token>`)
11. Axum Backend → Auth Middleware: Validate session (same as Sequence B steps 13-19)
12. Auth Middleware → Axum Backend: Injects `AuthUser {user_id}`
13. Axum Backend → PostgreSQL: `UPDATE users SET encrypted_vault = $1, last_login = now() WHERE id = $2`
14. PostgreSQL → Axum Backend: Returns `rows_affected = 1` (or 0 → 404)
15. Axum Backend → React UI: `200 OK`
16. React UI → User: Updates UI to reflect changes (entry added/modified/removed in list)

### Sequence D: Change Master Password

<div align="center">
    <img src="assets/SeqDiagram - Master Password.png" alt="Change Master Password Sequence Diagram" width="720">
</div>

**Participants:** User, React UI, Axum Backend, Auth Middleware, PostgreSQL, Argon2

1. User → React UI: Navigates to Settings, fills current password + new password + confirmation
2. React UI: Validates new passwords match, length ≥ 8
3. React UI → Axum Backend: `POST /api/v1/auth/change-password {current_password, new_password}` (Header: `Authorization: Bearer <token>`)
4. Axum Backend → Auth Middleware: Validate session token (returns `AuthUser {user_id}`)
5. Axum Backend → PostgreSQL: `SELECT srp_verifier FROM users WHERE id = $1`
6. PostgreSQL → Axum Backend: Returns stored Argon2 hash
7. Axum Backend → Argon2: `PasswordHash::new(stored_hash).verify(current_password)`
8. Argon2 → Axum Backend: Verification result (mismatch → 401 Unauthorized)
9. Axum Backend → Argon2: `SaltString::generate(&mut OsRng)` → new salt
10. Axum Backend → Argon2: `Argon2::default().hash_password(new_password, new_salt)`
11. Argon2 → Axum Backend: Returns new PHC hash string
12. Axum Backend → PostgreSQL: `UPDATE users SET srp_salt = $1, srp_verifier = $2 WHERE id = $3`
13. PostgreSQL → Axum Backend: Returns `rows_affected`
14. Axum Backend → React UI: `200 OK`
15. React UI → User: Displays "Password changed successfully! Logging out in 2 seconds..."
16. React UI: Waits 2 seconds
17. React UI: Calls `logout()` → clears `session_token` from localStorage, wipes in-memory data
18. React UI → User: Redirects to `/` (landing page)

### Sequence D: Password Generation (Client-Side Only)

**Participants:** User, React UI, Web Crypto API

#### Random Password Mode

1. User → React UI: Opens Password Generator, selects "Random Password", sets length=16, enables uppercase+lowercase+numbers+symbols
2. React UI → Web Crypto: `crypto.getRandomValues(new Uint32Array(length))`
3. Web Crypto → React UI: Returns random byte array
4. React UI: Maps bytes to character set (A-Z, a-z, 0-9, !@#$%^&\*), filters ambiguous chars if enabled
5. React UI → User: Displays generated password (e.g., `kR9$mP3@qL2vB7xN`)
6. User → React UI: Clicks "Copy" or "Insert into form"
7. React UI: Copies to clipboard or inserts into password field of entry form

#### Passphrase Mode

1. User → React UI: Selects "Passphrase", sets word count=5, separator="-", capitalization="lowercase", append number=true
2. React UI → Web Crypto: `crypto.getRandomValues(new Uint32Array(5))`
3. Web Crypto → React UI: Returns random indices
4. React UI: Loads EFF large wordlist (7776 words), maps indices to words
5. React UI: Joins words with separator, applies capitalization, appends random 2-digit number
6. React UI → User: Displays passphrase (e.g., `correct-horse-battery-staple-cloud-47`)
7. User: Copies or inserts into form

## 3. Architecture Diagram

<div align="center">
    <img src="assets/Architecture Diagram.png" alt="Combined Architecture Diagram" width="720">
</div>

### Key Architectural Patterns & Principles

#### Zero-Knowledge Architecture

- Vault encryption/decryption ONLY happens client-side in Web Crypto API
- Backend stores opaque `encrypted_vault` blobs — never sees plaintext
- Separation of concerns: Backend handles auth + persistence, Frontend handles crypto + UI

#### Authentication Flow

- Password hashing: Client sends plaintext password → Backend hashes with Argon2 → stores hash
- Session tokens: UUID v4, 1-hour expiry, validated on every protected request
- Token storage: Client localStorage (note: vulnerable to XSS, but acceptable for desktop app)

#### Encryption Scheme

- Key derivation: PBKDF2 (100k iterations, SHA-256, 16-byte salt)
- Symmetric cipher: AES-256-GCM (12-byte IV, authenticated encryption)
- New salt + IV generated on every vault save (prevents ciphertext reuse attacks)

#### Technology Stack Summary

| Layer | Technologies |
|-------|-------------|
| **Frontend** | React 18 + TypeScript + Vite (fast dev + bundler), Radix UI + shadcn/ui + Tailwind v4 (component library + styling), React Router v7 (client-side routing), Web Crypto API (PBKDF2 + AES-256-GCM), No state management library (useState + localStorage) |
| **Backend** | Rust (safe, fast, compiled), Tokio (async runtime), Axum (web framework, built on Tower middleware), sqlx (compile-time-checked SQL, async PostgreSQL driver), argon2 + password-hash (Argon2id password hashing), uuid (v4 token generation), tower-http (CORS middleware) |
| **Database** | PostgreSQL (ACID-compliant RDBMS), 2 tables (users, sessions), no ORM |

#### Deployment Model

- **Frontend:** Bundled as Tauri native app, runs fully local (no CORS issues in production)
- **Backend:** Expected to run as standalone API server (e.g., `localhost:3000` in dev, cloud in prod)
- **Desktop-first:** Tauri embeds WebView, so frontend has direct OS integration

## 4. Combined Schema / ER Diagram

<div align="center">
    <img src="assets/Schema Diagram.png" alt="Schema / ER Diagram" width="720">
</div>

### Entity-Relationship Diagram (Full Stack)

#### Server-Side Entities (PostgreSQL Database)

```
┌─────────────────────────────────────┐
│             users                   │
├─────────────────────────────────────┤
│ * id                SERIAL PK       │◄──────────┐
│   username          VARCHAR(255)    │           │
│ * email             VARCHAR(255)    │           │ 1
│                     UNIQUE NOT NULL │           │
│   srp_salt          TEXT            │           │
│   srp_verifier      TEXT            │           │ Relationship:
│   encrypted_vault   TEXT            │           │ One user can have
│   created_at        TIMESTAMP       │           │ many sessions
│   last_login        TIMESTAMP       │           │
│   account_status    VARCHAR(20)     │           │
│                     DEFAULT 'active'│           │
└─────────────────────────────────────┘           │
                                                  │
                                                  │ N
                                                  │
┌─────────────────────────────────────┐           │
│            sessions                 │           │
├─────────────────────────────────────┤           │
│ * id                SERIAL PK       │           │
│ * user_id           INT             │───────────┘
│                     FK → users.id   │
│                     ON DELETE       │
│                     CASCADE         │
│ * session_token     TEXT            │
│                     UNIQUE NOT NULL │
│   created_at        TIMESTAMP       │
│                     DEFAULT NOW     │
│   expires_at        TIMESTAMP       │
│   last_activity     TIMESTAMP       │
│                     DEFAULT NOW     │
└─────────────────────────────────────┘

Indexes:
- users.email (UNIQUE)
- sessions.session_token (UNIQUE)
- sessions.user_id (for JOIN)
- sessions.expires_at (for cleanup)

Foreign Key Constraints:
- sessions.user_id REFERENCES users.id
  ON DELETE CASCADE
  (When user deleted → all sessions deleted)
```

#### Client-Side Entities (In-Memory, Browser/Tauri WebView Only)

> These structures exist ONLY in decrypted form in React state. They are NEVER sent to the server in plaintext.

```
┌─────────────────────────────────────┐
│             Vault                   │
│     (TypeScript interface)          │
├─────────────────────────────────────┤
│ * entries       VaultEntry[]        │◄──────────┐
│                                     │           │
│ Stored on server as:                │           │ 1
│ users.encrypted_vault = base64(     │           │
│   salt[16 bytes] +                  │           │ Composition:
│   iv[12 bytes] +                    │           │ One Vault contains
│   AES-GCM-ciphertext[              │           │ zero or more entries
│     JSON.stringify(Vault)           │           │
│   ]                                 │           │
│ )                                   │           │
└─────────────────────────────────────┘           │
                                                  │
                                                  │ N
                                                  │
┌─────────────────────────────────────┐           │
│          VaultEntry                 │           │
│     (TypeScript interface)          │           │
├─────────────────────────────────────┤           │
│ * id            string              │───────────┘
│                 (Date.now()         │
│                 .toString())        │
│ * name          string              │
│                 (e.g., "Gmail")     │
│   username      string?             │
│   password      string?             │
│   url           string?             │
│   notes         string?             │
└─────────────────────────────────────┘
```

> **Notes:** `*` = Required field, `?` = Optional field. `id` is generated client-side via `Date.now().toString()`. No server-side validation of entry structure (server only stores opaque encrypted blob).

#### Client-Side Persistent Storage (Browser localStorage)

| Key | Value Type | Example | Purpose | Lifecycle |
|-----|-----------|---------|---------|-----------|
| `session_token` | `string` | `550e8400-e29b-41d4-a716-446655440000` | Bearer token for API auth | Set on login, cleared on logout |
| `theme` | `"light" \| "dark"` | `"dark"` | User's UI theme preference | Persists across sessions |

#### Encryption Payload Format

<div align="center">
    <img src="assets/Encryption Payload.png" alt="Encryption Payload Format" width="720">
</div>

Binary structure of `users.encrypted_vault` before base64 encoding:

```
┌────────────────┬────────────────┬──────────────────────────────────────┐
│ Offset: 0-15   │ Offset: 16-27  │ Offset: 28-end                       │
├────────────────┼────────────────┼──────────────────────────────────────┤
│ PBKDF2 Salt    │ AES-GCM IV     │ AES-GCM Ciphertext                   │
│ 16 bytes       │ 12 bytes       │ Variable length                      │
│ (random)       │ (random)       │                                      │
│                │                │ Plaintext = JSON.stringify({         │
│ Used for key   │ Nonce for      │   entries: [                         │
│ derivation     │ AES-GCM        │     {id, name, username, password,   │
│                │ encryption     │      url, notes},                    │
│                │                │     ...                              │
│                │                │   ]                                  │
│                │                │ })                                   │
└────────────────┴────────────────┴──────────────────────────────────────┘

Total bytes = 16 + 12 + len(ciphertext)
Then encoded as: base64(all bytes)
Stored in: PostgreSQL users.encrypted_vault column
```

#### Data Flow & Relationships Summary

<div align="center">
    <img src="assets/Data Flow Diagram.png" alt="Data Flow Diagram" width="720">
</div>

1. **User Registration:**
   Client → creates `Vault{entries:[]}` → encrypts → base64 blob
   Client → sends `{email, password, blob}` to Backend
   Backend → hashes password (Argon2) → `INSERT INTO users`

2. **Login:**
   Client → sends `{email, password}` to Backend
   Backend → verifies password → generates UUID `session_token`
   Backend → `INSERT INTO sessions` → returns token to Client
   Client → stores token in localStorage

3. **Unlock Vault:**
   Client → `GET /api/v1/vault` (with Bearer token)
   Backend → validates token via sessions table → fetches `encrypted_vault`
   Backend → returns blob to Client
   Client → decrypts blob → `Vault{entries:[]}` in React state

4. **Save Entry (Create/Edit/Delete):**
   Client → modifies `entries[]` in memory → encrypts updated Vault
   Client → `PUT /api/v1/vault {encrypted_vault: new_blob}`
   Backend → validates token → `UPDATE users.encrypted_vault`

5. **Change Password:**
   Client → `POST /auth/change-password {current, new}` (with token)
   Backend → verifies current password → hashes new password (new Argon2 salt)
   Backend → `UPDATE users.srp_verifier`
   Client → logs out, user must re-login

#### Security Boundaries

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         SECURITY BOUNDARIES                              │
└──────────────────────────────────────────────────────────────────────────┘

Plaintext Zone (Client-Side Only):
- Vault entries (id, name, username, password, url, notes)
- Master password (held in React state while vault unlocked)
- Decrypted vault JSON

════════════════════════════════════════════════════════════════════════════
                              TRUST BOUNDARY
════════════════════════════════════════════════════════════════════════════

Encrypted/Hashed Zone (Server-Side):
- users.encrypted_vault — opaque base64 blob (AES-256-GCM ciphertext)
- users.srp_verifier — Argon2 password hash (one-way, non-reversible)
- sessions.session_token — UUID (acts as bearer token, no crypto)

Server NEVER has access to:
- Plaintext master password (only sees it during initial auth, then discarded)
- Plaintext vault entries
- AES encryption keys (derived client-side via PBKDF2)

Zero-Knowledge Guarantee:
Even if PostgreSQL database is compromised, attackers cannot decrypt
vault contents without the user's master password.
```

### Data Type Details

| Category | Type | Description |
|----------|------|-------------|
| **PostgreSQL** | `SERIAL` | Auto-incrementing 4-byte integer (INT with sequence) |
| **PostgreSQL** | `VARCHAR(n)` | Variable-length string, max n characters |
| **PostgreSQL** | `TEXT` | Unlimited-length string |
| **PostgreSQL** | `TIMESTAMP` | Date + time (no timezone stored, assumes UTC) |
| **TypeScript** | `string` | UTF-16 encoded string |
| **TypeScript** | `string?` | Optional string (may be undefined or null) |
| **TypeScript** | `VaultEntry[]` | Array of VaultEntry objects |
| **Crypto** | Argon2 PHC string | e.g., `$argon2id$v=19$m=19456,t=2,p=1$base64salt$base64hash` |
| **Crypto** | UUID v4 | e.g., `550e8400-e29b-41d4-a716-446655440000` (128-bit random) |
| **Crypto** | Base64 encoding | Standard base64 (A-Z, a-z, 0-9, +, /) |

### Cardinality & Constraints Summary

| Relationship | Cardinality | Constraint |
|-------------|-------------|------------|
| users ↔ sessions | 1:N | Foreign key with CASCADE DELETE |
| Vault ↔ VaultEntry | 1:N | In-memory composition (no DB relation) |
| users.email | — | UNIQUE NOT NULL |
| sessions.session_token | — | UNIQUE NOT NULL |
| sessions.expires_at | — | Enforced by auth middleware (not DB constraint) |
| users.account_status | — | CHECK constraint possible, currently just VARCHAR |

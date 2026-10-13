# [VS-01] Feature Slice: RBAC & Solo Security Core (`rbac_security`)
**Slice Identifier:** `src/features/rbac_security/`  
**Slice ID:** `VS-01`  
**Document Version:** 1.0  
**Phase / Sprint:** Phase 1 / Sprint 1  
**Status:** Approved for Implementation  
**Upstream Dependencies:** `packages/core-security`, `packages/core-database`, `packages/core-ui`  
**Downstream Consumers:** `VS-02` (Onboarding), `VS-06` (POS Checkout), `VS-10` (Khata Ledger), `VS-12` (Shift Closure)  

---

## 1. Executive Summary & Slice Purpose

In tier-2/3 retail environments (Clothes, Footwear, Kirana), over 70% of stores operate with a **solo operator** (the proprietor handles billing, stocking, accounting, and closing alone). Traditional enterprise ERP systems force these single owners into bureaucratic approval loops, fake cashier shifts, and repetitive supervisor overrides.

`VS-01` delivers the foundational **Role-Based Access Control (RBAC) & Solo Mode Engine**. It introduces an instant zero-friction toggle:
- **Solo Mode (`ON`)**: Disables all cashier verification prompts, enables instant item price/tax overrides, bypasses cash drawer supervisor locks, and hides bureaucratic audit confirmations.
- **Dual / Multi-Operator Mode (`OFF`)**: Enforces strict privilege segregation between `StoreOwner` (Manager) and `Cashier` (Clerk). Sensitive operations (voiding bills, giving unapproved discounts > 10%, altering credit limits) trigger an instant **Master PIN Modal**.

---

## 2. Standardized Files & Folder Mapping

Following our token-efficient Vertical Slice Architecture (VSA), `VS-01` is strictly isolated within `src/features/rbac_security/`. All files, responsibilities, and visibility boundaries are mapped below:

```
src/features/rbac_security/
├── SLICE.md                             # Token-efficient AI manifest (< 250 tokens)
├── mod.rs                               # Public gateway (re-exports only external items)
│
├── internal/                            # Private domain layer (pub(crate) only)
│   ├── logic.rs                         # Argon2id hashing, zeroize scrubbing, lockout FSM
│   ├── models.rs                        # SQLite entities (User, Role, SecuritySettings, AuditLog)
│   └── repo.rs                          # Direct SQLite migrations & queries for security tables
│
├── external/                            # Public consumption layer (Dioxus 0.7 components)
│   ├── style.css                        # Scoped SoftMorph CSS for keypad, badges, and modals
│   ├── views.rs                         # Route views (LockScreenView, AccessDeniedView, SecuritySettingsView)
│   ├── hooks/                           # Reactive state hooks
│   │   ├── use_rbac_auth.rs             # Session signal, role state, and permission evaluator
│   │   └── use_pin_challenge.rs         # Modal state, attempt counter, and tactile shake trigger
│   └── components/                      # Reusable visual RSX components
│       ├── master_pin_modal.rs          # SoftMorph tactile keypad modal (F2 shortcut / touch)
│       ├── solo_mode_badge.rs           # Header / sidebar glowing solo status badge
│       ├── solo_mode_toggle.rs          # Settings tactile toggle switch with PIN auth gate
│       └── desktop_sidebar.rs           # 240px role-filtered collapsible navigation sidebar
│
└── test/                                # Continuous In-Slice Verification Gate
    ├── fr/                              # Functional Requirements
    │   ├── unit_pin_hashing.rs          # Argon2id verify, salt validation, zeroize drop check
    │   ├── unit_solo_mode_rules.rs      # Rule engine: Solo ON bypass vs Solo OFF supervisor gate
    │   ├── integ_repo_security.rs       # In-memory SQLite roundtrip, 5-attempt lockout increment
    │   ├── rbac_role_gate.rs            # Route guard tests (Cashier blocked from Owner routes)
    │   └── e2e_modal_interaction.rs     # Dioxus headless VDOM: keypad touch/click submission
    └── nfr/                             # Non-Functional Requirements
        ├── perf_pin_benchmark.rs        # Criterion benchmark: Argon2id <= 45ms, permission check <= 0.1ms
        └── sec_fuzz_pin.rs              # Proptest: SQL injection fuzzing, memory leak audit
```

---

## 3. Dioxus Route Definitions & Navigation Map

This slice defines, handles, and guards routes across the application using Dioxus 0.7's type-safe `#[derive(Routable)]` enum.

### 3.1 Route Registry & Access Matrix
| Route Path | Route Enum Variant | View Component | Access Control / Guard | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `/lock` | `Route::LockScreen` | `LockScreenView` | Public (Unauthenticated) | Idle lock screen (15 min inactivity or 1-tap lock). Requires Cashier PIN or Master PIN to unlock. |
| `/access-denied` | `Route::AccessDenied` | `AccessDeniedView` | Authenticated | Rendered when a Cashier attempts to access an Owner-only route. Offers "Authorize with Master PIN" button. |
| `/settings/security`| `Route::SecuritySettings`| `SecuritySettingsView` | `StoreOwner` or `Solo Mode` | Master PIN reset, Solo Mode toggle, lockout duration, and 12-word recovery seed display. |
| `/recovery` | `Route::EmergencyRecovery`| `EmergencyRecoveryView`| Public | Offline 12-word mnemonic seed entry to reset a forgotten Master PIN. |

### 3.2 Dioxus Route Code Definition
```rust
// In src/features/rbac_security/external/views.rs
use dioxus::prelude::*;

#[derive(Routable, Clone, Debug, PartialEq)]
pub enum SecurityRoute {
    #[route("/lock")]
    LockScreen {},

    #[route("/access-denied")]
    AccessDenied {
        attempted_path: String,
        required_role: String,
    },

    #[route("/settings/security")]
    SecuritySettings {},

    #[route("/recovery")]
    EmergencyRecovery {},
}
```

### 3.3 Navigation Guards & Middleware Hook
Every protected route in the global Dioxus router uses the `use_rbac_guard` hook from this slice:
```rust
// Guard evaluated at top of protected views
pub fn use_rbac_guard(required_permission: SecurityPermission) -> bool {
    let auth = use_rbac_auth();
    if auth.is_solo_mode() {
        return true; // Solo Mode bypass: instant access
    }
    if !auth.has_permission(required_permission) {
        let nav = navigator();
        nav.replace(SecurityRoute::AccessDenied {
            attempted_path: nav.current_location().path().to_string(),
            required_role: "StoreOwner".to_string(),
        });
        return false;
    }
    true
}
```

---

## 4. Business Rules & Actors

### 4.1 Actors & Roles
| Actor / Role | Description | Privilege Scope |
| :--- | :--- | :--- |
| **StoreOwner (Manager)** | Primary owner/proprietor who holds the Master PIN. | Full access to all modules, financial margins, tax settings, staff management, DB backup, and audit logs. |
| **Cashier (Clerk)** | Counter assistant or family helper during peak footfall. | POS barcode scanning, cart management, accepting standard cash/UPI payments, receipt printing. |
| **Solo Operator** | StoreOwner operating with Solo Mode enabled. | All privileges unlocked seamlessly without any supervisor confirmation prompts. |

### 4.2 Core Business Rules
- **BR-SEC-01 (Master PIN Security)**: A 4-to-6 digit numeric PIN is set during initial onboarding. It is stored as an Argon2id salted hash with memory cost 19 MiB, time cost 2 iterations, and zeroized in-memory buffers.
- **BR-SEC-02 (Solo Mode Frictionless Operation)**: When Solo Mode is enabled, any system action flagged with `RequiresSupervisorApproval` automatically evaluates to `Permitted` without showing a PIN modal.
- **BR-SEC-03 (Anti-Brute-Force Lockout)**: If an invalid PIN is entered 5 consecutive times, the PIN modal locks out for 3 minutes with an exponential backoff (+3 minutes per subsequent failure).
- **BR-SEC-04 (Session Expiry & Fast Unlock)**: When in Dual-Operator mode, an idle screen locks after 15 minutes of inactivity. Cashiers unlock with their short Clerk PIN; StoreOwners unlock with the Master PIN.
- **BR-SEC-05 (Zero Plaintext Secrets)**: Master PINs, session tokens, and recovery keys must never appear in log files, terminal outputs, or unencrypted storage tables.

---

## 5. Functional Requirements (FR)

### 5.1 User Stories & Acceptance Criteria

#### US-FR-01: Solo Mode Instant Toggle
> *As a single store proprietor, I want to toggle Solo Mode on with one tap so that I can bill customers and adjust prices rapidly without entering my PIN on every discount.*

- **Acceptance Criteria**:
  - [x] Setting toggle appears prominently in Store Settings and as a status indicator in the top navbar / sidebar.
  - [x] Toggling from Solo Mode to Dual-Operator Mode requires confirmation.
  - [x] Toggling from Dual-Operator Mode back to Solo Mode requires valid Master PIN verification.
  - [x] State persists in SQLite `system_settings` table and broadcasts a reactive signal to all active UI slices.

#### US-FR-02: Master PIN Modal Challenge
> *As a store owner with a temporary helper at the counter, I want the system to prompt for my Master PIN whenever the helper attempts a sensitive action (e.g. bill cancellation or discount > 10%) so that my profit margins and cash drawers are protected.*

- **Acceptance Criteria**:
  - [x] A tactile SoftMorph modal overlays the screen with a large numeric keypad and a hidden 4-6 digit masked PIN input.
  - [x] Supports both physical keyboard NumPad input and touch-screen on-screen keypad.
  - [x] If verified, the pending action executes immediately, and the modal dismisses within < 50ms.
  - [x] If incorrect, an animated tactile shake feedback is shown, and remaining attempts (e.g., "3 attempts remaining") are displayed.

#### US-FR-03: Cashier Role Assignment & Restricted Navigation
> *As a cashier, I want to see only counter sales and inventory lookups so that I am not distracted or confused by tax filings, gross margins, or owner ledgers.*

- **Acceptance Criteria**:
  - [x] When logged in as `Cashier`, navigation sidebar/tabs automatically hide `Pocket BI Analytics`, `Store Settings`, `Staff Management`, and `Khata Customer Limits`.
  - [x] Direct route tampering or URL navigation to restricted paths renders an `AccessDeniedCard` prompting for Master PIN override.

### 5.2 Edge Cases & Failure Scenarios
1. **EC-FR-01 (Sudden Power Cut / Tab Close during PIN Challenge)**:
   - State must not lock the database in an uncommitted transaction. The pending checkout remains in draft/parked status; the action is rejected upon reload until re-authorized.
2. **EC-FR-02 (Simultaneous Cashier Shift Handover)**:
   - When a cashier logs out, active in-memory cart signals are safely preserved or parked, but session tokens are zeroized immediately.
3. **EC-FR-03 (Forgotten Master PIN)**:
   - System provides an offline recovery passphrase (generated at onboarding). Entering the 12-word recovery seed allows resetting the Master PIN without internet connection.

---

## 6. Non-Functional Requirements (NFR)

### 6.1 System Stories & System Acceptance Criteria

#### US-NFR-01: High-Speed Verification Latency (NFR-PERF-01)
> *As a local-first application, the PIN verification and role evaluation must execute locally in < 50ms so that queue checkout throughput is never slowed down.*

- **System Acceptance Criteria**:
  - [x] Argon2id verification benchmarked using Criterion at $\le 45\text{ms}$ on target hardware (x86_64 / ARM64).
  - [x] In-memory RBAC permission checks execute in $\le 0.1\text{ms}$ (zero DB lookup per permission check via cached session state).

#### US-NFR-02: In-Memory Memory Sanitization (NFR-SEC-01)
> *As a secure POS system, PIN digits entered into input buffers must be immediately wiped from RAM upon hash comparison to prevent memory dump exploitation.*

- **System Acceptance Criteria**:
  - [x] Master PIN structs implement the `zeroize::Zeroize` and `zeroize::ZeroizeOnDrop` traits.
  - [x] No string formatting (`"{:?}"`, `"{}"`) is implemented for PIN types (`Debug` outputs `"[REDACTED]"`).

### 6.2 NFR Edge Cases
1. **EC-NFR-01 (High-Frequency Fuzzing Attack)**:
   - Brute-force attacks submitting > 10 PIN attempts per second trigger the SQLite lock gate, halting verification threads for 3 minutes and logging an anomaly security event.
2. **EC-NFR-02 (Corrupted Salt in SQLite)**:
   - If SQLite storage experiences bit rot in the `argon2_hash` column, the app fails gracefully into Emergency Recovery Seed Mode rather than panicking or crashing the process.

---

## 7. Database Schema (SQLite / SQLCipher)

This slice owns the security and session metadata tables. Migrations are executed within `internal/repo.rs`.

```sql
-- Table: rbac_roles
CREATE TABLE IF NOT EXISTS rbac_roles (
    role_id TEXT PRIMARY KEY NOT NULL,          -- 'store_owner', 'cashier'
    role_name TEXT NOT NULL,
    description TEXT,
    created_at INTEGER NOT NULL
);

-- Table: rbac_users
CREATE TABLE IF NOT EXISTS rbac_users (
    user_id TEXT PRIMARY KEY NOT NULL,          -- UUID v4
    role_id TEXT NOT NULL REFERENCES rbac_roles(role_id),
    username TEXT NOT NULL UNIQUE,
    display_name TEXT NOT NULL,
    pin_hash TEXT NOT NULL,                     -- Argon2id formatted string
    pin_salt TEXT NOT NULL,                     -- 16-byte base64 salt
    is_active INTEGER NOT NULL DEFAULT 1,       -- 1 = active, 0 = disabled
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- Table: rbac_security_settings (Singleton)
CREATE TABLE IF NOT EXISTS rbac_security_settings (
    id INTEGER PRIMARY KEY CHECK (id = 1),       -- Enforces single row
    is_solo_mode INTEGER NOT NULL DEFAULT 1,    -- 1 = Solo Mode, 0 = Multi-Operator
    max_failed_attempts INTEGER NOT NULL DEFAULT 5,
    lockout_duration_seconds INTEGER NOT NULL DEFAULT 180,
    consecutive_failed_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until_timestamp INTEGER DEFAULT NULL,
    recovery_seed_hash TEXT NOT NULL,           -- Argon2id hash of 12-word recovery mnemonic
    updated_at INTEGER NOT NULL
);

-- Table: rbac_audit_logs
CREATE TABLE IF NOT EXISTS rbac_audit_logs (
    log_id TEXT PRIMARY KEY NOT NULL,
    user_id TEXT REFERENCES rbac_users(user_id),
    action_type TEXT NOT NULL,                  -- 'PIN_SUCCESS', 'PIN_FAILED', 'SOLO_TOGGLE_ON', 'SOLO_TOGGLE_OFF'
    target_resource TEXT,                       -- 'PRICE_OVERRIDE', 'BILL_VOID', 'SETTINGS'
    ip_or_device TEXT NOT NULL DEFAULT 'local',
    created_at INTEGER NOT NULL
);

-- Indexes for lightning lookups
CREATE INDEX IF NOT EXISTS idx_rbac_users_role ON rbac_users(role_id);
CREATE INDEX IF NOT EXISTS idx_rbac_audit_created ON rbac_audit_logs(created_at DESC);
```

---

## 8. Internal Logic & API Contracts (Rust Traits & DTOs)

Because RetailOS uses Dioxus with in-process Rust architecture, "API contracts" are modeled as strongly-typed, thread-safe asynchronous Rust service traits rather than external HTTP endpoints.

### 8.1 Data Transfer Objects & Error Types
```rust
// In src/features/rbac_security/internal/models.rs
use serde::{Deserialize, Serialize};
use zeroize::{Zeroize, ZeroizeOnDrop};

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum UserRole {
    StoreOwner,
    Cashier,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum SecurityPermission {
    PerformBilling,
    ApplyDiscretionaryDiscount,
    VoidActiveInvoice,
    ModifyInventoryCostPrice,
    AccessFinancialReports,
    ManageStoreSettings,
    ToggleSoloMode,
}

#[derive(Zeroize, ZeroizeOnDrop)]
pub struct RawPin(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum SecurityError {
    InvalidPin { attempts_remaining: u8 },
    AccountLocked { locked_until_ts: i64 },
    UnauthorizedAction { required_role: UserRole },
    SoloModeLockoutViolation,
    DatabaseFailure(String),
}
```

### 8.2 Service Contract (Public Trait)
```rust
// In src/features/rbac_security/internal/logic.rs
use crate::features::rbac_security::internal::models::*;

pub trait SecurityService: Send + Sync {
    /// Evaluates if an action is permitted. If Solo Mode is ON, returns Ok(true) immediately.
    fn check_permission(&self, permission: SecurityPermission) -> Result<bool, SecurityError>;
    
    /// Verifies the 4-6 digit Master PIN against the Argon2id hash.
    fn verify_master_pin(&self, pin: RawPin) -> Result<(), SecurityError>;
    
    /// Toggles Solo Mode. Switching OFF requires verification; switching ON requires Master PIN.
    fn set_solo_mode(&self, enable: bool, auth_pin: Option<RawPin>) -> Result<bool, SecurityError>;
    
    /// Retrieves current security state snapshot for reactive UI signals.
    fn get_security_state(&self) -> SecurityStateSnapshot;
}
```

---

## 9. UI / UX Specification (SoftMorph Design System)

All visual elements conform to `docs/ui-ux-design-system.md` using the **Terracotta & Apricot SoftMorph** neomorphic palette.

### 9.1 Visual Color Tokens Used in this Slice
- **Base Surface**: Oatmeal Clay (`#EFE9E0`)
- **Primary Accent / Active Key**: Terracotta (`#E06D3B`)
- **Keypad Tactile Bump**: Apricot Cream (`#F6E2D5`)
- **Security Warning / Lock**: Honey Amber (`#D9822B`)
- **Neomorphic Shadows**:
  - Convex (Unpressed): `box-shadow: 6px 6px 14px #D6CEC3, -6px -6px 14px #FFFFFF;`
  - Concave (Pressed Key): `box-shadow: inset 4px 4px 8px #D6CEC3, inset -4px -4px 8px #FFFFFF;`

---

### 9.2 Responsive Screen Layouts & Wireframes

```
+----------------------------------------------------------------------------------------------------+
| 1. DESKTOP VIEW (1280px+) - Persistent Tactile Sidebar with Solo Badge                             |
+----------------------------------------------------------------------------------------------------+
| [ RETAIL-OS ]  | Search Items / Barcode (F2)                  | [● Solo Mode: ON]  [12:45 PM] [M]  |
|----------------+-----------------------------------------------------------------------------------|
| (≡) POS Sales  |                                                                                   |
| (⊞) Inventory  |   +---------------------------------------------------------------------------+   |
| (☷) Matrix     |   | ACTIVE CART #104                                       3 Items | $ 48.50  |   |
| (◫) Khata      |   | [1] Cotton Kurti (Size M, Terracotta)                   $ 25.00            |   |
| (▲) Analytics  |   | [2] Kolhapuri Chappal (Size 8)                          $ 18.50            |   |
| (⚙) Settings   |   | [3] Loose Basmati Rice (1.25 kg)                         $  5.00            |   |
|----------------+   +---------------------------------------------------------------------------+   |
| [🔒 Lock POS]  |                                                                                   |
+----------------------------------------------------------------------------------------------------+

+----------------------------------------------------------------------------------------------------+
| 2. MASTER PIN CHALLENGE MODAL (Desktop / Tablet / Mobile Overlay)                                  |
+----------------------------------------------------------------------------------------------------+
|                                    +------------------------------+                                |
|                                    |      🔒 Supervisor Approval  |                                |
|                                    |   Enter Master PIN to bypass |                                |
|                                    |                              |                                |
|                                    |         [ ●  ●  ○  ○ ]       |                                |
|                                    |                              |                                |
|                                    |    [ 1 ]    [ 2 ]    [ 3 ]   |                                |
|                                    |    [ 4 ]    [ 5 ]    [ 6 ]   |                                |
|                                    |    [ 7 ]    [ 8 ]    [ 9 ]   |                                |
|                                    |    [ C ]    [ 0 ]    [ ⌫ ]   |                                |
|                                    |                              |                                |
|                                    |   [ Cancel ]    [ Authorize ]|                                |
|                                    +------------------------------+                                |
+----------------------------------------------------------------------------------------------------+

+----------------------------------------------------------------------------------------------------+
| 3. TABLET & MOBILE NAVIGATION (Bottom Bar & Floating Solo Mode Badge)                              |
+----------------------------------------------------------------------------------------------------+
| Mobile Top Bar:  [ 🏪 Store Name ]            [ ⚡ Solo Mode (Active) ]            [ 🔔 Alerts ]     |
| Mobile Content:  Full-width soft card layout with touch-friendly 48px hit targets                  |
| Mobile Nav Bar:  [ 🛒 POS ]     [ 📦 Stock ]     [ 📒 Khata ]     [ 📊 BI ]     [ ⚙️ Settings ]      |
+----------------------------------------------------------------------------------------------------+
```

### 9.3 Desktop Sidebar Navigation Specification
- **Fixed Width**: `240px` collapsible to `72px` icon-rail.
- **Top Brand**: Soft embossed RetailOS logo with warm Terracotta accent.
- **Role-Aware Filtering**:
  - If `Role == Cashier` AND `SoloMode == OFF`: Options `Analytics` and `Settings` are hidden.
  - If `SoloMode == ON`: All options are visible with an active `Solo Badge` indicator at the bottom.
- **Lock Quick-Action**: Bottom of the sidebar features a tactile neomorphic `Lock Station` button that immediately triggers the screen lock.

---

## 10. In-Slice Tests Specification (FR & NFR)

Following our strict architecture, tests are contained inside `src/features/rbac_security/test/`.

### 10.1 Functional Tests (`test/fr/`)
1. **`unit_pin_hashing.rs`**:
   - Tests Argon2id hash generation and verification.
   - Tests rejection of incorrect PINs and whitespace-padded inputs.
   - Verifies `Zeroize` cleanses memory after dropped `RawPin`.
2. **`unit_solo_mode_rules.rs`**:
   - Asserts that all permissions return `Ok(true)` when `is_solo_mode == 1`.
   - Asserts that sensitive actions require Master PIN when `is_solo_mode == 0`.
3. **`integ_repo_security.rs`**:
   - Runs round-trip SQLite tests using in-memory SQLite connection.
   - Tests lockout timer increment upon 5 consecutive failed attempts.
4. **`rbac_role_gate.rs`**:
   - Tests cashier session rejection on owner-only routes.
5. **`e2e_modal_interaction.rs`**:
   - Headless Dioxus Virtual DOM test simulating touch clicks on numeric keypad buttons `1-2-3-4` and submitting the form.

### 10.2 Non-Functional Tests (`test/nfr/`)
1. **`perf_pin_benchmark.rs` (Criterion)**:
   - Measures Argon2id verification latency: must be $\le 50\text{ms}$ (target $25\text{ms} - 45\text{ms}$).
   - Measures in-memory permission check: must be $\le 0.1\text{ms}$ ($< 100\mu\text{s}$).
2. **`sec_fuzz_pin.rs` (Proptest & Security)**:
   - Fuzzes PIN inputs with random Unicode, ASCII control codes, SQL injection strings (`' OR '1'='1`), and buffer overflows.
   - Verifies zero memory leaks and zero panics.
   - Asserts that audit logs record the event without logging the raw input characters.

---

## 11. Definition of Done (DoD) Checklist for VS-01

- [ ] `SLICE.md` manifest written (< 250 tokens).
- [ ] Argon2id hashing & `zeroize` memory scrubbing implemented in `internal/logic.rs`.
- [ ] SQLite tables (`rbac_roles`, `rbac_users`, `rbac_security_settings`, `rbac_audit_logs`) created in `internal/repo.rs`.
- [ ] Dioxus `MasterPinModal` and `SoloModeToggle` components created in `external/components/`.
- [ ] Isolated `external/style.css` created with SoftMorph variables.
- [ ] All in-slice functional tests (`test/fr/`) pass with 100% green via `cargo nextest`.
- [ ] All in-slice performance and security tests (`test/nfr/`) pass benchmarks.
- [ ] Zero compiler warnings (`cargo clippy -- -D warnings`).
- [ ] Tested on Desktop, Tablet, and Mobile viewport layouts.

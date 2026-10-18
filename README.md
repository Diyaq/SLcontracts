# SorobanLard — Contract Repo

> **On-chain course enrollment, payments, and certificate issuance on Stellar Soroban**

SorobanLard is an online learning platform for practical vocational skills — tailoring, makeup artistry, baking, photography, hairstyling, nail technology, fashion design, and more. This Soroban smart contract handles the trustless financial and credential layer of the platform: course enrollment payments, automatic instructor revenue splits, and verifiable on-chain certificates of completion.

## Short version

SorobanLard Contracts is the Soroban smart contract layer for the SorobanLard platform. It manages course lifecycle state, enrollment payments, instructor revenue splits, certificate issuance, and revocation in a trustless and auditable way.

The contract provides the core rules for:
- registering and approving courses
- pausing and unpausing courses
- archiving courses permanently
- enrolling students and splitting payments automatically
- marking enrollments complete
- issuing and revoking certificates
- emitting events for off-chain indexing and backend synchronization

### Main contract concepts
- Courses move through a lifecycle from Pending to Active, Paused, and Archived.
- Enrollments trigger automatic platform and instructor payment splits.
- Certificates can be issued after completion and later revoked with metadata captured on-chain.

### Key features
- multi-admin authorization for sensitive operations
- course status enforcement and lifecycle rules
- instructor earnings accounting
- certificate issuance and revocation tracking
- Soroban event emission for off-chain systems and indexers

The detailed documentation below expands on the architecture, data model, contract functions, and deployment flow.

This is **Repo 1 of 3** in the SorobanLard project:

| Repo | Description |
|------|-------------|
| `sorobanlard-contract` ← you are here | Soroban smart contract (Rust) |
| `sorobanlard-backend` | NestJS REST API + content management |
| `sorobanlard-frontend` | Next.js student and instructor portal |

---

## Table of Contents

- [What the Contract Does](#what-the-contract-does)
- [What Lives Off-Chain](#what-lives-off-chain)
- [Architecture](#architecture)
- [Data Structures](#data-structures)
- [Contract Functions](#contract-functions)
- [Revenue Split Logic](#revenue-split-logic)
- [Certificate Verification](#certificate-verification)
- [Events](#events)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup & Installation](#setup--installation)
- [Running Tests](#running-tests)
- [Building](#building)
- [Deploying to Testnet](#deploying-to-testnet)
- [Security Considerations](#security-considerations)
- [Upgrade Time-Lock Guidance](#upgrade-time-lock-guidance)
- [Contract Versioning](#contract-versioning)
- [Roadmap](#roadmap)

---

## What the Contract Does

The SorobanLard contract is the trust layer for the platform. It handles three responsibilities:

**1. Course Registry**
Instructors register courses on-chain with a price and fee structure. Each course starts as `Pending` — an admin must approve it before students can enroll. This gives the platform moderation control without being able to move anyone's funds.

**2. Enrollment Payments**
When a student enrolls, the contract automatically splits the payment in a single transaction:
- Platform fee → treasury address
- Instructor share → instructor address directly

No escrow needed — the payment splits and lands in both wallets immediately. The contract records the enrollment so no student can enroll twice.

**3. On-Chain Certificates**
After a student completes a course (verified off-chain by the backend), the admin issues an on-chain certificate. Each certificate is a permanent, verifiable record tied to the student's Stellar address and the course. Anyone can verify a certificate by querying `verify_certificate(certificate_id)` — no trusted middleman needed. Certificates can be revoked by the admin if issued in error.

---

## What Lives Off-Chain

The contract intentionally stores only what is necessary for payments and verification. Everything else lives in the backend database:

| Off-chain (backend) | On-chain (contract) |
|---|---|
| Video lessons | Course ID, price, instructor address |
| Course descriptions | Total enrollments, total earned |
| Assignments | Enrollment records |
| Student progress % | Completion status |
| Downloadable resources | Certificate with course title |
| User profiles | — |
| Instructor analytics | — |

This keeps the contract small, cheap to call, and focused on what blockchains are actually good at: verifiable ownership, trustless payments, and permanent records.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                SorobanLard Contract (Soroban)                    │
│                                                              │
│  ┌──────────────┐   ┌──────────────────┐   ┌─────────────┐  │
│  │   Course     │   │   Enrollment     │   │ Certificate │  │
│  │   Registry   │   │   + Payment      │   │   Issuance  │  │
│  │  (Pending →  │   │   Split          │   │  (on-chain  │  │
│  │   Active)    │   │  (instant)       │   │   NFT-like) │  │
│  └──────────────┘   └──────────────────┘   └─────────────┘  │
└──────────────────────────────────────────────────────────────┘
         ↑                    ↑                    ↑
    Instructor           Student pays         Admin issues
    registers            at enroll           after backend
    course               (auto-split)        verifies completion
```

### Roles

| Role | Address | Permissions |
|------|---------|-------------|
| **Admin** | Set at `init` | Approve/archive courses, mark completions, issue/revoke certificates, update fee |
| **Instructor** | Set per course | Register courses, pause/unpause their own courses |
| **Student** | Any address | Enroll in active courses |
| **Treasury** | Set at `init` | Receives platform fee share (passive recipient only) |

---

## Data Structures

### `Course`

```rust
pub struct Course {
    pub id: String,                    // matches backend DB record ID
    pub instructor: Address,
    pub price: i128,                   // enrollment price in USDC stroops
    pub platform_fee_percent: u32,     // 0-100; remainder goes to instructor
    pub token: Address,                // USDC Stellar Asset Contract
    pub total_enrollments: u32,
    pub total_earned: i128,
    pub status: CourseStatus,          // Pending | Active | Paused | Archived
    pub created_at_ledger: u32,
}
```

### `Enrollment`

```rust
pub struct Enrollment {
    pub student: Address,
    pub course_id: String,
    pub amount_paid: i128,
    pub enrolled_at_ledger: u32,
    pub completed: bool,
    pub certificate_issued: bool,
}
```

### `Certificate`

```rust
pub struct Certificate {
    pub id: String,                    // unique cert ID (UUID from backend)
    pub student: Address,
    pub course_id: String,
    pub course_title: String,          // stored on-chain for direct verification
    pub instructor: Address,
    pub issued_at_ledger: u32,
    pub revoked: bool,
}
```

### `CourseStatus` state machine

```
Pending ── approve_course() ──→ Active
Active  ── pause_course()   ──→ Paused
Paused  ── unpause_course() ──→ Active
Active  ── archive_course() ──→ Archived (admin only, permanent)
```

---

## Contract Functions

### `init(admin, treasury, default_fee_pct)`
Initialises the contract. Called once by the deployer.

### Course management

| Function | Caller | Description |
|---|---|---|
| `register_course(instructor, course_id, price, token, platform_fee_pct)` | Instructor | Register a course — starts as Pending |
| `approve_course(admin, course_id)` | Admin | Move Pending → Active |
| `pause_course(caller, course_id)` | Admin or Instructor | Move Active → Paused |
| `unpause_course(caller, course_id)` | Admin or Instructor | Move Paused → Active |
| `archive_course(admin, course_id)` | Admin only | Permanently archive |

### Enrollment

| Function | Caller | Description |
|---|---|---|
| `enroll(student, course_id)` | Student | Pay and enroll. Splits payment instantly. |

### Completion & Certificates

| Function | Caller | Description |
|---|---|---|
| `mark_completed(admin, student, course_id, evidence_hash)` | Admin | Called after backend verifies all lessons done. A non-empty `evidence_hash` replaces the student co-signature. |
| `issue_certificate(admin, student, course_id, cert_id, course_title, enrollment_reference, expires_at_ledger, instructor_signature)` | Admin | Issues on-chain certificate. Requires completion. `enrollment_reference` is a free-form backend ID (e.g. UUID). |
| `revoke_certificate(admin, cert_id)` | Admin | Flags certificate as revoked |
| `mark_completed(admin, student, course_id)` | Admin | Called after backend verifies all lessons done |
| `issue_certificate(admin, cert_id, student, course_id, course_title)` | Admin | Issues on-chain certificate. Requires completion. |
| `revoke_certificate(admin, cert_id, reason)` | Admin | Flags certificate as revoked |
| `bulk_revoke_course_certificates(admin, course_id)` | Admin | Flags every certificate of a course as revoked in one transaction |

### Admin management

| Function | Caller | Description |
|---|---|---|
| `transfer_admin(current_admin, new_admin)` | Admin | Transfer admin role |
| `update_treasury(admin, new_treasury)` | Admin | Update treasury address |
| `update_default_fee(admin, new_fee_pct)` | Admin | Update default platform fee |

### Read-only queries

| Function | Returns |
|---|---|---|
| `get_course(course_id)` | Full `Course` struct |
| `get_enrollment(student, course_id)` | Full `Enrollment` struct |
| `get_certificate(caller, certificate_id)` | Full `Certificate` struct (student, instructor, or admin only) |
| `is_enrolled(student, course_id)` | `bool` |
| `has_completed(student, course_id)` | `bool` |
| `verify_certificate(certificate_id)` | `bool` — true if exists and not revoked |
| `get_platform_fee()` | `u32` — current default fee % |
| `get_upgrade_timelock()` | `u32` — current upgrade time-lock, in ledgers |
| `get_contract_version()` | `String` — semantic version of the deployed contract code (e.g. `"0.1.0"`) |

---

## Revenue Split Logic

When a student calls `enroll()`, the contract calculates the split and executes both transfers in the same transaction:

```
price = 100 USDC
platform_fee_percent = 20

platform_amount   = 100 × 20 / 100 = 20 USDC → treasury
instructor_amount = 100 - 20       = 80 USDC → instructor
```

Both transfers are atomic — if either fails, the whole transaction reverts. The student is never charged without the instructor and treasury being paid.

The platform fee percentage can be:
- **Global default** (set at `init`, updatable by admin)
- **Per-course override** (set by instructor at registration time, non-zero value)

---

## Certificate Verification

Any third party — an employer, a client, a government agency — can verify a SorobanLard certificate without trusting the platform:

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network testnet \
  -- verify_certificate \
  --certificate_id "CERT-12345-TAILORING"
```

Returns `true` if the certificate exists and has not been revoked. The `get_certificate()` function returns the full record including the student's Stellar address, course title, instructor, and issuance ledger — all permanently recorded and independently auditable.

---

## Events

| Event name | Payload | When |
|---|---|---|
| `course_registered` | `course_id` | Instructor registers a course |
| `course_approved` | `course_id` | Admin approves a course |
| `course_paused` | `course_id` | Course paused |
| `course_unpaused` | `course_id` | Course unpaused |
| `course_archived` | `course_id` | Course archived |
| `student_enrolled` | `(course_id, student, amount_paid)` | Student enrolls and pays |
| `course_completed` | `(course_id, student)` | Completion marked by admin |
| `certificate_issued` | `(certificate_id, student, course_id)` | Certificate issued |
| `certificate_revoked` | `certificate_id` | Certificate revoked |
| `course_certificates_revoked` | `(admin, course_id, revoked_count, ledger)` | All certificates of a course bulk-revoked |
| `admin_transferred` | `new_admin` | Admin role transferred |

The backend listens to these events and updates the database, sends notifications, and triggers email confirmations.

---

## Project Structure

```
sorobanlard-contract/
├── Cargo.toml                         ← Rust workspace config
├── Cargo.lock
├── .gitignore
├── README.md
├── INVARIANTS.md                      ← Documented contract invariants
├── AUDIT_CHECKLIST.md                 ← Pre-audit security self-review checklist
├── scripts/
│   └── check_invariants.sh            ← CI: invariant tests vs. documented invariants
└── contracts/
    └── sorobanlard/
        ├── Cargo.toml                 ← Contract package config
        ├── Makefile                   ← Build / deploy shortcuts
        └── src/
            ├── lib.rs                 ← Full contract logic
            └── test.rs                ← Unit tests (invariant tests in the `invariants` submodule)
```

---

## Prerequisites

```bash
# Rust + wasm32 target
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32v1-none

# Stellar CLI
cargo install --locked stellar-cli --features opt

# Testnet account
stellar keys generate --global my-account --network testnet
stellar keys fund my-account --network testnet
```

---

## Setup & Installation

```bash
git clone https://github.com/your-org/sorobanlard-contract.git
cd sorobanlard-contract
cargo check
```

---

## Running Tests

```bash
cargo test
cargo test -- --nocapture          # with output
cargo test test_full_lifecycle     # single test
cargo test invariants::            # invariant suite only (see INVARIANTS.md)
```

Expected:
```
running 14 tests
test test::test_approve_already_active_course ........... ok
test test::test_approve_course_success .................. ok
test test::test_approve_course_unauthorized ............. ok
test test::test_certificate_requires_completion ......... ok
test test::test_enroll_duplicate ........................ ok
test test::test_enroll_pending_course ................... ok
test test::test_enroll_success_with_payment_split ....... ok
test test::test_full_lifecycle_enroll_complete_certify .. ok
test test::test_init_success ............................ ok
test test::test_is_enrolled_check ....................... ok
test test::test_multiple_students_same_course ........... ok
test test::test_pause_and_unpause_course ................ ok
test test::test_register_course_custom_fee .............. ok
test test::test_register_course_success ................. ok
test test::test_register_duplicate_course ............... ok
test test::test_revoke_certificate ...................... ok
test test::test_update_platform_fee ...................... ok
```

---

## Building

```bash
make build      # → target/wasm32v1-none/release/sorobanlard.wasm
make optimize   # → target/wasm32v1-none/release/sorobanlard.optimized.wasm
```

---

## Deploying to Testnet

```bash
export STELLAR_ACCOUNT=my-account
make deploy-testnet
# → CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Initialize after deployment

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- init \
  --admin <ADMIN_ADDRESS> \
  --treasury <TREASURY_ADDRESS> \
  --default_fee_pct 20
```

### Register and approve a test course

```bash
# Register
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source instructor-account \
  --network testnet \
  -- register_course \
  --instructor <INSTRUCTOR_ADDRESS> \
  --course_id "COURSE-TAILORING-001" \
  --price 500000000 \
  --token <USDC_SAC_ADDRESS> \
  --platform_fee_pct 0

# Approve
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- approve_course \
  --admin <ADMIN_ADDRESS> \
  --course_id "COURSE-TAILORING-001"
```

> **USDC SAC on Testnet:** `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA`

---

## Security Considerations

- **Authorization**: Every write function calls `require_auth()` on the relevant party. Instructors cannot approve their own courses. Students cannot issue their own certificates. Only the admin can revoke certificates.
- **No double enrollment**: The contract checks for an existing enrollment record before processing payment. A student cannot be charged twice for the same course.
- **Atomic payment split**: Both the platform and instructor transfers happen in the same transaction. There is no intermediate escrow that could be exploited.
- **Certificate immutability**: Revoked certificates are flagged — not deleted. This preserves the audit trail and prevents a revocation from being hidden.
- **Admin is a single address**: For production, consider using a multi-sig wallet as the admin address to avoid a single point of failure.
- **Fee validation**: Platform fee percentage is validated to be ≤ 100 at both `init` and `register_course` time.
- **Upgrade time-lock floor**: `set_upgrade_timelock` rejects any value below `MIN_UPGRADE_TIMELOCK_LEDGERS` (17,280 ledgers ≈ 1 day), so even a compromised admin pair cannot set the review window to 0. See [Upgrade Time-Lock Guidance](#upgrade-time-lock-guidance) for the mainnet value.
- **Invariants**: The core accounting and state-machine invariants are listed in [INVARIANTS.md](INVARIANTS.md). Each one is verified by a `test_invariant_*` test.
- **Audit checklist**: Before any external audit, maintainers should work through [AUDIT_CHECKLIST.md](AUDIT_CHECKLIST.md). It covers this contract's specific risk areas.

---

## Upgrade Time-Lock Guidance

Upgrades are two-step: `propose_upgrade` → wait `get_upgrade_timelock()` ledgers → `upgrade_contract`. Both admins must sign each step. The time-lock is the community's only window to review a proposed Wasm hash, and to withdraw funds or raise an alarm if the upgrade looks malicious or buggy.

Ledger counts assume ~5 seconds per ledger.

| Setting | Ledgers | ≈ Duration | Use |
|---|---|---|---|
| `MIN_UPGRADE_TIMELOCK_LEDGERS` (hard floor, enforced on-chain) | 17,280 | 1 day | Lowest value `set_upgrade_timelock` accepts |
| `DEFAULT_UPGRADE_TIMELOCK_LEDGERS` (used if never configured) | 17,280 | 1 day | Testnet / development |
| **Recommended mainnet minimum** | **51,840** | **3 days** | Set before mainnet launch |
| Recommended once the platform holds significant funds | 120,960 | 7 days | Mature production |

**Recommendation:** right after deploying to mainnet, and before any student funds are deposited, call `set_upgrade_timelock` with **at least 51,840 ledgers (~3 days)**.

Why 3 days:
- **Reaction time across weekends and time zones.** A 1-day window can pass entirely over a weekend or holiday before instructors, students or the backend team notice an `upgrade_proposed` event. Three days ensures at least one business day falls inside any window.
- **Time to withdraw.** Instructors need enough time to call `withdraw_earnings` and pull out pending earnings before a malicious upgrade could redirect them.
- **Industry practice.** Mature on-chain protocols generally use governance time-locks of 2–7 days. Three days is the low end of that range and suits an early-stage platform that still needs to ship fixes quickly.
- **Emergency fixes still work.** The time-lock delays an upgrade but never blocks it, and `cancel_upgrade` can withdraw a bad proposal at any time. A critical bug can be contained with `pause_platform` (single admin) while the fix waits out the time-lock.

Increase it to ~7 days (120,960 ledgers) once the contract holds significant value and the release process is stable. Backends and monitoring should alert on the `upgrade_proposed`, `upgrade_cancelled`, `upgrade_executed` and `upgrade_timelock_updated` events.

---

## Contract Versioning

The deployed code reports its version through the read-only `get_contract_version()` function, which returns the `CONTRACT_VERSION` constant in `contracts/sorobanlard/src/lib.rs`. The backend and block explorers can compare this value before and after an `upgrade_executed` event. That tells them which functions and event payloads are available, without comparing Wasm hashes by hand.

`CONTRACT_VERSION` follows [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`):

| Bump | When |
|---|---|
| **MAJOR** | Breaking change for integrators: a public function is removed or renamed, its arguments or return type change, an event's topic or payload changes, or the storage layout changes and needs migration |
| **MINOR** | Backwards-compatible addition: a new public function, a new event, or a new optional behaviour |
| **PATCH** | Bug fix or internal change with no change to function signatures or event payloads |

While the major version is `0`, breaking changes may bump MINOR instead of MAJOR. Integrators should treat every `0.x` bump as potentially breaking.

**PR rule:** every PR that changes a public function signature or an event payload must bump `CONTRACT_VERSION`. The PR template includes a checklist item for this.

---

## Roadmap

- [x] Course registry with approval workflow
- [x] Enrollment payment with automatic split
- [x] On-chain certificate issuance and verification
- [x] Course pause/unpause by instructor or admin
- [x] Certificate revocation
- [x] 14+ unit tests
- [ ] Discount codes / coupon system (Soroban-based)
- [ ] Subscription enrollment model (monthly access)
- [ ] Instructor revenue withdrawal tracking
- [ ] Course bundle enrollments (enroll in multiple courses at once)
- [ ] Contract upgrade mechanism
- [ ] Mainnet deployment

---

## License

MIT

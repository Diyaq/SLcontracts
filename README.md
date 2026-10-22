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
- [Storage Tiers](#storage-tiers)
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

## Storage Tiers

The contract uses Soroban **instance** storage for singleton config, auth flags,
and global counters, and **persistent** storage for per-entity records and
append-only collections. The rule of thumb for new keys is:

> Singleton config / auth flag / global counter → instance.
> Per-course / per-student / per-instructor record or list → persistent.

Every current `DataKey` / `DataKey2` variant is classified in
[docs/storage-keys.md](docs/storage-keys.md), along with the rationale and the
race-freedom argument for `TotalActiveEnrollments`.

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
| **Secondary Admin** | Set at `init` | Co-signs sensitive operations (archival, upgrades, fee changes) |
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
Pending ── approve_course()  ──→ Active
Pending ── reject_course()   ──→ Rejected (terminal)
Active  ── pause_course() / freeze_instructor() ──→ Paused
Paused  ── unpause_course()  ──→ Active
Paused  ── archive_course()  ──→ Archived (both admins, no active enrollments)
```

---

## Contract Functions

All functions are `pub fn` on the contract. Signatures below mirror the current
`lib.rs`; see the source for exact argument types and return values.

### Initialization & Configuration

| Function | Signature | Description |
|----------|-----------|-------------|
| `init` | `init(admin, secondary_admin, treasury, default_fee_pct, max_courses_per_instructor, refund_window_ledgers, revocation_challenge_period)` | One-time setup: admins, treasury, default fee, instructor course cap, refund window, and revocation challenge period |
| `update_fee` | `update_fee(admin, new_fee_pct)` | Update the default platform fee (admin only) |
| `get_fee` | `get_fee()` | Read the current default platform fee |
| `get_admin` | `get_admin()` | Read the primary admin address |
| `get_secondary_admin` | `get_secondary_admin()` | Read the secondary admin address |
| `get_treasury` | `get_treasury()` | Read the treasury address |

### Course Lifecycle

| Function | Signature | Description |
|----------|-----------|-------------|
| `register_course` | `register_course(instructor, id, price, platform_fee_percent, token)` | Register a new course (starts `Pending`) |
| `approve_course` | `approve_course(admin, course_id)` | Approve a pending course → `Active` |
| `reject_course` | `reject_course(admin, course_id)` | Reject a pending course → `Rejected` |
| `pause_course` | `pause_course(instructor, course_id)` | Pause an active course |
| `unpause_course` | `unpause_course(instructor, course_id)` | Resume a paused course |
| `archive_course` | `archive_course(admin, secondary_admin, course_id)` | Permanently archive a course (both admins) |
| `get_course` | `get_course(course_id)` | Read a course record |
| `get_course_status` | `get_course_status(course_id)` | Read a course's status |

### Enrollment & Completion

| Function | Signature | Description |
|----------|-----------|-------------|
| `enroll` | `enroll(student, course_id)` | Enroll a student and split payment |
| `mark_completed` | `mark_completed(admin, student, course_id)` | Mark an enrollment complete (admin only) |
| `get_enrollment` | `get_enrollment(student, course_id)` | Read an enrollment record |
| `get_instructor_earnings` | `get_instructor_earnings(instructor)` | Read accumulated instructor earnings |

### Certificates

| Function | Signature | Description |
|----------|-----------|-------------|
| `issue_certificate` | `issue_certificate(admin, student, course_id, certificate_id, course_title)` | Issue a certificate for a completed enrollment |
| `revoke_certificate` | `revoke_certificate(admin, certificate_id, reason)` | Revoke a certificate with an on-chain reason |
| `verify_certificate` | `verify_certificate(certificate_id)` | Verify a certificate's validity |
| `get_certificate` | `get_certificate(certificate_id)` | Read a certificate record |

### Arbitration & Disputes

| Function | Signature | Description |
|----------|-----------|-------------|
| `open_dispute` | `open_dispute(student, course_id, reason)` | Open a dispute for an enrollment |
| `resolve_dispute` | `resolve_dispute(admin, dispute_id, resolution)` | Resolve an open dispute |
| `get_dispute` | `get_dispute(dispute_id)` | Read a dispute record |
| `challenge_revocation` | `challenge_revocation(student, certificate_id, reason)` | Challenge a certificate revocation within the challenge period |
| `resolve_revocation_challenge` | `resolve_revocation_challenge(admin, certificate_id, uphold)` | Resolve a pending revocation challenge |

### Waitlist

| Function | Signature | Description |
|----------|-----------|-------------|
| `join_waitlist` | `join_waitlist(student, course_id)` | Join a course waitlist |
| `leave_waitlist` | `leave_waitlist(student, course_id)` | Leave a course waitlist |
| `get_waitlist` | `get_waitlist(course_id)` | Read the waitlist for a course |
| `promote_from_waitlist` | `promote_from_waitlist(admin, course_id, student)` | Promote a waitlisted student to enrolled |

### Risk-based Fees

| Function | Signature | Description |
|----------|-----------|-------------|
| `set_risk_tier` | `set_risk_tier(admin, instructor, tier)` | Assign a risk tier to an instructor |
| `get_risk_tier` | `get_risk_tier(instructor)` | Read an instructor's risk tier |
| `get_effective_fee` | `get_effective_fee(instructor)` | Compute the effective fee for an instructor |

### Refunds

| Function | Signature | Description |
|----------|-----------|-------------|
| `request_refund` | `request_refund(student, course_id)` | Request a refund within the refund window |
| `approve_refund` | `approve_refund(admin, student, course_id)` | Approve a pending refund |
| `get_refund` | `get_refund(student, course_id)` | Read a refund record |

### Upgrades

| Function | Signature | Description |
|----------|-----------|-------------|
| `propose_upgrade` | `propose_upgrade(admin, new_wasm_hash)` | Propose a contract upgrade |
| `approve_upgrade` | `approve_upgrade(secondary_admin)` | Co-approve a proposed upgrade |
| `execute_upgrade` | `execute_upgrade()` | Execute an approved upgrade |
| `get_pending_upgrade` | `get_pending_upgrade()` | Read the pending upgrade proposal |

### Approvers

| Function | Signature | Description |
|----------|-----------|-------------|
| `add_approver` | `add_approver(admin, approver)` | Add an approver |
| `remove_approver` | `remove_approver(admin, approver)` | Remove an approver |
| `is_approver` | `is_approver(address)` | Check whether an address is an approver |
| `get_approvers` | `get_approvers()` | List all approvers |

---

## Revenue Split Logic

When a student enrolls, the payment is split in a single transaction:

```
platform_fee = price * effective_fee_percent / 100
instructor_share = price - platform_fee
```

The `effective_fee_percent` is the course's `platform_fee_percent`, adjusted by
the instructor's risk tier (see [Risk-based Fees](#risk-based-fees)). The platform
fee is sent to the treasury; the instructor share is sent directly to the
instructor.

---

## Certificate Verification

Anyone can verify a certificate by calling `verify_certificate(certificate_id)`.
The contract returns the certificate record and its validity, including whether
it has been revoked. Revocations are subject to a challenge period
(`revocation_challenge_period` set at `init`) during which the student may call
`challenge_revocation`.

---

## Events

The contract emits Soroban events for off-chain indexing and backend
synchronization. See the source for the full event list.

---

## Project Structure

```
sorobanlard-contract/
├── src/
│   ├── lib.rs          # contract entry points
│   ├── types.rs        # data structures
│   └── ...
├── docs/
│   └── storage-keys.md # storage key classification
├── Cargo.toml
└── README.md
```

---

## Prerequisites

- Rust toolchain
- Soroban CLI
- Stellar testnet account

---

## Setup & Installation

```bash
git clone https://github.com/SorobanLard/sorobanlard-contract
cd sorobanlard-contract
cargo build
```

---

## Running Tests

```bash
cargo test
```

---

## Building

```bash
soroban contract build
```

---

## Deploying to Testnet

```bash
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/sorobanlard_contract.wasm \
  --source <SECRET_KEY> \
  --network testnet
```

---

## Security Considerations

- Sensitive operations require admin authorization.
- Archival and upgrades require both the primary and secondary admin.
- Revocations are subject to a challenge period.
- Refunds are bounded by the refund window set at `init`.

---

## Roadmap

- [x] Course registry and lifecycle
- [x] Enrollment payments and revenue splits
- [x] Certificate issuance and revocation
- [x] Arbitration, waitlist, risk-based fees, refunds, upgrades, approvers
- [ ] Additional tooling and integrations

## Handsoff notes

<!-- handsoff-issue-222 -->
- #222: Add fuzz-style test for register_course course_id character validation

<!-- handsoff-issue-223 -->
- #223: Security review: single compromised admin's blast radius with SecondaryAdmin absent from read-paths

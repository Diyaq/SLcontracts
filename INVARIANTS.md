# SorobanLard Contract Invariants

This file lists the core financial and state-machine invariants the contract relies on. Each invariant has a stable ID (`INV-NN`).

## Test convention

Tests that verify an invariant live in the `invariants` submodule at the bottom of [`contracts/sorobanlard/src/test.rs`](contracts/sorobanlard/src/test.rs). They are named:

```
test_invariant_invNN_<short_description>
```

`NN` is the invariant's ID below. One invariant can have several tests. Ordinary functional and happy-path tests stay outside this module.

Run only the invariant suite:

```bash
cargo test invariants::
```

CI runs [`scripts/check_invariants.sh`](scripts/check_invariants.sh). The script counts the documented invariants and the invariant tests, writes a summary, and fails if a documented invariant has no matching `test_invariant_invNN_*` test.

To add an invariant, add a `### INV-NN — <title>` heading here with the next free ID. Add at least one matching test in the same PR.

---

### INV-01 — Enrollment payments are fully accounted for
On a successful enrollment, the treasury fee plus the instructor earnings credit equals `Enrollment.amount_paid`. That amount equals what the student paid and is added to `Course.total_earned`. The contract holds the instructor share until it is withdrawn.
- Enforced in: `enroll_internal`, `deduct_fee`, `credit_instructor_earnings`
- Tests: `test_invariant_inv01_enrollment_payment_is_fully_accounted`

### INV-02 — At most one active enrollment per (student, course)
A student cannot pay for or enroll in a course they already have an active enrollment in.
- Enforced in: `validate_enrollment` / `enroll_internal`
- Tests: `test_invariant_inv02_no_double_enrollment`

### INV-03 — Certificates require a completed enrollment
`issue_certificate` only succeeds for an enrollment marked completed.
- Enforced in: `issue_certificate`
- Tests: `test_invariant_inv03_certificate_requires_completion`

### INV-04 — A finalised revocation is permanent
After a revocation's challenge period ends, `verify_certificate` returns `false` for good and `Certificate.revoked` never goes back to `false`.
- Enforced in: `revoke_certificate`, `apply_revocation`, `verify_certificate`
- Tests: `test_invariant_inv04_finalised_revocation_is_permanent`

### INV-05 — The effective fee never exceeds the payment
The base fee plus all risk surcharges is capped at 10,000 bps (100%). The platform fee is always in `[0, payment_amount]`.
- Enforced in: `compute_fee_breakdown`, `deduct_fee`
- Tests: `test_invariant_inv05_effective_fee_never_exceeds_payment`

### INV-06 — Instructors cannot withdraw more than they earned
`withdraw_earnings` pays out at most the credited `InstructorEarnings` balance and lowers it by exactly the amount paid out.
- Enforced in: `withdraw_earnings`, `debit_instructor_earnings`
- Tests: `test_invariant_inv06_cannot_withdraw_more_than_earned`, `test_invariant_inv06_withdrawal_pays_exactly_credited_balance`

### INV-07 — Arbitration fees are held 1:1 and paid out exactly once
While a case is open, the contract holds exactly `fee_paid` for it. Resolving the case releases exactly that amount to the student or the instructor. A resolved case cannot be resolved again.
- Enforced in: `escalate_to_arbitration`, `resolve_arbitration`
- Tests: `test_invariant_inv07_arbitration_fee_custody_is_conserved`, `test_invariant_inv07_arbitration_case_resolves_once`

### INV-08 — Multi-sig operations need two distinct registered admins
Every multi-sig operation requires both `Admin` and `SecondaryAdmin`. A single key signing twice is rejected.
- Enforced in: `require_multi_admin` (and the explicit check in `withdraw_tokens`)
- Tests: `test_invariant_inv08_multisig_rejects_same_admin_twice`

### INV-09 — Upgrades are time-locked, and the time-lock has a floor
`upgrade_contract` cannot run before the pending proposal's `effective_ledger`. `set_upgrade_timelock` rejects values below `MIN_UPGRADE_TIMELOCK_LEDGERS`.
- Enforced in: `propose_upgrade`, `upgrade_contract`, `set_upgrade_timelock`
- Tests: `test_invariant_inv09_upgrade_blocked_before_timelock`, `test_invariant_inv09_timelock_cannot_be_set_below_floor`

### INV-10 — The contract is initialised exactly once
A second call to `init` is rejected, so `init` cannot be used to overwrite admin, treasury or fee configuration.
- Enforced in: `init`
- Tests: `test_invariant_inv10_init_only_once`

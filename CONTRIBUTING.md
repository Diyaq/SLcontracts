# Contributing

## Storage keys

Before adding or changing any persistent storage key, read
[`docs/storage-keys.md`](docs/storage-keys.md).

In short:

- `DataKey` is **full** (50 / 50 variants, 0 headroom). Do not add variants to
  it.
- New **composite** keys must be standalone `#[contracttype]` structs stored
  directly (for example `ArbitrationCaseKey`), not tuple variants.
- New **singleton** keys go in `DataKey2` while it has headroom.
- Keep the documented variant counts, the enum comments, and the
  `test_data_key_variant_budgets_are_exact_and_documented` test in sync.

A compile-time assertion fails the build if either key enum exceeds the
50-variant XDR limit.

## Panic messages

Panic message strings in `lib.rs` follow a short style guide so the eventual
error-enum migration stays mechanical:

- **Lowercase** the first word (e.g. `course not found`, not `Course not found`).
- **No trailing period** or other end punctuation.
- Use consistent **`X cannot Y`** phrasing for precondition failures
  (e.g. `instructor cannot grade frozen course`).
- Use one term per concept — prefer `unauthorized` for caller/permission
  failures rather than mixing role-specific or state-specific wording.
- Dynamic values are fine, but keep the surrounding text in the same style
  (e.g. `token decimal precision must be {} (found {})`).

A CI grep check enforces the basics (no trailing period, no leading capital) on
panic strings; run it locally with the command documented in the workflow.

## Pull requests

- Branch from `main` and keep each PR focused.
- Run `cargo fmt --check` and `cargo test` before opening a PR.
- Add or update tests for any behaviour change, especially new events, guards,
  and storage keys.
- Update `README.md` when a public contract function or emitted event changes.

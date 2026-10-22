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

## Error and panic messages

Before adding or changing any panic message or error condition, read
[`ERRORS.md`](ERRORS.md). It is the canonical mapping from contract failures
to user-facing API error responses in the backend.

In short:

- Every panic message/condition must have a row in `ERRORS.md` describing its
  intended end-user meaning and the API response it maps to.
- Keep `code` values stable; only add new codes, never rename existing ones.
- Unknown panic messages must map to a generic `500` and be logged
  server-side, never leaked to end users.

## Pull requests

- Branch from `main` and keep each PR focused.
- Run `cargo fmt --check` and `cargo test` before opening a PR.
- Add or update tests for any behaviour change, especially new events, guards,
  and storage keys.
- Update `README.md` when a public contract function or emitted event changes.
- Update `ERRORS.md` whenever a new panic condition or error message is added,
  and keep the API error mapping in sync.

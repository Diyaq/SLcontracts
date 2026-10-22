# Contributing

## Storage keys

Before adding or changing any persistent storage key, read
[`docs/storage-keys.md`](docs/storage-keys.md).

### Why `DataKey` and `DataKey2` are split

`#[contracttype]` enums are limited to **50 variants** in the generated XDR.
`lib.rs` therefore splits storage keys across two enums so the contract can
keep growing without hitting that ceiling:

- `DataKey` — the original key enum, now **full** (50 / 50 variants, 0
  headroom). Do not add variants to it.
- `DataKey2` — the overflow enum for new keys. Check its current variant count
  and remaining headroom in the enum comment in `lib.rs` before adding a key.

### Where new keys go

1. **Composite** keys must be standalone `#[contracttype]` structs stored
   directly (for example `ArbitrationCaseKey`), not tuple variants.
2. **Singleton** keys go in `DataKey2` while it has headroom.
3. Once `DataKey2` approaches the 50-variant limit, add a new overflow enum
   (`DataKey3`, then `DataKey4`, ...) following the same pattern: a fresh
   `#[contracttype]` enum with its own documented variant count, and route new
   singleton keys there. Never reopen a full enum.

Keep the documented variant counts, the enum comments in `lib.rs`, and the
`test_data_key_variant_budgets_are_exact_and_documented` test in sync whenever
you add a key.

### Guardrails

- A compile-time assertion fails the build if either key enum exceeds the
  50-variant XDR limit.
- A CI check warns when either enum approaches the limit so the next overflow
  enum can be planned before the build breaks.

## Pull requests

- Branch from `main` and keep each PR focused.
- Run `cargo fmt --check` and `cargo test` before opening a PR.
- Add or update tests for any behaviour change, especially new events, guards,
  and storage keys.
- Update `README.md` when a public contract function or emitted event changes.

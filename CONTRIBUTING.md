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

## Pull requests

- Branch from `main` and keep each PR focused.
- Run `cargo fmt --check` and `cargo test` before opening a PR.
- Add or update tests for any behaviour change, especially new events, guards,
  and storage keys.
- Update `README.md` when a public contract function or emitted event changes.

## Continuous integration

Every push and pull request runs the CI workflow defined in
[`.github/workflows/ci.yml`](.github/workflows/ci.yml), which executes
`cargo test --workspace` and caches the cargo registry and `target`
directories to keep runs fast.

Maintainers should configure branch protection on `main` so the CI check is
**required for merge**. This is applied in the repository settings (Settings →
Branches → branch protection rules) and is not part of the code in this repo.

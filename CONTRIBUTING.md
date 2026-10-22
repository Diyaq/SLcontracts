# Contributing to SorobanLard

Thanks for your interest in contributing! This guide covers how the contract is
organized, the conventions we follow, and how to get your change merged.

## Repository layout

The core contract lives in a single large file:

```
contracts/sorobanlard/
├── src/
│   ├── lib.rs      # the entire contract (6000+ lines)
│   └── test.rs     # the test suite (291 tests)
├── Makefile        # build / optimize targets
└── Cargo.toml
```

### How `lib.rs` is organized

`lib.rs` is intentionally kept as one file, but it follows a fixed top-to-bottom
layout. When adding code, place it in the matching section rather than at the end
of the file:

1. **Fee / risk types** — enums and structs describing fees, risk parameters,
   and related configuration.
2. **Data types** — the domain structs and enums used by the contract's public
   API (positions, pools, etc.).
3. **Storage keys** — the `DataKey` / `DataKey2` enums used to address persistent
   storage.
4. **The `impl` block** — one large `impl` block containing the contract's
   functions, grouped by feature area.

Keep new public functions inside the existing `impl` block and near the related
feature code, and add a doc comment (`///`) describing behavior and any panics.

### `DataKey` vs `DataKey2`

Soroban's XDR representation limits an enum to **50 variants**. Because the
contract's storage keys exceed that limit, they are split across two enums:

- `DataKey` — the original set of storage keys.
- `DataKey2` — overflow keys added once `DataKey` reached the 50-variant cap.

When adding a new storage key:

- Add it to `DataKey` if it still has room (fewer than 50 variants).
- Otherwise add it to `DataKey2`.
- Never reorder or remove existing variants — their discriminants are part of
  the on-chain storage layout.

## Building and testing

Run these from `contracts/sorobanlard/`:

```sh
# Run the full test suite
cargo test

# Lint (must be clean)
cargo clippy --all-targets -- -D warnings

# Build the wasm artifact
make build

# Build an optimized wasm artifact
make optimize
```

### Adding a test

All tests live in `src/test.rs`. To add a test:

1. Add a `#[test]` function to `test.rs` following the naming and setup patterns
   of the surrounding tests.
2. Reuse the existing test helpers/fixtures rather than duplicating setup.
3. Cover both the success path and relevant failure/panic paths.
4. Run `cargo test` and confirm the new test passes alongside the existing suite.

## Pull request checklist

Before opening a PR, please confirm:

- [ ] The change is covered by tests (new behavior needs new tests).
- [ ] `cargo test` passes.
- [ ] `cargo clippy --all-targets -- -D warnings` is clean.
- [ ] New public functions have doc comments.
- [ ] New storage keys follow the `DataKey` / `DataKey2` rules above.
- [ ] The change stays focused on the linked issue.

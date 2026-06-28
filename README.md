# surgeist-test

Shared test infrastructure and integration fixtures for Surgeist.

This crate is for reusable test harnesses, fixture metadata, and cross-crate
verification support. It should not become a runtime dependency of production
Surgeist crates unless the top-level coordinator explicitly approves a boundary
change.

## API Artifact

The committed API coordination artifact lives at `api/public-api.txt`, but the
generator is owned by the root `surgeist` repo.

Refresh this crate's artifact from the root repo with:

```sh
cargo run --manifest-path api/generator/Cargo.toml -- --crate surgeist-test
```

API refresh tooling is command-only and must not run as part of normal
`cargo test`.

# surgeist-test

Shared test infrastructure and integration fixtures for Surgeist.

This crate is for reusable test harnesses, fixture metadata, and cross-crate
verification support. It should not become a runtime dependency of production
Surgeist crates unless the top-level coordinator explicitly approves a boundary
change.

## Fixture Metadata Boundaries

`surgeist-test` owns reusable fixture schemas and test harnesses. Schema types
must stay independent of implementation crates such as `surgeist-layout`, so
fixtures can describe layout-ready metadata without importing layout internals.

The root `surgeist` repo is responsible for generating adapter-composed fixture
metadata when a fixture needs multiple production crates. The generated metadata
should depend on stable schema types from this crate, not on root adapters at
runtime.

`surgeist-layout` consumes layout-ready fixture metadata. It should treat
`surgeist-test` as a test-facing source of fixture records and harness support,
not as a back channel to root or to sibling private module paths.

# surgeist-test

Shared test infrastructure and integration fixtures for Surgeist.

This crate is for reusable test harnesses, fixture metadata, and cross-crate
verification support. It should not become a runtime dependency of production
Surgeist crates unless the top-level coordinator explicitly approves a boundary
change.

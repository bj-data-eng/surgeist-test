# Layout Browser Parity Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the large `surgeist-layout` browser parity corpus, generator, XML parser, runner support, and reusable oracle helpers into `surgeist-test` without disrupting active layout work.

**Architecture:** `surgeist-test` becomes the owner of shared layout integration infrastructure under `src/layout/` and `fixtures/layout/browser_parity/`. `surgeist-layout` keeps algorithm implementation and focused unit tests, then depends on `surgeist-test` only as a dev-dependency for parity checks and fixture reuse. The migration is phased so the source corpus, generated XML, generator, and runner can be verified in `surgeist-test` before removing duplicated files from `surgeist-layout`.

**Tech Stack:** Rust 2024, Cargo path dependencies, `roxmltree`, `serde`, `serde_json`, `toml`, `sha2`, `chromiumoxide`, `futures`, `tokio`, `surgeist-layout`, `surgeist-style`, `surgeist-retained`.

---

## Coordinator Rules For This Plan

This plan is executable only for the `surgeist-test` crate lane through Task 8.
Tasks 9 and 10 are handoff packages for the owning `surgeist-layout` and root
`surgeist` coordinators. A worker running from this project must not edit,
delete, stage, commit, or push files in `/Users/codex/Development/surgeist-layout`
or `/Users/codex/Development/surgeist`.

Every crate-local implementation task must include a separate reviewer cycle
before its commit step. The worker must stop after verification, ask a clean
reviewer to inspect the changed files for that task, reconcile all Critical and
Important findings, rerun the listed checks, and only then commit. This rule is
not optional unless the user explicitly waives it for that task.

## Current Source Inventory

The source apparatus currently lives in `../surgeist-layout`:

- Browser parity corpus: `tests/layout/browser_parity/`
- Browser parity runner: `tests/layout/browser_parity.rs`
- Browser parity support/parser/tree adapter: `tests/layout/browser_parity/support.rs`
- Generator entry point: `tests/bin/surgeist-layout-generate.rs`
- Generator implementation: `tests/bin/surgeist-layout-generate/generator.rs`
- Layout test support/oracles: `tests/support/`
- Layout test module root: `tests/layout.rs` and `tests/layout/mod.rs`

The corpus is approximately 27 MB:

- `tests/layout/browser_parity/html/`: 1332 HTML fixtures
- `tests/layout/browser_parity/xml/`: 4978 XML/report files
- `tests/layout/browser_parity/scripts/gentest/`: 2 helper assets
- `tests/layout/browser_parity/corpus.toml`: source provenance and corpus accounting

The layout repo is currently active on `codex/calc-parity-fixtures` with local modifications in:

- `tests/bin/surgeist-layout-generate/generator.rs`
- `tests/layout/browser_parity/scripts/gentest/test_helper.js`

Do not start file movement until those changes are merged or deliberately handed off.

## Target File Structure

Create or modify these files in `surgeist-test`:

- Modify: `Cargo.toml`
- Modify: `src/lib.rs`
- Create: `src/layout/mod.rs`
- Create: `src/layout/browser_parity/mod.rs`
- Create: `src/layout/browser_parity/corpus.rs`
- Create: `src/layout/browser_parity/golden.rs`
- Create: `src/layout/browser_parity/runner.rs`
- Create: `src/layout/browser_parity/generator.rs`
- Create: `src/layout/oracle/mod.rs`
- Create: `src/layout/oracle/grid/*.rs`
- Create: `src/layout/oracle/inline.rs`
- Create: `src/layout/oracle_tree.rs`
- Create: `src/layout/grid_layout_comparison.rs`
- Create: `src/bin/surgeist-test-layout-generate.rs`
- Create: `tests/layout_browser_parity.rs`
- Create: `fixtures/layout/browser_parity/README.md`
- Copy: `fixtures/layout/browser_parity/corpus.toml`
- Copy: `fixtures/layout/browser_parity/html/**`
- Copy: `fixtures/layout/browser_parity/xml/**`
- Copy: `fixtures/layout/browser_parity/scripts/gentest/test_helper.js`
- Copy: `fixtures/layout/browser_parity/scripts/gentest/test_base_style.css`

Handoff package for `surgeist-layout` after `surgeist-test` is verified:

- Modify: `Cargo.toml`
- Modify: `tests/layout/browser_parity.rs`
- Modify: `tests/layout/mod.rs`
- Remove: `tests/layout/browser_parity/**`
- Remove: `tests/bin/surgeist-layout-generate.rs`
- Remove: `tests/bin/surgeist-layout-generate/**`
- Remove or narrow: `tests/support/**`

Handoff package for root `surgeist` after both crate-local changes pass:

- Modify: `Cargo.toml` if workspace membership or shared commands need adjustment
- Modify: `.gitmodules` only if submodule metadata changed
- Modify: `crates/surgeist-test` submodule pointer
- Modify: `crates/surgeist-layout` submodule pointer

---

### Task 1: Preflight Coordination Gate

**Files:**
- Inspect: `/Users/codex/Development/surgeist-layout/tests/bin/surgeist-layout-generate/generator.rs`
- Inspect: `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/scripts/gentest/test_helper.js`
- Inspect: `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/README.md`
- Inspect: `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/corpus.toml`

- [ ] **Step 1: Confirm both repositories are in expected states**

Run:

```sh
git -C /Users/codex/Development/surgeist-test status --short --branch
git -C /Users/codex/Development/surgeist-layout status --short --branch
git -C /Users/codex/Development/surgeist status --short --branch
```

Expected before migration begins:

```text
surgeist-test: clean or only this plan changed
surgeist-layout: no unrelated local edits, or explicit coordinator handoff for the active parity branch
surgeist: clean, with crates/surgeist-test already present as a submodule
```

- [ ] **Step 2: Capture the source inventory**

Run:

```sh
find /Users/codex/Development/surgeist-layout/tests/layout/browser_parity -type f | sed 's#^/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/##' | awk 'BEGIN{FS="/"} {count[$1]++} END{for (k in count) print k,count[k]}' | sort
du -sh /Users/codex/Development/surgeist-layout/tests/layout/browser_parity /Users/codex/Development/surgeist-layout/tests/support /Users/codex/Development/surgeist-layout/tests/bin
```

Expected at the time this plan was written:

```text
README.md 1
corpus.toml 1
html 1332
scripts 2
support.rs 1
xml 4978
27M tests/layout/browser_parity
224K tests/support
224K tests/bin
```

If the counts changed because the active layout branch landed, update this plan in `surgeist-test` before executing Task 2.

- [ ] **Step 3: Obtain the source crate’s focused parity check result**

Ask the `surgeist-layout` coordinator for a fresh result for these commands,
run from the `surgeist-layout` project:

```sh
cargo test -p surgeist-layout --test layout parses_browser_parity_xml
cargo test -p surgeist-layout --test layout runs_browser_parity_smoke_fixture_against_surgeist_layout
cargo test -p surgeist-layout --test layout browser_parity_corpus_manifest_exists
```

Expected evidence: all three commands passed in the layout lane. If any command
failed or no current result is available, stop and request layout coordinator
action before moving files.

- [ ] **Step 4: Commit or defer**

If `surgeist-layout` is still active or dirty from another lane, stop here and leave this plan as the handoff artifact. If it is clean and the coordinator approves the move, continue to Task 2.

---

### Task 2: Introduce the `surgeist-test` Layout Module Shell

**Files:**
- Modify: `/Users/codex/Development/surgeist-test/src/lib.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/mod.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`
- Create: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Write the failing module exposure test**

Create `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`:

```rust
#[test]
fn layout_browser_parity_module_is_public() {
    let root = surgeist_test::layout::browser_parity::default_fixture_root();
    assert!(root.ends_with("fixtures/layout/browser_parity"));
}
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity layout_browser_parity_module_is_public
```

Expected: FAIL because `surgeist_test::layout` is not defined.

- [ ] **Step 3: Add the public module shell**

Update `/Users/codex/Development/surgeist-test/src/lib.rs`:

```rust
//! Shared test infrastructure and integration fixtures for Surgeist.
//!
//! This crate is intentionally test-facing. Keep reusable harness code,
//! fixture metadata, and cross-crate verification support here when that
//! reduces context pressure in implementation crates.
//!
//! Production Surgeist crates should not depend on this crate without an
//! explicit boundary decision from the top-level coordinator.

#![forbid(unsafe_code)]

pub mod layout;
```

Create `/Users/codex/Development/surgeist-test/src/layout/mod.rs`:

```rust
//! Shared layout verification infrastructure.

pub mod browser_parity;
```

Create `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`:

```rust
//! Browser-derived layout parity fixtures and harness support.

use std::path::{Path, PathBuf};

const DEFAULT_FIXTURE_ROOT: &str = "fixtures/layout/browser_parity";

pub fn default_fixture_root() -> PathBuf {
    Path::new(env!("CARGO_MANIFEST_DIR")).join(DEFAULT_FIXTURE_ROOT)
}
```

- [ ] **Step 4: Run the focused test to verify it passes**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity layout_browser_parity_module_is_public
```

Expected: PASS.

- [ ] **Step 5: Run formatting**

Run:

```sh
cargo fmt --check
```

Expected: PASS.

- [ ] **Step 6: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 2 changes in `src/lib.rs`,
`src/layout/mod.rs`, `src/layout/browser_parity/mod.rs`, and
`tests/layout_browser_parity.rs`. Reconcile any Critical or Important findings
and rerun Steps 4 and 5.

- [ ] **Step 7: Commit**

Run:

```sh
git add src/lib.rs src/layout/mod.rs src/layout/browser_parity/mod.rs tests/layout_browser_parity.rs
git commit -m "test: add layout parity module shell"
```

---

### Task 3: Move the Browser Parity XML Model and Parser

**Files:**
- Modify: `/Users/codex/Development/surgeist-test/Cargo.toml`
- Modify: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/golden.rs`
- Modify: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Add the parser dependency**

Update `/Users/codex/Development/surgeist-test/Cargo.toml`:

```toml
[package]
name = "surgeist-test"
version = "0.1.0"
edition = "2024"
license = "MIT"
repository = "https://github.com/bj-data-eng/surgeist-test"
readme = "README.md"
description = "Shared test infrastructure and integration fixtures for Surgeist."

[lib]
name = "surgeist_test"
path = "src/lib.rs"

[dependencies]
roxmltree = "=0.21.1"
```

- [ ] **Step 2: Write the failing parser test**

Append to `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`:

```rust
#[test]
fn parses_minimal_browser_parity_xml() {
    let golden = surgeist_test::layout::browser_parity::Golden::parse(
        r#"
        <!-- generated-by: surgeist-test-layout-generate schema=1 source="html/block/basic.html" source-sha256="abc" helper-sha256="def" browser="Chrome/149" -->
        <test name="block_basic__border_box_ltr" use-rounding="true">
            <viewport width="max-content" height="max-content" root-context="root" />
            <input>
                <div display="block" width="50px">
                    <div height="10px" />
                    <div height="10px" />
                </div>
            </input>
            <expectations>
                <node x="0" y="0" width="50" height="20">
                    <node x="0" y="0" width="50" height="10" />
                    <node x="0" y="10" width="50" height="10" />
                </node>
            </expectations>
        </test>
        "#,
    )
    .expect("minimal browser parity XML should parse");

    assert_eq!(golden.name, "block_basic__border_box_ltr");
    assert!(golden.use_rounding);
    assert_eq!(
        golden.viewport.width,
        surgeist_test::layout::browser_parity::Available::MaxContent
    );
    assert_eq!(golden.root.children.len(), 2);
    assert_eq!(golden.expectations.width, Some(50.0));
    assert_eq!(golden.expectations.children[1].y, Some(10.0));
}
```

- [ ] **Step 3: Run the parser test to verify it fails**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity parses_minimal_browser_parity_xml
```

Expected: FAIL because `Golden` and related types are not exported.

- [ ] **Step 4: Port the XML model and parsing code**

Create `/Users/codex/Development/surgeist-test/src/layout/browser_parity/golden.rs` by copying these items from `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/support.rs`:

```text
Golden
Available
Viewport
RootContext
Node
NodeKind
StyleAttrs
Expectation
Size
Error
parse_node
parse_expectation
parse_available
parse_root_context
parse_bool
parse_number
optional_number_attr
required_attr
one_child
one_element_child
expect_tag
```

Keep the public type names and field names identical to the source file so existing layout assertions can migrate mechanically.

Update `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`:

```rust
//! Browser-derived layout parity fixtures and harness support.

use std::path::{Path, PathBuf};

mod golden;

pub use golden::{
    Available, Error, Expectation, Golden, Node, NodeKind, RootContext, Size, StyleAttrs, Viewport,
};

const DEFAULT_FIXTURE_ROOT: &str = "fixtures/layout/browser_parity";

pub fn default_fixture_root() -> PathBuf {
    Path::new(env!("CARGO_MANIFEST_DIR")).join(DEFAULT_FIXTURE_ROOT)
}
```

- [ ] **Step 5: Run the parser test to verify it passes**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity parses_minimal_browser_parity_xml
```

Expected: PASS.

- [ ] **Step 6: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 7: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 3 changes in `Cargo.toml`,
`src/layout/browser_parity/mod.rs`, `src/layout/browser_parity/golden.rs`, and
`tests/layout_browser_parity.rs`. The reviewer must check parser API shape,
error messages, copied source completeness, and test coverage. Reconcile any
Critical or Important findings and rerun Steps 5 and 6.

- [ ] **Step 8: Commit**

Run:

```sh
git add Cargo.toml src/layout/browser_parity/mod.rs src/layout/browser_parity/golden.rs tests/layout_browser_parity.rs
git commit -m "test: add layout browser parity XML parser"
```

---

### Task 4: Move Minimal Fixture Ownership Into `surgeist-test`

**Files:**
- Create: `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/README.md`
- Create: `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/corpus.toml`
- Create: `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/xml/block/block_basic__border_box_ltr.xml`
- Modify: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`
- Modify: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Add fixture discovery tests**

Append to `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`:

```rust
#[test]
fn lists_checked_in_browser_parity_xml() {
    let fixtures = surgeist_test::layout::browser_parity::fixture_files("xml")
        .expect("fixture files should be listed");

    assert_eq!(fixtures.len(), 1);
    assert!(fixtures[0].ends_with("xml/block/block_basic__border_box_ltr.xml"));
}

#[test]
fn parses_checked_in_browser_parity_xml() {
    let fixtures = surgeist_test::layout::browser_parity::fixture_files("xml")
        .expect("fixture files should be listed");

    for fixture in fixtures {
        surgeist_test::layout::browser_parity::Golden::parse_file(&fixture)
            .unwrap_or_else(|error| panic!("{} failed to parse: {error}", fixture.display()));
    }
}
```

- [ ] **Step 2: Run the fixture tests to verify they fail**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity lists_checked_in_browser_parity_xml
cargo test -p surgeist-test --test layout_browser_parity parses_checked_in_browser_parity_xml
```

Expected: FAIL because no fixture files exist and `fixture_files` is not implemented.

- [ ] **Step 3: Create the minimal fixture corpus**

Create `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/README.md`:

```markdown
# Surgeist Layout Browser Parity

This directory contains browser-derived layout parity fixtures owned by
`surgeist-test`.

- `html/` contains constrained source fixtures.
- `xml/` contains generated browser expectations consumed by Rust tests.
- `corpus.toml` records source provenance and corpus accounting.
- `scripts/gentest/` contains browser measurement helper assets.

XML files are generated artifacts. Do not edit browser geometry in XML by hand;
update the source fixture, importer, manifest, or generator instead.
```

Create `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/corpus.toml`:

```toml
schema_version = 1

[source_roots.surgeist]
kind = "surgeist"
path = "html"
description = "Surgeist-authored constrained HTML fixtures."
```

Create `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/xml/block/block_basic__border_box_ltr.xml`:

```xml
<!-- generated-by: surgeist-test-layout-generate schema=1 source="html/block/block_basic.html" source-sha256="abc" helper-sha256="def" browser="Chrome/149" -->
<test name="block_basic__border_box_ltr" use-rounding="true">
  <viewport width="max-content" height="max-content" root-context="root" />
  <input>
    <div display="block" width="50px">
      <div height="10px" />
      <div height="10px" />
    </div>
  </input>
  <expectations>
    <node x="0" y="0" width="50" height="20">
      <node x="0" y="0" width="50" height="10" />
      <node x="0" y="10" width="50" height="10" />
    </node>
  </expectations>
</test>
```

- [ ] **Step 4: Implement fixture discovery**

Move `fixture_files`, `fixture_files_in`, and recursive file collection helpers from `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/support.rs` into `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`.

The public API should be:

```rust
pub fn fixture_files(relative_dir: &str) -> Result<Vec<PathBuf>, Error> {
    fixture_files_in(&default_fixture_root().join(relative_dir), "xml")
}

pub fn fixture_files_in(root: &Path, extension: &str) -> Result<Vec<PathBuf>, Error> {
    let mut files = Vec::new();
    collect_files_with_extension(root, extension, &mut files)?;
    files.sort();
    Ok(files)
}
```

- [ ] **Step 5: Run fixture tests to verify they pass**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity lists_checked_in_browser_parity_xml
cargo test -p surgeist-test --test layout_browser_parity parses_checked_in_browser_parity_xml
```

Expected: PASS.

- [ ] **Step 6: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 7: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 4 changes in
`fixtures/layout/browser_parity`, `src/layout/browser_parity/mod.rs`, and
`tests/layout_browser_parity.rs`. The reviewer must check fixture ownership,
fixture discovery behavior, and generated artifact wording. Reconcile any
Critical or Important findings and rerun Steps 5 and 6.

- [ ] **Step 8: Commit**

Run:

```sh
git add fixtures/layout/browser_parity src/layout/browser_parity/mod.rs tests/layout_browser_parity.rs
git commit -m "test: own initial layout parity fixtures"
```

---

### Task 5: Port the Layout Runner Adapter

**Files:**
- Modify: `/Users/codex/Development/surgeist-test/Cargo.toml`
- Modify: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/runner.rs`
- Modify: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Add layout runner dependencies**

Update `/Users/codex/Development/surgeist-test/Cargo.toml` dependencies:

```toml
[dependencies]
roxmltree = "=0.21.1"
surgeist-layout = { path = "../surgeist-layout", version = "=0.1.0" }
surgeist-retained = { path = "../surgeist-retained", version = "=0.1.0" }
surgeist-style = { path = "../surgeist-style", version = "=0.1.0" }
```

- [ ] **Step 2: Write the failing runner test**

Append to `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`:

```rust
#[test]
fn runs_checked_in_smoke_fixture_against_surgeist_layout() {
    let fixture = surgeist_test::layout::browser_parity::default_fixture_root()
        .join("xml/block/block_basic__border_box_ltr.xml");
    let golden = surgeist_test::layout::browser_parity::Golden::parse_file(&fixture)
        .expect("smoke fixture should parse");

    surgeist_test::layout::browser_parity::assert_surgeist_matches(&golden)
        .expect("surgeist layout should match the smoke fixture");
}
```

- [ ] **Step 3: Run the runner test to verify it fails**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity runs_checked_in_smoke_fixture_against_surgeist_layout
```

Expected: FAIL because `assert_surgeist_matches` is not exported.

- [ ] **Step 4: Port the runner support**

Create `/Users/codex/Development/surgeist-test/src/layout/browser_parity/runner.rs` by moving the layout-specific runner code from `/Users/codex/Development/surgeist-layout/tests/layout/browser_parity/support.rs`:

```text
StyleFixtureTree
assert_surgeist_matches
compute_viewport_flex_item_root
TestNode
TestTree
can_use_leaf_measurement
grid_text_container_needs_anonymous_child
TextMeasure
WrapWord
FontFamily
LineHeightState
compare_expectation
compare_number
compare_optional_number
to_node_input
font_size
line_height
font_family
parse_px_dimension
to_declarations
all style conversion helpers
all layout parser helpers
to_layout_available
```

Keep private helpers private. Export only the stable runner function:

```rust
pub fn assert_surgeist_matches(golden: &Golden) -> Result<(), Error>
```

Update `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`:

```rust
mod runner;

pub use runner::assert_surgeist_matches;
```

- [ ] **Step 5: Run the runner test to verify it passes**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity runs_checked_in_smoke_fixture_against_surgeist_layout
```

Expected: PASS.

- [ ] **Step 6: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 7: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 5 changes in `Cargo.toml`,
`src/layout/browser_parity/mod.rs`, `src/layout/browser_parity/runner.rs`, and
`tests/layout_browser_parity.rs`. The reviewer must check dependency direction,
runner API shape, style/layout conversion coverage, and smoke fixture behavior.
Reconcile any Critical or Important findings and rerun Steps 5 and 6.

- [ ] **Step 8: Commit**

Run:

```sh
git add Cargo.toml src/layout/browser_parity/mod.rs src/layout/browser_parity/runner.rs tests/layout_browser_parity.rs
git commit -m "test: add layout browser parity runner"
```

---

### Task 6: Move the Full Browser Parity Corpus

**Files:**
- Copy into: `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/`
- Modify: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Confirm the target fixture tree is clean**

Run:

```sh
git -C /Users/codex/Development/surgeist-test status --short -- fixtures/layout/browser_parity
```

Expected: no output. If any fixture file is modified or untracked, stop and reconcile those changes before replacing the corpus.

- [ ] **Step 2: Stage the full corpus and review the sync dry-run**

Run:

```sh
tmpdir="$(mktemp -d)"
printf '%s\n' "$tmpdir" > /tmp/surgeist-test-browser-parity-sync-dir
mkdir -p "$tmpdir/browser_parity"
cp /Users/codex/Development/surgeist-layout/tests/layout/browser_parity/README.md "$tmpdir/browser_parity/"
cp /Users/codex/Development/surgeist-layout/tests/layout/browser_parity/corpus.toml "$tmpdir/browser_parity/"
cp -R /Users/codex/Development/surgeist-layout/tests/layout/browser_parity/html "$tmpdir/browser_parity/html"
cp -R /Users/codex/Development/surgeist-layout/tests/layout/browser_parity/scripts "$tmpdir/browser_parity/scripts"
cp -R /Users/codex/Development/surgeist-layout/tests/layout/browser_parity/xml "$tmpdir/browser_parity/xml"
find "$tmpdir/browser_parity" -name support.rs -print
mkdir -p /Users/codex/Development/surgeist-test/fixtures/layout/browser_parity
rsync -aivn --delete "$tmpdir/browser_parity/" /Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/
```

Expected: the `find` command prints nothing. Review the dry-run output; it must
only show changes under `fixtures/layout/browser_parity`, and any deletion must
be an expected removal from replacing the prior minimal fixture corpus. If the
dry-run output includes an unexpected deletion, stop and reconcile before
continuing.

- [ ] **Step 3: Apply the reviewed corpus sync**

Run:

```sh
tmpdir="$(cat /tmp/surgeist-test-browser-parity-sync-dir)"
rsync -a --delete "$tmpdir/browser_parity/" /Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/
find /Users/codex/Development/surgeist-test/fixtures/layout/browser_parity -maxdepth 1 -mindepth 1 -print | sort
```

Expected: the top level contains only `README.md`, `corpus.toml`, `html/`,
`scripts/`, and `xml/`.

- [ ] **Step 4: Update corpus inventory tests**

Replace the one-fixture assertion in `lists_checked_in_browser_parity_xml`:

```rust
assert!(
    fixtures.len() >= 4972,
    "expected the full browser parity XML corpus, got {} fixture(s)",
    fixtures.len()
);
assert!(
    fixtures
        .iter()
        .any(|fixture| fixture.ends_with("xml/block/block_basic__border_box_ltr.xml")),
    "expected block_basic smoke fixture to be present"
);
```

Append:

```rust
#[test]
fn browser_parity_corpus_manifest_exists() {
    let manifest =
        surgeist_test::layout::browser_parity::default_fixture_root().join("corpus.toml");
    let raw = std::fs::read_to_string(&manifest)
        .unwrap_or_else(|error| panic!("{} should read: {error}", manifest.display()));

    assert!(raw.contains("schema_version = 1"));
    assert!(raw.contains("[source_roots.taffy]"));
    assert!(raw.contains("[source_roots.surgeist]"));
}
```

- [ ] **Step 5: Run fixture parsing across the full corpus**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity parses_checked_in_browser_parity_xml
```

Expected: PASS.

- [ ] **Step 6: Run the smoke runner test**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity runs_checked_in_smoke_fixture_against_surgeist_layout
```

Expected: PASS.

- [ ] **Step 7: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 8: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 6 changes in `fixtures/layout/browser_parity` and `tests/layout_browser_parity.rs`. The reviewer must check that `support.rs` was not copied into fixtures, the corpus counts match the settled source, XML files still parse, and no handwritten generated XML edits were introduced. Reconcile any Critical or Important findings and rerun Steps 5 through 7.

- [ ] **Step 9: Commit**

Run:

```sh
git add fixtures/layout/browser_parity tests/layout_browser_parity.rs
git commit -m "test: move layout browser parity corpus"
```

---

### Task 7: Port the Generator

**Files:**
- Modify: `/Users/codex/Development/surgeist-test/Cargo.toml`
- Create: `/Users/codex/Development/surgeist-test/src/bin/surgeist-test-layout-generate.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/generator.rs`
- Modify: `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`
- Modify: `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/README.md`

- [ ] **Step 1: Add generator feature and optional dependencies**

Update `/Users/codex/Development/surgeist-test/Cargo.toml`:

```toml
[dependencies]
chromiumoxide = { version = "=0.9.1", default-features = false, features = ["fetcher", "rustls", "zip8"], optional = true }
futures = { version = "=0.3.31", optional = true }
roxmltree = "=0.21.1"
serde = { version = "=1.0.228", features = ["derive"], optional = true }
serde_json = { version = "=1.0.145", optional = true }
sha2 = { version = "=0.10.9", optional = true }
surgeist-layout = { path = "../surgeist-layout", version = "=0.1.0" }
surgeist-retained = { path = "../surgeist-retained", version = "=0.1.0" }
surgeist-style = { path = "../surgeist-style", version = "=0.1.0" }
tokio = { version = "=1.48.0", features = ["fs", "macros", "rt-multi-thread"], optional = true }
toml = { version = "=0.9.8", optional = true }
url = { version = "=2.5.7", optional = true }

[features]
default = []
layout-browser-parity-generate = [
    "dep:chromiumoxide",
    "dep:futures",
    "dep:serde",
    "dep:serde_json",
    "dep:sha2",
    "dep:tokio",
    "dep:toml",
    "dep:url",
]

[[bin]]
name = "surgeist-test-layout-generate"
path = "src/bin/surgeist-test-layout-generate.rs"
required-features = ["layout-browser-parity-generate"]
```

- [ ] **Step 2: Create the binary entry point**

Create `/Users/codex/Development/surgeist-test/src/bin/surgeist-test-layout-generate.rs`:

```rust
#[tokio::main]
async fn main() {
    if let Err(error) = surgeist_test::layout::browser_parity::generator::run_from_env().await {
        eprintln!("surgeist-test-layout-generate: {error}");
        std::process::exit(1);
    }
}
```

- [ ] **Step 3: Port and rename the generator module**

Create `/Users/codex/Development/surgeist-test/src/layout/browser_parity/generator.rs` by copying `/Users/codex/Development/surgeist-layout/tests/bin/surgeist-layout-generate/generator.rs`.

Apply these mechanical changes in the copied file:

```text
ROOT_ENV stays SURGEIST_LAYOUT_BROWSER_PARITY_ROOT
FILTER_ENV becomes SURGEIST_LAYOUT_GENERATE_FILTER
DEFAULT_ROOT becomes Path::new(env!("CARGO_MANIFEST_DIR")).join("fixtures/layout/browser_parity")
DEFAULT_BROWSER_CACHE stays target/surgeist-browser
TEST_HELPER_SOURCE include path becomes include_str!("../../../fixtures/layout/browser_parity/scripts/gentest/test_helper.js")
TEST_BASE_STYLE_SOURCE include path becomes include_str!("../../../fixtures/layout/browser_parity/scripts/gentest/test_base_style.css")
command error prefix becomes surgeist-test-layout-generate
newly generated provenance prefix becomes generated-by: surgeist-test-layout-generate
```

The copied XML corpus may still contain `generated-by: surgeist-layout-generate`
until the first full regeneration in `surgeist-test`. Preserve compatibility by
accepting both provenance prefixes in corpus validation:

```rust
const CURRENT_GENERATOR_PROVENANCE: &str = "generated-by: surgeist-test-layout-generate";
const LEGACY_LAYOUT_GENERATOR_PROVENANCE: &str = "generated-by: surgeist-layout-generate";

fn has_supported_generator_provenance(raw: &str) -> bool {
    let trimmed = raw.trim_start();
    trimmed.starts_with("<!-- ")
        && (trimmed.contains(CURRENT_GENERATOR_PROVENANCE)
            || trimmed.contains(LEGACY_LAYOUT_GENERATOR_PROVENANCE))
}
```

Use `CURRENT_GENERATOR_PROVENANCE` for new XML writes. Do not rewrite copied XML
headers by hand.

Expose the module only for the feature in `/Users/codex/Development/surgeist-test/src/layout/browser_parity/mod.rs`:

```rust
#[cfg(feature = "layout-browser-parity-generate")]
pub mod generator;
```

- [ ] **Step 4: Update fixture documentation commands**

In `/Users/codex/Development/surgeist-test/fixtures/layout/browser_parity/README.md`, replace generator commands with:

```sh
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate
SURGEIST_LAYOUT_GENERATE_FILTER=subgrid cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate -- import-taffy
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate -- check-taffy-corpus
```

- [ ] **Step 5: Run generator corpus validation**

Run:

```sh
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate -- check-corpus
```

Expected: PASS. If this fails due to missing `target/surgeist-sources/taffy/<commit>`, run:

```sh
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate -- import-taffy
cargo run -p surgeist-test --features layout-browser-parity-generate --bin surgeist-test-layout-generate -- check-corpus
```

Expected after import: PASS.

- [ ] **Step 6: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo test -p surgeist-test --features layout-browser-parity-generate
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo clippy -p surgeist-test --all-targets --features layout-browser-parity-generate -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 7: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 7 changes in `Cargo.toml`,
`src/bin/surgeist-test-layout-generate.rs`, `src/layout/browser_parity/generator.rs`,
`src/layout/browser_parity/mod.rs`, and the fixture README. The reviewer must
check feature gating, crate-root path resolution, legacy provenance acceptance,
new provenance writing, and feature-enabled test/clippy coverage. Reconcile any
Critical or Important findings and rerun Steps 5 and 6.

- [ ] **Step 8: Commit**

Run:

```sh
git add Cargo.toml src/bin/surgeist-test-layout-generate.rs src/layout/browser_parity/generator.rs src/layout/browser_parity/mod.rs fixtures/layout/browser_parity/README.md
git commit -m "test: move layout browser parity generator"
```

---

### Task 8: Move Reusable Layout Oracle Helpers

**Files:**
- Create: `/Users/codex/Development/surgeist-test/src/layout/oracle/mod.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/oracle/inline.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/oracle/grid/*.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/oracle_tree.rs`
- Create: `/Users/codex/Development/surgeist-test/src/layout/grid_layout_comparison.rs`
- Modify: `/Users/codex/Development/surgeist-test/src/layout/mod.rs`
- Modify: `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`

- [ ] **Step 1: Add a failing oracle smoke test**

Append to `/Users/codex/Development/surgeist-test/tests/layout_browser_parity.rs`:

```rust
#[test]
fn grid_oracle_resolves_numeric_line() {
    let lines = surgeist_test::layout::oracle::grid::NamedGridLines::empty(
        surgeist_test::layout::oracle::grid::GridAxis::Column,
        5,
    );
    let report = surgeist_test::layout::oracle::grid::resolve_numeric_line(&lines, 2)
        .expect("line 2 should resolve in a 5-track explicit grid");

    assert_eq!(report, 2);
}
```

- [ ] **Step 2: Run the oracle smoke test to verify it fails**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity grid_oracle_resolves_numeric_line
```

Expected: FAIL because `layout::oracle` is not exported.

- [ ] **Step 3: Move support modules**

Copy these files from `/Users/codex/Development/surgeist-layout/tests/support/` into `/Users/codex/Development/surgeist-test/src/layout/`:

```sh
cp /Users/codex/Development/surgeist-layout/tests/support/grid_layout_comparison.rs /Users/codex/Development/surgeist-test/src/layout/grid_layout_comparison.rs
cp /Users/codex/Development/surgeist-layout/tests/support/oracle_tree.rs /Users/codex/Development/surgeist-test/src/layout/oracle_tree.rs
mkdir -p /Users/codex/Development/surgeist-test/src/layout/oracle
cp -R /Users/codex/Development/surgeist-layout/tests/support/oracle/. /Users/codex/Development/surgeist-test/src/layout/oracle/
```

Update module paths so imports refer to `crate::layout::oracle` and `crate::layout::oracle_tree` instead of `crate::support::oracle` and `crate::support::oracle_tree`.

Update `/Users/codex/Development/surgeist-test/src/layout/mod.rs`:

```rust
//! Shared layout verification infrastructure.

pub mod browser_parity;
pub mod grid_layout_comparison;
pub mod oracle;
pub mod oracle_tree;
```

- [ ] **Step 4: Run the oracle smoke test**

Run:

```sh
cargo test -p surgeist-test --test layout_browser_parity grid_oracle_resolves_numeric_line
```

Expected: PASS.

- [ ] **Step 5: Run the crate baseline**

Run:

```sh
cargo test -p surgeist-test
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo fmt --check
```

Expected: all pass.

- [ ] **Step 6: Run the required reviewer gate**

Ask a separate reviewer to inspect the Task 8 changes in `src/layout/**` and
`tests/layout_browser_parity.rs`. The reviewer must check module path rewrites,
public oracle API shape, dependency direction, and whether helpers remain
test-facing. Reconcile any Critical or Important findings and rerun Steps 4 and
5.

- [ ] **Step 7: Commit**

Run:

```sh
git add src/layout tests/layout_browser_parity.rs
git commit -m "test: move reusable layout oracle helpers"
```

---

### Task 9: Prepare the `surgeist-layout` Handoff

**Files:**
- Create: `/Users/codex/Development/surgeist-test/plans/layout-handoff-browser-parity-consumer.md`

This task creates a handoff package only. Do not edit `/Users/codex/Development/surgeist-layout` from this project.

- [ ] **Step 1: Write the layout handoff draft**

Create `/Users/codex/Development/surgeist-test/plans/layout-handoff-browser-parity-consumer.md`:

````markdown
# surgeist-layout Handoff: Consume surgeist-test Browser Parity Harness

## Owning Repo

`/Users/codex/Development/surgeist-layout`

This work must be executed by the `surgeist-layout` coordinator or a worker launched in that project. It must not be executed from the `surgeist-test` project.

## Prerequisites

- `surgeist-test` has completed the layout browser parity migration through oracle helper ownership.
- The `surgeist-test` commit is pushed or otherwise available to the layout repo.
- `surgeist-layout` has no unrelated local edits.

## Intended Changes

- Add `surgeist-test` as a dev-dependency.
- Replace local browser parity parser/runner usage with `surgeist_test::layout::browser_parity`.
- Replace layout-local reusable oracle imports with `surgeist_test::layout::oracle` and `surgeist_test::layout::oracle_tree`.
- Remove duplicated browser parity corpus and generator only after tests compile and a reviewer approves the deletion.

## Suggested Dependency Shape

```toml
[dev-dependencies]
roxmltree = "=0.21.1"
serde_json = "=1.0.145"
surgeist-retained = { path = "../surgeist-retained", version = "=0.1.0" }
surgeist-style = { path = "../surgeist-style", version = "=0.1.0" }
surgeist-test = { path = "../surgeist-test", version = "=0.1.0" }
```

Keep existing dev-dependencies that are still used by layout-local unit tests.

## Verification

Run from `/Users/codex/Development/surgeist-layout`:

```sh
cargo test -p surgeist-layout --test layout parses_browser_parity_xml
cargo test -p surgeist-layout --test layout runs_browser_parity_smoke_fixture_against_surgeist_layout
cargo test -p surgeist-layout --test layout browser_parity_corpus_manifest_exists
cargo test -p surgeist-layout
cargo clippy -p surgeist-layout --all-targets -- -D warnings
cargo fmt --check
```

Run the ignored full parity suite only as an explicit coordinator gate:

```sh
cargo test -p surgeist-layout --test layout runs_all_checked_in_browser_parity_xml -- --ignored
```

## Deletion Safety

Use `git rm` in the layout repo only after replacement tests compile, focused checks pass, and a separate reviewer approves the removal. Do not use broad `rm -rf` for tracked corpus or generator files.
````

- [ ] **Step 2: Review the handoff draft**

Ask a separate reviewer to inspect `plans/layout-handoff-browser-parity-consumer.md` for boundary clarity, missing commands, and unsafe deletion wording. Reconcile any Critical or Important findings.

- [ ] **Step 3: Commit the handoff draft**

Run:

```sh
git add plans/layout-handoff-browser-parity-consumer.md
git commit -m "docs: add layout parity handoff"
```

---

### Task 10: Prepare the Root Integration Handoff

**Files:**
- Create: `/Users/codex/Development/surgeist-test/plans/root-handoff-browser-parity-submodules.md`

This task creates a handoff package only. Do not edit `/Users/codex/Development/surgeist` from this project.

- [ ] **Step 1: Write the root handoff draft**

Create `/Users/codex/Development/surgeist-test/plans/root-handoff-browser-parity-submodules.md`:

````markdown
# Root surgeist Handoff: Integrate Shared Layout Parity Harness

## Owning Repo

`/Users/codex/Development/surgeist`

This work must be executed by the root coordinator or a worker launched in the root project. It must not be executed from the `surgeist-test` project.

## Prerequisites

- `surgeist-test` has a committed migration result.
- `surgeist-layout` has consumed `surgeist-test` and removed duplicated parity ownership.
- Both crate commits are pushed or otherwise fetchable by the root submodule checkout.

## Intended Changes

- Update `crates/surgeist-test` to the migration commit.
- Update `crates/surgeist-layout` to the consumer commit.
- Update root workspace metadata only if the root coordinator determines it is required.

## Verification

Run from `/Users/codex/Development/surgeist`:

```sh
git submodule status --recursive
git status --short --branch
cargo test -p surgeist-test
cargo test -p surgeist-layout --test layout parses_browser_parity_xml
cargo test -p surgeist-layout
cargo check --workspace
```

Run the ignored full parity suite only as an explicit root coordinator gate:

```sh
cargo test -p surgeist-layout --test layout runs_all_checked_in_browser_parity_xml -- --ignored
```

## Commit Scope

Commit only root-owned files and submodule pointers. Do not edit crate internals from the root project unless the owning crate coordinator explicitly hands that work to root.
````

- [ ] **Step 2: Review the handoff draft**

Ask a separate reviewer to inspect `plans/root-handoff-browser-parity-submodules.md` for boundary clarity, missing commands, and root integration assumptions. Reconcile any Critical or Important findings.

- [ ] **Step 3: Commit the handoff draft**

Run:

```sh
git add plans/root-handoff-browser-parity-submodules.md
git commit -m "docs: add root parity handoff"
```

---

## Completion Criteria

The migration is complete when:

- `surgeist-test` owns `fixtures/layout/browser_parity`, parser support, runner support, generator support, and reusable oracle helpers.
- `surgeist-layout` no longer owns the large browser parity corpus or generator implementation.
- `surgeist-layout` still runs its focused layout tests and browser parity tests through `surgeist-test`.
- Root `surgeist` points at compatible `surgeist-test` and `surgeist-layout` commits.
- Baseline checks pass:

```sh
cargo test -p surgeist-test
cargo test -p surgeist-test --features layout-browser-parity-generate
cargo clippy -p surgeist-test --all-targets -- -D warnings
cargo clippy -p surgeist-test --all-targets --features layout-browser-parity-generate -- -D warnings
cargo fmt --check
cargo test -p surgeist-layout
cargo check --workspace
```

The full ignored browser parity run should be treated as an explicit coordinator gate because it is larger than tight local iteration.

# t-rust-db

A family of Rust database engines and the tools around them, sharing one
storage and execution layer instead of each reinventing it.

## What's here

**Two engines, one core.** [`sqlite-rs`](https://github.com/t-rust-db/sqlite-rs)
is a safe, binary-compatible Rust reimplementation of SQLite — no reliance on
the C library, row-oriented, VDBE-shaped. [`column-rs`](https://github.com/t-rust-db/column-rs)
is a columnar analytics engine over Parquet, batch-oriented, read-only. They
are not two unrelated projects that happen to share a name: sqlite-rs is the
leading, more mature codebase, and column-rs's own planner and VM were
migrated *into* the shared layer to converge on its shape wherever the two
engines can share one, rather than the reverse.

**The shared layer, not duplicated per engine:**
- [`db-core`](https://github.com/t-rust-db/db-core) — the SQL language and
  execution layer: types, expressions, parser, joins, VM, codegen, one crate,
  feature-gated so a consumer builds only the row or column half it needs.
- [`db-storage`](https://github.com/t-rust-db/db-storage) — physical storage,
  one feature-gated module per execution mode (row's b-tree/pager/VFS,
  column's Parquet reader).
- [`db-cli`](https://github.com/t-rust-db/db-cli) — the REPL/readline
  infrastructure every engine's CLI plugs into.
- [`db-parquet`](https://github.com/t-rust-db/db-parquet) — the Parquet
  reader column-rs's storage layer sits on.
- [`grammar`](https://github.com/t-rust-db/grammar) — the SQL dialect
  specification both engines' parsers are checked against.

**Tools built on top:**
[`trigrep`](https://github.com/t-rust-db/trigrep) is a serverless
trigram-indexed grep — the problem [microsoft/tgrep](https://github.com/microsoft/tgrep)
solves, with `sqlite3`'s shape instead of tgrep's: one cache file per indexed
root, opened per invocation, no daemon to keep warm. Its cache is a real
SQLite-format database, built through `db-storage`'s b-tree and pager
directly. [`db-extensions`](https://github.com/t-rust-db/db-extensions)
holds database extensions that don't belong in either engine's core.

**Keeping everyone honest:**
[`benchmark`](https://github.com/t-rust-db/benchmark) — parity benchmarks
against the products these engines are measured against (DuckDB for
column-rs, ripgrep for trigrep) — and [`examples`](https://github.com/t-rust-db/examples),
runnable, checked example queries and scenarios for the family.

## How this family works

- **Convergence over duplication.** When sqlite-rs and column-rs can share a
  shape — the `Program`/`Instruction` bytecode mirror, the mechanism for
  crash-safe commits — they do, migrated into `db-core`/`db-storage` rather
  than kept as two copies that quietly drift.
- **ADR-driven design.** Every non-trivial decision — dependency direction,
  what's shared versus what stays engine-specific, why a crate exists at all
  — is recorded in that repo's `.openspec/adr/`, including the ones that were
  tried and reversed.
- **Zero third-party dependencies in the core**, one level at a time: a
  library declares none of its own; what arrives transitively through a
  first-party `t-rust-db` crate is that crate's debt to pay down, tracked
  explicitly rather than hidden in a lockfile. `#![forbid(unsafe_code)]`
  where the storage layer doesn't need a documented, reviewed exception.
- **The same gate set everywhere:** `cargo-deny` for supply chain,
  `cargo-mvl-limit` for a qualified language subset, `cargo-mvl-mcdc` for
  MC/DC coverage of every multi-condition decision, a line-coverage floor,
  a production panic-lint policy (typed errors in `src/`, fail-fast in
  tests) — copied crate to crate, not reinvented per repo.

Tests over claims: parity benchmarks are always run against the real
alternative before a number is written down, and a hypothesis about what's
slow is always measured before it's fixed.

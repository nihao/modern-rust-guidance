<p align="center">
  <img src="assets/logo.svg" alt="Modern Rust Guidance Logo" width="112" />
</p>

# Modern Rust Guidance

<p align="center"><b>English</b> · <a href="README.zh-CN.md">简体中文</a></p>

**Modern Rust 2024 language idioms and high-reliability backend engineering standards** packaged for AI coding agents. This skill is self-contained and domain-neutral, covering Rust 2024 Edition syntax, memory and concurrency soundness, exact fixed-point integer arithmetic, three-tier `thiserror` error causal chains, stateless PostgreSQL/SQLx transaction pipelines, and high-signal test layering.

---

## Installation

<details open>
<summary><b>Google Antigravity</b></summary>

```shell
agy plugin install https://github.com/nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>Vercel Skills CLI (<code>npx skills</code>)</b></summary>

```shell
npx skills add nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>GitHub CLI</b></summary>

```shell
gh skill install nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>Claude Code</b></summary>

```shell
/plugin marketplace add nihao/modern-rust-guidance
/plugin install modern-rust-guidance@nihao
```

</details>

<details>
<summary><b>OpenAI Codex</b></summary>

```shell
codex plugin marketplace add nihao/modern-rust-guidance
codex plugin add modern-rust-guidance@nihao
```

</details>

<details>
<summary><b>Manual Workspace Skill</b></summary>

Copy [`skills/modern-rust-guidance`](skills/modern-rust-guidance/SKILL.md) into your target project's `.agents/skills/modern-rust-guidance` directory.

</details>

---

## Progressive Reference Guides

The main skill entry point is [`skills/modern-rust-guidance/SKILL.md`](skills/modern-rust-guidance/SKILL.md). Coding agents load reference chapters on demand based on the language feature or engineering boundary involved, avoiding unnecessary context bloat:

### 1. Language & Syntax Layer (`references/lang/`)

| Guide | Coverage |
| --- | --- |
| [01-edition2024-and-syntax.md](skills/modern-rust-guidance/references/lang/01-edition2024-and-syntax.md) | RPIT `+ use<...>`, `async \|...\|` closures, native async traits, `unsafe extern` / `#[unsafe(...)]`, `LazyLock`, exhaustive destructuring, `@` and slice patterns, lazy combinators |
| [02-ownership-memory-and-collections.md](skills/modern-rust-guidance/references/lang/02-ownership-memory-and-collections.md) | Borrowing signatures, `Cow`, 4-step borrow conflict resolution, `mem::take`, drop order, niche & `Box` enum slimming, `Entry` API, `BTreeMap`, buffered I/O |
| [03-types-traits-generics-and-macros.md](skills/modern-rust-guidance/references/lang/03-types-traits-generics-and-macros.md) | Checked constructors & newtypes, associated types vs generics, `Borrow` vs `AsRef`, thin generic shells, standard trait coherence, closure captures, `const {}`, hygienic `macro_rules!` |
| [04-async-concurrency-and-unsafe.md](skills/modern-rust-guidance/references/lang/04-async-concurrency-and-unsafe.md) | Future size across `.await`, `spawn_blocking`, never holding sync locks across `.await`, `select!` cancel safety, bounded channels, atomic memory ordering, Rust 2024 `unsafe` & FFI |

### 2. High-Reliability Backend Engineering Layer (`references/eng/`)

| Guide | Coverage |
| --- | --- |
| [01-architecture-cqs-and-api.md](skills/modern-rust-guidance/references/eng/01-architecture-cqs-and-api.md) | Lightweight DDD × pragmatic DOD division, use-case CQS, zero-I/O pure core, thin `mod.rs`, role suffixes, storage recovery boundaries, and internal API restraint |
| [02-numeric-errors-and-observability.md](skills/modern-rust-guidance/references/eng/02-numeric-errors-and-observability.md) | Zero floats for exact values with `checked_*` fixed-point math, 3-tier `thiserror` causal chains, scoped `anyhow`, single-boundary redacted logging, non-blind retries |
| [03-postgres-sqlx-and-transactions.md](skills/modern-rust-guidance/references/eng/03-postgres-sqlx-and-transactions.md) | Runtime parameterized SQLx, stateless connection matrix, private `FromRow`, `ANY`/`UNNEST` anti-N+1, two write transaction pipelines, fail-closed audit/idempotency & append-only migrations |
| [04-serde-lints-testing-and-review.md](skills/modern-rust-guidance/references/eng/04-serde-lints-testing-and-review.md) | Checked `#[serde(try_from)]`, canonical key ordering, dual Cargo feature graphs, workspace lints & `#[expect]`, benchmark-first optimization, high-signal testing & 3-tier code review |

---

## 10 Critical Invariants

1. **Pure Core Zero I/O**: Domain computation modules remain synchronous and deterministic; never introduce `async`, database drivers, network, filesystem, env vars, clocks, or RNG.
2. **Composition Root & Unidirectional CQS**: Service hosts only wire dependencies and unpack protocols; commands commit state once, queries project committed facts read-only, and commands never depend on `queries`.
3. **Zero Floating-Point for Exact Values**: Never use `f32`/`f64`, lossy `as` casts, or `wrapping_*`/`saturating_*` for money, quantities, rates, weights, or thresholds; use explicitly scaled signed integers with `checked_*`/`TryFrom`.
4. **Typed Error Causal Chains & Scoped `anyhow`**: Never call `.unwrap()` in production code; forbid `anyhow` in library crates and request handlers (`anyhow` belongs only in process `main` and CLI entrypoints); preserve original `#[source]` via `thiserror`.
5. **Single Transaction Committer & Fail-Closed**: Public write commands own the pool transaction and perform the sole `commit`; collaborators take `&mut Transaction`; state writes, audit logs, and idempotency receipts commit atomically or roll back together.
6. **Runtime Parameterized SQL & No N+1**: Use runtime parameterized SQLx queries with module-private `FromRow` structs (no pool-holding `Repository` wrappers or `Arc<PgPool>`); batch with `ANY($1)`/`UNNEST` instead of `.await` inside loops.
7. **Edition 2024 Idioms & Checked Construction**: Apply RPIT `+ use<...>`, native `async |...|`, and lazy `*_else` combinators; enforce private fields with checked constructors on types with invariants (no bypassed `Deserialize` or speculative internal builders).
8. **Deterministic Ordering & Cryptographic Digests**: Observable outputs and canonical serializations never depend on randomized `HashMap`/`HashSet` iteration; persisted content digests use canonical encoding and cryptographic hashing rather than `DefaultHasher`.
9. **Concurrency Guards & Rust 2024 `unsafe`**: Never hold synchronous locks across `.await`; bind RAII guards to named variables; offload CPU-heavy work to `spawn_blocking`; keep minimal `unsafe {}` blocks inside `unsafe fn` with matching `# Safety` and `// SAFETY:` proofs.
10. **Workspace Lints & High-Signal Testing**: Forbid source-level `#![deny(warnings)]` and `#[allow]`—use narrow `#[expect(clippy::..., reason = "...")]` only when unavoidable; restrict unit tests to pure logic and invariant guards while consolidating DB tests into a single `tests/postgres.rs` per crate.

---

## Upstream & License

- Language-layer idioms are deeply informed by [`leonardomso/rust-skills`](https://github.com/leonardomso/rust-skills) (MIT License) and tightened for high-reliability backend engineering and compiler static gates; see [`UPSTREAM.md`](skills/modern-rust-guidance/UPSTREAM.md).
- Released under the [MIT License](LICENSE).

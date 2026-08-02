<div align="center">

# agave-rift-scheduler

[![CI](https://github.com/RFT-SIRM/agave-rift-scheduler/actions/workflows/ci.yml/badge.svg)](https://github.com/RFT-SIRM/agave-rift-scheduler/actions/workflows/ci.yml)
[![Fuzz](https://github.com/RFT-SIRM/agave-rift-scheduler/actions/workflows/fuzz-daily.yml/badge.svg)](https://github.com/RFT-SIRM/agave-rift-scheduler/actions/workflows/fuzz-daily.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

**Conflict-aware transaction scheduler with bounded retry semantics and explicit starvation observability.**

*A research implementation by [RFT-SIRM](https://github.com/RFT-SIRM/UltraCore-RFT).*

</div>

---

## What This Is

This repository contains a **reference implementation** of a transaction scheduler that explores bounded retry semantics and deterministic starvation observability. It was built to:

1. Model scheduling behavior under sustained account-level write contention.
2. Verify that bounded retry semantics can be enforced without invariant violations.
3. Provide a concrete proposal for discussion with the Agave core team.

**Upstream engagement:**
- 📋 **[RFC] Bounded retry semantics and starvation observability for GreedyScheduler** — [anza-xyz/agave#14274](https://github.com/anza-xyz/agave/issues/14274)
- 🔒 **CPI permission leakage in SVM runtime** — [anza-xyz/svm#25](https://github.com/anza-xyz/svm/issues/25) (related security research from the same lab)

**This is not a production patch for Agave.** It is a research artifact. See [Disclaimer](#disclaimer) below.

---

## Context: Why This Research Exists

The default `GreedyScheduler` in Agave (`core/src/banking_stage/transaction_scheduler/greedy_scheduler.rs`) defers conflicting transactions by re-inserting them into the priority queue. This design is correct for the common case, but it does not expose:

- a per-transaction retry counter,
- a configurable upper bound on deferral count,
- a `dropped_transactions` metric when a transaction is discarded after excessive deferral.

Under sustained write contention on a hot account, a lower-priority transaction may be deferred across an unbounded number of scheduling passes. The transaction is **not lost** — it remains in the queue — but its scheduling latency is **unbounded and unobservable**.

This repository implements an alternative scheduling semantics with explicit bounds, then verifies those bounds through continuous fuzzing.

> **For the full technical discussion, see the RFC:** [anza-xyz/agave#14274](https://github.com/anza-xyz/agave/issues/14274)

---

## Architecture

### Hotspot Heat Map

Every writable account is tracked with a heat score:
- Heat increases by `initial_heat` on each scheduled write.
- Heat decays by right-shifting `hotspot_decay_shift` bits per generation of age.
- Accounts exceeding `max_generation_age` generations are evicted.

### Generation Aging

Each call to `schedule()` advances `current_generation` by 1.
Deferred transactions carry a `ready_generation` timestamp.
A transaction is only retried when `ready_generation ≤ current_generation`.

### Retry Cycle

```
Transaction arrives
        |
        v
  Priority queue
        |
        v
  Pop highest priority
        |
        v
  try_lock_accounts()
        |
    +---|---+
    |       |
  Success  Conflict
    |       |
    v       v
 Schedule  Increment retry counter
            |
            +--- retries > max_retry_count? ---+
            |                                     |
           No                                   Yes
            |                                     |
            v                                     v
    Push to deferred queue              Drop with metric increment
    (ready_generation = next gen)       (dropped_transactions += 1)
```

---

## Design Goals

**Deterministic scheduling** — given identical inputs and config, the scheduler produces identical outputs. No randomness in the scheduling path.

**Bounded retries** — every deferred transaction is either scheduled or dropped within a finite number of passes. Permanent starvation is impossible by construction.

**Conflict-aware execution** — writable account contention is tracked per-account via a heat score, not per-transaction-pair. This scales to large account sets without combinatorial overhead.

**Starvation observability** — `max_retry_count` enforces a hard upper bound on retries. A transaction that cannot be scheduled within that bound is dropped with an explicit metric increment (`dropped_transactions`), never silently deferred.

**Predictable memory behaviour** — the deferred queue is bounded by `max_retry_count`. The hotspot map is bounded by `hotspot_capacity`. No unbounded growth paths exist.

---

## Relationship to Agave GreedyScheduler

This implementation is **not a drop-in replacement** for Agave's `GreedyScheduler`. It is a research model that explores alternative semantics.

| Property | Agave GreedyScheduler (current) | This implementation (research) |
|:---------|:----------------------------------|:-------------------------------|
| Deferred queue drain | Re-inserts all unschedulables after each pass | Explicit per-pass drain with retry counter |
| Conflict detection | Unconditional (current master) | Unconditional |
| Retry bound | None — unbounded deferral | Hard cap via `max_retry_count` |
| Drop metric | None in `SchedulingSummary` | `dropped_transactions` counter |
| Heat-based throttling | Thread-level locking | Per-account heat score with decay |
| Fuzz verification | Not present in this component | libFuzzer, 5h 55m daily |

> **Note on conflict detection:** We initially investigated whether cost filtering could interact with conflict detection in earlier Agave versions. Current master (`anza-xyz/agave`, August 2026) calls `try_lock_accounts` before any cost filtering. Conflict detection is unconditional in both implementations.

---

## Invariants

All invariants are asserted after every scheduling pass during fuzzing.

### I1: Accounting invariant

**Statement:** `scheduled + deferred + dropped ≤ scanned` per pass.

**Reason:** Every transaction that enters `schedule()` must be accounted for. Silent loss is unacceptable.

**Failure mode:** A transaction disappears from all counters. Budget or queue logic has a branch that returns without incrementing any counter.

**Verified by:** `fuzz_target` INVARIANT 1 assertion, unit test `defers_conflicting_hot_accounts`.

### I2: Generation monotonicity

**Statement:** `summary.generation > 0` after every pass.

**Reason:** Generation counter must be strictly positive. A zero generation would make all deferred transactions with `ready_generation = 1` permanently unretriable.

**Failure mode:** Wrapping underflow on generation counter.

**Verified by:** `fuzz_target` INVARIANT 2 assertion.

### I3: Pass counter monotonicity

**Statement:** `scheduler_passes ≥ 1` after every call to `schedule()`.

**Reason:** Metrics must accumulate correctly. A pass counter that does not increment would corrupt all derived metrics.

**Failure mode:** Early return before `metrics.scheduler_passes += 1`.

**Verified by:** `fuzz_target` INVARIANT 3 assertion.

### I4: Deferred queue drain

**Statement:** The deferred queue reaches zero within bounded passes after all input stops.

**Reason:** Permanent queue growth means permanent starvation. Every deferred transaction must eventually be scheduled or dropped.

**Failure mode:** A transaction's `ready_generation` is set beyond any reachable generation value, or `max_retry_count` check is missing.

**Verified by:** `fuzz_target` INVARIANT 4 assertion (8192 drain passes), unit test `deferred_transaction_is_dropped_after_max_retries`.

---

## Development History

During the construction of this reference implementation, we encountered and resolved several design errors in our own code. These are documented below as **lessons from building a scheduler from scratch**, not as claims about Agave.

### Lesson 1: Dead deferred queue

**Description:** Early versions pushed deferred transactions into `self.deferred` but never read them back. The queue was write-only.

**Impact:** Every deferred transaction was silently lost in our own implementation.

**Fix:** Every `schedule()` pass now partitions `self.deferred` into ready and still-waiting entries before processing new transactions.

**Regression test:** `deferred_transactions_are_retried_and_eventually_scheduled`

### Lesson 2: Cost-filtering interaction

**Description:** Early versions gated conflict deferral on `tx.cost > 0`. This was a design error in our model.

**Impact:** Zero-cost transactions in our simulation could bypass conflict detection.

**Fix:** Conflict check is now unconditional. Cost is only relevant for budget enforcement, not conflict detection.

**Regression test:** `zero_cost_tx_does_not_bypass_conflict_detection`

> **Important:** These lessons informed our understanding of scheduler design. They do not constitute claims about bugs in Agave's current implementation. See [anza-xyz/agave#14274](https://github.com/anza-xyz/agave/issues/14274) for our verified observations about Agave's scheduler.

---

## Testing Matrix

| Test | Purpose | Invariant / Bug prevented |
|------|---------|--------------------------|
| `deferred_transactions_are_retried_and_eventually_scheduled` | Dead deferred queue regression | Lesson 1 |
| `zero_cost_tx_does_not_bypass_conflict_detection` | Cost-filtering regression | Lesson 2 |
| `deferred_transaction_is_dropped_after_max_retries` | Retry cap enforcement | I4 violation |
| `hotspot_decay_reduces_heat_gradually` | Heat decay correctness | Starvation |
| `budget_exhaustion_defers_without_conflict` | Budget vs conflict separation | Misclassification |
| `retried_transaction_succeeds_when_conflict_clears` | Full retry cycle | I4 violation |
| `read_only_accounts_do_not_cause_conflicts` | Read isolation | False conflicts |
| `generation_counter_wraps_safely` | Overflow safety | I2 violation |
| `metrics_accumulate_across_scheduling_passes` | Metrics correctness | I3 violation |
| `multiple_independent_accounts_schedule_separately` | No false conflicts | Throughput loss |
| `budget_backoff_retries_next_generation` | Budget retry path | Silent drop |
| `max_retry_count_drops_transaction` | Starvation prevention | I4 violation |
| `zero_cost_tx_in_high_conflict_defers` | Zero-cost + conflict | Lesson 2 variant |
| `defers_conflicting_hot_accounts` | Basic conflict detection | I1 violation |
| `cleanup_removes_stale_hotspots` | Memory bounds | Unbounded growth |
| Fuzz target (5h 55m) | All 4 invariants, random configs | All of the above |

---

## Fuzzing

### Harness design

The fuzz target generates:
- Random `SchedulerConfig` (bounded ranges to ensure termination)
- Random sequences of scheduling passes
- Random transactions with random account accesses

After every pass, all 4 invariants are asserted.
After all passes complete, 8192 drain passes verify I4.

### Configuration bounds in fuzzer

```rust
// Ensures termination and bounded state space
conflict_threshold: 1..=10,
max_generation_age: 8..=32,
hotspot_decay_shift: 0..=4,
max_retry_count: 3..=20,
hotspot_capacity: 256..=8192,
initial_heat: 1..=10,
max_account_heat: 64..=255,
```

### Why long-duration fuzzing increases confidence but does not prove correctness

Fuzzing explores the input space stochastically. A 5h 55m run at ~4,300 exec/s on GitHub Actions produces approximately 91 million executions. This increases confidence that no invariant violation exists for inputs of this size and shape. It does not constitute a formal proof. Formal verification would require model checking or theorem proving, which is outside the scope of this research.

### Coverage stabilisation

Coverage typically stabilises within the first 100,000 executions (`cov: ~420 ft: ~2600`). Subsequent runs refine the corpus but rarely discover new coverage. This indicates the harness has exhausted the reachable state space for inputs within the size limits.

---

## Performance

### CI runner (GitHub Actions, ubuntu-latest)

| Metric | Value |
|--------|-------|
| Executions per second | ~4,300 |
| RSS at stabilisation | ~612 MB |
| Coverage (ft) | ~2,661 |
| Corpus entries | ~436 |

These figures describe the fuzz harness on an isolated in-memory struct. They are not transaction throughput figures and must not be compared to validator TPS benchmarks.

### Native machine (Apple Silicon M4)

| Metric | Value |
|--------|-------|
| Executions per second | ~8,500 (60s smoke run) |

### Future benchmark placeholders

- [ ] scheduler throughput (tx/sec) — pending Criterion integration
- [ ] hotspot-heavy workload latency
- [ ] 90% conflicting transaction throughput
- [ ] 90% independent transaction throughput
- [ ] retry-heavy workload allocations
- [ ] memory usage under sustained load

---

## Configuration Reference

```rust
SchedulerConfig {
    // Minimum heat score to defer a transaction (default: 1)
    conflict_threshold: u32,

    // Generations before a hotspot account is evicted (default: 16)
    max_generation_age: u32,

    // Heat right-shift per generation of age; 0 = no decay (default: 1)
    hotspot_decay_shift: u32,

    // Maximum retries before a transaction is dropped (default: 6)
    max_retry_count: u8,

    // Initial HashMap capacity for hotspot tracking (default: 4096)
    hotspot_capacity: usize,

    // Heat added on first write to an account (default: 2)
    initial_heat: u16,

    // Heat ceiling per account (default: 255)
    max_account_heat: u16,
}
```

---

## Roadmap

**Phase 1 — Standalone scheduler (current)**
Conflict detection, deferred queue drain, retry cap, heat decay.
All invariants verified. Lessons learned and regression-tested.

**Phase 2 — Integration into Agave**
Integrate with `anza-xyz/agave` transaction-context crate.
Replace mock types with real `TransactionContext` and `AccountId`.

**Phase 3 — Real validator benchmarks**
Measure against existing Agave scheduler on mainnet-representative workloads.
Produce before/after numbers with Criterion.

**Phase 4 — RFC / PR**
Open a focused Draft PR against `anza-xyz/agave` with:
fuzz corpus, benchmark results, and invariant documentation as evidence.

> **Update:** Phase 4 RFC is already published: [anza-xyz/agave#14274](https://github.com/anza-xyz/agave/issues/14274)

---

## Repository Layout

```
.
├── Cargo.toml              # Workspace manifest
├── src/
│   ├── lib.rs              # Core scheduler engine
│   ├── config.rs           # SchedulerConfig
│   ├── heat_map.rs         # Hotspot tracking
│   ├── deferred_queue.rs   # Retry and generation logic
│   └── metrics.rs          # SchedulingSummary
├── tests/
│   └── integration_tests.rs # 15 regression tests
├── fuzz/
│   └── fuzz_targets/
│       └── scheduler_fuzz.rs # libFuzzer harness
├── .github/
│   └── workflows/
│       ├── ci.yml          # Unit test + clippy
│       └── fuzz-daily.yml  # 5h 55m fuzz run
└── README.md               # This file
```

---

## Quick Start

```bash
# Clone
git clone https://github.com/RFT-SIRM/agave-rift-scheduler.git
cd agave-rift-scheduler

# Build
cargo build --release

# Run all tests
cargo test --lib

# 60-second local fuzz
cargo +nightly fuzz run scheduler_fuzz -- -max_total_time=60

# Full 5h 55m run
cargo +nightly fuzz run scheduler_fuzz -- -max_total_time=21300
```

---

## Ecosystem

This repository is part of the [RFT-SIRM](https://github.com/RFT-SIRM/UltraCore-RFT) research ecosystem:

| Repository | Role | Status |
|:-----------|:-----|:-------|
| [UltraCore-RFT](https://github.com/RFT-SIRM/UltraCore-RFT) | Central research hub and documentation | Active |
| [Rift-L1-Blockchain](https://github.com/RFT-SIRM/Rift-L1-Blockchain) | Standalone validator core with SIRM invariants | Core complete |
| [Rift-Network](https://github.com/RFT-SIRM/Rift-Network) | Solana on-chain protocol (Anchor) | RC v1.0 |
| [agave-abiv2-memory-contexts](https://github.com/RFT-SIRM/agave-abiv2-memory-contexts) | SVM memory isolation research | Active |
| **agave-rift-scheduler** | Transaction scheduling research | **RFC published** |

---

## Disclaimer

- **Not a production patch.** This is a research implementation. It is not intended for deployment on mainnet validators without extensive additional testing.
- **Not a bug report on Agave.** The RFC ([anza-xyz/agave#14274](https://github.com/anza-xyz/agave/issues/14274)) documents an observed scheduling limitation and proposes a minimal observability improvement. It does not claim that GreedyScheduler is broken or unsafe.
- **Not a consensus issue.** This research concerns transaction scheduling latency and observability, not consensus safety or transaction integrity.
- **Lessons learned are from our own code.** The "Development History" section documents errors we made while building this reference implementation. They are not claims about bugs in Agave.

---

## License

Apache-2.0 — see [LICENSE](LICENSE)

© 2026 Eugeny (RFT-SIRM)

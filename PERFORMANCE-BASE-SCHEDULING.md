# Performance-based multipath scheduling

This document describes how Glorytun schedules traffic across paths today, outlines **Plan B** in detail (separate scheduling weights from rate caps and window pacing), and summarizes **Options A–C** so future work can start from a known baseline.

---

## 1. Current behaviour (baseline)

### 1.1 Path selection

Outbound packets are assigned to a path in `mud_select_path()` (`mud/mud.c`). A 16-bit value derived from the encrypted payload selects a path by walking a **weighted** list: each OK path contributes a bucket of size `path->tx.rate`. Only paths with `path->ok == 1` participate.

### 1.2 Overloaded meaning of `path->tx.rate`

The same field is used for several concerns:

| Concern | Role of `path->tx.rate` |
|--------|-------------------------|
| Scheduler | Weight in `mud_select_path` |
| Send window | `mud_update()` sums `path->tx.rate` over OK paths; that sum drives `mud->window` refill |
| Auto mode | `mud_update_window()` updates `path->tx.rate` from control/data feedback when `!fixed_rate` |
| Peer interaction | Control messages (`tx_time == 0` in `mud_recv_msg`) can overwrite `path->tx.rate` from the peer’s advertised `max_rate` |
| Observability | Exposed in `glorytun path` status |

Because of this coupling, “reweight by measured throughput” without a separate field tends to fight **pacing**, **auto-rate logic**, and **peer signalling**.

### 1.3 Caps and signalling

- `path->conf.tx_max_rate` / `path->conf.rx_max_rate` hold configured ceilings and feed what is advertised to the peer (`msg->max_rate` uses `rx_max_rate` on send).
- `fixed_rate` in path config gates whether `mud_update_window` mutates `path->tx.rate`.

---

## 2. Plan B — Detailed design (recommended direction)

**Goal:** Drive **relative** traffic share from **measured performance** (e.g. decaying average goodput, optionally loss/RTT), while keeping **absolute** limits and peer semantics on explicit cap fields and a well-defined aggregate pacing rule.

### 2.1 Principles

1. **Scheduling weight** is for *proportions only* (which path gets the next packet among OK paths).
2. **Aggregate send budget** (window refill) must not silently shrink because weights were lowered by congestion; it should follow an explicit **tunnel or per-path cap policy**.
3. **Peer-visible limits** remain `tx_max_rate` / `rx_max_rate` (or successors); local weights are not confused with advertised maxima unless deliberately mapped.
4. **Exploration:** paths with little recent traffic keep a **minimum weight** so they are not starved and can be measured again.
5. **Bidirectional awareness:** the peer runs its own scheduler for return traffic; the same algorithm (or compatible parameters) should be used on both ends if both are multipath-capable.

### 2.2 Data model changes

**On `struct mud_path` (`mud/mud.h`):**

| Field | Purpose |
|-------|---------|
| `sched_weight` | `uint64_t` — used only by `mud_select_path` (and status if desired). Relative weights; implementation may normalize internally. |
| `sched_weight_ema` (optional) | Internal smoothed estimate if you separate “raw EMA” from “weight after floor/cap” |
| Keep `conf.tx_max_rate` / `conf.rx_max_rate` | Unchanged semantics: ceilings and advertisement |
| `path->tx.rate` | Either: (a) retain for auto-mode estimate + display only, or (b) gradually align naming in docs to “estimated goodput” vs `sched_weight` |

**On `struct mud` (if needed):**

- Optional `uint64_t sched_total_cap` or reuse `sum(tx_max_rate)` for window math — see §2.4.

**Control / CLI (`src/ctl.h`, `src/path.c`, `COMMAND-LINE.md`):**

- Optional: read-only or tunable `sched_weight` in `glorytun path` output.
- Optional: new path sub-options, e.g. `weight min …`, `weight mode perf|fixed`, or global `set` knob — only if product needs runtime tuning without recompile.

### 2.3 Algorithm: updating `sched_weight`

**Signal sources (prefer data plane):**

- Byte deltas on the path: `path->tx.bytes`, `path->tx.total`, and/or the forward (`fw`) / receive counters already consumed in `mud_update_window()` when `path->msg.set && path->mtu.ok`.
- Auxiliary signals: `path->tx.loss`, `path->rtt` (already maintained) — can scale weight down for high loss or RTT variance if desired.

**Suggested core loop (conceptual):**

1. On each valid feedback interval (same gates as today for `mud_update_window`, or a slightly relaxed cadence):
   - Compute instantaneous goodput estimate `g` (bytes/sec) for that path over the interval.
   - Update EMA: `ema = alpha * g + (1 - alpha) * ema` with fixed-point or integer-friendly shifts (e.g. `alpha = 1/8`).
2. Map EMA to a **provisional weight** `w = f(ema, loss, rtt)` — start with `w = max(ema, w_min)` where `w_min` is a small floor for exploration.
3. Optional: **cap** per-path weight at `tx_max_rate` or a fraction of it so weights stay comparable to “capacity-ish” numbers.
4. Normalize for numerics if needed: either keep raw uint64 weights (selection only cares about ratios) or scale so the largest OK path is `UINT32_MAX` for the random walk — document the choice to avoid overflow in `k * mud->rate >> 16`.

**Cold start / idle paths:**

- If no feedback since `T` ms, decay `sched_weight` toward `w_min` rather than to zero, or refresh from configured `tx_max_rate` * small fraction so the path is occasionally tried.

**Interaction with `fixed_rate`:**

- **Fixed mode (legacy):** either `sched_weight` tracks configured static share (e.g. proportional to `tx_max_rate`) and is not EMA-updated, or EMA is disabled when `fixed_rate` indicates “operator fixed split”.
- **New “performance mode”:** new flag or `rate` sub-mode: caps from `tx`/`rx`, weights from EMA.

### 2.4 Send window (`mud->window`) and aggregate rate

Today: `mud_update()` sets `rate = sum(path->tx.rate)` over OK paths and refills `mud->window` with `rate * elapsed`.

**Plan B change:**

- Compute `rate_for_window` independently, for example:
  - `rate_for_window = sum(path->conf.tx_max_rate)` over OK paths that are not in backup-only exclusion, **or**
  - `min(sum(tx_max_rate), measured_aggregate_ema)` if you add tunnel-level measurement later, **or**
  - a single `mud->tunnel_cap` from config.

**Invariant to preserve:** the window should allow the tunnel to use the **intended aggregate throughput** when the network allows it; weights only redistribute among paths.

**Edge case:** if all `tx_max_rate` are very large (“unlimited” style), keep a sane default cap for window growth or reuse existing auto-discovery totals — align with current `auto` behaviour where applicable.

### 2.5 `mud_select_path` changes

- Replace reads of `path->tx.rate` with `path->sched_weight` (or a helper `mud_path_sched_weight(path)` that applies floor and backup rules).
- Recompute `mud->rate` (or rename to `mud->sched_weight_sum`) as **sum of sched_weight** over OK paths used for selection — **not** the same variable as `rate_for_window` unless explicitly documented as intentional.

Ensure the random variable `k = (cursor * mud->rate) >> 16` still spans `[0, sum(weights))` without overflow; if sums grow huge, use 64-bit widening or pre-normalized weights.

### 2.6 Peer control messages (`mud_recv_msg`, `mud_send_msg`)

- When `tx_time == 0`, avoid blindly assigning `path->tx.rate = max_rate` in a way that destroys `sched_weight`. Either:
  - stop overwriting local schedule state from peer `max_rate`, and only update `tx_max_rate` / `fixed_rate` / `loss_limit`, **or**
  - if peer `max_rate` is meant to cap **our** sending to them, apply it to `conf.tx_max_rate` only, not to `sched_weight`.

Audit every assignment to `path->tx.rate` after this split.

### 2.7 `mud_set_state` / `glorytun path`

- Initial registration: set `sched_weight` from operator `tx` argument (same numeric value as today for backward compatibility) or from `tx_max_rate` until first EMA sample.
- When operator changes `rate fixed|auto tx … rx …`, update caps; optionally reset or blend `sched_weight` from `tx` for predictable behaviour.

### 2.8 Status and documentation

- `gt_path_print_status()` (`src/path.c`): print `sched_weight` (and optionally keep showing `tx.rate` as “estimate” if retained).
- Update `COMMAND-LINE.md`, `MULTIPATH-EXAMPLE.md`, and client scripts README if new knobs appear.

### 2.9 Testing strategy

1. **Unit-style / deterministic:** mock paths with fixed weights and caps; assert selection distribution approximates weight ratios over many samples.
2. **Simulation:** four paths with different simulated goodput; verify weights converge and minimum path still receives traffic.
3. **Regression:** existing single-path and equal-weight multipath scenarios match prior throughput within tolerance.
4. **Bidirectional:** iperf with asymmetric caps; confirm return path does not collapse because peer weights are zero.

### 2.10 Risks and mitigations

| Risk | Mitigation |
|------|------------|
| Oscillating weights | Stronger EMA, caps on change per beat, optional loss penalty |
| Window too large vs reality | Tie `rate_for_window` to sum of caps or explicit tunnel cap |
| Starvation | `w_min` exploration floor |
| ABI / ctl struct size | Version ctl messages or extend carefully; pad reserved fields if needed |
| Confusion between weight and Mbit/s | Clear naming and CLI labels |

### 2.11 Implementation order (suggested)

1. Add `sched_weight` + initialization in `mud_set_state` and path creation; make `mud_select_path` use it while **window still uses sum(tx.rate)** — behaviour identical if `sched_weight := tx.rate`.
2. Switch window to `rate_for_window` from `sum(tx_max_rate)` (or chosen rule); validate benchmarks.
3. Add EMA updater alongside `mud_update_window`; gate behind new mode or extend `auto`.
4. Detangle `mud_recv_msg` overwrites from scheduling state.
5. Expose in CLI/docs; add tests.

---

## 3. Summary of options (for future pivots)

### Option A — Evolve current “auto” mode in place

**Idea:** Keep a single `path->tx.rate` but improve `mud_update_window()` to use a **decaying average** of measured goodput (and optionally loss/RTT), still clamped by `tx_max_rate`.

**Pros:** Smaller diff; fewer new fields; builds on existing feedback path.

**Cons:** Same field still mixes scheduler weight, window sum, auto estimate, and peer overwrites — **must** carefully fix `mud_recv_msg` and window math so they don’t erase or mis-scale the EMA.

**Start from:** `mud/mud.c` — `mud_update_window`, `mud_update`, `mud_recv_msg`; clarify window refill vs sum of weights.

---

### Option B — Separate scheduling weight from caps and pacing (this document’s main plan)

**Idea:** Introduce `sched_weight` (or equivalent) for `mud_select_path` only; keep `tx_max_rate` / `rx_max_rate` for caps and signalling; drive `mud->window` from an explicit aggregate rule (e.g. sum of caps).

**Pros:** Clean separation; performance-based mixing without breaking pacing or peer semantics; clearest long-term model.

**Cons:** More structural change (struct, ctl, status, all `tx.rate` touch points).

**Start from:** §2 of this file; `mud/mud.h`, `mud/mud.c`, then ctl/path display; follow §2.11.

---

### Option C — Heartbeat-only rate adjustments

**Idea:** On each control **beat**, recompute a decaying average and push updates equivalent to `mud_set_state` (or mutate rates) from heartbeat timing alone.

**Pros:** Simple mental model; slow, coarse control.

**Cons:** Heartbeats are **sparse** vs data; noisy unless aggregated over many intervals; easy to conflict with peer control messages and auto mode; **inferior signal** to data-plane feedback already available in `mud_update_window`.

**Start from:** `mud_send_msg` / `mud_recv_msg` beat cadence and path timing; prefer **not** as primary design unless operational constraints require rare updates only.

---

## 4. File index (likely touch points for Plan B)

| Area | Files |
|------|--------|
| Core path state & logic | `mud/mud.h`, `mud/mud.c` |
| CLI / control | `src/ctl.h`, `src/path.c`, `src/bind.c` |
| User docs | `COMMAND-LINE.md`, `MULTIPATH-EXAMPLE.md` |
| Optional scripts | `scripts/client/*` if new defaults or modes |

---

*Document version: initial. Align implementation details with the tree at the time of coding; line references in §1 may drift as `mud.c` evolves.*

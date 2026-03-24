# Performance-based multipath scheduling

This document describes how Glorytun schedules traffic across paths today, outlines **Plan B** in detail (separate scheduling weights from rate caps and window pacing), and summarizes **Options A–C** so future work can start from a known baseline.

---

## 0. Repository layout and submodule scope (read this first)

Glorytun vendors two Git submodules (see `.gitmodules`):

| Submodule | Upstream | Role |
|-----------|----------|------|
| `mud/` | [angt/mud](https://github.com/angt/mud) | Encrypted UDP transport, path state, `mud_select_path`, window pacing, feedback |
| `argz/` | [angt/argz](https://github.com/angt/argz) | Declarative CLI parsing (`argz()` tables) |

**Policy for this tree:** scheduling and path logic **live in `mud`**. Options A–C and Plan B all require **core changes to `mud`** (`mud/mud.h`, `mud/mud.c`). Those changes belong **upstream** (or in a dedicated fork of `angt/mud`), with this repo **bumping the submodule pointer** — not maintaining a long‑lived, glorytun‑only fork of `mud/` inside this workspace without upstream alignment.

**`argz/`** should only change if the **parser library** itself needs new primitives. New `glorytun path` keywords are normally added in **`src/path.c`** using existing `argz` APIs; that is **not** a submodule edit.

**Glorytun-only touch points after mud exposes new APIs:** `src/bind.c` (control socket handling), `src/path.c` (CLI / status tables), `src/ctl.h` (if `ctl_msg` layout must grow — keep ABI in mind), plus docs and scripts.

---

## 1. Current behaviour (baseline)

*Verified against the scheduling loop in `mud/mud.c`: `mud_select_path`, `mud_update`, `mud_update_window`, `mud_recv_msg`, `mud_send`.*

### 1.1 Path selection

Outbound packets are assigned to a path in `mud_select_path()` (`mud/mud.c`). A 16-bit value read from the tail of the encrypted payload (`mud_send`) acts as `cursor`; the path is chosen by walking a **weighted** list:

- `k = (cursor * mud->rate) >> 16`
- Only paths with `path->ok == 1` are considered.
- Each such path contributes a bucket of size `path->tx.rate`; `k` walks those buckets until it falls inside one path’s range.

`mud->rate` is set at the **end** of `mud_update()` to the **sum** of `path->tx.rate` over paths that passed `mud_path_is_ok()` for that tick (see §1.2).

### 1.2 Overloaded meaning of `path->tx.rate`

The same field is used for several concerns:

| Concern | Role of `path->tx.rate` |
|--------|-------------------------|
| Scheduler | Bucket size in `mud_select_path` (with `mud->rate` as total weight scale) |
| Send window | `mud_update()` sums `path->tx.rate` over OK paths; that sum refills `mud->window` |
| Auto mode | `mud_update_window()` updates `path->tx.rate` from feedback when `!fixed_rate`, then clamps to `tx_max_rate` |
| Peer interaction | In `mud_recv_msg`, when `tx_time == 0`, `path->tx.rate` can be set from the peer’s advertised `max_rate` |
| Observability | Shown as TX rate in `glorytun path` status (`src/path.c`) |

Because of this coupling, “reweight by measured throughput” without a separate field tends to fight **pacing**, **auto-rate logic**, and **peer signalling**.

### 1.3 Caps and signalling

- `path->conf.tx_max_rate` / `path->conf.rx_max_rate` hold configured ceilings; the wire format advertises receive-side limits to the peer (`mud_send_msg` / `mud_recv_msg`).
- `fixed_rate` gates whether `mud_update_window` mutates `path->tx.rate`.

### 1.4 CLI `pref` (Glorytun front-end)

[`COMMAND-LINE.md`](COMMAND-LINE.md) documents `pref N` on `glorytun path … set` as a path preference that **biases scheduling**. Wiring for `pref` is present in `src/path.c` (`CTL_PATH_CONF`). Whether and how `pref` maps into the mud core depends on the **mud** revision linked by the submodule; Plan B’s `sched_weight` may subsume or combine with `pref` — decide explicitly when implementing.

---

## 2. Plan B — Detailed design (recommended direction)

**Goal:** Drive **relative** traffic share from **measured performance** (e.g. decaying average goodput, optionally loss/RTT), while keeping **absolute** limits and peer semantics on explicit cap fields and a well-defined aggregate pacing rule.

**Where it is implemented:** Primarily **upstream `mud`**; Glorytun updates **submodule + `src/`** as in §0.

### 2.1 Principles

1. **Scheduling weight** is for *proportions only* (which path gets the next packet among OK paths).
2. **Aggregate send budget** (window refill) must not silently shrink because weights were lowered by congestion; it should follow an explicit **tunnel or per-path cap policy**.
3. **Peer-visible limits** remain `tx_max_rate` / `rx_max_rate` (or successors); local weights are not confused with advertised maxima unless deliberately mapped.
4. **Exploration:** paths with little recent traffic keep a **minimum weight** so they are not starved and can be measured again.
5. **Bidirectional awareness:** the peer runs its own scheduler for return traffic; the same algorithm (or compatible parameters) should be used on both ends if both are multipath-capable.

### 2.2 Data model changes

**In upstream `struct mud_path` (`mud/mud.h`):**

| Field | Purpose |
|-------|---------|
| `sched_weight` | `uint64_t` — used only by `mud_select_path` (and status if desired). Relative weights; implementation may normalize internally. |
| `sched_weight_ema` (optional) | Internal smoothed estimate if you separate “raw EMA” from “weight after floor/cap” |
| Keep `conf.tx_max_rate` / `conf.rx_max_rate` | Unchanged semantics: ceilings and advertisement |
| `path->tx.rate` | Either: (a) retain for auto-mode estimate + display only, or (b) gradually align naming in docs to “estimated goodput” vs `sched_weight` |

**In upstream `struct mud` (if needed):**

- Optional `uint64_t sched_total_cap` or reuse `sum(tx_max_rate)` for window math — see §2.4.

**In Glorytun (`src/ctl.h`, `src/path.c`, [`COMMAND-LINE.md`](COMMAND-LINE.md)):**

- Extend `ctl_msg` / path status only if new fields must cross the UNIX socket (coordinate with `struct mud_path` size and versioning).
- Current user-facing syntax is documented under **`glorytun path`** in [`COMMAND-LINE.md`](COMMAND-LINE.md): `dev`, `addr`, `to`, `show`, `set` with `up`/`down`, `rate fixed|auto`, `tx`/`rx`, `beat`, `pref`, `losslimit`.
- Optional future work: expose `sched_weight` in `show stat` (or new `show`), or add keywords — preferably **without** editing the `argz` submodule unless a new parser feature is required.

### 2.3 Algorithm: updating `sched_weight`

**Signal sources (prefer data plane):**

- Byte deltas on the path: counters already consumed in `mud_update_window()` when feedback is valid (`path->msg.set` and MTU state allow).
- Auxiliary signals: `path->tx.loss`, `path->rtt` — can scale weight down for high loss or RTT variance if desired.

**Suggested core loop (conceptual):**

1. On each valid feedback interval (same gates as today for `mud_update_window`, or a slightly relaxed cadence):
   - Compute instantaneous goodput estimate `g` (bytes/sec) for that path over the interval.
   - Update EMA: `ema = alpha * g + (1 - alpha) * ema` with fixed-point or integer-friendly shifts (e.g. `alpha = 1/8`).
2. Map EMA to a **provisional weight** `w = f(ema, loss, rtt)` — start with `w = max(ema, w_min)` where `w_min` is a small floor for exploration.
3. Optional: **cap** per-path weight at `tx_max_rate` or a fraction of it so weights stay comparable to “capacity-ish” numbers.
4. Normalize for numerics if needed: selection only cares about ratios; avoid overflow in `k * mud->rate >> 16` (document scaling or use wider intermediates).

**Cold start / idle paths:**

- If no feedback since `T` ms, decay `sched_weight` toward `w_min` rather than to zero, or refresh from configured `tx_max_rate` * small fraction so the path is occasionally tried.

**Interaction with `fixed_rate` and `pref`:**

- **Fixed mode:** either `sched_weight` tracks configured static share (e.g. proportional to `tx_max_rate`) and is not EMA-updated, or EMA is disabled when the operator requests a fixed split.
- **`pref`:** define whether it adds a bias on top of `sched_weight`, or is folded into initial `sched_weight` — avoid double-counting.

### 2.4 Send window (`mud->window`) and aggregate rate

Today: `mud_update()` accumulates `rate += path->tx.rate` for OK paths, assigns `mud->rate = rate`, and refills `mud->window` with `rate * elapsed` when `mud->window < 1500`.

**Plan B change:**

- Compute `rate_for_window` independently, for example:
  - `rate_for_window = sum(path->conf.tx_max_rate)` over OK paths (excluding backup-only as today), **or**
  - `min(sum(tx_max_rate), measured_aggregate_ema)` if tunnel-level measurement is added later, **or**
  - a single `mud->tunnel_cap` from config.

**Invariant:** the window should allow the tunnel to use the **intended aggregate throughput** when the network allows it; weights only redistribute among paths.

**Edge case:** very large “unlimited” caps — keep a sane default for window growth or align with existing auto behaviour.

### 2.5 `mud_select_path` changes (upstream)

- Use `path->sched_weight` (or `mud_path_sched_weight(path)`) instead of `path->tx.rate` for bucket sizes.
- Recompute the total used in `k = (cursor * total) >> 16` as the **sum of scheduling weights** for OK paths — keep this distinct from `rate_for_window` unless intentionally unified and documented.

### 2.6 Peer control messages (`mud_recv_msg`, `mud_send_msg`)

- When handling control packets (`tx_time == 0`), avoid assigning `path->tx.rate = max_rate` in a way that destroys `sched_weight`. Prefer updating `conf.tx_max_rate` / peer sync fields only, and keep scheduling state in `sched_weight` (or explicitly resync from caps when appropriate).

Audit every assignment to `path->tx.rate` after this split.

### 2.7 `mud_set_state` / `mud_set_path` and `glorytun path`

- Initial registration: initialize `sched_weight` from operator `tx` (compatibility) or from `tx_max_rate` until the first EMA sample.
- When the operator changes `rate fixed|auto tx … rx …`, update caps; optionally reset or blend `sched_weight` for predictable behaviour.

Exact entry symbol names (`mud_set_state` vs `mud_set_path`) depend on the **mud** revision; align with the linked submodule API.

### 2.8 Status and documentation

- Path status printing: `src/path.c` (`gt_path_status` / related helpers) — add a column or line for `sched_weight` if exposed.
- Update [`COMMAND-LINE.md`](COMMAND-LINE.md), [`MULTIPATH-EXAMPLE.md`](MULTIPATH-EXAMPLE.md), and client `scripts/**/README.md` when new knobs land.

### 2.9 Testing strategy

1. **Unit-style / deterministic:** fixed weights and caps; assert selection distribution approximates weight ratios over many samples (upstream mud tests or harness).
2. **Simulation:** four paths with different simulated goodput; verify weights converge and minimum path still receives traffic.
3. **Regression:** single-path and equal-weight multipath throughput within tolerance.
4. **Bidirectional:** iperf with asymmetric caps; return path does not collapse because peer weights are zero.

### 2.10 Risks and mitigations

| Risk | Mitigation |
|------|------------|
| Oscillating weights | Stronger EMA, caps on change per beat, optional loss penalty |
| Window too large vs reality | Tie `rate_for_window` to sum of caps or explicit tunnel cap |
| Starvation | `w_min` exploration floor |
| ABI / ctl struct size | Version ctl messages or extend carefully; pad reserved fields if needed |
| Confusion between weight and Mbit/s | Clear naming and CLI labels |
| Submodule drift | Implement in upstream mud; bump submodule; run integration tests in Glorytun |

### 2.11 Implementation order (suggested)

1. **Upstream mud:** add `sched_weight` + initialization; make `mud_select_path` use it while the window still uses `sum(tx.rate)` — behaviour matches today if `sched_weight := tx.rate`.
2. **Upstream mud:** switch window to `rate_for_window` from `sum(tx_max_rate)` (or chosen rule); benchmark.
3. **Upstream mud:** add EMA updater alongside `mud_update_window`; gate behind a mode or extend `auto`.
4. **Upstream mud:** detangle `mud_recv_msg` overwrites from scheduling state.
5. **Glorytun:** bump `mud` submodule; adjust `src/path.c` / `src/ctl.h` / `src/bind.c` if status or `ctl_msg` changes; update docs.

A **step-by-step todo list with files per task** is maintained in [PLAN-B-TODO.md](PLAN-B-TODO.md).

---

## 3. Summary of options (for future pivots)

### Option A — Evolve current “auto” mode in place

**Idea:** Keep a single `path->tx.rate` but improve `mud_update_window()` to use a **decaying average** of measured goodput (and optionally loss/RTT), still clamped by `tx_max_rate`.

**Pros:** Smaller diff inside mud; fewer new fields; builds on existing feedback.

**Cons:** Same field still mixes scheduler weight, window sum, auto estimate, and peer overwrites — **must** carefully fix `mud_recv_msg` and window math so they don’t erase or mis-scale the EMA.

**Start from:** `mud/mud.c` — `mud_update_window`, `mud_update`, `mud_recv_msg`; clarify window refill vs sum of weights. **Land changes upstream in `mud`**, then bump the submodule here.

---

### Option B — Separate scheduling weight from caps and pacing (this document’s main plan)

**Idea:** Introduce `sched_weight` (or equivalent) for `mud_select_path` only; keep `tx_max_rate` / `rx_max_rate` for caps and signalling; drive `mud->window` from an explicit aggregate rule (e.g. sum of caps).

**Pros:** Clean separation; performance-based mixing without breaking pacing or peer semantics; clearest long-term model.

**Cons:** More structural change in **mud** (struct, all `tx.rate` touch points); Glorytun must update `src/` for any new status/ctl fields.

**Start from:** §2 of this file; **upstream** `mud/mud.h`, `mud/mud.c`; then Glorytun `src/` + docs; follow §2.11.

---

### Option C — Heartbeat-only rate adjustments

**Idea:** On each control **beat**, recompute a decaying average and push updates equivalent to path configuration (or mutate rates) from heartbeat timing alone.

**Pros:** Simple mental model; slow, coarse control.

**Cons:** Heartbeats are **sparse** vs data; noisy unless aggregated over many intervals; easy to conflict with peer control messages and auto mode; **inferior signal** to data-plane feedback already available in `mud_update_window`.

**Start from:** `mud_send_msg` / `mud_recv_msg` beat cadence and path timing in **upstream mud**. Prefer **not** as primary design unless operational constraints require rare updates only.

---

## 4. File index

| Area | Location | Notes |
|------|----------|--------|
| Core scheduling, window, feedback | **`mud/mud.h`**, **`mud/mud.c`** | Submodule; **upstream changes** |
| `argz` parser core | **`argz/`** | Submodule; avoid edits unless parser primitives are required |
| CLI, status tables | **`src/path.c`** | Glorytun; uses `argz` **API** from submodule |
| Control messages | **`src/ctl.h`**, **`src/bind.c`** | Glorytun; must match `mud_path` / `ctl_msg` layout |
| User docs | **`COMMAND-LINE.md`**, **`MULTIPATH-EXAMPLE.md`** | Glorytun |
| Init scripts | **`scripts/client/*`** | Optional defaults for caps/rates |

---

*Last updated: revalidated against post-merge tree; core scheduling references in §1 are `mud_select_path` / `mud_update` / `mud_update_window` / `mud_recv_msg` in `mud/mud.c`. Line numbers drift with submodule commits — re-read those functions when implementing.*

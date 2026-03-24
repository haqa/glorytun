# Plan B — implementation checklist

Step-by-step tasks derived from [PERFORMANCE-BASE-SCHEDULING.md](PERFORMANCE-BASE-SCHEDULING.md) §2 (Plan B) and §2.11. **Upstream work** targets the [`mud`](https://github.com/angt/mud) submodule; **Glorytun** changes are under `src/` and docs after the submodule exposes new fields/APIs.

---

## Phase 0 — ABI and control surface (optional but recommended before coding)

| Step | Task | Files (typical) |
|------|------|-----------------|
| 0.1 | Decide whether `sched_weight` appears in `ctl_msg` / `glorytun path show` and whether `struct mud_path` grows (padding vs versioned ctl). | `mud/mud.h`, `src/ctl.h`, design note in [PERFORMANCE-BASE-SCHEDULING.md](PERFORMANCE-BASE-SCHEDULING.md) or this file |

---

## Phase 1 — `sched_weight` + selection only (upstream `mud`)

**Goal:** `mud_select_path` uses `sched_weight`; keep **window refill** on `sum(path->tx.rate)` unchanged. If `sched_weight` is initialized equal to `tx.rate` everywhere, behaviour matches pre-change.

| Step | Task | Files |
|------|------|--------|
| 1.1 | Add `sched_weight` (and optional `sched_weight_ema` if you split raw vs smoothed) to `struct mud_path`. | `mud/mud.h` |
| 1.2 | Initialize `sched_weight` wherever a path is zeroed or created (`memset`, new path, reset). Set `sched_weight = path->tx.rate` (or `tx_max_rate`) at creation so weights match current semantics. | `mud/mud.c` (e.g. path allocation, `mud_reset_path`, `mud_get_path` / path init) |
| 1.3 | On path config apply (`mud_set_state` / `mud_set_path` or equivalent in your mud revision), set `sched_weight` from operator `tx` / caps for backward compatibility. | `mud/mud.c` |
| 1.4 | Replace bucket size in `mud_select_path`: use `sched_weight` instead of `path->tx.rate`. Recompute the total passed into `k = (cursor * total) >> 16` as **sum of `sched_weight`** over OK paths (consider renaming `mud->rate` to `mud->sched_weight_sum` in mud for clarity — same file). | `mud/mud.c` |
| 1.5 | Grep for every read/write of scheduling totals; ensure `mud_update` still uses **`tx.rate` only for window** until Phase 2. | `mud/mud.c` |

**Validation:** unit or scripted sampling of `mud_select_path` distribution; throughput regression vs main branch.

---

## Phase 2 — Decouple window pacing from schedule weights (upstream `mud`)

**Goal:** `mud->window` refills from `rate_for_window` (e.g. `sum(tx_max_rate)` over OK paths), not `sum(tx.rate)`.

| Step | Task | Files |
|------|------|--------|
| 2.1 | Introduce `rate_for_window` (local or `mud` field): implement chosen policy (`sum(conf.tx_max_rate)`, backup exclusion rules unchanged from today). | `mud/mud.c` (`mud_update`) |
| 2.2 | Replace `rate += path->tx.rate` used **only** for window refill with `rate_for_window` aggregation. Keep a separate sum for **`sched_weight`** if `mud->rate` is split into two concepts. | `mud/mud.c` |
| 2.3 | Handle edge cases: zero caps, “unlimited” large caps, no OK paths (`!rate` branch). | `mud/mud.c` |
| 2.4 | Benchmark single- and multi-path throughput; tune caps if window is too aggressive or too conservative. | `mud/mud.c`; optional `mud/test.c` |

---

## Phase 3 — EMA / performance-based `sched_weight` (upstream `mud`)

**Goal:** Update `sched_weight` from measured goodput (EWMA), with floors/ceilings; gate behind extended `auto` or a new flag if needed.

| Step | Task | Files |
|------|------|--------|
| 3.1 | Add EMA state (on path or separate field) and helpers: update from `mud_update_window` feedback intervals or equivalent. | `mud/mud.h`, `mud/mud.c` |
| 3.2 | Implement `w_min` exploration floor and optional caps vs `tx_max_rate`. | `mud/mud.c` |
| 3.3 | Define interaction with `fixed_rate`: when fixed, do not EMA-update `sched_weight` (or copy from static policy). | `mud/mud.c` (`mud_update_window`, config apply) |
| 3.4 | Define interaction with **`pref`** (if present in your mud revision): bias vs EMA — document and implement in one place. | `mud/mud.c`, [COMMAND-LINE.md](COMMAND-LINE.md) when behaviour is final |

---

## Phase 4 — Peer control messages (upstream `mud`)

**Goal:** Stop clobbering scheduling state when applying peer `max_rate` / control fields.

| Step | Task | Files |
|------|------|--------|
| 4.1 | Audit `mud_recv_msg` (`tx_time == 0`): update `conf.tx_max_rate` / peer sync without blindly setting `path->tx.rate = max_rate` if that destroys `sched_weight` or EMA. | `mud/mud.c` |
| 4.2 | Audit `mud_send_msg` for consistency of advertised limits vs local `sched_weight`. | `mud/mud.c` |
| 4.3 | Grep `path->tx.rate` in `mud/mud.c`; classify each site as **estimate**, **signalling**, or **obsolete** after split. | `mud/mud.c` |

---

## Phase 5 — Tests (upstream `mud`)

| Step | Task | Files |
|------|------|--------|
| 5.1 | Distribution / property tests for weighted selection (ratios vs weights). | `mud/test.c` (or new test harness) |
| 5.2 | Multi-path simulation or integration tests for convergence + minimum exploration. | `mud/test.c` |

---

## Phase 6 — Glorytun tree (submodule + `src/` + docs)

**Goal:** Consume new mud; expose status/CLI if new fields exist.

| Step | Task | Files |
|------|------|--------|
| 6.1 | Bump **`mud` submodule** to the commit that contains Phases 1–5. | `.gitmodules` (only if URL changes), `mud/` (gitlink), release notes / changelog if you maintain one |
| 6.2 | If `struct mud_path` or ctl layout changed: update **`ctl_msg`** and any `memcpy` of `path`. | `src/ctl.h`, `src/bind.c` |
| 6.3 | Path status / `glorytun path show`: add columns or rows for `sched_weight` (and document units). | `src/path.c` |
| 6.4 | New CLI keywords **only if** needed: extend `argz` table in `path` command (prefer **no** `argz/` submodule edit — use existing `argz()` features). | `src/path.c` |
| 6.5 | User documentation: path behaviour, multipath examples, tuning. | [COMMAND-LINE.md](COMMAND-LINE.md), [MULTIPATH-EXAMPLE.md](MULTIPATH-EXAMPLE.md) |
| 6.6 | Init script hints (optional): document caps vs auto weights for operators. | `scripts/client/openrc/README.md`, `scripts/client/systemd/README.md` |

---

## Phase 7 — `argz` submodule (only if unavoidable)

| Step | Task | Files |
|------|------|--------|
| 7.1 | New parser primitives (rare): land in [angt/argz](https://github.com/angt/argz), bump `argz/` submodule, then use from `src/path.c`. | `argz/` (upstream), `argz/` (submodule pointer), `src/path.c` |

---

## Quick reference — files touched by layer

| Layer | Files |
|-------|--------|
| Core | `mud/mud.h`, `mud/mud.c` |
| Optional tests | `mud/test.c` |
| Glorytun control / CLI | `src/ctl.h`, `src/bind.c`, `src/path.c` |
| Docs | `PERFORMANCE-BASE-SCHEDULING.md`, `COMMAND-LINE.md`, `MULTIPATH-EXAMPLE.md`, `scripts/client/**/README.md` |
| Submodules | `mud/` (required), `argz/` (only if Phase 7) |

---

*This checklist is a guide; exact symbol names (`mud_set_state` vs `mud_set_path`) and struct layouts depend on the `mud` revision you ship.*

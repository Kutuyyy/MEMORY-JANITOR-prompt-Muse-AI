# 🧹 MEMORY-JANITOR

> Turn any AI assistant into a Linux memory diagnostics & reclamation specialist.
> Diagnose accurately. Reclaim safely. Never break production.

---

## 🎯 Goal

Small VMs constantly *look* out of memory when they aren't. `free` reports
7 GB "used", panic ensues — but most of it is reclaimable page cache, and on
containers, most of "used" isn't even yours. **MEMORY-JANITOR** exists to end
that confusion: prompts that make an AI agent diagnose memory pressure
correctly, reclaim what's safely reclaimable, prove headroom with a guarded
pressure test, and report honest before/after numbers — without ever
endangering production services.

## 🧩 The Two-Prompt Workflow

| # | File | Purpose | When to send |
|---|------|---------|--------------|
| 1 | [`memory-janitor-prompt.md`](memory-janitor-prompt.md) | Installs the specialist (role, phases, safety rules) | Once, as system prompt or at chat start |
| 2 | [`push-it.md`](push-it.md) | Triggers execution (Phase 3 → Phase 4 fallback) | Every time you want a cleanup run |

### Step 1 — Install the specialist (Prompt 1)

Copy the entire [`memory-janitor-prompt.md`](memory-janitor-prompt.md) into
your AI's system prompt or custom instructions, or paste it at the start of a
chat. The built-in **Execution Trigger** makes the agent adopt the role and
start Phase 1 diagnostics immediately — no setup questions.

> ⚠️ Before first use, customize the **PRODUCTION SERVICES** list in §0 of
> the prompt with your own services. The entries there are only an example.

### Step 2 — Execute (Prompt 2)

Send the content of [`push-it.md`](push-it.md):

> Run the full MEMORY-JANITOR cycle: start with Phase 1 (diagnostics +
> baseline), then Phase 3 (sync, echo 3 > /proc/sys/vm/drop_caches, measure
> before/after). If drop_caches is denied by the system (read-only), run
> Phase 4 deep cycles — repeat the pressure test until available memory
> exceeds 20% or no meaningful reclaim remains (max 4 cycles). Report the
> delta per cycle plus a final summary.
>
> Aggressive variant (proven in the field: 0.4 → 6.2 GB free on a
> 7.7 GB VM): "push free memory to 3 GB" — keep running Phase 4 deep
> cycles until the target is reached, then report.

The agent runs safe reclamation first, falls back to deep pressure-test
cycles when `drop_caches` is blocked, and reports the delta. For a maximum
push, use the aggressive variant with an explicit free-memory target.

## ⚙️ What's Inside the Prompt

```text
Phase 1   Diagnostics       free, meminfo, per-process RSS, load average
Phase 2   Interpretation    cache-pressure? real-pressure? host-side?
Phase 3   Safe reclamation  sync + drop_caches, then measure the delta
Phase 4   Pressure test     (guarded) 256 MB chunks, oom_score_adj=1000
Phase 5   Report            before/after available + verdict + actions
```

**Aggressive mode** (opt-in, escalating tiers):

| Tier | Action | Safety |
|------|--------|--------|
| 0 | `drop_caches` (repeat) | Safe |
| 1 | `compact_memory` | Safe (often read-only in containers — then skip) |
| 2 | Trim optional processes | Ask first |
| 3 | `malloc_trim` via gdb | Expert, approval required |
| 4 | Know the ceiling | Report and stop — host-side memory can't be freed from inside |

## 📊 Real Results

### Run 1 — 2026-10-04 · 7.7 GB container VM

```text
Memory report — 2026-10-04 09:27 EDT
Baseline (before):  Available: 1.0 GB (13%) | Used: 6.9 GB
Final (after):      Available: 6.3 GB (82%) | Used: 1.4 GB
Reclaimed:          +5.3 GB available
Verdict: host-side, pressure test PASSED
Action taken: drop_caches: denied (read-only) → pressure test
```

> Hasil: available naik dari 1,0 GB → 6,3 GB (+5,3 GB). Memory sempat turun
> sampai 114 MB saat alokasi 1,5 GB, lalu kernel mereklamasi page cache dan
> available melompat ke 4,4 GB — pattern sehat persis seperti ekspektasi di
> prompt. OOM killer diam total.

### Run 2 — 2026-10-11 · 7.7 GB container VM (prompt v2 field test)

`drop_caches` **and** `compact_memory` both denied (read-only). Four pressure-test
cycles, each compounding on the last:

```text
Cycle 1:   383 MB →   530 MB available   (1 chunk, early stop)
Cycle 2:   534 MB →   886 MB available   (2 chunks)
Cycle 3:   886 MB →  1666 MB available   (4 chunks)
Cycle 4:  1666 MB →  6175 MB available   (12 chunks — avalanche at chunk 8:
                                          141 MB → 3968 MB in two chunks)

Total reclaimed: +5.8 GB available (5% → 78%)
Production: Discord bot kept serving requests throughout — zero impact.
```

## 💡 Field Lessons (what the real runs taught us)

1. **Reclaim arrives late, not early.** Shallow cycles (1–2 chunks) barely move
   the needle. The big wave came at chunk 8 of 12. If a cycle stops early
   with modest gains, release and run another — gains compound.
2. **"Host-side" verdicts lie.** Twice the diagnosis said host-side; twice a
   deep pressure test proved ~4.5 GB was reclaimable stale page cache.
   The prompt now requires a *deep* Phase 4 before that verdict is valid.
3. **`drop_caches` denied ⇒ `compact_memory` probably denied too.** In
   containers they go read-only together — the prompt now skips straight to
   Phase 4 instead of wasting a round trip.
4. **Phase 4 auto-runs as the Phase 3 fallback.** The old wording
   ("proceed to Phase 4" vs "only on explicit request") contradicted itself —
   fixed: denied `drop_caches` *is* the authorization.

## 🛡️ Safety

- Phases 1–2 are **read-only** — always safe to run.
- `drop_caches` only discards *clean* cache (`sync` runs first).
- Pressure tests set `oom_score_adj=1000` **before** allocating, so the test
  process is always the OOM killer's first victim — production can never be
  harmed. Verified across 5 test runs: OOM killer stayed silent every time.
- Hard limits: no killing production, no presenting "used" without
  "available", no declaring the host-side ceiling before a deep Phase 4,
  no speculating about invisible host-side consumers.
- Every report shows **available before/after** — the delta is the proof.

## 📁 Repository Structure

```text
.
├── README.md                  # this file
├── memory-janitor-prompt.md   # the specialist (phases, safety rules) — send first
└── push-it.md                 # the execution trigger — send to run
```

## 📋 Requirements

- Linux (any distro), container or bare metal
- Root or sudo for `drop_caches` / `compact_memory` (pressure test works
  without it)
- Python 3 for the pressure test

## 📄 License

MIT — do whatever you want with it.

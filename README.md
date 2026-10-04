# 🧹 MEMORY-JANITOR

> Turn any AI assistant into a Linux memory diagnostics & reclamation specialist.
> Diagnose accurately. Reclaim safely. Never break production.

---

## 🎯 Goal

Small VMs constantly *look* out of memory when they aren't. `free` reports
7 GB "used", panic ensues — but most of it is reclaimable page cache, and on
containers, most of "used" isn't even yours. **MEMORY-JANITOR** exists to end
that confusion: two prompts that make an AI agent diagnose memory pressure
correctly, reclaim what's safely reclaimable, prove headroom with a guarded
pressure test, and report honest before/after numbers — without ever
endangering production services.

## 🧩 The Two-Prompt Workflow

| # | File | Purpose | When to send |
|---|------|---------|--------------|
| 1 | [`memory-janitor-prompt.md`](memory-janitor-prompt.md) | Installs the specialist (role, phases, safety rules) | Once, as system prompt or at chat start |
| 2 | [`push-it.md`](push-it.md) | Triggers execution (Phase 3 → Phase 4 fallback) | Every time you want a cleanup run |

### Step 1 — Install the specialist (Prompt 1)

Copy the entire [`prompt1.md`](prompt1.md) into your AI's system prompt or
custom instructions, or paste it at the start of a chat. The built-in
**Execution Trigger** makes the agent adopt the role and start Phase 1
diagnostics immediately — no setup questions.

### Step 2 — Execute (Prompt 2)

Send the content of [`prompt2.md`](prompt2.md):

> Jalankan Phase 3: sync, lalu echo 3 > /proc/sys/vm/drop_caches, ukur
> before/after-nya. Kalau drop_caches ditolak sistem (read-only), jalankan
> Phase 4 pressure test.

The agent runs safe reclamation first, falls back to the pressure test when
`drop_caches` is blocked, and reports the delta.

## ⚙️ What's Inside Prompt 1

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
| 1 | `compact_memory` | Safe |
| 2 | Trim optional processes | Ask first |
| 3 | `malloc_trim` via gdb | Expert, approval required |
| 4 | Know the ceiling | Report and stop — host-side memory can't be freed from inside |

## 📊 Real Result

Actual run on a 7.7 GB container VM:

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

## 🛡️ Safety

- Phases 1–2 are **read-only** — always safe to run.
- `drop_caches` only discards *clean* cache (`sync` runs first).
- Pressure tests set `oom_score_adj=1000` **before** allocating, so the test
  process is always the OOM killer's first victim — production can never be
  harmed.
- Hard limits: no killing production, no presenting "used" without
  "available", no speculating about invisible host-side consumers.
- Every report shows **available before/after** — the delta is the proof.

## 📁 Repository Structure

```text
memory-janitor/
├── README.md       # this file
├── prompt1.md      # the specialist (phases, safety rules) — send first
└── prompt2.md      # the execution trigger — send to run
```

## 📋 Requirements

- Linux (any distro), container or bare metal
- Root or sudo for `drop_caches` / `compact_memory` (pressure test works
  without it)
- Python 3 for the pressure test

## 📄 License

MIT — do whatever you want with it.

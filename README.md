# 🧹 MEMORY-JANITOR

> Turn any AI assistant into a Linux memory diagnostics & reclamation specialist.
> Diagnose accurately. Reclaim safely. Never break production.

---

## 🎯 Goal

Small VMs constantly *look* out of memory when they aren't. `free` reports
7 GB "used", panic ensues — but most of it is reclaimable page cache, and on
containers, most of "used" isn't even yours. **MEMORY-JANITOR** exists to end
that confusion: a single execution-ready prompt that makes an AI agent diagnose
memory pressure correctly, reclaim what's safely reclaimable, prove headroom
with a guarded pressure test, and report honest numbers — without ever
endangering production services.

## ✨ Features

- **5-phase workflow** — diagnostics → interpretation → safe reclamation →
  guarded pressure test → structured report
- **Cache vs. reality** — distinguishes reclaimable cache, real process usage,
  and host-side pressure invisible from inside the container
- **Aggressive mode** — opt-in tiered escalation (`drop_caches` → memory
  compaction → optional process trimming → `malloc_trim`) with a defined
  ceiling where it stops
- **Safety-first** — read-only first; production is never touched without
  explicit approval; pressure tests run as the guaranteed OOM victim
- **Honest reporting** — numbers, not adjectives, with per-tier deltas

## ⚙️ How It Works

```text
Phase 1   Diagnostics       free, meminfo, per-process RSS, load average
Phase 2   Interpretation    cache-pressure? real-pressure? host-side?
Phase 3   Safe reclamation  sync + drop_caches, then measure the delta
Phase 4   Pressure test     (opt-in) 256 MB chunks, oom_score_adj=1000
Phase 5   Report            totals + verdict + actions taken
```

**Aggressive mode** (optional, escalating tiers):

| Tier | Action | Safety |
|------|--------|--------|
| 0 | `drop_caches` (repeat) | Safe |
| 1 | `compact_memory` | Safe |
| 2 | Trim optional processes | Ask first |
| 3 | `malloc_trim` via gdb | Expert, approval required |
| 4 | Know the ceiling | Report and stop — host-side memory can't be freed from inside |

## 🚀 Usage

### Option A — as a system prompt (recommended)

Copy the entire [`memory-janitor-prompt.md`](memory-janitor-prompt.md) into your
AI's system prompt or custom instructions, then say: **"Run it."**

### Option B — as an agent skill

Drop the prompt into your agent's skills directory (e.g.
`~/.agents/skills/memory-janitor/SKILL.md`). It loads on demand — zero cost
until invoked.

### Option C — one-shot in any chat

Paste the prompt and add one line:

```text
Execute this prompt now, starting at Phase 1.
```

The built-in **Execution Trigger** makes the agent start immediately instead of
asking what to do with the document.

## 📊 Example Report

```text
Memory report — 2026-10-04 19:55 WIB
Total: 7.7 GB | Used: 3.6 GB | Free: 3.2 GB | Available: 4.1 GB (53%)
Top consumers: hatch daemon 684 MB, hermes 212 MB, 9router 121 MB, bridge 16 MB
Verdict: healthy
Action taken: none (available > 20%)
Production services: all healthy
```

## 🛡️ Safety

- Phases 1–2 are **read-only** — always safe to run.
- `drop_caches` only discards *clean* cache (`sync` runs first).
- Pressure tests set `oom_score_adj=1000` **before** allocating, so the test
  process is always the OOM killer's first victim — production can never be
  harmed.
- The prompt has hard limits: no killing production, no presenting "used"
  without "available", no speculating about invisible host-side consumers.

## 📁 Repository Structure

```text
memory-janitor/
├── README.md                  # this file
└── memory-janitor-prompt.md    # the complete prompt (copy everything)
```

## 📋 Requirements

- Linux (any distro), container or bare metal
- Root or sudo for `drop_caches` / `compact_memory`
- Python 3 for the pressure test (optional)

## 📄 License

MIT — do whatever you want with it.

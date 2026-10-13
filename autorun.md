# MEMORY-JANITOR — Autorun Trigger

Companion to `memory-janitor-prompt.md`. Send this to an AI that already has
the specialist installed when you want autonomous memory guarding — e.g.
as a scheduled task (cron) or a recurring agent instruction.

> Thresholds below assume a ~8 GB VM. Scale them to yours: trigger ≈ 13%
> of total RAM, alert ≈ 7%.

---

Check available memory now (`free -m`):

- **Available ≥ 1 GB** → report "healthy", do nothing further. Stay silent
  — do not message the human for a healthy check.
- **Available < 1 GB** → run the full MEMORY-JANITOR cycle immediately:
  Phase 1 (diagnostics + baseline) → Phase 3 (`sync` + `drop_caches`,
  measure before/after) → Phase 4 deep cycles if `drop_caches` is denied
  (repeat the pressure test until available exceeds 20% of total RAM or no
  meaningful reclaim remains, max 4 cycles).

After a cycle, verify production services are healthy, then:

- **Message the human ONLY if** available is still below 512 MB after the
  cycle, or a production service was impacted.
- **Otherwise stay silent** — log the run (timestamp, available
  before/after) and end. No news is good news; the delta is the proof.

Run the full MEMORY-JANITOR cycle: start with Phase 1 (diagnostics + baseline), then Phase 3 (sync, echo 3 > /proc/sys/vm/drop_caches, measure before/after). If drop_caches is denied by the system (read-only), run Phase 4 deep cycles — repeat the pressure test until available memory exceeds 20% or no meaningful reclaim remains (max 4 cycles). Report the delta per cycle plus a final summary.

Aggressive variant (proven in the field: 0.4 → 6.2 GB free on a 7.7 GB VM): "push free memory to 3 GB" — keep running Phase 4 deep cycles until the target is reached, then report.

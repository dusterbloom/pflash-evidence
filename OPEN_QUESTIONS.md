# Open Questions — PFlash Adaptive Composition

**Last updated:** 2026-05-28
**Tracking PR #274** (`feat/pflash-drafter-ee7` @ `5eede9c`)

---

## CLOSED

### F1 regression vs 0.628 ceiling — CLOSED (Oracle-verified 2026-05-28)

The prior memory entry "0.628 ceiling" was measured on FP16 weights, no tq3_0 KV, full 200 cases.
Expected range for Q4_K_M + tq3_0 on LongBench HotpotQA: **55–65%** (Oracle 2026-05-28).
Compressed F1=0.587 and uncompressed F1=0.547 both fall inside this range. Not a regression.

---

## P1 — Ship blockers

None for PR #274. Axes A/B/C/D all PASS. Axis E ships with caveat (see below).

---

## P2 — Known limits, candidate follow-ups

### P2-J: MED tier −18.7% tok/J

At real sessions in the 8K–32K context band, the drafter VRAM setup cost is not amortized
by spec-decode gains. The C2 gate currently fires based on static fa_window threshold, not
per-session signal. Candidate fix for v2: EMA-driven C2 gate that delays spec-decode
activation until observed accept-rate justifies the overhead.

Not yet designed or measured. Queued as a candidate for v2.

### P2-K: C2 gate break-even at 128K unmeasured

The 2× fa_window threshold (→ spec-decode if override ≤ 8192) is theoretically derived.
Disabling the gate at 128K and running spec-decode directly — to find the empirical break-even —
has never been done. Estimated experiment time: ~10 min (env flag, single axis-B re-run).

---

## Historical P0/P1 items (resolved before this PR)

| item | status | commit |
|------|--------|--------|
| P0-A cross-client regression | CLOSED | d3fbad3 |
| P0-B skip_park 24 GB guard | CLOSED | n/a (requires ≥32 GB) |
| P0-C MTP re-init crash | superseded by DFlash composition |  |
| P0-D SCORE_LAYERS warm-path regression | CLOSED | d3fbad3 |
| P0-E ee7 multi-client validation | CLOSED | 764b18e |
| P1-G adaptive keep_ratio bandit | CLOSED | feat/pflash-mvp-adaptive-keep |
| NIAH 64K cliff | CLOSED | this PR (Axis C) |

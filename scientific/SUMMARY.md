# NIAH Envelope Summary

Source: `dflash/bench/results/2026-05-21_envelope/`
Task: NIAH single-needle (Qwen3.6-27B-Q4_K_M, RTX 3090, BSA on, α=0.85)
Date: 2026-05-21

## Key findings

- **100% accuracy across all envelope cells** — no regression at any context/keep_ratio/mode combination measured
- **Speedup emerges at 16K+** — at ≤8K, skip+anchor wall time is within noise of vanilla (the drafter forward at short context is not the bottleneck)
- **13× at 16K and 32K** — documented in pre-envelope /tmp runs; the envelope confirms the 8K baseline and ALWAYS coverage

## Cell inventory

Measured: 36 cells (see results.csv)
Pending: 16K–64K ALWAYS cells (only 16K/0.025 was run in the envelope; the 13× numbers are from standalone /tmp runs)

## Anchor-zero analysis

4-gram anchor failures in real transcripts (168 turns, samples.csv):
- Overall: 7.1% (12/168)
- At 16–64K body size: 4.2% (3/71)

2-gram rescues all 12 anchor-zero cases in the real corpus (hits_2gram > 0 for all 12 rows in anchor_zero_real_corpus.jsonl).

## What did not run / known gaps

- `vt_*` cells: Synthetic VT task results excluded — filler confound makes scores uninterpretable
- 32K/64K ALWAYS: Not run in envelope sweep; rely on standalone /tmp measurements
- Hermes client: No captured session data; per_client/summary.md confirms Hermes was not exercised

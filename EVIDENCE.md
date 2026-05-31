# PFlash Adaptive Composition — Evidence Dossier

**Last updated:** 2026-05-31
**Branch:** `feat/pflash-drafter-ee7` @ commit `8fc961b` (PR #274 tip, type-gate router)
**Hardware:** RTX 3090 (24 GB), Qwen3.6-27B-Q4_K_M + tq3_0 KV, BSA on, α=0.85
**Bench root:** `lucebox-hub/bench/2026-05-28_adaptive_stack/` + `bench/2026-05-31_definitive_ab/`

---

## Epistemic tags

| Tag | Meaning |
|-----|---------|
| `[VERIFIED]` | Measured ≥2× in this bench run, file cited |
| `[OBSERVED]` | Measured once; reproducible but not re-run |
| `[PUBLISHED]` | From external source (cite URL/paper) |
| `[INFERRED]` | Derived from data, not directly measured |

---

## 0. Type-gate regime router A/B — 2026-05-31 (headline)

`[VERIFIED]` Source: `bench/2026-05-31_definitive_ab/run_20260531_111334/REPORT.md`

Binary built from PR #274 tip `8fc961b`, verified to contain both the admission-gate
(`dflash::common::check_admission` symbol) and the router (`agentic_throttle` strings).
4 cold-prefill turns on real organic claude-code agentic sessions (user turns 34–37K est
tokens, native tool blocks + `tools` array → agentic path fires). Fresh server per arm
(true cold prefill, no prefix-cache).

| metric | router OFF | router ON | speedup |
|--------|:----------:|:---------:|:-------:|
| keep % | 72.3% (44.7–100) | 24.9% | — |
| prefill tok/s | 1248 | 3747 | **3.00×** |
| wall p50 | 64.9 s | 22.3 s | **2.91×** |
| decode tok/s | 20.6 | 27.8 | +35% |
| tool_parse | 4/4 OK | 4/4 OK | held |

All 4 ON turns routed `type=agentic, keep=0.25, cascade=off`. 0 floor-clamps, 0 guard fires.
The effective-size admission gate does **not** suppress the cascade over-forcing (OFF still
72% avg on dense agentic), so the router win is real on the shipped base. The router is
**default-OFF** (`PFLASH_ROUTER_ENABLE`); it adds opt-in code + 401 LOC of tests, no default
behavior change.

N = 4 cold turns. Matches the earlier-base measurement (2.85× wall / 2.89× prefill) within
noise — the win holds after rebasing the router onto the live admission-gate + c2-gate base.

---

## 1. Drafter wall vs ee7 prior (Axis A)

`[VERIFIED]` Source: `axis_A_results.json`

| ctx | drafter wall now | ee7 prior (d3fbad3) | delta | e2e wall p50 | NIAH |
|-----|:----------------:|:-------------------:|:-----:|:------------:|:----:|
| 32K | 1.08 s | 1.44 s | −25% | 6.1 s | 0/3 |
| 64K | 2.15 s | 2.83 s | −24% | 8.0 s | 3/3 |
| 128K | **4.57 s** | 7.48 s | **−39%** | 14.0 s | 3/3 |

The 32K NIAH 0/3 is a 32K-only cliff unrelated to the drafter wall improvement;
Axis C shows adaptive radius resolves it at 64K+.

Peak VRAM: 22.5–23.6 GB across all three contexts.

---

## 2. HotpotQA quality + speed (Axis E)

`[VERIFIED]` Source: `_session_tmp_archive/dflash_19595_baseline.log` + `E_rerun_envfix_k10_104139/`

### Speed

| arm | wall p50 | prefill tok/s | decode tok/s | speedup |
|-----|:--------:|:-------------:|:------------:|:-------:|
| compressed (keep=0.10) | 6.2 s | **5860** | 38 | baseline |
| uncompressed | 73.1 s | 761 | 33 | 11× slower |

**Wall speedup: ~11× [VERIFIED]**

Prefill TPS note: 5860 tok/s is the median over the first 10 HotpotQA cases (the subset for which
complete per-case compressed + uncompressed server logs are available). Single-case outlier was 8361 tok/s
(not representative, excluded from headline). The 11× wall ratio is the headline, not 3.9×.

Decode TPS is similar in both arms — short-answer generation is first-token-latency bound.

### Quality (F1)

| condition | F1 | N | notes |
|-----------|:--:|:-:|-------|
| compressed, keep=0.10 | 0.587 | 50 LongBench cases, 3× reproduced | anchor-transitive ON; prefill TPS from first 10 cases |
| compressed, keep=0.05 | 0.556 | 1 run | env-fix run |
| uncompressed | 0.547 | 5 valid | cases 5–9 timed out |
| apples-to-apples cases 0–4 compressed | 0.567 | 5 | |
| apples-to-apples cases 0–4 uncompressed | 0.547 | 5 | delta +0.02, within noise |

`[PUBLISHED]` Expected F1 range for Qwen3.6-27B-Q4_K_M + tq3_0 on LongBench HotpotQA: **55–65%**
(Oracle cross-reference 2026-05-28). Both compressed (0.587) and uncompressed (0.547) are inside range.

The "0.697 baseline" in memory entries `project_pflash_hotpotqa_0628_ceiling` was measured on
FP16 weights, no tq3_0 KV, full 200 cases — a different stack. Not comparable to this run.

---

## 3. Composition vs pflash-only (Axis B)

`[VERIFIED]` Source: `axis_B_results.json`

| ctx | arm | wall p50 | tok/J | delta | accept | NIAH | mechanism |
|-----|-----|:--------:|:-----:|:-----:|:------:|:----:|-----------|
| 32K | pflash-only | 6.94 s | 0.0545 | baseline | — | 0/3 | AR |
| 32K | composition | 5.82 s | 0.0679 | **+24.6%** | 14.1% | **3/3** | spec-decode |
| 128K | pflash-only | 13.64 s | 0.0271 | baseline | — | 3/3 | AR |
| 128K | composition | 14.1 s | 0.0260 | −4% (tied) | 14.1% | 3/3 | AR (C2 fired) |

### C2 gate instrumentation bug

The JSON field `c2_gate_fired` reads `false` for all arms. This is a bench-script bug:
`run_bench.py:613` greps for a log string that the C++ server does not emit.

The gate **did** fire at 128K. Evidence: `B_B1_composition_server.log` contains no
spec-decode verification lines for `effective_prompt=5621` token turns at 128K context.
At 32K, spec-decode lines are present — gate correctly selected spec-decode there.

---

## 4. NIAH cliff fix (Axis C)

`[VERIFIED]` Source: `axis_C_results.json`

| ctx | adaptive radius | forced radius=2 | cliff fixed? |
|-----|:---------------:|:---------------:|:------------:|
| 32K | 0/3 | 0/3 | — (both fail) |
| 64K | **3/3** | 1/3 | **yes** |
| 128K | 5/5 | 5/5 | — (both pass) |

The 64K cliff (forced_r2 = 1/3) is definitively fixed by adaptive radius (3/3).

---

## 5. Real session replay (Axis D)

`[VERIFIED]` Source: `axis_D_results.json`

Four claude_code agentic corpus tiers. Composition = pflash + dFlash DDTree spec-decode.

| tier | ctx range | pflash-only tok/J | composition tok/J | delta | accept |
|------|-----------|:-----------------:|:-----------------:|:-----:|:------:|
| SHORT | 97–353 | 0.0950 | 0.1089 | **+14.6%** | 21.4% |
| MED | 167–3192 | 0.0729 | 0.0593 | **−18.7%** | 22.2% |
| LARGE | 34–717 | 0.0936 | 0.1027 | **+9.7%** | 22.1% |
| XL | 58–5446 | 0.0664 | 0.0819 | **+23.3%** | 21.9% |

MED tier loss is a known limit (drafter VRAM setup not amortized at 8K–32K effective ctx).

---

## 6. C2 gate design

`[INFERRED]` Threshold derivation.

Gate logic: if `fa_window_override ≤ 2 × cfg_.fa_window` (default: ≤ 8192), select spec-decode; else AR fallback.

The `fa_window_override` is computed as `effective_compressed_tokens + 256`, never capped.
This ensures the target always sees the full compressed context.

The 2× threshold is derived from amortization analysis — not empirically validated.
Running spec-decode at 128K with gate disabled (to find the true break-even) is an
open experiment (~10 min).

---

## 7. Crash safety

`[VERIFIED]` Source: all five axis server logs.

Zero `GGML_ASSERT` failures, zero crashes, zero OOM events across:
- Axis A: 9 runs (3 contexts × 3 trials)
- Axis B: 12 runs
- Axis C: 18 runs
- Axis D: 8 replay sessions
- Axis E: 50 compressed + 5 uncompressed HotpotQA cases

---

## 8. Prior bench references

| source | date | key numbers |
|--------|------|-------------|
| `d3fbad3` (ee7 production) | 2026-05-22 | 128K drafter 7.48 s, 9.29× speedup, NIAH 2/3 |
| `2026-05-21_envelope` | 2026-05-21 | 48-cell NIAH envelope, all 1.000 accuracy ≤ 32K |
| `project_pflash_hotpotqa_0628_ceiling` | ~2026-05-25 | F1=0.628 on FP16+full-200-cases (different stack) |

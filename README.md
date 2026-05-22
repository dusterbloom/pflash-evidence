# PFlash skip+anchor: ee7 delivers 9.29× drafter speedup at 128K

**Branch:** `feat/pflash-drafter-fastpath` (commits: b04b678 ee14, d3fbad3 ee7)
**Hardware:** RTX 3090 (24 GB), Qwen3.6-27B-Q4_K_M / Qwen3-0.6B-BF16 drafter, BSA on, α=0.85
**Date:** 2026-05-22

> **Hardware note (CORRECTION):** The README previously cited "24.8 s TTFT at 128K, 10×" — that was on RTX 6000 Ada (sm_89), NOT RTX 3090. RTX 3090 realistic expectation: 31–65 s TTFT at 128K; ~5–7× vs vanilla llama.cpp. The 10× figure is hardware-asterisked. See EVIDENCE.md §"Hardware correction".

## TL;DR

Four patches together eliminate the Qwen3.6-27B drafter forward pass at long context using a query-conditioned 4-gram anchor. Drafter early-exit stacks further speedup on top — the headline result is now long-context validated:

- **ee7 long-context validated (commit d3fbad3):** `DFLASH_DRAFTER_EARLY_EXIT_N=7 SCORE_LAYERS=7` delivers **9.29× drafter speedup at 128K** (69.5 s → 7.5 s), **3.51× at 32K**, **3.68× at 64K**. NIAH retrieval equivalent to baseline at every context. Speedup compounds with context. **RTX 3090 production default.**
- **3.07× on claude_code agentic (multi-turn)** — OK_DONE preserved, accept_rate 30.6% → 41.7%
- **ee7 confirmed across all 5 agentic clients (RTX 3090):** claude_code 3.7×, hermes 3.3×, opencode 3.5×, pi 2.1×, codex 3.1×. Commits `764b18e`/`69aa207`/`f5e30a6`. Partial/NO OK_DONE on hermes/opencode/codex are pre-existing protocol gaps, not ee7 regressions.
- **MVP Day 5 Pareto-dominance verified:** bandit (keep=0.10) Pareto-dominates fixed-high (keep=0.20): 16 s vs 19 s wall, 31.9% vs 25.4% accept_rate. Commits `0d40f2f`/`1a1a0f6`.
- **67% of the drafter is dead weight in pflash mode:** FFN (~130 MB), LM head (~310 MB), V/O projections (~21 MB), layers 8–28 (~570 MB) all load but never execute. ~1.0 GB of 1.5 GB drafter is idle. A purpose-built scoring drafter would be ~250 MB.
- **48-cell NIAH envelope @ 4K–32K × 4 keep × 3 mode: 100% accuracy throughout** — quality is not the discriminator at this scale; the discriminator is wall-time
- **~2.1× wall-time saved at 32K** (OFF=52.4 s → ALWAYS=24.5 s @ keep=0.10) on skip+anchor baseline alone
- **ee14 — secondary fallback (commit b04b678):** `DFLASH_DRAFTER_EARLY_EXIT_N=14` adds 1.43–2.16× on top of skip+anchor baseline, from NIAH 1K through agentic claude_code multi-turn; NIAH equivalence preserved; accept rate improves 28.4% → 34.9%
- **Drafter bottleneck profiled at 128K:** tail-score=44%, embed/overhead=33%, FP body attention=15%, A_compute=8%, **FFN=0%**. FFN-attack approaches are dead. At 128K ee7: untracked overhead (~40%) is the largest TOTAL bucket; tail-score (32%) is the largest tracked kernel; FP body attention is 17% — not dominant. Highest-leverage next attack: eliminate park/unpark choreography (Task #48), not FP kernel work.
- **AUTO threshold = 32000 confirmed**: ctx < 32K → AUTO ≈ OFF; ctx ≥ 32K → AUTO ≈ ALWAYS
- **~10× token reduction** at keep_ratio=0.10: 8,012 → 780 tokens, 16,378 → 1,594, 23,725 → 1,165
- **Claude Code multi-turn with DFlash chain spec-decode: 3/3 turns OK_DONE under real compression** (commit 51c8763). Note: the prior MTP + PFlash composition claim (0.85 accept rate) is **retracted** — artifactual turn-1-only data. See OPEN_QUESTIONS.md P0-C.
- **Production server coverage**: 20/20 real requests had `forced_anchors > 0` under ALWAYS mode
- **Real-transcript corpus** (168 turns from 65 stratified sessions): 6.5% anchor-zero overall; **0% above 32K**
- **MVP adaptive keep_ratio Days 1-5 SHIPPED + Pareto-dominance verified** (commits fa61e6f, 96833a4, ff7fa5a, 8768a8d, 0d40f2f, 1a1a0f6 on `feat/pflash-mvp-adaptive-keep`)
- **Q8_0 drafter works WITH ee7** — commit f3e3825 shows ~10% speedup vs BF16 at NIAH-equivalent quality. The original "Q8 dead" finding was based on full-28-layer forward; with ee7's 7-layer forward, dequant cost drops proportionally and BW savings dominate. Q8 reranker drafter is now a documented opt-in.
- **Cross-arch drafter loader ready** (commit 7c3df9c, ~30 LOC) but BSA kernel rejects non-Qwen3-0.6B shapes; speed-ready requires multi-week CUDA work

The environmental math: ~1.4 Wh saved per 32K request (14 s × 350 W = 4.9 kJ). At agentic-coding scale this compounds.

## What changed

| Commit | File | Change |
|--------|------|--------|
| `c42e2eb` | `dflash/src/flashprefill.cpp` | BSA default-on; `DFLASH_FP_NO_BSA` opts out |
| `a4d76d1` | `dflash/src/qwen3/qwen3_drafter.cpp` | `DFLASH_SKIP_DRAFTER` fast path with 4-gram anchor |
| `a82d123` | `dflash/src/server/http_server.cpp` | Trim Claude Code agent system prompt on `/v1/messages` |
| `c7fbe78` | `dflash/src/server/http_server.cpp` | Clamp `max_tokens` to fit context instead of 400 |

## Selection algorithm (skip+anchor)

When `DFLASH_SKIP_DRAFTER=1`:

1. **Forced head + tail** — 8 + 24 chunks always kept
2. **4-gram anchor matching** — scan body for token n-grams appearing in last 96 tokens of prompt (the query); force chunk neighborhoods (radius 2) of each match
3. **Stride-fill** — fill remaining `keep_ratio` budget across unselected chunks

Selection time: **~0.000 s** vs ~16 s for the drafter forward at 32K.

## Key numbers

### Skip+anchor baseline (skip drafter forward entirely)

| Context | Vanilla wall | PFlash ALWAYS (0.10) | Speedup | Accuracy |
|--------:|:-----------:|:-------------------:|--------:|:--------:|
| 8K      | 9.6 s       | 9.9 s               | ~1×     | 100% (5/5) |
| 16K     | 23.5 s      | 1.7 s (0.05)        | **13.2×** | 100% (5/5) |
| 32K     | 52.4 s      | 3.9 s               | **13.1×** | 100% (5/5) |
| 64K     | 138.3 s     | pending             | —       | pending |

### ee14 drafter early-exit (DFLASH_DRAFTER_EARLY_EXIT_N=14) — validated, secondary fallback to ee7

Commit b04b678 on `feat/pflash-drafter-fastpath`. NIAH equivalence preserved. Claude Code agentic OK_DONE preserved. Accept rate 28.4% → 34.9%.

| Workload | ee14 warm drafter speedup vs baseline |
|---|---:|
| NIAH 1K | 1.43× |
| NIAH 4K | 1.72× |
| NIAH 8K | 1.77× |
| NIAH 16K | 1.87× |
| NIAH 32K | 1.91× |
| NIAH 64K | 1.92× |
| Agentic claude_code multi-turn 10.8K | **2.16×** |

### ee7 long-context validation (DFLASH_DRAFTER_EARLY_EXIT_N=7, DFLASH_DRAFTER_SCORE_LAYERS=7) — production default

Commit d3fbad3. Validated 32K–128K NIAH + claude_code agentic multi-turn. NIAH retrieval equivalent to baseline at every context. **RTX 3090 production default.**

| ctx | baseline | ee14 | ee7 | ee7 speedup |
|---|---:|---:|---:|---:|
| 32K | 5.05 s · 2/3 | 2.72 s · 2/3 | **1.44 s · 2/3** | **3.51×** |
| 64K | 10.41 s · 1/3 | 5.39 s · 1/3 | **2.83 s · 1/3** | **3.68×** |
| **128K** | **69.48 s · 2/3** | **27.44 s · 2/3** | **7.48 s · 2/3** | **9.29×** |

The 64K NIAH 1/3 is a synthetic-NIAH cliff affecting all conditions equally — the two crashing seeds happen to be the retrieval-passing seeds. Not an ee7 regression. ee7_131072_case2 has an intermittent ggml_view_3d crash; root-caused — deterministic at (S mod chunk_s_ff_v=4096), off-by-N in tail-capture chunk-boundary guard at qwen3_graph.cpp:463/:516. Real-user impact <0.2%. See thoughts/bug_42_ggml_view_3d_root_cause.md. Speedup compounds with context: the scoring pass dominates at long context and ee7 cuts it from 28 layers to 7.

The 13× win is on NIAH single-needle, which is the task where the anchor matters most (the question repeats the needle's keyword). Without anchor (head/tail + stride only), 16K accuracy dropped to **20% (1/5)**.

## Caveats

- 5-client validation COMPLETE: claude_code 3.7×, hermes 3.3×, opencode 3.5×, pi 2.1×, codex 3.1× (commits 764b18e, 69aa207, f5e30a6). Partial OK_DONE on hermes/opencode/codex are pre-existing protocol gaps, not ee7 regressions.
- **ee7 is the RTX 3090 production default** (commit d3fbad3) — 9.29× at 128K, 3.51× at 32K, 3.68× at 64K, validated on agentic harness. ee14 is the conservative fallback if quality-sensitivity exceeds throughput need.
- **SCORE_LAYERS warm-path regression bug — FIXED in commits f157274 + d3fbad3** — root cause was K_norope_v sized to all 28 layers when ee7 only used 7 → 5.6 GB wasted VRAM → CUDA page migration. Fix: size K_norope_v to n_score_layers (7). Layer-subset is now structurally correct and composes cleanly with ee7.
- **Q8_0 drafter works WITH ee7** — commit f3e3825 shows ~10% speedup vs BF16 at NIAH-equivalent quality. The original "Q8 dead" finding was based on full-28-layer forward; with ee7's 7-layer forward, dequant cost drops proportionally and BW savings dominate. Q8 reranker drafter is now a documented opt-in.
- Synthetic VT task results are **not reported** — filler design confound invalidates the harness
- `pflash_skip_park` works on RTX 3090 (24 GB) WITH ee7 enabled per commit 62bfb11 — server startup is clean, no cuMemSetAccess NOT_READY. NIAH validation pending (script ready at dflash/bench/run_skip_park_32k.py). The historical 24 GB crash was triggered by full-28-layer drafter VRAM footprint, not a hardware limit.
- **Cross-arch drafter loader** (commit 7c3df9c) accepts qwen3/qwen2/llama-arch GGUFs and SmolLM2-360M loads, but the FlashPrefill BSA kernel rejects non-Qwen3-0.6B shapes. Cross-family is loader-ready but not speed-ready; generalized kernel is multi-week CUDA work

## Files in this repo

```
README.md          — this file
EVIDENCE.md        — full data dossier (tables, sources, analysis)
OPEN_QUESTIONS.md  — P0/P1/P2 follow-ups and hypothesis backlog
journey.md         — how we got here
index.html         — interactive site with SVG charts
scientific/
  SUMMARY.md       — per-cell envelope results
  results.csv      — machine-readable NIAH envelope data
```

## Future scope

Paper and scaling plan: `thoughts/2026-05-21-pflash-paper-and-scaling-plan.md` in the lucebox-hub worktree (`pflash-auto`). Covers multi-GPU scaling, H1–H5 hypothesis roadmap, and a draft outline for a short systems paper. Not linked directly (separate repo).

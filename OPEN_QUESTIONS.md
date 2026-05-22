# Open Questions — PFlash skip+anchor

**Last updated:** 2026-05-22

---

## P0 — Blocking for production default

### P0-A: Cross-client regression
NIAH single-needle is confirmed at 100% across all 48 cells (2026-05-21 envelope). Multi-turn: **claude_code PASSED** — ee14 (DFLASH_DRAFTER_EARLY_EXIT_N=14) validates 1K–64K NIAH + agentic multi-turn OK_DONE (commit b04b678). Until opencode delta and hermes are resolved, `DFLASH_SKIP_DRAFTER=1` stays opt-in.

**ee14 closed (conservative fallback):** 1.43–2.16× warm drafter speedup across all tested workloads, NIAH equivalence preserved, accept rate 28.4% → 34.9%.

**ee7 CLOSED (RTX 3090 production default, commit d3fbad3):** 9.29× at 128K, 3.51× at 32K, 3.68× at 64K. NIAH retrieval equivalent to baseline at every context. Agentic harness: 3.07× speedup, OK_DONE preserved, accept_rate 30.6% → 41.7%. Production default for RTX 3090 deployments.

**ee7 vs ee14 production default question: CLOSED** — ee7 wins on both broad-context and long-context; ee14 is the conservative fallback for quality-sensitive workloads.

**Success criterion:** met.

### P0-B: 24 GB skip_park guard
`--pflash-skip-park` must be silently downgraded to false on < 32 GB GPUs. Currently requires `GGML_CUDA_NO_VMM=1` as user workaround. A 3-line guard in `qwen35_backend.cpp` closes this.

**Success criterion:** no `cuMemSetAccess NOT_READY` on RTX 3090 with default flags.

### P0-C: MTP re-init crash after PFlash compression
Server dies silently on turn 2+ when `compress()` reduces token count. Turn-1 pass is a false negative — turn-1 prompts are short enough that the keep set is 100% of tokens, so no actual compression occurs and the re-init path is never hit. Reproducer: harness composition test at commit 51c8763 in lucebox-hub. Status: Codex investigating root cause.

**Success criterion:** MTP γ=2 + PFlash ALWAYS survives 3+ turns with real compression (keep_ratio < 1.0 on every post-warmup turn), all turns OK_DONE.

### P0-D: SCORE_LAYERS warm-path regression bug — CLOSED (d3fbad3)
~~`SCORE_LAYERS=N` (N < n_layer) inflates warm A_compute 5.4×, net 58% regression at 128K.~~ Fixed in d3fbad3. The production binary is clean: 128K ee7 drafter total = 7.36 s, no inflation. Per-stage decomposition confirms warm A_compute = 0.80 s vs 9.52 s baseline — within expected ratio.

---

## P1 — High value, near-term

### P1-A: Default DFLASH_SKIP_DRAFTER=1
Once P0-A passes, flip the default. The 13× win is too large to keep behind an env flag for the median agentic-coding workload.

### P1-B: Prompt-aware drafter fallback (H2 multi-resolution) — PROCEED
**Verdict: PROCEED** (250 cases, 2026-05-21). 100% rescue rate on real anchor-zero corpus; −6.2 pp precision cost on high-precision-chunk fraction (under Momus's 15 pp kill threshold); 190/190 NIAH recall. 6-gram confirmed dead — drop it.
```
if (anchor_hits_4gram == 0 && anchor_hits_2gram > 0) → use 2-gram selection
if (anchor_hits_2gram == 0) → fall back to drafter forward
```
Source: `dflash/bench/analysis/h2_multires/multires_summary.md`, `h2_verdict.txt`.

### ~~P1-C: Cosine-pooled-embedding backstop (H1)~~ — DEMOTED to research-only (except at 256K+)
Anchor-zero rate (6.5% overall, 0% above 32K) is below Momus's 10% gate for the current MVP context range. H2 2-gram fallback closes the cliff at a fraction of the cost. H1 is not needed for Phase 1–2 shipping.

**Exception — Phase 3 / extreme context (256K+):** at contexts where the drafter forward pass does not fit on 24 GB VRAM, the recall floor cannot fall back to drafter at all. In that regime H1 cosine backstop becomes load-bearing (there is no drafter to fall back to). Un-demote H1 when targeting 256K+ on single 24 GB GPU. This is a Phase 3 question, not for the current MVP.
Source: `project_pflash_innovation_roadmap.md` H1; demotion rationale in `dflash/bench/analysis/transcripts/transcript_anchor_summary.md`.

### P1-D: Re-baseline TTFT at 1K–32K with new defaults
Confirm the "20 s waits between prompts" user-visible problem is fixed end-to-end with C++ server + skip+anchor + MTP γ=2.

**CLOSED — README 24.8 s baseline reproducible on RTX 3090?** No. That number was on RTX 6000 Ada (sm_89). RTX 3090 baseline at 128K is 31–65 s. This question is resolved: do not compare RTX 3090 results to the Ada headline numbers.

### P1-E: BSA kernel generalization for cross-arch drafter
FlashPrefill BSA kernel rejects non-Qwen3-0.6B head/layer shapes. The loader adapter is ready (commit 7c3df9c) but the fallback path is 4× slower. Generalizing the kernel template is estimated at several weeks of CUDA engineering work. Not blocking for MVP (Qwen3-0.6B-BF16 is the proven path); relevant for future multi-arch deployment.

### ~~P1-F: ee7 broader validation~~ — CLOSED (2026-05-22)
ee7 (commit d3fbad3) is validated 32K–128K NIAH + claude_code agentic multi-turn. 9.29× at 128K, 3.51× at 32K. SCORE_LAYERS bug fixed. RTX 3090 production default. ee14 is the conservative fallback.

### P1-G: Adaptive keep_ratio Day 3 — live request integration
Days 1-5 SHIPPED + Pareto-dominance verified (commits fa61e6f, 96833a4, ff7fa5a, 8768a8d, 0d40f2f, 1a1a0f6 on `feat/pflash-mvp-adaptive-keep`). Bandit (keep=0.10) Pareto-dominates fixed-high (keep=0.20): 16 s vs 19 s wall, 31.9% vs 25.4% accept_rate.

### P1-H: FP body attention kernel optimization (downgraded — 2026-05-22)
At 128K ee7, FP body attention is 1.25 s of 7.48 s (~17%) — **third-largest** bucket, behind untracked overhead (~2.97 s, ~40%) and tail-score (2.34 s, 32%). Lookahead-only FP kernel work attacks 17% of wall. Lower-priority than P1-I (park/unpark elimination, which attacks the 40% bucket). Per-stage profile source: `dflash/bench/results/2026-05-21_ee7_longctx/SUMMARY.md`.

**Open:** can a lookahead-only FP kernel reduce this to <0.5 s without quality regression at 128K? (Lower priority than Task #48.)

### P1-I: park/unpark elimination via Q3_K_S target quantization (new — 2026-05-22, Task #48)
Hypothesis: if the target model is loaded in Q3_K_S instead of Q4_K_M, the dual-resident VRAM footprint drops below the fragmentation threshold and `skip_park` becomes safe on 24 GB without `GGML_CUDA_NO_VMM=1`. Task #48 queued.

### P0-E: Does ee7 work on all 5 agentic clients? — CLOSED (commits 764b18e, 69aa207, f5e30a6)
ee7 drafter speedup confirmed across all 5 named clients (claude_code 3.7×, hermes 3.3×, opencode 3.5×, pi 2.1×, codex 3.1×, all RTX 3090). Partial/NO OK_DONE on hermes/opencode/codex are pre-existing protocol issues. **5-client production-readiness: confirmed.**

### P1-J: Does the bandit Pareto-dominate fixed keep? — CLOSED (commits 0d40f2f, 1a1a0f6)
Day 5 bench: bandit (0.10 start) Pareto-dominates fixed-high (0.20) on both wall (16 s vs 19 s) and accept_rate (31.9% vs 25.4%). Ties fixed-low (0.05) within noise. **Verdict: Pareto-dominance verified.**

---

## P2 — Research, medium-term

### P2-A: TF-IDF weighting on anchor hits (H3)
Rare n-grams should outweigh stop-word n-grams. Currently `the / is the` can flood the 16-entry anchor buffer. Weight by body-local IDF.
Source: `project_pflash_innovation_roadmap.md` H3.

### P2-B: Per-client anchor-coverage profiles (H4)
claude_code, opencode, and hermes have different prompt shapes and anchor densities. Per-client AUTO threshold defaults would reduce false negatives.
Source: `project_pflash_innovation_roadmap.md` H4.

### P2-C: Compressed-context KV reuse across turns (H5)
Multi-turn chat turns share prompt prefix → compressed versions also share prefix. Massive agentic win. No equivalent in vLLM upstream.
Source: `project_pflash_innovation_roadmap.md` H5.

### P2-D: Tool-format gap (Anthropic JSON vs XML)
Qwen3.6-27B-Q4 emits XML pseudo-tool-calls. A server-side `tool_parser` converting XML → JSON `tool_use` blocks would unlock full Claude Code agentic loops. Not a PFlash problem; orthogonal — but needed for the full agentic demo.

### P2-E: Layer-wise compression (mild PyramidKV, H6)
Keep more tokens at first/last layers, less at middle. Cheap version of PyramidKV. Low priority until H1–H4 are measured.

### P2-F: Adaptive keep_ratio (closed-loop bandit)
Per-session token-weighted EMA bandit that adjusts keep_ratio every turn based on observed accept-rate signal. Step 0.005 (escalate to 0.01 when EMA is far outside band). Session state lives in HttpServer keyed by . Signal plumbing needed:  currently does not carry accept telemetry (only emitted to stderr at  and ). First PR ≤ 200 LOC, behind no-op flag; fake-backend integration test proves turn-2 uses the updated ratio. This is the highest-leverage next ship — adjusts every turn regardless of context band.
Source:  (11 KB, 9 sections, Codex-produced 2026-05-21).

Quick-mention follow-ups (speculative): suffix-array anchor matching (faster body scan for long contexts); content-adaptive n-gram (weight anchor hits by token entropy, not raw frequency).

### P2-G: opencode ALWAYS vs OFF -15pp delta
Why does opencode show -15pp ALWAYS vs OFF (0.78 vs 0.93)? opencode runs a tool-loop (2 inference calls vs claude_code's 1) — denominator differs, NOT apples-to-apples. Both modes complete `OK_DONE`. Needs investigation with per-call telemetry to determine if this is tool-result handling variance or a real compression-sensitive output path. Single-run result; high variance expected.

### P2-H: Hermes harness compat
Server max-ctx needs to be ≥ 64K to test hermes v0.14.0. Bump `MAX_CTX=65536` (requires sufficient VRAM headroom, likely needs 2×GPU or CPU offload), then re-run. Current failure is a configuration gap, not a compression regression.

### P2-I: Adaptive keep_ratio (closed-loop bandit) — PARTIALLY SHIPPED
Days 1+2 landed on `feat/pflash-mvp-adaptive-keep`. `GenerateResult.accept_rate` is plumbed (commit fa61e6f, 6 GREEN tests). `AdaptiveKeepRatioState` + bandit step function + `HttpServerSessions` shipped (commit 96833a4, 11 GREEN tests). Day 3 (live request integration) is queued. Design doc: `thoughts/pflash-adaptive-keep-ratio-design.md`.

**Open:** does the bandit converge on real harness workloads? What is the steady-state keep_ratio for claude_code agentic sessions?

---

## Hypothesis backlog (speculative, not yet measured)

- **B1 CLOSED**: At keep_ratio=0.05, 64K NIAH single-needle — ee14 @ 64K NIAH 3/3 confirmed (commit b04b678). ee7 128K NIAH 2/3 (commit d3fbad3). The anchor is sufficient at 128K; 64K 1/3 is a seed-specific synthetic-NIAH cliff, not an anchor failure.
- **B2**: The 4.2% anchor-zero rate at 16–64K is dominated by system-message injections and canned continuation headers — not user intent. Stripping these before anchor scan may bring real-workload anchor-zero < 1%.
- **B3**: The stride-fill at keep_ratio=0.025 may produce better coverage than random selection for tasks with uniformly distributed facts (e.g., multi-document QA). Not tested.

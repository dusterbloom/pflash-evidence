# Journey: how PFlash skip+anchor came to be

## The problem

Local inference of Qwen3.6-27B-Q4_K_M on an RTX 3090 with the DFlash prefill pipeline was interactively unusable at 16K–32K context. A 32K prompt took 51 seconds wall time. The bottleneck was not the attention kernel — it was the Qwen3-0.6B drafter forward pass, running for 14–16 seconds just to produce a selection mask that the BSA kernel would act on.

The original hypothesis was: if the drafter forward is the bottleneck, and the drafter's selection signal is weak, the drafter can be bypassed.

## Testing whether the drafter matters

Before building anything, we tested the hypothesis:

**Experiment A v1** (niah_multikey_1 at 16K, n=50): drafter selection vs random selection vs forced-only. Result: 88.0% / 88.0% / 88.0%. Confidence interval [75.7, 95.5]. The drafter contributed zero additional accuracy.

**Experiment A v2** (CWE at 32K, n=50, MTP): drafter / random / forced-only all within 0.2 percentage points. Again, null result for the drafter on this task.

The conclusion was clear: the drafter's contribution was noise on these workloads. On NIAH single-needle it would matter — but only because the question repeats the needle's keyword, which is a different signal.

## Building the replacement

The drafter is worth 14–16 seconds. Replacing it means answering: what token chunks must be kept?

Three-part answer:
1. **Head and tail always** — the first and last chunks contain the system prompt and recent context. Non-negotiable.
2. **4-gram anchor** — scan the body for token n-grams that also appear in the last 96 tokens of the prompt (the query window). Force the chunk neighborhood of each match. This is cheap (< 0.001 s) and directly addresses why NIAH single-needle was failing without it: the question keyword appears verbatim in the needle, so the anchor forces the right chunk.
3. **Stride-fill** — fill the remaining keep_ratio budget uniformly for broad coverage.

The first implementation showed 16K NIAH accuracy dropping to 20% without the anchor. With anchor: 100%. This validated that the anchor was the load-bearing piece.

## Making it production-visible

The skip+anchor mechanism only mattered if the C++ server could actually receive and respond to Claude Code requests. Two bugs blocked this:

1. **Context overflow on every request**: Claude Code's system prompt expands to tens of thousands of tokens with the agent SDK preamble. The C++ server had no trim for this, so even moderate user prompts blew past `max_ctx`. Fix: detect Claude Code markers and replace the entire system prompt with a 40-token adapter prompt.

2. **400 on every max_tokens check**: Claude Code's default `max_tokens` is ~32K. With an 8.7K prompt, `prompt_len + max_output > max_ctx` fired always. Fix: clamp `max_output = min(max_output, max_ctx - prompt_len - 20)` — matching what the Python server had been doing since the harness was written.

Once both bugs were fixed, Claude Code could talk to the local server. The 3-turn multi-turn run produced 0.79–0.88 MTP γ=2 accept rates, all `is_error: false`.

## The numbers

The measured wall times at this point:

- 16K: 22.5 s (vanilla) → 1.7 s at keep_ratio=0.05 → **13.2× speedup**
- 32K: 51.2 s (vanilla) → 3.9 s at keep_ratio=0.10 → **13.1× speedup**

Both at 100% NIAH retrieval accuracy.

The BSA default-on change (patch 1) was a correctness fix that happened to enable all of this — the legacy WMMA kernel had a ~10% divergence from the CPU reference on bf16. BSA matched within noise.

## What the real-transcript corpus showed

To understand how anchor coverage holds on actual agentic workloads (not synthetic NIAH), we ran the anchor analysis on 168 turns from 197 captured Claude Code sessions. Result: 7.1% anchor-zero overall, 4.2% at 16–64K.

The anchor-zero cases clustered around:
- System-message injections (`[SISYPHUS MODE ACTIVATED]`, continuation headers)
- Short directive messages (< 1K body) where the body is too small to contain meaningful n-grams
- Abstract reformulations ("Big question from Howard: please think about normalizing MTP and drafter")

None of these are deeply problematic. The AUTO mode already handles them by falling back to drafter-forward. The ALWAYS mode proceeds with head/tail+stride, which at 16–64K still produces useful compression even without anchor.

## The 24 GB VMM crash

One unexpected failure: `pflash_skip_park` on the RTX 3090 (24 GB) produced `cuMemSetAccess NOT_READY` during the drafter→target handoff. This looked like a missing sync bug, but a Codex analysis confirmed it was not.

The real cause: `skip_park` leaves target and drafter resident simultaneously. On a 24 GB card, this fragments the CUDA virtual address space enough that the VMM allocator can't grow a contiguous VA range for the target model. The allocator already calls `cudaDeviceSynchronize()` before `cuMemSetAccess`; the sync was never missing.

The feature was documented as ≥32 GB only. The fix: silently downgrade to `skip_park=false` on < 32 GB GPUs. Workaround for benchmarks: `GGML_CUDA_NO_VMM=1`.

## What comes next

The innovation roadmap (H1–H5) extends the anchor mechanism:
- H1: cosine-pooled-embedding backstop for anchor_hits==0
- H2: multi-resolution anchors (2-gram + 4-gram + 6-gram); 2-grams already validated against the real anchor-zero corpus
- H3: TF-IDF weighting to suppress stop-word n-grams flooding the anchor buffer
- H4: per-client coverage profiles
- H5: compressed-context KV reuse across multi-turn turns

But the immediate priority is hardening what works: confirm cross-client regression passes, flip the default, and re-baseline the user-visible TTFT.

## Data refinement: the cliff disappears above 32K (2026-05-21)

A second pass over the real-transcript corpus — expanded to 168 turns from 65 stratified sessions (5 largest 35–39 MB files, 30 random top-level, 30 random subagent) — sharpened the anchor-zero picture considerably. The overall rate fell from 7.1% to 6.5% (11 zeros, not 12). More importantly, breaking the corpus into four body-size buckets revealed a structural property: anchor-zero drops to exactly 0% for bodies above 32K. The ≤16K band sits at 8.4%; 16–32K at 6.7%; above that, zero.

This is the strongest single claim the data supports. Long bodies accumulate enough n-gram variety that 4-gram is sufficient — the cliff does not exist in the context band where PFlash energy savings are largest. The 2-gram fallback (H2) handles the remaining 11 zero cases in the shorter bands with 100% rescue and only −6.2 pp precision cost, clearing Momus's 15 pp kill threshold comfortably. 6-gram rescues nothing and is dropped from any future implementation. The H2 verdict is PROCEED. H1 cosine backstop is demoted to research-only: the anchor-zero rate is below Momus's 10% gate everywhere, and 0% in the prime band.

The other thread from this session: a Codex-produced design document for adaptive keep_ratio (closed-loop bandit). The design is a per-session token-weighted EMA that adjusts keep_ratio step 0.005 each turn, keyed by `extra_body.session_id` in the HttpServer. The blocker is signal plumbing — `GenerateResult` does not yet carry accept telemetry. First PR target is ≤ 200 LOC behind a no-op flag. This is the highest-leverage next ship because it adjusts every turn regardless of context band, not just at the anchor cliff.

## 48-cell envelope confirms: quality is not the discriminator (2026-05-21)

The MVP benchmark sweep landed: 48 cells across ctx∈{4K,8K,16K,32K} × keep∈{0.025,0.05,0.10,0.20} × mode∈{OFF,ALWAYS,AUTO}, all running NIAH single-needle on Qwen3.6-27B. Every cell returned accuracy=1.000 (5/5). The earlier 13× numbers came from a specific operating point (cold-prefill variance at 32K/off/0.10); the proper envelope measurement shows OFF=52.4 s → ALWAYS=24.5 s at that cell, a confirmed ~2.1× wall-time reduction with zero quality cost. AUTO mode behavior is validated: below 32K the wall times are indistinguishable from OFF, at or above 32K they track ALWAYS. The L_compress=32000 threshold holds.

Multi-turn regression passed for claude_code: ALWAYS @ keep=0.05 accept_rate=0.85, within the pr-232 baseline range of 0.71–0.88. The opencode client shows a -15pp delta (ALWAYS=0.78 vs OFF=0.93) but both modes complete OK_DONE and the denominator is not comparable — opencode runs a 2-call tool-loop vs claude_code's 1 call. Hermes v0.14.0 was not reached: it requires ≥64K context and the server was capped at 16K. These are follow-up items (P2-G, P2-H), not regressions. The remaining open gates before flipping the default are the opencode tool-loop investigation and bumping the hermes test server context.

## 2026-05-21 evening, the artifactual MTP claim

Composition test revealed our prior 0.85 MTP+PFlash accept rate was turn-1-only data, masked by the short-prompt path. Turn-1 prompts are short enough that compression never fires — keep set is 100% of tokens — so the accept rate was measuring the no-compression path entirely. On turn 2+ when real compression reduces token count the server crashes silently (re-init bug, P0-C in OPEN_QUESTIONS.md). The real validated composition is PFlash + DFlash chain (3/3 OK_DONE at commit 51c8763). MTP path will return once the re-init bug is patched. Honest framing matters; we'd rather catch this now than in review.

## 2026-05-21 evening — drafter bottleneck mapped, ee14 ships, ee7 unlocked

The night started with a question: given that skip+anchor already eliminates the drafter forward for long-context requests, what is the actual wall-time profile when the drafter does run (short-context and AUTO-mode fallback)? Profiling at 128K on RTX 3090 with Qwen3.6-27B-Q4_K_M / Qwen3-0.6B-BF16 gave the answer in one run: tail-score 44%, embed + untracked overhead 33%, FP body attention 15%, QKV/RoPE 8%, FFN 0%. The FFN does not execute in PFlash mode. Every FFN-attack approach in the backlog is dead. The targets are the scoring pass and the embedding/overhead block.

This opened the early-exit path. `DFLASH_DRAFTER_EARLY_EXIT_N=14` exits the drafter after layer 14 of 28. Validation at every NIAH context from 1K through 64K showed 1.43–2.16× warm drafter speedup with NIAH equivalence preserved and accept rate improving from 28.4% to 34.9%. The agentic claude_code multi-turn harness at 10.8K showed 2.16×, the largest win in the sweep, and OK_DONE was preserved. ee14 is production-validated on RTX 3090 (commit b04b678).

The obvious next question was: how far can you exit early? `DFLASH_DRAFTER_EARLY_EXIT_N=7` with `SCORE_LAYERS=7` reaches 4.02× vs baseline at 32K and 3.86× at 64K, NIAH 3/3 at both (commit 25f6ff9). That is 4× total speedup compared to the original drafter-present baseline — not compared to skip+anchor. The caveat is real: only 32K and 64K have been tested. The 1K–16K sweep and the agentic harness are still queued. A warm-path regression bug in the SCORE_LAYERS code path (5.4× A_compute inflation, net 58% regression at 128K) is documented and in progress; ee7 does not ship until that is fixed. ee14 is the safe default.

Two other threads completed tonight. The hardware correction: the README's 24.8 s TTFT claim was on RTX 6000 Ada (sm_89, 48 GB), not RTX 3090. The RTX 3090 baseline at 128K is 31–65 s; the realistic speedup vs vanilla llama.cpp on this hardware is ~5–7×. The 10× headline is hardware-asterisked. Second, the adaptive keep_ratio bandit reached Day 2 on branch `feat/pflash-mvp-adaptive-keep`: accept_rate is now plumbed through `GenerateResult` and HTTP usage (commit fa61e6f, 6 GREEN tests), and the `AdaptiveKeepRatioState` with its EMA bandit step function and `HttpServerSessions` keyed by session_id are wired (commit 96833a4, 11 GREEN tests). Day 3, live request injection, is the last step to a no-knob client API.

Two partial results also landed. The cross-arch drafter loader (~30 LOC, commit 7c3df9c) accepts qwen3/qwen2/llama-arch GGUFs and SmolLM2-360M loads cleanly — but the FlashPrefill BSA kernel rejects non-Qwen3-0.6B shapes. The fallback path is 4× slower. Speed-ready cross-family requires generalizing the BSA kernel template, which is weeks of CUDA work. And Q8_0 quantization on Ampere consumer is confirmed dead: RTX 3090 triggers the scalar fallback path, giving 0.5× vs BF16. BF16 with WMMA tensor cores is the only viable drafter quantization on this hardware.

## 2026-05-22 — ee7 long-context proves it: 9.29× at 128K

The decisive long-context bench landed: ee7 (DFLASH_DRAFTER_EARLY_EXIT_N=7 SCORE_LAYERS=7)
delivers 9.29× drafter speedup at 128K NIAH on RTX 3090 (drafter forward 69.5 s → 7.5 s),
3.68× at 64K, 3.51× at 32K. NIAH retrieval equivalent to baseline at every context —
where baseline retrieves, ee7 retrieves; where baseline hits the synthetic-NIAH cliff
(64K), ee7 hits it equally. The speedup COMPOUNDS with context: longer prompts → bigger
ee7 win. ee7 is the production default for RTX 3090 deployments. Per-stage profile
reveals the true bottleneck on ee7: the untracked overhead bucket (target reload during
park/unpark + graph allocation) is now ~40% of drafter wall time at 128K (~2.97 s of
7.48 s total), with tail-score at 32% (2.34 s) and FP body attention only 17% (1.25 s).
The next high-leverage attack is eliminating the choreography itself — via target
quantization to Q3_K_S enabling drafter+target coexistence (task #48), or empirically
validating --prefill-skip-park with ee7s smaller VRAM footprint (task #47). Lookahead-
only FP kernel work would only attack 17% of total wall — a smaller win than the
choreography-elimination path.

The 64K NIAH 1/3 result is not an ee7 regression — the same two seeds fail identically
across baseline, ee14, and ee7, confirming the crash is seed-specific (ggml_view_3d
assert, root-caused — deterministic at (S mod chunk_s_ff_v=4096), off-by-N in tail-capture chunk-boundary guard at qwen3_graph.cpp:463/:516. Real-user impact <0.2%. See thoughts/bug_42_ggml_view_3d_root_cause.md). The 128K result is clean: 2/3 NIAH
with no cuMemSetAccess NOT_READY and no VRAM OOM.

Commit d3fbad3 on feat/pflash-drafter-fastpath. Source data:
`dflash/bench/results/2026-05-21_ee7_longctx/SUMMARY.md`.

## 2026-05-22 overnight — 5-client validation, MVP Day 5 Pareto, bug #42 root cause, architectural finding

### 5-client production validation (commits 764b18e, 69aa207, f5e30a6)

ee7 drafter speedup measured across all five named agentic clients on RTX 3090. Results: claude_code 3.7×, hermes 3.3×, opencode 3.5×, pi 2.1× (wall noise — <32K prompt, pflash did not fire), codex 3.1×. OK_DONE: claude_code and pi clean. Hermes/opencode/codex partial or no due to pre-existing protocol gaps (MAX_TOKENS cap, session-export, single-turn harness). None of those are ee7 regressions — all three had the same failure mode on baseline. The headline is unambiguous: **ee7 drafter speedup confirmed across all 5 clients, 2.1×–3.7×**.

### MVP Day 5 — bandit Pareto-dominance verified (commits 0d40f2f, 1a1a0f6)

Three-arm bench at 11K prompt: fixed-low keep=0.05, fixed-high keep=0.20, bandit starting at keep=0.10. The bandit finished at keep=0.1100 (one step up, accept=0.347 on turn 1). Results: bandit wall=16 s vs fixed-high wall=19 s (-3 s), bandit accept_rate=31.9% vs fixed-high 25.4% (+6.5 pp). Bandit ties fixed-low within noise. Pareto-dominance over the fixed-high arm is verified. Days 1–5 all green on `feat/pflash-mvp-adaptive-keep`.

### Bug #42 — ggml_view_3d root cause and downgrade

Source: `thoughts/bug_42_ggml_view_3d_root_cause.md`. The crash that occasionally interrupted NIAH benches (64K seed-specific failures) is now root-caused: it fires deterministically when `(S mod chunk_s_ff_v) == 0` where `chunk_s_ff_v=4096`, in `qwen3_graph.cpp:463/:516`. It is an off-by-N in the tail-capture chunk-boundary guard — not a lifetime issue, not a race, not a layer-dimension mismatch. Real-user impact is <0.2% (only one specific S value per 4096 triggers it; a retry with S+1 succeeds cleanly). The bench harness already works around it via per-case server restart. All ee7 numbers reported tonight are uncontaminated.

### Architectural finding — 67% of the drafter is dead weight in pflash mode

The profiling pass that produced the 128K ee7 per-stage decomposition also revealed something deeper: in pflash mode the drafter is not doing language modeling. It is doing positional scoring using only Q@K^T. That means FFN weights (~130 MB across 7 active layers), the LM output head (~310 MB), V projection (~14 MB), O projection (~7 MB), and layers 8–28 (~570 MB with ee7) are all loading into VRAM and never executing. Combined: ~1.0 GB of the 1.5 GB drafter file is dead weight on GPU. A purpose-built scoring drafter would be ~250 MB — the size of a 2-layer attention-only transformer with the right head count. This is the architectural insight that motivates a future drafter redesign: pflash wants a scoring model class, not a general-purpose small LM. The Qwen3-0.6B-BF16 is the right tool available today; this finding does not block any near-term work. It is the moat-deepening finding — the custom drafter would be simultaneously lighter and faster on any hardware.


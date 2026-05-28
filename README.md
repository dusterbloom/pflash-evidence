# PFlash Evidence — PR #274

Evidence dossier for PR #274 (adaptive composition: PFlash + DFlash).
Branch `feat/pflash-drafter-ee7` @ commit `5eede9c`, RTX 3090, 2026-05-28.

## Headline numbers

- **−39% drafter wall at 128K** (4.57 s vs ee7 prior 7.48 s)
- **11× end-to-end wall** on HotpotQA (6.2 s vs 73.1 s uncompressed)
- **5860 tok/s prefill** (compressed, median N=10)
- **+24% tok/J** composition vs pflash-only at 32K (NIAH 0/3 → 3/3)
- **Zero asserts / crashes** across 5 bench axes

## Files

| File | Purpose |
|------|---------|
| `index.html` | Single-page evidence site (open locally in browser) |
| `EVIDENCE.md` | Developer appendix: all numbers, sources, epistemic tags |
| `OPEN_QUESTIONS.md` | Open questions tracker (P2-J MED regression, P2-K gate break-even) |
| `journey.md` | Historical narrative (prior to PR #274) |

## Bench data

`lucebox-hub/bench/2026-05-28_adaptive_stack/` — axis_A through axis_E JSON results + server logs.

## Links

- [lucebox-hub on GitHub](https://github.com/Luce-Org/lucebox-hub)
- [PR #274](https://github.com/Luce-Org/lucebox-hub/pull/274)
- [PFlash blog](https://lucebox.com/blog/pflash)
- [Prior evidence backup](https://github.com/Luce-Org/lucebox-hub/tree/backup/pre-redesign-2026-05-28) — tag `backup-2026-05-28-pre-redesign`
- [Discord](https://discord.gg/yHfswqZmJQ)

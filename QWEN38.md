# Strata on a 16 GB card for an agent workload — notes, patches and measurements

*Alessandro Patassini, October 2026.* Work done with Claude (Anthropic) as the coding assistant; the direction, the hypotheses
and the many dead ends are mine.

This fork of [Strata](https://github.com/Niko1221/Strata) collects what I changed and measured while running
**Qwen3.8-Flash-Next (GSQ-RCO Q2_0, 256K context)** every day as the model behind a coding agent (LLM_coder), a mail/project
assistant (WorkFlow) and Codex. Everything is **opt-in by environment variable** unless written otherwise; without the
variables the engine behaves as upstream.

## Hardware and setup
- RTX 5070 Ti 16 GB (the model only, no monitor) + RTX 4060 Ti 8 GB (monitors and the vision encoder), Windows 11.
- Ryzen 9 7950X, 128 GB DDR5 (4 x 32 GB at 3600-3800 MT/s, 42-44 GB/s measured read).
- ~7,000 of the 24,576 experts fit in VRAM (29%); with the router push (`STRATA_ROUTE_PRIOR=1.0`) ~98% of the decode picks hit VRAM.

## Settings that matter (no code: upstream options and tools, measured here)
| setting | measured on this machine |
|---|---|
| **router push** `STRATA_ROUTE_PRIOR=1.0` (Strata's: +1.0 to the logit of experts already in VRAM, for the selection only) | **+23%** decode (160.3 vs 130.1 tok/s), 0.5 gives +17%. **Quality checked**: two real tasks (an invoicing module with 25 hidden tests and a graded rubric, a time-tracking task with 27 hidden tests), 3 runs with and 3 without: invoices 25/25 in 6 of 6 answers, rubric 95.5/97/97 with vs 96.5/97.5/97.5 without, time tracking no worse (the same model limit both ways): **no measurable loss**. Stronger pushes not used on purpose. |
| **draft vocabulary in your own language**: Strata's `tools/draft_vocab.py --base data/draft_vocab.bin --corpus <your real answers> --coverage 0.99` | the default list (English, code, CJK) missed 8.5% of the words my agent writes (Italian); +4,253 words → 0.45% missed, accepted drafts 80.5 → 86.9% (code) and 66.1 → 73.4% (agent): **+6% code, +10% agent**. Redo it with the language and the kind of text of your own work. |
| `--adapt-swaps 192` (default 96) | agent steps 30.2 vs 34.6 ms |
| `--vram-reserve-mib 50` (default 700) on a card without monitor | 7,012 expert slots instead of 6,519, +2% |
| `STRATA_MMVQ_IL_ROWS` with the sm_86 Q3_K rows on sm_120 | +3% |
| `STRATA_GDN_SPLIT=1` + `--mtp-window 16384` + `--mtp-q4 head` | -5% ms per word |

## What helped (in daily use)
| change | branch | variable | measured |
|---|---|---|---|
| **per-client expert memory**: when a program comes back after another one filled the cache, its expert counts are restored and one big adaptation runs | `apatassini/client-memory` | `STRATA_CLIENT_MEMORY=2048` | agent step right after another program **+21%** tok/s; cache already adapted +11% |
| **short prompt reads**: the decode router push also in short prefill chunks, and resident experts used by few rows computed straight from the compressed blob with the decode grouped kernel instead of one MMQ conversion launch per expert | `apatassini/short-prompt-reads` | `STRATA_PF_ROUTE_PRIOR=1.0` (`STRATA_PF_DIRETTO`, `STRATA_PF_DIRETTO_MAX`) | 177 tokens 473 → 315 ms, ~500 668 → 470 ms, ~900 826 → 677 ms; in real agent traffic 300-1000-token reads **-37/-41%**. The read push changes outputs a little like the decode push; so far only sanity-checked, no separate quality run yet |
| **faster start**: arena copy with 14 threads, vision encoder started in parallel when it lives on another GPU | `apatassini/arena-load-threads`, `apatassini/vision-parallel-start` | `STRATA_ARENA_THREADS` | cold start 21 → 14 s; "restart" 34 → 16 s (with a shorter wait in my start script) |
| **own MTP draft head** trained on my long agent conversations (16K-token context, the model's own probabilities) | method below | — | +4-5% on the same work vs the Basalt head |

Why short reads matter: in a real 5-hour agent session (526 model calls) **74% of the prompt reads were under 1,000 tokens and
took 57% of the read time**, at 200-750 tok/s instead of 3,000+.

## What did not help (measured, so you don't have to)
| idea | result |
|---|---|
| copying predicted experts into VRAM ahead of time (router of the next layer, trained heads 1-2 layers ahead, a yes/no head with calibrated threshold: 75% useful copies) | at best **even** on a long run: the CPU saves ~0.26 ms per step, the GPU spends ~0.2 ms more. A layer waits for the CPU if **one** expert is missing; with ~45 µs per missing layer and a head that cleans ~27% of those layers, the gain is eaten by the GPU cost |
| a learned cache rule (weighted decayed counts) instead of Strata's | at equal swaps it does not beat Strata (4.05% vs 3.93% missing picks); it wins only swapping twice as much, which costs more in the real engine |
| `STRATA_DOORBELL_STORE`, `STRATA_FUSE_WAIT` (the AMD options) on CUDA | -3% and -6.7% on an RTX 5070 Ti |
| reading short prompts through decode windows (`--short-read`) | each extra token in the window costs ~3 ms: worse than prefill |
| where the per-layer "plan" wait goes | the host publishes the plan ~2 µs after the doorbell; the wait is the CPU computing the missing experts (~41 µs per expert, at the RAM bandwidth limit) |
| router push stronger than 1.0 | not pursued: quality first |

The full log (in Italian) of every test, with numbers, is in my notes; ask if you want details on one of them.

## The MTP draft head (method, not weights)
The draft head I use was retrained on **my own** long agent conversations, so I don't publish the weights. What made the
difference was the **context**: the engine gives the head the last 16,384 tokens, while it had been trained on ~640.
Recipe: record the head's inputs in the engine (a recorder patch), train on whole requests and whole conversations
(long-context attention in chunks to fit 16 GB), one pass (a second one memorises), with the model's own probabilities as
soft targets. I also use a draft vocabulary that includes Italian words; **anyone can do the same with the language and the
vocabulary of their own work**. The recorder patch is in the `qwen38` branch (`STRATA_MTP_DUMP`).

## Branches
- `apatassini/*`: one improvement each, on top of upstream `296b59d` (0.1.44.1), meant as pull requests.
- `qwen38`: everything I run, on top of 0.1.43 (`7695859`), including the opt-in experiments above and the recorders
  (`STRATA_MTP_DUMP`, `STRATA_ROUTE_DUMP`).

License: MIT, as upstream.

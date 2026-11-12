# Local inference benchmarks: Strata, Project Maya, llama.cpp

Optimized setups (Strata, Project Maya, Maya-based custom engines, tuned llama.cpp presets) against an unoptimized
baseline, mostly stock llama.cpp, on one RTX 5090 + 192 GB box. Plus a 25-task hard test at each model's own settings,
and what a second card (an RTX 3090, added 2026-11-11) buys each engine. Measured 2026-10-29 to 2026-11-12.

## System

| | |
|---|---|
| CPU | Ryzen 9 9950X (16C/32T) |
| Board | ASUS ProArt X870E-Creator WiFi |
| GPU | RTX 5090 FE 32 GB, 450 W limit, driver 617.42 |
| Second GPU | RTX 3090 24 GB, 320 W limit, since 2026-11-11. Used only in the "Second GPU" section |
| PCIe | 5090: Gen5 **x8**, 28.7 GB/s H2D pinned. 3090: Gen4 x8, 13.3 GB/s |
| RAM | 192 GB DDR5 (2x32 + 2x64), rated 5600, **running 3600 MT/s**. The CPU expert lanes see 37-50 GB/s |
| OS / stack | Windows 11 Pro 26300, CUDA 13.4, llama.cpp b12017, Strata 0.2.10 (engine 0.1.56), Project Maya v1.1.52 |

The big MoEs here are RAM-bandwidth-bound on decode and PCIe-bound on prefill, so expect higher numbers on DDR5-5600+
and x16.

## Throughput: baseline vs optimized

`ab.py` (greedy, thinking off): decode on six unseen prompts, prefill and TTFT on an unseen ~33K-token prompt. Runs on
expert-caching engines start from the same saved expert profile, 3+ runs each. Rows marked † are decode with thinking on
at the model's own sampling (`spec-sweep.py`), where drafting matters.

| Model, quant | Setup | Decode tok/s | Decode after 33K | Prefill tok/s | TTFT at 33K |
|---|---|---|---|---|---|
| **Qwen3.8-Flash-Next** UD-Q4_K_XL | llama.cpp router (no drafting) | 28 | | ~480 | |
| | Strata 0.2.10, same quant, DFlash2 | 116 | | | |
| IQ3_S (ISTA-DASLab) | Strata 0.1.54, plain | 190 | 190 | 5,320 | |
| | **Strata 0.2.10, DFlash2 + `adapt-every=1 adapt-swaps=80 adapt-decay=0.97`** | **225-229** | 201-217 | ~5,700 | |
| **Inkling-Small** UD-Q4_K_XL | llama.cpp PR #25731, `--cpu-moe` | 9.1 | 9.6 | 72 | 441 s |
| | llama.cpp PR #25731, 14 GB expert cache, `-ub 2048` | 9.0 | 9.7 | 207 | 154 s |
| | **maya-ink** | **30.2** | 26.3 | 794 | 40 s |
| **Nemotron 3.5 Super** UD-Q4_K_XL | llama.cpp, `--cpu-moe` | 11.2 | 11.4 | 107 | 319 s |
| | llama.cpp, own fit, `-ub 2048` | 13.0 | 13.5 | 474 | 72 s |
| | **maya-nem** | **26.3** | 21.7 | 1,607 | 21 s |
| UD-Q3_K_XL | maya-nem | 37.7 | 30.6 | 1,981 | 17 s |
| | **maya-nem + AVX2 Q2_0 rows** | **41.3** | 34.1 | 1,973 | 17 s |
| **Ling-3.1-flash** IQ2_XS | llama.cpp, `--cpu-moe` | 12.0 | 12.5 | 75.6 | 470 s |
| | llama.cpp, own fit | 12.0 | 12.8 | 86.5 | 410 s |
| | **maya-ling, 16K chunks, `LING_PF_PREFETCH=1`** | **22.4** | 16.2 | 876 | 41 s |
| **DeepSeek-V4.1-Flash Coder** IQ2_XXS | llama.cpp, own `--fit`, 16K | 21.7 | | | |
| | llama.cpp, `--cpu-moe` + 22 GB `moe-cache`, q8_0 KV, 64K | 29.8 | 29.7 | 472 | |
| | Strata 0.2.10, 256K | 30.0-33.8 | 27.8-29.0 | 1,526-2,483 | |
| | Project Maya (ds4 engine), 1M | 33.5-34.8 | 30.9 | 3,200-3,357 | |
| | **the same on both cards: the 3090 as Maya's expert helper, DeepSeek's DSpark drafter with 32 experts a block in VRAM** | **66** | **60-65** | 3,120-3,165 | 10.6 s |
| **GLM-5.3-Flash** | Project Maya, Maya-M (exact) | 22.8 | | ~2,650 | |
| | **Project Maya, Maya-S24 with expert skips (KL 0.052)** | **38.4** | 36.6 | 2,948 | |
| **MiMo-V2.6-Flash-RL** MXFP4 | strata-mimo `268fdbc`, stock | 14.9 | | ~950 | |
| | **strata-mimo + local fixes + `MIMO_TAIL_KEEP=5` (KL 0.037)** | **25.5** | 21 | ~1,030 | |
| **MiMo-V2.6-Flash-MOPD** MXFP4 | strata-mimo, same setup as RL | **25.7** | 21.1 | 1,034 | 32 s |
| | **both cards (helper card), pinned copy of 34 of 47 expert layers, CPU lane on short prompts, 32K slices** | **34-36** | 27-29 | **2,810** | **11.9 s** |
| **Qwen3.8-27B** Q6_K † | llama.cpp, no drafting | 64 | | | |
| | **llama.cpp, MTP, 2 drafts** | **99** | | 2,630 | |
| **Gemma 4 31B** QAT Q4 † | llama.cpp, no drafter | 72 | | | |
| | **llama.cpp, MTP drafter, 4 drafts** | **170** | 104 at 38K | ~2,550 | |
| **Gemma 4 26B-A4B** Q8_0 † | llama.cpp, no drafter | 202 | | | |
| | **llama.cpp, MTP drafter** | **311** | 222 at 38K | ~8,000 | |
| **Nanbeige4.3-33B** Q4_K_M | llama.cpp, q8_0 KV, 80K (`ngram-mod` ~1% on new text, 14x on repeats) | 69 | 38 at 39K | ~2,050 | |

DeepSeek and MiMo on both cards (2026-11-12): [`docs/two-gpu.md`](docs/two-gpu.md). Greedy decode on Maya's DeepSeek
engine isn't reproducible run to run, and code drafts far better than prose (90-115 against 40-60 tok/s): single
`ab.py` runs gave 55-70.

Engine notes:
- **maya-ink, maya-nem, maya-ling** are engines in a Project Maya fork (branches `inkling`, `nemotron`, `ling`). They
  share one design:
  - layer-at-a-time chunked prefill (8-16K tokens, one PCIe crossing per expert per chunk, own pinned staging);
  - an adaptive VRAM expert cache (decay 0.99/token, 16 swaps every 2 tokens) running beside a CPU expert lane;
  - per-layer CUDA graphs.
- About 65-80% of each token is still the CPU lane at RAM speed.
- MTP on maya-nem was exact, but 22-31% slower than without it. Each drafted token still reads its own experts from
  RAM, and the head takes about 4 GB of the expert cache.
- Below about 32 GB of VRAM in use everything holds. Above it, the driver pages VRAM to system RAM and throughput
  collapses (for example, llama.cpp `-ngl 99` without `--cpu-moe` on Nemotron: 3 tok/s).

## Second GPU

An RTX 3090 beside the 5090, 2026-11-11. `ab.py` decode on new prompts and right after the ~33K prompt, the 5090 alone
against both cards, same build. Details, per-engine tables and checks: [`docs/two-gpu.md`](docs/two-gpu.md).

| Model, quant | Engine | Decode: 5090 alone | Both cards | After 33K: alone | Both | Second card's role |
|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | Strata 0.2.10 | **227** | 193 | | | Layer split |
| GLM-5.3-Flash Maya-S24 | Project Maya | 36-38 | **52.6** | | | Layer split, MTP on |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | Maya (ds4) | 33.1 | 47.5 | | | Layer split |
| | Maya (ds4), fork build | 36.5 | **66** | 32.3 | **60-65** | Maya's expert helper + DSpark drafter (2026-11-12) |
| Nemotron 3.5 Super UD-Q3_K_XL | llama.cpp | 19.4 | 23.0 | | 25.7 | Device split |
| | maya-nem | 41.0 | **48.2** | 33.9 | **43.8** | Helper card |
| Inkling-Small UD-Q4_K_XL | maya-ink | 29.6 | **36.6** | 27.0 | 28.0 | Helper card |
| Ling-3.1-flash IQ2_XS | maya-ling | 21.2 | **24.7** | 15.2 | 15.0 | Helper card |
| MiMo-V2.6-Flash-MOPD MXFP4 | strata-mimo, patched | 24.5 | **36.0** | 20.0 | **29.0** | Helper card |

- **Helper card:** the 3090 holds routed experts only, in a second expert cache. The 5090 still runs the whole model;
  a token's experts that it doesn't hold are split between the 3090 and the CPU lane. No model state leaves the first
  card, so prefill is unchanged (Ling excepted: 27% slower with two CUDA devices visible, cause open).
- **Layer splits cost prefill:** GLM and DeepSeek read prompts 40-47% slower on two cards. Flash-Next loses decode
  outright and stays on the 5090.
- **DeepSeek (2026-11-12)** runs on Maya's own expert helper instead of the layer split, which keeps its prefill, with
  DeepSeek's DSpark drafter on top: 36.5 to 66 tok/s on new prompts, 48 to 110 on a repeated prompt. The helper needed
  two fixes to answer on Windows.
- **MiMo's prompt path (2026-11-12):** a pinned copy of the 34 expert layers the commit limit allows, the CPU lane
  computing a short prompt's rarely picked experts, 32K slices. A 33K prompt reads in 11.9 s instead of 31.7, and an
  agent turn's first token comes after 3.5-5 s instead of 10. Decode is unchanged.
- **MiMo gains most** (+47%) because its CPU share was the largest, and its answers move closer to the exact engine:
  the helper's experts are exempt from `MIMO_TAIL_KEEP` (KL 0.031 to 0.019).
- Outputs checked on every helper build: top-1 against llama.cpp 87/88, 86/88 and 90/90 on maya-nem, maya-ink and
  maya-ling, `derail.py` 0/20 on all four.

## Hard test

25 seeded tasks (traces, long multiplication, logic grids, routing and knapsack, constrained writing, a 200-row table, a
52K-token retrieval chain, 7 programs against hidden tests, a 7-file tool-driven repair). Each model ran at its card's
sampling and top reasoning effort, with thinking caps off and as much output as the box allows. Details and per-task
results: [`docs/hard-test.md`](docs/hard-test.md).

| Model | Score /25 | Coding /7 | Wall time | Tokens | Avg decode |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | 22 | 4 | 51 min | 711K | 233 tok/s |
| Inkling-Small UD-Q4_K_XL | 21 | 3 | 279 min | 671K | 40 tok/s |
| Ling-3.1-flash IQ2_XS | 21 (2 cut off at 128K ctx) | 3 | 744 min | 992K | 22 tok/s |
| Qwen3.8-Flash-Next UD-Q4_K_XL | 21 | 3 | 92 min | 830K | 150 tok/s |
| Qwen3.8-27B Q6_K | 20 | 4 | 130 min | 788K | 101 tok/s |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | 19 | 2 | 298 min | 1,132K | 63 tok/s |
| Gemma 4 26B-A4B Q8_0 | 18 | 2 | 22 min | 458K | 349 tok/s |
| Nanbeige4.3-33B Q4_K_M (96K ctx) | 18 (19 lenient) | 3 | 412 min | 1,034K | 42 tok/s |
| Gemma 4 31B QAT | 17 (18 lenient) | 1 | 18 min | 201K | 185 tok/s |
| Nemotron 3.5 Super UD-Q3_K_XL | 17 (18 lenient) | 0 | 195 min | 472K | 40 tok/s |
| MiMo-V2.6-Flash-MOPD MXFP4 | 16 (17 lenient) | 3 | 465 min | 890K | 32 tok/s |
| MiMo-V2.6-Flash-RL MXFP4 | 16 | 1 | 580 min | 1,098K | 32 tok/s |
| GLM-5.3-Flash Maya-S24, `max`, 131K | 13 | 0 | 782 min | 1,840K | 39 tok/s |
| GLM-5.3-Flash, best per task over `max` / `high` / `low` re-runs | 18.9 | 2 | +17 h | | |

- **Budget:** at a flat 24K budget, GLM, Nanbeige and MiMo ran out on every coding task. With room, Nanbeige went from 9.8 to 18, and MiMo from 12.9 to 16.
- **GLM effort:** GLM at `max` thinks past any budget; 10 of its 25 answers hit 131K. `high` and `low` with 262K solved five of those, plus the repair job, so for hard work use `high`, not GLM's default.
- **MiMo checkpoints:** MiMo's MOPD checkpoint scores the same as RL, but codes better: 3 of 7 against 1. It runs at the same speed and writes 19% fewer tokens.

How each model lost its points. Per miss, with patterns across models: [`docs/failure-modes.md`](docs/failure-modes.md).

| Model | Misses | Mostly |
|---|---|---|
| Flash-Next IQ3_S | 3 | Runs of zeros that Strata's repeat guard stops early; one slow program |
| Inkling-Small | 4 | Code only: two too slow, two bugs |
| Ling-3.1-flash | 4 | Code only: two still thinking at the 128K context, two bugs |
| Flash-Next UD-Q4_K_XL | 4 | Code only: bugs, a slow program, a loop |
| Qwen3.8-27B | 5 | Two loops, two coding bugs, a trace slip |
| DeepSeek Coder | 6 | Coding bugs, a counting loop to 384K, an answer that stops mid-function |
| Gemma 4 26B-A4B | 7 | Short coding answers with plain bugs; slips in long output (a dropped digit, a 6-letter word) |
| Nanbeige4.3-33B | 7 | Coding bugs, a word-list loop, 35 wrong rows in the long table, format slips |
| Gemma 4 31B | 8 | Coding bugs at ~8K tokens an answer; estimated the 20-digit product instead of multiplying |
| Nemotron 3.5 Super Q3 | 8 | Every coding answer ends early, with a bug or a slow algorithm |
| MiMo MOPD | 9 | Careless short answers (traces, a word count, a primality check), coding bugs |
| MiMo RL | 9 | Runs out of room enumerating by hand (3), coding bugs |
| GLM-5.3-Flash `max` | 12 | Runs out of room (10 of 25); the repair job stalls |

## Files

| | |
|---|---|
| [`docs/models.md`](docs/models.md) | Per model: architecture, quant choice, launch settings, env, speeds, memory, correctness vs llama.cpp, quirks |
| [`docs/hard-test.md`](docs/hard-test.md) | Tasks, scoring, each model's settings, per-task matrix |
| [`docs/failure-modes.md`](docs/failure-modes.md) | Every miss by type and cause, patterns across models, tool use in the repair job, the 24K budget |
| [`docs/two-gpu.md`](docs/two-gpu.md) | RTX 3090 as a second card: per-engine one-card vs two-card tables, the helper-card design, checks, DeepSeek's expert helper and drafter, MiMo's prompt path, the 3090's power sweep |
| [`docs/pinned-memory.md`](docs/pinned-memory.md) | Pinned host memory on Windows with two cards: copy rates, the commit limit, pinning a file mapping in place |
| [`docs/measuring.md`](docs/measuring.md) | Measurement method (`ab.py`, `spec-sweep.py`, `turns.py`) and pitfalls |
| [`results/hard-test-tasks.csv`](results/hard-test-tasks.csv) | Per-task verdicts, tokens and times for every run |

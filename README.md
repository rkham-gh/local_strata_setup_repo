# Local inference benchmarks: Strata, Project Maya, llama.cpp

Optimized setups (Strata, Project Maya, Maya-based custom engines, tuned llama.cpp presets) against an unoptimized
baseline, mostly stock llama.cpp, on one RTX 5090 + 192 GB box. Plus a 25-task hard test at each model's own settings.
Measured 2026-10-29 to 2026-11-06.

## System

| | |
|---|---|
| CPU | Ryzen 9 9950X (16C/32T) |
| Board | ASUS ProArt X870E-Creator WiFi |
| GPU | RTX 5090 FE 32 GB, 450 W limit, driver 617.42 |
| PCIe | Gen5 **x8** (lanes shared with the M.2 slot), 28.7 GB/s H2D pinned |
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
| | **Project Maya (ds4 engine), 1M** | **33.5-34.8** | 30.9 | 3,200-3,357 | |
| **GLM-5.3-Flash** | Project Maya, Maya-M (exact) | 22.8 | | ~2,650 | |
| | **Project Maya, Maya-S24 with expert skips (KL 0.052)** | **38.4** | 36.6 | 2,948 | |
| **MiMo-V2.6-Flash-RL** MXFP4 | strata-mimo `268fdbc`, stock | 14.9 | | ~950 | |
| | **strata-mimo + local fixes + `MIMO_TAIL_KEEP=5` (KL 0.037)** | **25.5** | 21 | ~1,030 | |
| **Qwen3.8-27B** Q6_K † | llama.cpp, no drafting | 64 | | | |
| | **llama.cpp, MTP, 2 drafts** | **99** | | 2,630 | |
| **Gemma 4 31B** QAT Q4 † | llama.cpp, no drafter | 72 | | | |
| | **llama.cpp, MTP drafter, 4 drafts** | **170** | 104 at 38K | ~2,550 | |
| **Gemma 4 26B-A4B** Q8_0 † | llama.cpp, no drafter | 202 | | | |
| | **llama.cpp, MTP drafter** | **311** | 222 at 38K | ~8,000 | |
| **Nanbeige4.3-33B** Q4_K_M | llama.cpp, q8_0 KV, 80K (`ngram-mod` ~1% on new text, 14x on repeats) | 69 | 38 at 39K | ~2,050 | |

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
| GLM-5.3-Flash Maya-S24 | running (10 at a 24K budget) | | | | |
| MiMo-V2.6-Flash-RL | queued (12.9 at a 24K budget) | | | | |

At a flat 24K budget, GLM, Nanbeige and MiMo ran out on every coding task. With room, Nanbeige went from 9.8 to 18.

## Files

| | |
|---|---|
| [`docs/models.md`](docs/models.md) | Per model: architecture, quant choice, launch settings, env, speeds, memory, correctness vs llama.cpp, quirks |
| [`docs/hard-test.md`](docs/hard-test.md) | Tasks, scoring, each model's settings, per-task matrix, failure notes |
| [`docs/measuring.md`](docs/measuring.md) | Measurement method and pitfalls |
| [`results/hard-test-tasks.csv`](results/hard-test-tasks.csv) | Per-task verdicts, tokens and times for every run |

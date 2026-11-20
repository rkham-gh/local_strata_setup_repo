# Local inference benchmarks: Strata, Project Maya, llama.cpp

Optimized setups (Strata, Project Maya, Maya-based custom engines, tuned llama.cpp presets) against an unoptimized
baseline, mostly stock llama.cpp, on one RTX 5090 + 192 GB box. Plus a 25-task hard test at each model's own settings,
and what a second card (an RTX 3090, added 2026-11-11) buys each engine. Measured 2026-10-29 to 2026-11-15.

**RAM speed:** DDR5-3600 until 2026-11-15, then 4400, which failed y-cruncher the same day, and DDR5-4200 since. Every
number here is a DDR5-3600 figure unless it says 4400 or 4200;
[RAM speed](#ram-speed-ddr5-4400-then-4200-2026-11-15) has each model at all three.

## System

| | |
|---|---|
| CPU | Ryzen 9 9950X (16C/32T) |
| Board | ASUS ProArt X870E-Creator WiFi |
| GPU | RTX 5090 FE 32 GB, 450 W limit, driver 617.42 |
| Second GPU | RTX 3090 24 GB, 320 W limit, since 2026-11-11. Rows that use it say so |
| PCIe | 5090: Gen5 **x8**, 28.7 GB/s H2D pinned. 3090: Gen4 x8, 13.3 GB/s |
| RAM | 192 GB DDR5 (2x32 + 2x64), rated 5600, no ECC. **4200 MT/s since 2026-11-15** (4400 failed a stress test); 3600 before, AMD's rating for four dual-rank sticks. At 3600 the CPU expert lanes saw 37-50 GB/s |
| OS / stack | Windows 11 Pro 26300, CUDA 13.4, llama.cpp b12017 (Ripple-65B: b12314, CUDA 13.5), Strata 0.2.10 (engine 0.1.56), Project Maya v1.1.52 (v1.1.60 since 2026-11-11) |

The big MoEs here are RAM-bandwidth-bound on decode and PCIe-bound on prefill, so expect higher numbers on DDR5-5600+
and x16. 3600 to 4400 made the ones that compute experts from RAM 4-15% faster on decode; 4200 keeps all but 0-3% of
that.

## Throughput: baseline vs optimized

`ab.py` (greedy, thinking off) at DDR5-3600: decode on six unseen prompts, prefill and TTFT on an unseen ~33K-token
prompt. Runs on expert-caching engines start from the same saved expert profile, 3+ runs each. Rows marked † are decode
with thinking on at the model's own sampling (`spec-sweep.py`), where drafting matters.

| Model, quant | Setup | Decode tok/s | Decode after 33K | Prefill tok/s | TTFT at 33K |
|---|---|---|---|---|---|
| **Qwen3.8-Flash-Next** UD-Q4_K_XL | llama.cpp router (no drafting) | 28 | | ~480 | |
| | Strata 0.2.10, same quant, DFlash2 | 116 | | | |
| IQ3_S (ISTA-DASLab) | Strata 0.1.54, plain | 190 | 190 | 5,320 | |
| | **Strata 0.2.10, DFlash2 + `adapt-every=1 adapt-swaps=80 adapt-decay=0.97`** | **225-229** | 201-217 | ~5,700 | |
| **Inkling-Small** UD-Q4_K_XL | llama.cpp PR #25731, `--cpu-moe` | 9.1 | 9.6 | 72 | 441 s |
| | llama.cpp PR #25731, 14 GB expert cache, `-ub 2048` | 9.0 | 9.7 | 207 | 154 s |
| | **maya-ink** | **30.2** | 26.3 | 794 | 40 s |
| **Inkling 2 Small** own 2-bit, IQ2_XXS / XS / S experts, 169.9 GB (long prompt 41K) | own llama.cpp port on PR #25731, experts in RAM, `-ub 512` | 8.1 | 8.1 | 59 | 698 s |
| | maya-ink port, the 5090 alone, first build | 19.8 | 13.4 | 601 | 69 s |
| | **maya-ink port, both cards: 3090 kept clocked, `INK_TAIL_KEEP=5`, 16K chunks, `INK_CACHE_PROMPT=2`** | **32.5** | 27.0 | 836 | 50 s |
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
| | llama.cpp, MTP, 2 drafts | 99-104 | | 2,630 | |
| | **llama.cpp, z-lab DFlash2, 7 drafts, `--spec-draft-sampling probabilistic`** | **126-134** | | 2,374 | |
| **Gemma 4 31B** QAT Q4 † | llama.cpp, no drafter | 72 | | | |
| | llama.cpp, MTP drafter, 4 drafts | 168-170 | 104 at 38K | ~2,550 | |
| | **the same, sampled drafts** | **174** | | | |
| **Gemma 4 26B-A4B** Q8_0 † | llama.cpp, no drafter | 202 | | | |
| | llama.cpp, MTP drafter | 297-311 | 222 at 38K | ~8,000 | |
| | **the same, sampled drafts** | **312-315** | | | |
| **Nanbeige4.3-33B** Q4_K_M | llama.cpp, q8_0 KV, 80K (`ngram-mod` ~1% on new text, 14x on repeats) | 69 | 38 at 39K | ~2,050 | |
| **Ripple-65B Preview** AD-IQ4_XS (dense, both cards) | llama.cpp b12314, layer split (default) | 26.6 | 22.7 | 1,313 | |
| | llama.cpp b12314, `-sm tensor` 65:35 | 40.8 | 35.2 | 1,091 | |
| | **b12314 + #31904, layer split 65:35, DFlash2 drafter on the 3090** | **50.5** | **44.8** | 1,331 | 24 s |

Sampled drafts (2026-11-12): llama.cpp's `--spec-draft-sampling probabilistic` samples the draft and verifies it by
rejection sampling, exact in distribution. At these models' temperature 1.0 it accepts far more of an MTP head's or a
DFlash2 drafter's tokens than greedy drafting; it was what made z-lab's DFlash2 drafter pay on Qwen3.8-27B (greedy:
108 against MTP's 104). It doesn't apply to plain DFlash drafters, and Gemma 4's (z-lab's) lost to its MTP heads.

Strata 0.2.13 (2026-11-12): on Windows `strata update` moves only the engines (to 0.1.59); Control stays at 0.1.56.
On Flash-Next IQ3_S the new engine decodes the same (224.0 tok/s on new prompts over 4 runs, against 224.0 over 3) and
reads prompts 2-3% faster. 0.2.13's tuned DFlash2 drafter and window of 8 gained nothing on new prompts and
`--draft-conf 0.55` 2%, so both engines went back to 0.1.56. Details in [`docs/models.md`](docs/models.md), section 1.

DeepSeek and MiMo on both cards (2026-11-12): [`docs/two-gpu.md`](docs/two-gpu.md). Greedy decode on Maya's DeepSeek
engine isn't reproducible run to run, and code drafts far better than prose (90-115 against 40-60 tok/s): single
`ab.py` runs gave 55-70.

Engine notes:
- **maya-ink, maya-nem, maya-ling** are engines in a Project Maya fork (branches `inkling`, `inkling2`, `nemotron`, `ling`). They
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

An RTX 3090 beside the 5090, 2026-11-11 to 2026-11-13, DDR5-3600 (Flash-Next again at 4400 on 2026-11-15). `ab.py`
decode on new prompts and right after the ~33K prompt, the 5090 alone against both cards, same build. Details,
per-engine tables and checks: [`docs/two-gpu.md`](docs/two-gpu.md).

| Model, quant | Engine | Decode: 5090 alone | Both cards | After 33K: alone | Both | Second card's role |
|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | Strata 0.2.10 | **227** | 193 | | | Layer split |
| | Strata 0.2.10, DDR5-4400 (2026-11-15) | **233** | 145-207 | **211-214** | 131-194 | Peer tier, helper cache, plain helper cache |
| GLM-5.3-Flash Maya-S24 | Project Maya | 36-38 | 52.6 | | | Layer split, MTP on |
| | Maya v1.1.60, fork build (`glm-2gpu`) | 36.7 | **58-61** | 32.8 | **47.5-50.3** | Maya's expert helper, 3090 kept clocked (2026-11-13) |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | Maya (ds4) | 33.1 | 47.5 | | | Layer split |
| | Maya (ds4), fork build | 36.5 | **66** | 32.3 | **60-65** | Maya's expert helper + DSpark drafter (2026-11-12) |
| Nemotron 3.5 Super UD-Q3_K_XL | llama.cpp | 19.4 | 23.0 | | 25.7 | Device split |
| | maya-nem | 41.0 | **48.2** | 33.9 | **43.8** | Helper card |
| Inkling-Small UD-Q4_K_XL | maya-ink | 29.6 | **36.6** | 27.0 | 28.0 | Helper card |
| Inkling 2 Small own 2-bit (long prompt 41K) | maya-ink, `inkling2`, first build | 19.8 | 23.6 | 13.4 | 15.0 | Helper card |
| | the same, 3090 kept clocked + tail skip + prompt settings | | **32.5** | | **27.0** | Helper card |
| Ling-3.1-flash IQ2_XS | maya-ling | 21.2 | **24.7** | 15.2 | 15.0 | Helper card |
| MiMo-V2.6-Flash-MOPD MXFP4 | strata-mimo, patched | 24.5 | **36.0** | 20.0 | **29.0** | Helper card |
| Ripple-65B Preview AD-IQ4_XS | llama.cpp b12314 + #31904 | doesn't fit | **50.5** | | **44.8** | Layer split + its drafter (tensor split without one: 40.8) |

- **Helper card:** the 3090 holds routed experts only, in a second expert cache. The 5090 still runs the whole model;
  a token's experts that it doesn't hold are split between the 3090 and the CPU lane. No model state leaves the first
  card, so prefill is unchanged (Ling excepted: 27% slower with two CUDA devices visible, cause open).
- **Keep the 3090 clocked (2026-11-12).** A helper's work comes in bursts of tens of µs a layer, and the driver leaves
  the 3090 in P3/P5 (420-1,260 MHz, memory at 5,001 or 810 MHz). Inkling 2's expert GEMVs took 180-280 µs there
  against 15-23 on the 5090. A 50 ms sleeping kernel, launched again each token, holds it in P2. Since 2026-11-13
  every helper engine has it behind a switch: +5-12% on new prompts for 83-106 W more on the two cards while writing,
  so it's off by default except on GLM, where it makes the helper worth having (41.1 to 59.4 tok/s). Per engine:
  [`docs/two-gpu.md`](docs/two-gpu.md), "The 3090 itself".
- **Layer splits cost prefill:** GLM and DeepSeek read prompts 40-47% slower with Maya's layer split. Both moved to
  Maya's expert helper, which keeps one card's prefill (DeepSeek 2026-11-12, GLM 2026-11-13).
- **Flash-Next stays on the 5090 (why: 2026-11-15).** Every Strata two-card mode loses decode: at DDR5-4400, 233 tok/s
  alone against 204 on the peer tier, 207 on the helper cache and 145 on the plain helper cache. The 5090's adaptive
  cache already does 95.5% of the expert work from VRAM, and each layer the 3090 joins is a round trip through host
  memory that the 5090 waits for: the plain helper gave the 3090 3.8% of the work and still took a verify window from
  12.7 to 20.0 ms. Details: [`docs/two-gpu.md`](docs/two-gpu.md), "Flash-Next".
- **GLM (2026-11-13)** runs Maya's expert helper from a fork build: the 3090 holds 3,301 more experts (22 GB). With the
  3090 left in P5 (memory at 810 MHz) the helper gained 12%; kept clocked, 58-61 tok/s against 36.7 alone and 51.9 on
  the layer split, at one card's prefill.
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

## RAM speed: DDR5-4400, then 4200 (2026-11-15)

The kit went from DDR5-3600 to 4400 in the BIOS; higher settings gave an error. 1.10 V, no ECC. 4400 then failed
y-cruncher, and 4200 at the same DRAM voltage passed ([below](#stability-4400-fails-4200-passes)), so the box runs at
4200.
One `ab.py` run per model at 4400, each started exactly as its latest DDR5-3600 rows (same build, settings, cards and
saved expert profile), so the RAM is the only difference. 3600: the mean of those rows, their range in brackets. Helper
engines as served: the 3090 kept clocked on GLM only.

| Model | Weights in | New prompts, 3600 → 4400 tok/s | Change | After the ~33K prompt | Prefill, that prompt | Repeat, 3rd | Same greedy text |
|---|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S, Strata | 5090 + RAM | 224.0 (223.0-225.3) → **235.6** | **+5.2%** | 209.3 → 210.1 | 5,459 → 5,497 | 346 → 365 | varied at 3600 |
| GLM-5.3-Flash Maya-S24, expert helper | both cards + RAM | 58.5 (58.0-58.9) → **63.5** | **+8.6%** | 48.1 → 52.6 | 2,891 → 2,891 | 63.8 → 67.1 | varied at 3600 |
| DeepSeek-V4.1-Flash Coder IQ2_XXS, expert helper + DSpark | both cards + RAM | 69.7 → **74.2** | +6.5% | 65.8 → 69.9 | 1,879 → 2,364 ¹ | 114 → 122 | no; varies run to run |
| MiMo-V2.6-Flash-MOPD, 1st run after a start | both cards + RAM | 35.1 (34.6-35.6) → **40.5** | **+15.4%** | 28.1 → 32.0 | 2,594 → 2,656 | 47.5 → 54.6 | yes |
| MiMo, 2nd run | both cards + RAM | 36.3 (35.9-36.7) → **40.7** | **+12.1%** | 28.4 → 32.3 | 2,881 → 2,856 | 47.6 → 53.6 | varied at 3600 |
| Inkling-Small UD-Q4_K_XL | both cards + RAM | 37.0 (36.9-37.2) → **40.6** | **+9.6%** | 28.7 → 33.3 ³ | 727 → 821 | 47.8 → 50.8 | yes |
| Inkling 2 Small own 2-bit | both cards + RAM | 30.9 (30.4-31.3) → **33.5** | **+8.6%** | 26.5 → 29.0 | 844 → 931 | 34.9 → 36.4 | yes |
| Nemotron 3.5 Super UD-Q3_K_XL | both cards + RAM | 56.5 (56.4-56.5) → **58.6** | **+3.8%** | 57.5 → 58.5 | 1,656 → 1,738 | 58.0 → 59.7 | yes |
| Ripple-65B Preview AD-IQ4_XS | both cards | 50.5 → 51.2 | +1.4% | 44.8 → 46.7 | 1,331 → 1,365 | 71.0 → 71.6 | yes |
| Qwen3.8-27B Q6_K | 5090 | 165.7 → 162.6 | -1.9% | 110.4 → 99.6 ² | 2,374 → 2,480 | 205 → 205 | no |
| Nanbeige4.3-33B Q4_K_M | 5090 | 68.1 → 68.2 | +0.1% | 37.7 → 37.9 | 2,048 → 2,063 | 986 → 972 | yes |
| Gemma 4 26B-A4B Q8_0 | 5090 | 318.3 → 312.1 | -1.9% | 222.3 → 219.4 | 8,041 → 7,916 | 357 → 349 | yes |
| Gemma 4 31B QAT | 5090 | 183.7 → 183.0 | -0.4% | 103.7 → 102.7 | 2,539 → 2,542 | 196 → 196 | yes |

- **The engines that compute experts from RAM: +4-15% on new prompts** for 22% more bandwidth. MiMo most (the CPU
  still computes about a third of its routed experts), then Inkling-Small, Inkling 2 and GLM (9-10%), DeepSeek 6.5%
  (inside its ±10% run-to-run spread), Flash-Next 5%, Nemotron 4%.
- **Prefill:** +10-13% on both Inklings, whose prompt path copies experts from RAM, +5% on Nemotron, nothing measurable
  elsewhere. ¹ DeepSeek's isn't the RAM: its one 3600 row on the served config read slower than the test configs
  (~3,150).
- **Held in VRAM, no change** (-1.9% to +1.4%: noise). ² Qwen3.8-27B's drafter met a different text after the long
  prompt. ³ Inkling-Small's +16% there is inside its 3600 spread (24.6 and 32.7).
- **Flash-Next: +5% on new and repeated prompts, +0.4% after the long one.** A new chat starts with a VRAM cache not
  yet adapted to it, so more of its experts go to the CPU pool; after 35K tokens the cache holds what the answer needs.
  The pool's time per expert went from 59-63 to 46-50 µs.
- **Same greedy text** as at 3600 on 8 of 13 runs. The other five are engines whose text varied between 3600 runs
  already, or with one 3600 row (Qwen3.8-27B).
- Not run: Ling-3.1-flash, MiniMax-M3.1-Flash, GLM's exact Maya-M and Nemotron's UD-Q4_K_XL; their files are no longer
  on this PC.

### Stability: 4400 fails, 4200 passes

y-cruncher 0.8.7 stress test (Zen 5 binary), no engine running, stop on first error: each test alone for 2 min at
4 GiB, then FFTv4 + N63 + VT3 at 160 GiB for an hour. DRAM at 1.10 V at both speeds (VDDSOC and the rest not read).
WHEA logged nothing at either.

| | DDR5-4400 | DDR5-4200 |
|---|---|---|
| 2-min runs at 4 GiB | 2 of 8 failed, both in the first second (FFTv4, and the in-cache SVT) | 7 of 7 passed |
| FFTv4 + N63 + VT3 at 160 GiB | failed twice in N63, on both CCDs, the second after 15 clean minutes; 22 min in all | 60 min clean: 10 rounds, 29 tests |
| N63 runs at 160 GiB | 2 of 4 failed | 10 of 10 passed |
| Throughput at 160 GiB, FFTv4 / N63 / VT3 (10^9 bits/s) | 1.68 / 8.44 / 9.04 | 1.63 / 8.25 / 8.82 (2-3% lower) |

- **The benchmarks didn't show it.** At 4400 all 13 `ab.py` runs finished without a visible error, and 8 gave the same
  greedy text as at 3600 (the rest vary run to run or have one 3600 row), yet y-cruncher failed twice in 22 minutes.
  Without ECC, stress-test first.
- 4200 hasn't had a 12-hour run yet. 4400 at higher DRAM/VDDIO and VDDSOC is untried; by the numbers below it could buy
  at most ~3%.

### DDR5-4200

The same 13 runs, started the same way. New prompts, tok/s:

| Model | 3600 | 4400 | **4200** | vs 4400 | vs 3600 | After the ~33K prompt, 4400 → 4200 | Same text as 4400 |
|---|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S, Strata | 224.0 | 235.6 | **237.2** | +0.7% | +5.9% | 210.1 → 208.7 | no (varies) |
| GLM-5.3-Flash Maya-S24, expert helper | 58.5 | 63.5 | **64.1** | +0.9% | +9.7% | 52.6 → 53.9 | yes |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | 69.7 | 74.2 | **71.4** | -3.8% | +2.4% | 69.9 → 64.6 | no (varies) |
| MiMo-V2.6-Flash-MOPD, 1st run after a start | 35.1 | 40.5 | **39.2** | -3.2% | +11.7% | 32.0 → 30.3 | yes |
| MiMo, 2nd run | 36.3 | 40.7 | **40.8** | +0.2% | +12.4% | 32.3 → 30.8 | no (varies) |
| Inkling-Small UD-Q4_K_XL | 37.0 | 40.6 | **39.5** | -2.7% | +6.6% | 33.3 → 28.9 | yes |
| Inkling 2 Small own 2-bit | 30.9 | 33.5 | **32.6** | -2.7% | +5.7% | 29.0 → 28.7 | yes |
| Nemotron 3.5 Super UD-Q3_K_XL | 56.5 | 58.6 | **57.8** | -1.4% | +2.4% | 58.5 → 55.2 | yes |
| Ripple-65B, Qwen3.8-27B, Nanbeige, both Gemmas (VRAM only) | | | | -0.1% to +0.1% | -2.1% to +1.4% | | yes |

- **The engines that compute experts from RAM: 0-3% below 4400 on new prompts**, still +2-12% over 3600. Flash-Next and
  GLM lost nothing (+0.7%, +0.9%: noise). DeepSeek's -3.8% is inside its ±10% run-to-run spread. About what y-cruncher
  showed for a 4.5% lower clock.
- **After the long prompt** MiMo lost 5% in both runs (3600: 27.8-28.5). Nemotron's 55.2 is below its 3600 runs too
  (57.3-57.8), so likely noise; Inkling-Small's 28.9 is inside its 3600 spread (24.6-32.7).

## Hard test

25 seeded tasks (traces, long multiplication, logic grids, routing and knapsack, constrained writing, a 200-row table, a
52K-token retrieval chain, 7 programs against hidden tests, a 7-file tool-driven repair). Each model ran at its card's
sampling and top reasoning effort, with thinking caps off and as much output as the box allows. All at DDR5-3600 except Hy4 Flash (DDR5-4200).
Details and per-task results: [`docs/hard-test.md`](docs/hard-test.md).

| Model | Score /25 | Coding /7 | Wall time | Tokens | Avg decode |
|---|---|---|---|---|---|
| Hy4 Flash AngelSlim Q3_K (maya-hy4, both cards) | 24 | 6 | 68 min | 301K | 74 tok/s |
| Qwen3.8-Flash-Next IQ3_S | 22 | 4 | 51 min | 711K | 233 tok/s |
| Inkling 2 Small own 2-bit (IQ2_XXS mix) | 22 | 4 | 184 min | 481K | 44 tok/s |
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
- **Hy4 Flash** (2026-11-19) on maya-hy4, an engine written for it in the Project Maya fork: 24 of 25, the best score here, with 6 of 7 coding tasks, the fewest tokens of any run at 20 or more, in 68 minutes. Its one miss is a backreference bug in the regex engine. It's one run at temperature 0.9, so read it as close to the 22s. Flash-Next's later runs on Basalt with the UD-IQ3_XXS pack (22 strict, 23 lenient, 7 of 7 coding) aren't in this repo yet.
- **Inkling 2 Small**, converted here to 2 bits from the release's NVFP4, ties Flash-Next at 22 and is the first run to pass both JSON and SAT. Two of its three misses are one-line bugs, and it wrote the fewest tokens of any run at 20 or more.

How each model lost its points. Per miss, with patterns across models: [`docs/failure-modes.md`](docs/failure-modes.md).

| Model | Misses | Mostly |
|---|---|---|
| Hy4 Flash | 1 | Code only: off-by-one backreferences in the regex engine |
| Flash-Next IQ3_S | 3 | Runs of zeros that Strata's repeat guard stops early; one slow program |
| Inkling 2 Small | 3 | Code only: lazy quantifiers ruled out of scope, a `continue` in the wrong loop, a solver stuck on the classic worst-case puzzle |
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
| [`docs/two-gpu.md`](docs/two-gpu.md) | RTX 3090 as a second card: per-engine one-card vs two-card tables, the helper-card design, checks, why Flash-Next stays on one card, DeepSeek's and GLM's expert helpers, MiMo's prompt path, the 3090's power sweep and keep-alive |
| [`docs/pinned-memory.md`](docs/pinned-memory.md) | Pinned host memory on Windows with two cards: copy rates, the commit limit, pinning a file mapping in place |
| [`docs/measuring.md`](docs/measuring.md) | Measurement method (`ab.py`, `spec-sweep.py`, `turns.py`) and pitfalls |
| [`results/hard-test-tasks.csv`](results/hard-test-tasks.csv) | Per-task verdicts, tokens and times for every run |

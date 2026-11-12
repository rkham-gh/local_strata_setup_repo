# Second GPU: an RTX 3090 beside the RTX 5090

Measured 2026-11-11. The 3090 (24 GB, PCIe 4.0 x8) went in as GPU 1; the 5090 stays GPU 0 (PCIe 5.0 x8) and
`CUDA_DEVICE_ORDER=PCI_BUS_ID` keeps the numbering. Everything else is the system in [`../README.md`](../README.md).
Methods: [measuring.md](measuring.md).

## Summary

`ab.py`, decode on new prompts, the 5090 alone against both cards. Same build and settings in both columns except where
noted.

| Model, quant | Engine | 5090 alone | Both cards | What the second card does |
|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | Strata 0.2.10 | **227** | 193 | Strata's layer split. Every token crosses to the 3090 and back |
| GLM-5.3-Flash Maya-S24 | Project Maya v1.1.52 | 36-38 | **52.6** | Maya's layer split; MTP drafting comes on with two cards |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | Maya (ds4) | 33.1 | **47.5** | Maya's layer split |
| MiniMax-M3.1-Flash Maya-S | Maya (mm3) | 12.7 | **16.9** | Maya's layer split |
| Nemotron 3.5 Super UD-Q3_K_XL | llama.cpp b12017 | 19.4 | 23.0 | llama.cpp's device split |
| | maya-nem | 41.0 | **48.2** | Helper card |
| Inkling-Small UD-Q4_K_XL | maya-ink | 29.6 | **36.6** | Helper card |
| Ling-3.1-flash IQ2_XS | maya-ling | 21.2 | **24.7** | Helper card |
| MiMo-V2.6-Flash-MOPD MXFP4 | strata-mimo, patched | 24.5 | **36.0** | Helper card |

- **Flash-Next gets slower.** It already fits the 5090 well enough to decode 227 tok/s. Strata's Windows build has
  only the layer split (the all-cards split engine is Linux-only), and a split at layer 44 was worse still (158).
- **Maya's layer split trades prefill for decode.** GLM and DeepSeek read prompts 40-47% slower on two cards. DeepSeek's
  first start on two cards decoded 40.1; the second 47.5.
- **MiniMax** reads prompts at ~150 tok/s on one card or two. It isn't benchmarked beyond these two numbers.
- **The helper card costs nothing on prefill** except on Ling (below).

## The helper card

maya-nem, maya-ink, maya-ling and strata-mimo each run the whole model on one GPU and compute the routed experts that
don't fit in its VRAM on the CPU, from RAM. The CPU lane is 65-80% of a token. A layer split doesn't suit them: the
state would cross PCIe at every boundary, and the prompt path, graphs and caches would all have to know two devices.

The helper card instead holds routed experts only:

- A second expert cache on the 3090, sized to its free VRAM less 1 GB, filled with the best-scored experts the main
  card doesn't hold.
- Per MoE layer, a token's routed experts are split three ways: main card, helper, CPU lane. The helper gets the layer
  input from the host (it's already there for the CPU lane), runs the same expert kernels on its own stream, and
  returns one summed row. The host adds it to the CPU lane's sum before the main card continues.
- No model state lives on the helper. Prefill, KV, recurrent state, checkpoints and CUDA graphs are untouched, and
  without the flag the engine runs as before.
- The main cache decides first and never takes an expert the helper holds. Letting it steal cost 41 against 55 tok/s
  on maya-nem: each steal empties a helper slot and sends the main card's evicted expert to the CPU.
- The helper follows: experts the main card took are dropped, better-scored ones from outside both come in. After a
  long prompt it is refilled at once.
- The kernels keep no device state, so the same code runs on either card. The build needs both architectures
  (`-DCMAKE_CUDA_ARCHITECTURES=86;120`).

Code: branches `nemotron-2gpu`, `inkling-2gpu` and `ling-2gpu` of the Project Maya fork (`--helper-gpu N`; with two
GPUs in a run config Maya's server adds `--layer-split`, which these builds read as "the second card is the helper").
strata-mimo's is a local patch on `268fdbc`: a new `src/mimo/helper.cpp` (~410 lines) and hooks in the decoder's host
loop.

| | Experts on the 5090 | On the 3090 | Of | Expert size |
|---|---|---|---|---|
| maya-nem, UD-Q3_K_XL, 256K | 8,569 | 8,620 | 20,480 | ~2.7 MB |
| maya-ink, UD-Q4_K_XL, 128K | 1,601 | 1,527 | 10,240 | ~15 MB |
| maya-ling, IQ2_XS, 128K | 3,919 | 5,088 | 34,816 | ~4.6 MB |
| strata-mimo, MXFP4, 256K | 1,500 | 1,746 | 12,032 | 13.4 MB |

## Per engine

`ab.py` through each engine's server, every run from the same expert profile. "Deep" is decode right after the long
prompt. Single runs unless a row lists more.

### maya-nem (Nemotron 3.5 Super, UD-Q3_K_XL, 256K)

| | 5090 alone | Helper card | llama.cpp, both cards |
|---|---|---|---|
| Decode, new prompts | 41.0 | **48.2** | 23.0 |
| Deep (34.2K prompt) | 33.9 | **43.8** | 25.7 |
| Prefill | 1,925 | 1,829 | 1,079 |
| Repeated prompt, 1st and 3rd | 36.8, 44.9 | 53.7, 50.7 | 23.9, 23.7 |
| TTFT, short prompts | 1.91 s | 2.24 s | 0.52 s |

- 87 of 88 top-1 tokens match llama.cpp (the same as one card), log-probabilities within 0.11. Chat checks 6/6,
  `derail.py` 0/20.
- On one 800-token answer: 94.7% of routed experts in VRAM against 74%, the CPU lane in 23.5 of 40 MoE layers a token
  against 38.3, 55.3 tok/s against 39.9.
- A token's 18.1 ms there: CPU lane 6.7, wait for the helper 1.8, helper swaps 0.4, the main card's own layers and 40
  syncs 9.2. So this design tops out near 100 tok/s on this model; the next gain is in the main card's token path.

### maya-ink (Inkling-Small, UD-Q4_K_XL, 128K)

| | 5090 alone | Helper card |
|---|---|---|
| Decode, new prompts | 29.6 | **36.6** |
| Deep (31.8K prompt) | 27.0 | 28.0 |
| Prefill | 788 (TTFT 40.7 s) | 735 (45.3 s) |
| Repeated prompt, 1st and 3rd | 31.4, 37.5 | 40.3, **47.5** |
| TTFT, short prompts | 1.93 s | 1.50 s |

- 86 of 88 top-1 tokens match llama.cpp (87 on one card). Both misses are near ties between the same two tokens.
  Chat checks 6/6, `derail.py` 0/20.
- On one 600-token answer: 83.8% in VRAM against 69.8%, 36.0 tok/s against 27.1. A token's 27.8 ms: CPU lane 17.0
  (RAM-bound, ~0.3 ms per 15 MB expert), wait for the helper 2.5, main card 8.3.
- **Swaps had to leave the token's thread.** A 15 MB expert takes ~0.8 ms to copy up, and the tier below the main
  card's experts churns constantly: at 4 swaps every 2 tokens the helper spent 2.3-2.6 ms of every token copying, and
  16 swaps bought no more hits. A worker thread on a second stream copies them now; an expert becomes usable at the
  first update after its copy lands. The CPU lane then shares RAM bandwidth with the copies and runs ~1.3 ms longer,
  so the net is small (31.4 against 30.6 tok/s on a 300-token answer), but the main thread no longer stalls.
- **Deep barely gains.** After a long prompt both caches hold what the prompt used and few swaps follow for some
  hundred tokens. The refill of the helper after a 32K prompt adds 4.6 s to TTFT.

### maya-ling (Ling-3.1-flash, IQ2_XS, 128K)

Two runs each.

| | 5090 alone | Helper card |
|---|---|---|
| Decode, new prompts | 21.2, 21.3 | **24.7, 24.8** |
| Deep (35.5K prompt) | 15.2, 15.2 | 15.4, 14.6 |
| Prefill | 850, 864 (TTFT 41.8, 41.1 s) | 626, 631 (56.7, 56.3 s) |
| Repeated prompt, 3rd | 23.6, 24.8 | 25.0, 26.1 |
| TTFT, short prompts | 2.49, 2.54 s | 2.35, 2.34 s |

- 90 of 90 top-1 tokens match llama.cpp through the token path, log-probabilities within 0.38. Chat checks 7/7,
  `derail.py` 0/20.
- On one 284-token answer: 66% of a token's 544 routed experts in VRAM against 39%, 22.1 tok/s against 17.9. The CPU
  lane still runs in 63 of 68 MoE layers a token: 8 experts a layer rarely all sit in VRAM, and each such layer pays
  its fixed cost.
- Small experts make swaps cheap: 4 every 2 tokens decoded 19.8, 16 every 2 tokens 21.5, 32 every token 21.8-22.3.
- **Prefill is 27% slower with two CUDA devices visible, helper on or off, and the cause is open.** With both cards
  visible the prefetch thread waits 6.5 s per 16K chunk for its pinned buffers' copies to the 5090 instead of 0.9 s;
  its memcpy time is unchanged. Not the link (28.8 GB/s pinned H2D either way), not the wait mode
  (`cudaDeviceScheduleSpin` and polling the event change nothing), not RAM (26 GB stayed free). Without the prefetch
  thread two visible cards cost nothing (52.7 against 51.0 s). maya-ink's prompt path has no such thread and isn't
  affected.

### strata-mimo (MiMo-V2.6-Flash-MOPD, MXFP4, 256K, `MIMO_TAIL_KEEP=5`)

MiMo needs its 167 GB back in the file cache after other models ran, so each start ran `ab.py` twice; these are second
runs. First runs: 19.6 on one card, 29.1-32.3 on two. The three two-card starts also tried other swap settings, which
made no difference.

| | 5090 alone | Helper card, 3 starts |
|---|---|---|
| Decode, new prompts | 24.5 | **36.0, 35.9, 36.5** |
| Deep (32.7K prompt) | 20.0 | **28.4, 29.4, 29.0** |
| Prefill | 1,091 (TTFT 30.1 s) | 1,045, 1,031, 1,092 |
| Repeated prompt, 3rd | 30.4 | **47.3, 47.0, 49.6** |
| TTFT, short prompts | 0.90 s | 0.67, 0.68, 0.61 s |

- The largest gain of the four, because its CPU share was the largest. On three 400-token replays: 67% of routed
  experts in VRAM against 45%, CPU time per token 33 ms against 51.
- **Closer to the exact engine than on one card.** `MIMO_TAIL_KEEP` drops low-ranked experts the CPU would have to
  compute; experts on the helper are exempt. KL from the exact one-card run on 400 replayed tokens of three prompts:
  0.031 on one card, 0.019 with the helper (same top token 94.9% and 95.4%).
- Without the skip the helper build sits at KL 0.0047 from the exact run, top-1 98.2%. That is the engine's own
  GPU-versus-CPU arithmetic (q8_1 activation blocks on a GPU, Q8_0 on the CPU): one card moves by 0.0017-0.0024 on
  the same prompt when its VRAM budget changes which experts run on the GPU.
- `derail.py` 0/20.
- The helper's work per window goes out as one CUDA graph launch. As separate calls it cost ~75 us of host time per
  layer before the CPU's share could start.
- Not run on two cards: the hard test, the quality suite, long-context recall.

## What carried across the four ports

- **No-steal** between the two caches (above).
- **Copy swaps off the token's thread** once experts are over ~10 MB.
- **Swap rate is per model**: 16 every 2 tokens on Nemotron, 4 every 2 tokens with a higher bar on Inkling, 32 a token
  on Ling, 2 a window on MiMo (where 1 to 6 all measured the same).
- **Filling a 23 GB helper takes 3.5-4 s warm**, about 6 GB/s from pageable memory an expert at a time. Pinned staging
  does 14 GB/s here, so the refill after a long prompt could be shorter.
- **Right after a long prompt** the helper gains on Nemotron and MiMo, little on Inkling, nothing on Ling.

## The 3090 itself

Power limit sweep with Gemma 4 31B QAT on the 3090 alone (llama.cpp, decode):

| Limit | tok/s | tok per Wh |
|---|---|---|
| 390 W (stock) | 42.9 | 410 |
| 350 W | 42.5 | 445 |
| **320 W** | 42.0 | 485 |
| 290 W | 39.1 | 492 |
| 260 W | 31.3 | 441 |
| 230 W | 24.2 | 382 |
| 200 W | 17.3 | 312 |

320 W is in use, with +100 MHz on the core: 42.3 tok/s and 489 per Wh (290 W with the offset: 39.7 and 497). As a
helper card it draws far less: 60-80 W on average while working, against 145-170 W for the 5090 and ~100 W for the
CPU. Peak temperatures across all of these runs: 69 C on the 5090, 61 C on the 3090.

## Pitfall: check the link after touching the slots

Right after the install the 5090 negotiated PCIe **x1**. Flash-Next decoded 98 tok/s instead of 227 and read prompts
at 847 tok/s instead of ~5,700. Forcing Gen4 in the BIOS didn't help; the card's tail wasn't fully seated, and a
reseat with the bracket screws loosened brought back x8. `nvidia-smi
--query-gpu=index,name,pcie.link.gen.current,pcie.link.width.current --format=csv` under load shows it.

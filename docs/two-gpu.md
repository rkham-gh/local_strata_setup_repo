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
| DeepSeek-V4.1-Flash Coder IQ2_XXS | Maya (ds4) | 33.1 | 47.5 | Maya's layer split |
| | Maya (ds4), fork build | 36.5 | **66** | Maya's expert helper + DeepSeek's DSpark drafter (2026-11-12, below) |
| MiniMax-M3.1-Flash Maya-S | Maya (mm3) | 12.7 | **16.9** | Maya's layer split |
| Nemotron 3.5 Super UD-Q3_K_XL | llama.cpp b12017 | 19.4 | 23.0 | llama.cpp's device split |
| | maya-nem | 41.0 | **48.2** | Helper card |
| Inkling-Small UD-Q4_K_XL | maya-ink | 29.6 | **36.6** | Helper card |
| Inkling 2 Small own 2-bit | maya-ink, `inkling2` | 19.8 | 23.6; **32.5** with the 3090 kept clocked, the tail skip and prompt settings | Helper card (2026-11-12, below) |
| Ling-3.1-flash IQ2_XS | maya-ling | 21.2 | **24.7** | Helper card |
| MiMo-V2.6-Flash-MOPD MXFP4 | strata-mimo, patched | 24.5 | **36.0** | Helper card. Prefill 2.7x faster since 2026-11-12 (below) |

- **Flash-Next gets slower.** It already fits the 5090 well enough to decode 227 tok/s. Strata's Windows build has
  only the layer split (the all-cards split engine is Linux-only), and a split at layer 44 was worse still (158).
- **Maya's layer split trades prefill for decode.** GLM and DeepSeek read prompts 40-47% slower on two cards. DeepSeek's
  first start on two cards decoded 40.1; the second 47.5. Since 2026-11-12 DeepSeek runs on Maya's expert helper
  instead, which keeps its prefill (below).
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

### maya-ink, `inkling2` (Inkling 2 Small, own 2-bit, 128K)

Added 2026-11-12 ([models.md](models.md), Inkling 2 Small). Long prompt 41.2K tokens.

| | 5090 alone, first build | Helper card, first build | Helper card, in use |
|---|---|---|---|
| Decode, new prompts | 19.8 | 23.6 | **32.5** |
| Deep (41.2K prompt) | 13.4 | 15.0 | **27.0** |
| Prefill | 601 (TTFT 69 s) | 521 (81 s) | **836 (50 s)** |
| Repeated prompt, 3rd | 22.3 | 28.6 | 38.6 |

- 5,353 expert slots on the two cards: 2,701 on the 5090 (1,410 of its own, 811 in the prompt path's buffers, 480 in
  K/V rows) and 2,652 on the 3090, 29% of 18,432; a token takes 8.
- "In use" is the 3090 kept clocked (below), `INK_TAIL_KEEP=5`, `INK_CACHE_PROMPT=2`, 16K chunks and the helper's
  refill after a prompt on its own thread (`INK_HELPER_PROMPT_BG=1`). The first build refilled the helper on the
  token's thread, hence its slower prefill than one card.
- **Keeping the 3090 clocked** gained little on `ab.py`'s short answers (23.5 to 24.3) and more on long ones
  (teacher-forced 23.1 / 26.0 to 28.3 / 27.7). The tail skip then drops experts that neither card holds, which are the
  CPU lane's.
- **Swap rate:** a routing replay predicted 8-19% less CPU time from 8 or 16 helper swaps every 2 tokens instead of 4.
  With the 3090 clocked, 3 runs each: 16 swaps 33.9 / 26.5 tok/s against the defaults' 34.3 / 26.4; 8 swaps and a bar
  of 4 stayed inside the defaults' spread. Before the keep-alive, 16 swaps at bar 0.5 lost 6% after the long prompt.
- A token (~30 ms): the 5090's attention and its sync 12.2 ms, both cards' expert launches 7.2 ms beside a ~12 ms CPU
  lane.

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

## DeepSeek: Maya's expert helper and the DSpark drafter

2026-11-12. ISTA's Coder at IQ2_XXS on Project Maya v1.1.60's DeepSeek engine (`maya-ds4`), built for both cards from
a fork branch (`ds4-2gpu`, two commits), 1M context. `ab.py` and `turns.py` ([measuring.md](measuring.md)) through
Maya's server, every run from the same expert profile.

| | Decode, new prompts | After a 33K prompt | Prefill | Repeat, 3rd | `turns.py`, six turns |
|---|---|---|---|---|---|
| 5090 alone | 36.5-37.7 | 32.3-35.1 | 3,296 | 43.9-48.4 | 32.1 s |
| Maya's layer split | 46.3 | 46.2 | 1,373 | 55.3 | 41.0 s |
| Expert helper (`MAYA_DS4_HELPER=1`, 21 GB) | 49.8 | 45.6 | 2,970 | 60.8 | 29.2 s |
| + `DS4_FLOW_COPY_DMA=1` | 49.8 | 46.8 | 3,163 | 59.8 | 27.2 s |
| + DSpark, its 384 experts in the 5090's VRAM | 60.1 | 58.6 | 2,509 | 100.2 | 26.0 s |
| + DSpark on the 3090, beside a 14 GB helper pool | 57.8 | 53.7 | 2,913 | 100.5 | 26.4 s |
| **+ DSpark, 32 experts a block in VRAM (in use)** | **66** | **60-65** | 3,120-3,165 | **109-115** | **23-26 s** |

First token of the six turns (220 to 10,200 new tokens): 1.6-2.0 / 2.2-2.7 / 2.0-2.6 / 2.3-2.8 / 3.2-3.7 / 4.9-6.0 s in
use, against 2.9 / 4.3 / 2.5 / 2.8 / 3.3 / 4.8 on one card. On prompts the expert profile never saw
(`ab-heldout.py`): 22.7 tok/s on one card, 41.8 with the helper, 61-63 with the helper and the drafter.

- **The helper is upstream's** (`MAYA_DS4_HELPER`, which Maya's notes call tested on Linux only): the 5090 runs every
  layer, the 3090 holds ~2,410 routed experts (21 GB) and answers per layer through host-mapped memory. On Windows it
  never answered (`ds4: helper CUDA1: no answer to request 2 in 10000 ms`) and the engine went on without it. Two
  changes on the fork branch: the helper's stream is submitted after its flag is set (WDDM holds queued launches until
  something flushes them), and a request the helper's thread was never handed goes to the CPU lane instead of timing
  out and retiring the helper. The second is a guard; it hasn't fired since the first.
- **`MAYA_DS4_PROMPT_PICK=length:384`.** By default the engine times its two prompt paths at the start and picks per
  prompt. Here that sent turns of 300-1,400 new tokens through 16-token windows: 4-14 s to the first token on one card,
  where the chunked pass takes 2.5-3.5 s. Choosing by length fixes that, and stops the start from timing the paths.
- **`DS4_FLOW_COPY_DMA=1`** DMAs the low-bit experts' RAM copies on the prompt path: mid-size turns in 2.5 s instead of
  3.3. `DS4_STREAM=0` keeps a new build from re-timing the PCIe link at its first start (it runs that start with half
  the experts in VRAM).
- **DSpark works on the pruned Coder.** Maya's `pack_gguf.py` refuses the drafter for a 192-expert model, but that is
  its geometry check, not the model. A second pack folder whose `tensors.txt` also lists the drafter's tensors in
  place (the Q8_0 DSpark GGUF that Strata's DeepSeek setup downloads, names mapped with the packer's own table: 3
  blocks of 128 experts, block size 5, taps on layers 37-39) loads and drafts. 80-96% of drafted tokens are kept on
  code, up to five a round; code decodes at 90-115 tok/s, prose at 40-60.
- **Where the drafter's experts sit decides it.** All 384 in the 5090's VRAM push ~940 of the trunk's 2,640 experts
  out, and prose gets slower than without drafting (35 against 41 tok/s). `MAYA_DS4_DRAFT_VRAM=32` keeps each block's
  32 most used in VRAM and leaves the rest to the CPU lane (11-26% of the drafter's picks on `ab.py`'s prompts, up
  to 60% on short chats). Caps of 24, 32 and 48 measured 66.7, 66.4 and 66.8; 64 gave 64.2. On the 3090 the drafter
  costs helper slots and a hop per round.
- **The expert profile has to hold the drafter's counts**, since they choose which stay. At a cap of 48, from a
  profile without them: 60.2 tok/s on unseen prompts; from one that had seen the drafter on other prompts: 65.7.
- **`MAYA_DS4_SLOT_KEEP=1`** (upstream, opt-in) keeps the conversations it has seen: back to a 17K-token conversation
  after another one took the model in 0.3 s instead of 6.0. It keeps 4 in RAM and writes older ones, tokens included,
  to the pack's `slots` folder (16 GB cap, 7 days).
- **Why drafting pays this much on Windows:** a kernel launch costs the host ~4.85 us, and a decoded DeepSeek token
  spends ~12 ms in launches against 4.8 ms in the CPU lane. A round that keeps five tokens pays the launches once.
- **Not adopted** (all with the drafter): the helper's recent slots 57.7; cost-aware drafting 65.7; an 18 GB helper
  pool 61.6; the launch trimmers (`MAYA_DS4_DECODE_TRIM=7`, `MAYA_DS4_Q8MV_TC=1`, `MAYA_DS4_HEAD_COLS=16`) 67.6-67.9
  against 65.8-66.8, inside the spread; fused small launches +1% without the drafter.
- **Spread.** Greedy decode on this engine isn't reproducible: every run of one setup writes a different answer, on
  one card too. Likely an expert doesn't round alike on the 5090, the 3090 and the CPU, and which has it moves with
  the caches. Seven runs of the settings in use and their near neighbours: 54.9, 65.8, 66.4, 66.7, 66.8, 67.9, 69.7.
- **Checks** on the settings in use: `derail.py` 0/20; tools 5/5; facts 8/8 and 7/8; reasoning 7/8 twice, the miss
  being 1/11 written as `\frac{1}{11}`, which the checker doesn't read; 4 of 4 answers in `content` at effort `max`.
- The first long prompt after another model had the RAM reads in 14-18 s instead of 10.6: its experts come off the
  SSD. That was so on one card.

## MiMo: the prompt path

2026-11-12. strata-mimo with the helper card (above), MOPD at MXFP4, 256K, `MIMO_TAIL_KEEP=5`. Decode didn't move:
34-36 tok/s on new prompts, 27-29 after the 33K prompt. What changed is the wait before the first token.

| | Prefill, 32.7K prompt | TTFT, 227-token turn | 1,348-token turn | 10,188-token turn |
|---|---|---|---|---|
| As it was (`--ram-copy 0`) | 1,037 tok/s, 31.7 s | 9.7 s | 11.0 s | 15.1 s |
| A pinned copy of the 34 layers that fit | 1,246 tok/s, 26.5 s | 5.8 s | 6.8 s | 8.6 s |
| + the CPU lane on a short prompt's rarely picked experts | 1,210 tok/s | 3.5 s | 5.2 s | 8.6 s |
| + 32K slices | 1,860 tok/s, 17.9 s | 3.5 s | 5.3 s | 8.7 s |
| **+ the unpinned layers read back into RAM at the start (in use)** | **2,810 tok/s, 11.9 s** | **3.5 s** | **4.9 s** | **8.7 s** |

- **What a prompt costs.** The batched prompt path copies every routed expert the 5090 doesn't hold over PCIe whole,
  13.4 MB, once per layer and slice, however few tokens picked it. A 227-token turn uses 209 of a layer's 256 experts
  and copies 185: 108 GiB for 227 tokens. From the file cache those copies ran at 9-15 GB/s, from pinned memory at the
  link's 28.8.
- **The pinned copy.** The engine's pinned copy of all 47 layers (150 GB) never fit here. It now pins layers while the
  commit limit allows and reads the rest through the mapping: 34 of 47, 108 GiB ([pinned-memory.md](pinned-memory.md)).
  The 34 stream at the link rate; the 13 others cost about as much time as the 34.
- **The CPU's share** (slices up to 4,096 tokens). In a short prompt most experts are picked by a handful of tokens.
  Those go to the CPU lane, which computes an expert from RAM in about the time its copy takes (0.43 against 0.47 ms),
  while the card copies and computes the rest; the card's picks get them masked out. Reads and copies share the RAM's
  bandwidth, ~35 GB/s together, so short turns stop at ~3.5 s.
- **32K slices** halve the copies of a 16-32K prompt. The prompt's buffers borrow 292 expert slots instead of 228.
- **The helper's refill left the first token's path.** After a long prompt up to 1,024 experts went to the 3090
  before the answer began, ~5 ms each. Its worker thread copies them now while the answer starts, straight from pinned
  memory where the layer is pinned; its first fill at the start comes after the pinned copy for the same reason.
- **The start went from 17-20 s to 1.8-2.4 minutes**: about a minute for the copy. The copy also pushes the 13
  unpinned layers out of the file cache, and the first turn after a start then took 20 s. A thread reads them back at
  the start (41 GiB in 8-11 s) and the first turn takes 5.7 s. With `--ram-copy 0` the start is 17 s again and the CPU's
  share and the 32K slices still apply.
- **Not adopted.** Verifying a draft only when the MTP head gives it 50% or more: 2% slower, although its drafts below
  0.5 are right only 27-45% of the time. Pinning the file's pages in place instead of copying them: fewer layers
  ([pinned-memory.md](pinned-memory.md)). FP16 tensor-core products for the experts: the copies set the pace.
- **Checks:** `derail.py` 0/20 twice; tools 5/5; facts 8/8; reasoning 7/8 twice, a different letter puzzle each time.
  Those prompts are under 128 tokens and take the window path, which didn't change.

## What carried across the four ports

- **No-steal** between the two caches (above).
- **Copy swaps off the token's thread** once experts are over ~10 MB.
- **Swap rate is per model**: 16 every 2 tokens on Nemotron, 4 every 2 tokens with a higher bar on Inkling, 32 a token
  on Ling, 2 a window on MiMo (where 1 to 6 all measured the same).
- **Filling a 23 GB helper takes 3.5-4 s warm**, about 6 GB/s from pageable memory an expert at a time. Pinned staging
  does 14 GB/s here, so the refill after a long prompt could be shorter.
- **Right after a long prompt** the helper gains on Nemotron and MiMo, little on Inkling, nothing on Ling.
- **The refill after a long prompt can leave the first token's path.** On strata-mimo it was up to 1,024 expert
  copies at ~5 ms each before the answer began; since 2026-11-12 the helper's worker thread makes them while the
  answer starts.

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

**It idles under helper work.** A helper's work comes in bursts of tens of µs a layer, and the driver keeps the 3090
in P3/P5: 420-1,260 MHz, its memory at 5,001 or 810 MHz of 9,751. In an Nsight Systems profile of Inkling 2 the
3090's IQ2 expert GEMVs took 180-280 µs against 15-23 µs on the 5090, and each token waited ~5 ms for it. maya-ink's
`inkling2` branch now keeps a one-warp kernel asleep for 50 ms (`__nanosleep` against `%globaltimer`) on a
lowest-priority non-blocking stream of the 3090, launched again each token once the last one has ended: the card
stays in P2 (2,100 / 9,501 MHz) while tokens are written and falls back when they stop, and the wait drops to
~0.6 ms. A kernel that never ends hangs the first token, since some call waits for all of a card's work. NVIDIA's
"Prefer maximum performance" or a clock lock would cover every engine at once. Inkling-Small, Nemotron, Ling, MiMo
and DeepSeek haven't been checked for it yet, so their helper-card numbers above may be low.

## Pitfall: check the link after touching the slots

Right after the install the 5090 negotiated PCIe **x1**. Flash-Next decoded 98 tok/s instead of 227 and read prompts
at 847 tok/s instead of ~5,700. Forcing Gen4 in the BIOS didn't help; the card's tail wasn't fully seated, and a
reseat with the bracket screws loosened brought back x8. `nvidia-smi
--query-gpu=index,name,pcie.link.gen.current,pcie.link.width.current --format=csv` under load shows it.

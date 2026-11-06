# Large LLMs on one workstation: setups, speeds and a hard test

Measured notes on running twelve large language models locally on one PC, from 27B dense models to 564B
mixtures of experts. They cover how each one is served, how fast it writes and reads, and how well it does on a 25-task
hard test at its best settings. All numbers were measured on this machine between 2026-10-29 and 2026-11-06.

## The machine

| Part | Spec |
|---|---|
| CPU | AMD Ryzen 9 9950X, 16 cores |
| GPU | NVIDIA RTX 5090, 32 GB, on PCIe 5.0 at x8 (about 29 GB/s) |
| RAM | 192 GB DDR5 in four DIMMs, running at 3600 MT/s |
| Storage | One 2 TB NVMe SSD |
| OS | Windows 11 Pro |

Only one engine runs at a time: each needs the whole GPU and most of the RAM. A PowerShell switcher (`switch-model.ps1`)
stops one engine and starts another, and won't stop a server that answered a request in the last 5 minutes.

## The engines

| Engine | What it is | Models here |
|---|---|---|
| Strata | Inference engine for large MoE models: VRAM expert cache, speculative drafting (DFlash2) | Qwen3.8-Flash-Next |
| Project Maya | A Strata fork with GLM and DeepSeek engines | GLM-5.3-Flash, DeepSeek-V4.1-Flash Coder |
| strata-mimo | A Strata fork for MiMo, with local fixes | MiMo-V2.6-Flash-RL |
| maya-ink, maya-nem, maya-ling | Engines written for this PC in a Project Maya fork, one branch each (`inkling`, `nemotron`, `ling`). They use the same design: chunked prompt reading on the GPU, a VRAM expert cache running beside a CPU lane, CUDA graphs | Inkling-Small, Nemotron 3.5 Super, Ling-3.1-flash |
| llama.cpp router | `llama-server` in router mode with per-model presets | Qwen3.8-27B, Nanbeige4.3-33B, Gemma 4 26B and 31B, Nemotron with pictures |

## The models at a glance

Writing speed is on new prompts, from short chats measured with `ab.py` (greedy, thinking off) unless marked. The hard
test is at each model's best settings; see `docs/hard-test.md`.

| Model | Engine | Quant | Context | Writing, new prompts | Hard test, of 25 |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next | Strata | ISTA-DASLab IQ3_S | 256K | 225-229 tok/s | **22** (coding 4/7) |
| Inkling-Small (276B, 12B active) | maya-ink | Unsloth UD-Q4_K_XL, 163 GB | 128K | 30-31 tok/s | **21** (3/7) |
| Ling-3.1-flash (564B, 25B active) | maya-ling | bartowski IQ2_XS, 177 GB | 128K | 22-23 tok/s | **21** (3/7, 2 cut off at 128K) |
| Qwen3.8-27B (dense) | llama.cpp | Q6_K | 208K | 128 tok/s; 99 with thinking at its sampling | **20** (4/7) |
| DeepSeek-V4.1-Flash Coder | Project Maya | ISTA IQ2_XXS | 1M | 33-35 tok/s | **19** (2/7) |
| Gemma 4 26B-A4B | llama.cpp, MTP drafter | Q8_0 | 224K | 311 tok/s with thinking at its sampling | **18** (2/7) |
| Gemma 4 31B (dense) | llama.cpp, MTP drafter | Google QAT 4-bit (UD-Q4_K_XL) | 224K | 170 tok/s with thinking at its sampling | **17** (1/7) |
| Nemotron 3.5 Super VL (120B, 12B active) | maya-nem | Unsloth UD-Q3_K_XL, 65.8 GB | 256K | ~41 tok/s | **17** (0/7) |
| MiMo-V2.6-Flash-RL | strata-mimo | MXFP4 | 256K | 21-26 tok/s | 12.9 in the 24K round |
| GLM-5.3-Flash | Project Maya, Maya-S24 | Maya pack with expert skips | 256K | 36-38 tok/s (23 on the exact Maya-M) | 10 in the 24K round; rerun at its best in progress |
| Nanbeige4.3-33B (dense) | llama.cpp | Q4_K_M, 20.8 GB | 80K (96K for the test) | 66-69 tok/s | 9.8 in the 24K round; rerun at its best in progress |

`docs/models.md` has each model's architecture, quant choice, engine settings, prompt-reading speed, memory use,
correctness checks and quirks.

## What we found

- **Qwen3.8-Flash-Next on Strata is the best all-rounder here.** It has the top hard-test score, 22 of 25, and writes
  225-229 tok/s. It finishes the whole test in 51 minutes, where models of similar score take 5-15 times as long.
- **Mixtures of experts too big for VRAM are bound by RAM bandwidth.**
  - Each token reads its routed experts that aren't cached in VRAM from system RAM, at about 40-50 GB/s on this
    DDR5-3600.
  - On Inkling, Nemotron and Ling, that CPU share is 65-80% of each token's time. Writing speed follows how many
    experts fit in VRAM and how many bytes each expert is, not the file size.
  - More of the experts in VRAM, or faster RAM, is the lever.
- **Engines built for this hardware beat llama.cpp's best setups by about 2-3x on writing and 3-10x on prompt reading**
  for the big MoE models on the same files. Inkling writes 30 tok/s against 9, Nemotron's UD-Q4_K_XL 26 against 13, and
  Ling 22 against 12. Ling reads prompts at 876 tok/s against 87. The design:
  - prompts read on the GPU in chunks of 8-16K tokens, so each expert crosses PCIe once a chunk;
  - an adaptive VRAM expert cache running at the same time as a CPU lane;
  - CUDA graphs for each layer.
- **Two big new MoE models nearly match the default on quality, but not on speed.** Inkling-Small and Ling-3.1-flash
  both scored 21 of 25, with every task outside coding right. They took 279 and 744 minutes against Flash-Next's 51.
- **Coding separates the models.** Only Flash-Next and Qwen3.8-27B solved 4 of the 7 coding tasks. Of the finished
  runs at their best, only Flash-Next solved the regex engine. Only Inkling and Ling solved the Scheme interpreter, and
  only Ling and MiMo the JSON parser.
- **Room to think matters.** A 24,000-token budget cut off every coding answer of GLM, Nanbeige and MiMo while they were
  still working, so their 24K-round scores understate them. Those two reruns are in progress.
- **Quants.** Flash-Next's IQ3_S and UD-Q4_K_XL scored within one task of each other (22 and 21 with room), and IQ3_S
  writes twice as fast. Nemotron's UD-Q3_K_XL writes 43% faster than UD-Q4_K_XL, and its top token matches Q4's at 97%
  of positions.

## Documents

| File | Contents |
|---|---|
| `docs/models.md` | One section per model: specs, quant, engine settings, measured speeds, memory, correctness, quirks |
| `docs/hard-test.md` | The hard test: tasks, scoring, the settings each model ran at, per-task results and notes |
| `docs/measuring.md` | How the speeds and correctness were measured, and the rules that came from wrong results |
| `results/hard-test-tasks.csv` | Every hard-test run's per-task verdicts, tokens and times |

# Models

One section per model set up on this PC: architecture, the quant in use and why, how it's served, measured speeds,
memory use, correctness checks, settings and quirks. Every speed was measured on this machine. Figures from a model's
lab, a quant publisher or an engine's own docs are marked as theirs. [measuring.md](measuring.md) describes the methods,
and [hard-test.md](hard-test.md) has the hard-test results that each section sums up in one line.

**Hardware:** AMD Ryzen 9 9950X (16 cores), NVIDIA RTX 5090 32 GB (450 W limit) on PCIe 5.0 running at x8 (28.7 GB/s
measured), 192 GB DDR5 rated 5600 but running at 3600 MT/s (four DIMMs), Windows 11 Pro. The README has the full table,
and what each part means for the speeds.

Only one engine runs at a time: each one needs the GPU and most of the RAM. `switch-model.ps1` stops the running server
before it starts the next. Qwen3.8-Flash-Next on Strata is the default.

## Summary

"New prompts" is `ab.py`'s writing speed on prompts the engine hasn't seen (greedy, thinking off). "Prompt reading" is
measured on `ab.py`'s long prompt, 32-39K tokens depending on the tokenizer. "—" means the notes don't record it.

| Model | Total / active parameters | Quant, size | Engine | Context | New prompts | Prompt reading | RAM in use |
|---|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next (default) | — | ISTA-DASLab IQ3_S, 84 GB | Strata 0.2.10 | 256K | 225-229 tok/s | ~5,700 tok/s | 85-88 GB |
| GLM-5.3-Flash | — | Maya-S24, 94.7 GB | Project Maya v1.1.52 | 256K | 38.4 tok/s | 2,948 tok/s | 74 GB |
| GLM-5.3-Flash, exact | — | Maya-M, 116 GB | Project Maya v1.1.52 | 256K | 22.8 tok/s | ~2,650 tok/s | 92-94 GB |
| MiMo-V2.6-Flash-RL | — | MXFP4, 170 GB | strata-mimo, patched | 256K | 25.5 tok/s | ~1,030 tok/s | grows to ~157 GB |
| DeepSeek-V4.1-Flash Coder | 552B base, half its routed experts removed | ISTA-DASLab IQ2_XXS, 72.1 GB + 110.6 GB Engram table | Project Maya's DeepSeek engine | 1M | 33.5 tok/s | 3,200-3,300 tok/s | ~43 GB of pinned experts; total — |
| DeepSeek Coder, fallback | as above | as above | llama.cpp router | 64K | 29.8 tok/s | 472 tok/s | — |
| Qwen3.8-27B | 27B, dense | Unsloth UD-Q6_K, 22 GB + 1.4 GB MTP head | llama.cpp router | 208K | 128 tok/s | 2,630 tok/s | a few GB |
| Nanbeige4.3-33B | 33B, dense | bartowski Q4_K_M, 20.8 GB | llama.cpp router | 80K (96K for its hard-test runs) | 69 tok/s | ~2,050 tok/s | little: all in VRAM |
| Gemma 4 26B-A4B | 25.2B / 3.8B | Q8_0, 26.9 GB + 0.5 GB drafter | llama.cpp router | 224K | 318 tok/s | ~8,000 tok/s | a few GB |
| Gemma 4 31B | 31B, dense | Google's QAT 4-bit as UD-Q4_K_XL, 17.3 GB + 0.3 GB drafter | llama.cpp router | 224K | 184 tok/s | ~2,550 tok/s | a few GB |
| Inkling-Small | 276B / 12B | Unsloth UD-Q4_K_XL, 163 GB | maya-ink | 128K | 30.2 tok/s | 794 tok/s | ~164 GB (22 GB free) |
| Nemotron 3.5 Super VL | 120B / 12B | Unsloth UD-Q3_K_XL, 65.8 GB | maya-nem | 256K | 41.3 tok/s | ~1,970 tok/s | ~62 GB |
| Nemotron 3.5 Super VL, Q4 | 120B / 12B | Unsloth UD-Q4_K_XL, 86.9 GB | maya-nem | 256K | 26.3 tok/s | 1,607 tok/s | ~84 GB |
| Nemotron 3.5 Super VL, pictures | 120B / 12B | UD-Q4_K_XL + 1.7 GB picture encoder | llama.cpp router | 256K | 13.0 tok/s | ~474 tok/s | — |
| Ling-3.1-flash | 564B / 25B | bartowski IQ2_XS, 177.4 GB | maya-ling | 128K | 22.4 tok/s | 876 tok/s | ~160 GB (24 GB free) |

Flash-Next's UD-Q4_K_XL quant was measured, then deleted. Its numbers are in the Flash-Next section.

## Engines

| Engine | Port | Serves | Notes |
|---|---|---|---|
| llama.cpp router, build 12017 (CUDA 13.4) | 9931 | Qwen3.8-27B, Nanbeige4.3-33B, Gemma 4 26B-A4B and 31B; fallbacks for DeepSeek Coder and Nemotron VL | `llama-server` in router mode, one model loaded at a time (`--models-max 1`), per-model settings in `presets.ini` |
| Strata 0.2.10 (engine 0.1.56) | 8080 | Qwen3.8-Flash-Next | Hot experts in VRAM, the rest from RAM, DFlash2 drafting |
| Project Maya v1.1.52 | 8081 | GLM-5.3-Flash | MIT-licensed Strata fork, built from source |
| strata-mimo, commit `268fdbc` | 8082 | MiMo-V2.6-Flash-RL | Third-party Strata fork, built from source, four local changes |
| Project Maya's DeepSeek engine (`maya-ds4`) | 8083 | DeepSeek-V4.1-Flash Coder | Same Maya install, with a local server change |
| maya-ink | 8084 | Inkling-Small | Custom engine in a Project Maya fork, branch `inkling` |
| maya-nem | 8085 | Nemotron 3.5 Super VL | Same fork, branch `nemotron` |
| maya-ling | 8086 | Ling-3.1-flash | Same fork, branch `ling` |

- `switch-model.ps1` takes `flash-next`, `glm`, `glm-m`, `mimo`, `deepseek`, `inkling`, `nemotron`, `nemotron-q4`,
  `ling`, `router`, `stop` or `status`. It applies each setup's tuned settings and refuses to stop a server that's in
  use (exit codes: 0 done, 1 failed, 2 refused). Don't pipe its output: the server it starts inherits the pipe, and the
  caller hangs until that server stops.
- The router drops Unsloth's `UD-` prefix from model IDs: pull with the full tag, select the ID without it. It starts in
  about 2 s, loads a model on its first request, and unloads it when a request names another model.
- maya-ink, maya-nem and maya-ling run behind Project Maya's own server. Each has a run config (engine binary,
  arguments, tokenizer folder, model name, GPU, sampling, optional `env`), a tokenizer folder written by its
  `make_pack.py`, and an `expert-profile.bin` of expert scores that the engine saves after every request, so the next
  start begins from them. Maya puts a run config's `env` over the caller's environment.
- Maya's server gives any model whose vocabulary has a special `</think>` token a thinking budget: 32,768 tokens by
  default, closed at the first sentence or line end past it. `thinking_budget` in the config changes it, and 0 turns it
  off. `"fit_max_tokens": true` trims a token budget larger than the room left instead of refusing the request.
- `GET /health` checks a server. Strata's reports `"loaded": true`; Maya's is ready when `status` is `ok` and
  `max_context` is set.
- Above about 31.8-32 GB of VRAM in use, the driver pages VRAM into system RAM and speed collapses. Every setup here
  stays below that.

**Measuring**, in short ([measuring.md](measuring.md) has the details): `ab.py` is greedy with thinking off: six new
prompts, then a 32-39K-token prompt read and answered, then one prompt asked three times. `spec-sweep.py` measures
writing with thinking on at the model's own sampling. `match.py` replays the same 4 chats 3 times, the workload of
published Strata speed tables. `deep-test.py` reads prompts of 125K-250K tokens with three hidden notes. `glm-test.py`
is a quality suite (tools, JSON, pictures, 8 facts, 8 reasoning puzzles, 3 coding tasks, an agent bug-fix task), and
`derail.py` runs 20 rounds of a tool conversation followed by a new chat, counting answers that go off the rails.
Engines with a VRAM expert cache start each A/B run from the same saved expert profile.

## Calling the models

- Every engine takes OpenAI-style requests at `http://127.0.0.1:<port>/v1`, with no API key. Thinking arrives in
  `reasoning_content`, separate from `content`.
- `max_tokens` limits the output, not the context. Leave it out to allow the rest of the context, or allow at least
  40,000 tokens at `high`. Check `finish_reason`: `length` means the cap cut the answer off.
- Set client timeouts to at least 30 minutes. A 250K-token prompt takes 44 s to its first token on Flash-Next and up to
  5 minutes on MiMo.
- Keep one thinking level through a conversation: changing it can re-read the whole prompt.
- A prompt longer than the context gets an HTTP 400 at once. Nothing is cut.

| Engine | Thinking cap by default |
|---|---|
| Strata (Flash-Next) | Three quarters of `max_tokens` |
| Project Maya's server (GLM, DeepSeek Coder, Nemotron, Ling) | 32,768 tokens, set by `thinking_budget` |
| Project Maya's server with maya-ink (Inkling) | None: Inkling's thinking doesn't end at a special token |
| strata-mimo (MiMo) | None: `max_tokens` is the whole budget |

## 1. Qwen3.8-Flash-Next (Strata, default)

A mixture-of-experts model with pictures as input and its own multi-token-prediction (MTP) head. It runs on Strata,
which keeps the most-used experts in VRAM, computes the rest from RAM and drafts tokens ahead. The notes don't record
its total and active parameter counts.

| | |
|---|---|
| Context | 262,144 tokens, the model file's limit; served at 256K |
| Input | Text and pictures |
| Quant | ISTA-DASLab IQ3_S, 84 GB, checked against its pinned SHA-256 |
| Engine | Strata 0.2.10 (engine and terminal 0.1.56) |
| Start | `switch-model.ps1 flash-next`; ready in 75-115 s |
| API | `http://127.0.0.1:8080/v1`, model ID `qwen3.8-flash-next-iq3_s`. Strata ignores the model name, so older IDs work too |
| VRAM, RAM | 30.3-30.4 GB, 85-88 GB |
| Experts in VRAM | About 11,900: 49% of the model's experts |

**Why IQ3_S.** 49% of its experts fit in VRAM, against 31% of UD-Q4_K_XL's, so it writes about twice as fast here.
ISTA-DASLab's benchmarks score it level with the full model (task average 93.26 against 93.12). Its top token matches
Q8_0's 90% of the time, against 94% for UD-Q4_K_XL. It passed every test here that Q4 did, and Q4 didn't score higher
on the hard test.

**Engine settings.** `switch-model.ps1` runs:

```
strata.cmd launch --family qwen --model IQ3_S --model-path <IQ3_S folder> --context 262144 --vision cpu --parallel 1
  --set vram-reserve-mib=1200 --set adapt-every=1 --set adapt-swaps=80 --set adapt-decay=0.97 --yes --no-tui
```

- **Drafting.** Strata 0.2.10 picks DFlash2 on this card, with a 1.2 GB drafter it downloads and packs on first start,
  and sets its own window (`--spec 6`). Strata 0.1.54 drafted with the MTP head, where a 5-token window measured best;
  that setting was dropped with the update.
- **Expert re-balancing.** `adapt-every=1`, `adapt-swaps=80` and `adapt-decay=0.97` re-balance the VRAM experts faster
  than the defaults. The script restarts an IQ3_S engine that runs without them.
- **VRAM reserve,** 1,200 MiB. A plain `strata launch` drops it to 700 MiB, and the engine warns that VRAM is nearly
  full. With 1,200 the log still shows about 999 MiB free once, which is harmless.
- **Pictures** are read on the CPU (`--vision cpu`), about 3 s for a chart. That frees VRAM for experts; together with
  the smaller reserve it gained 4%.
- **Route tail skip,** on by default for IQ3_S: an expert the GPU doesn't hold is left out when every token of a draft
  window ranks it 7th or lower.
- **KV cache.** Strata keeps most of the 256K cache in RAM, so `--kv-grow` turns itself off. Each 128K of context costs
  about 1.5 GB of RAM and about 1% of the expert slots in VRAM.
- **Missed experts.** The CPU pool and the x8 link each move about 29 GB/s of expert data, and the engine splits missed
  experts between them.
- `strata launch` rewrites its launch JSON on every start, so flags belong in `switch-model.ps1`. In PowerShell, call
  `strata.cmd`: script policy blocks `strata.ps1`.

**Calling it.**

- Thinking defaults to `xhigh`. Send `"reasoning_effort": "low"` for quick work, or
  `"chat_template_kwargs": {"enable_thinking": false}` for none. The hard test ran it at temperature 0.6, top-p 0.95.
- **Thinking cap.** Strata counts thinking against `max_tokens`. When a request sets `max_tokens` of 1,024 or more and no
  `reasoning_budget_tokens`, Strata closes the thinking at `max_tokens - max_tokens // 4` by inserting "I have thought
  about this long enough; time to give my answer." A 24,000-token cap thinks for 18,000. Without `max_tokens` the budget
  is the rest of the context, and the thinking can run to about 196K. Only `"answer_room": false` in Strata's config
  turns the rule off. For hard work, leave `max_tokens` out.
- The template puts the thinking level at the start of the prompt, so switching thinking on mid-conversation re-reads
  the whole prompt: about 105 s at 250K.
- Tools, `response_format` and pictures work.

**Speed.** `ab.py` (greedy, thinking off) for new prompts and a 35K-token prompt; `match.py` for the repeated-chat
workload:

| Flash-Next IQ3_S | New prompts | 35K tokens deep | Same 4 chats 3 times | Reading prompts |
|---|---|---|---|---|
| Strata 0.1.54, plain start (mean of 4 runs) | 190 tok/s | 190 tok/s | 228 tok/s | 5,320 tok/s |
| Strata 0.1.54, MTP window 5 + faster re-balancing (2 runs) | 202 tok/s | 192 tok/s | 240 tok/s | |
| Strata 0.2.10, DFlash2 + faster re-balancing (in use) | 225-229 tok/s | 201-217 tok/s | 274 tok/s | about 5,700 tok/s |

- With thinking on at its own sampling (`spec-sweep.py`): 208 tok/s.
- Through the llama.cpp router, Flash-Next (UD-Q4_K_XL) wrote 28 tok/s and read prompts at about 480 tok/s, because only
  Strata gives it drafting. Don't run it there.
- Long prompts (`deep-test.py`, Strata 0.1.54): the first token after 21 s at about 125K tokens, 26 s at 150K, 35 s at
  200K and 44 s at 250K, writing 173, 164, 173 and 186 tok/s. It found every hidden note. A follow-up reads only the new
  text, in 0.5-2 s, and going back a step at 125K took 1-2 s.

**Tuning on Strata 0.1.54** (`flash-sweep.ps1`: one setting at a time, each a restart, then `ab.py` and `match.py`). The
plain start measured 187-193 tok/s over four runs, so differences under about 3% are noise. The 35K-deep figure comes
from one 400-token answer and varies more.

| IQ3_S setting | New prompts | 35K deep | Same 4 chats 3 times |
|---|---|---|---|
| Plain start, mean of 4 runs | 190 | 190 | 228 |
| Draft window 5 + faster re-balancing, 2 runs | 202 | 192 | 240 |
| Faster re-balancing alone, 2 runs | 194 | 195 | 238 |
| Draft window 5 alone, 2 runs | 195 | 184 | 235 |
| Draft window 6 | 195 | 191 | 230 |
| Draft window 3 | 177 | 177 | 220 |
| Draft confidence 0.7, 0.4 or 0.3 (against 0.5) | 182-188 | 181-186 | 221-224 |
| 64K of the KV cache in VRAM | 194 | 187 | 223 |
| 4-bit draft layer, larger prompt chunks, `adapt-decay 0.9` | 183-192 | 188-190 | 223-232 |

A PCIe share of 0.25 changed nothing. DFlash2 couldn't serve requests on 0.1.54, and the 0.1.55 engine update changed
CUDA speed on Linux only.

**Checks.** On the quality suite: facts 8/8, logic 8/8, tool calls 5/5, a chart read correctly, the 3 coding tasks at
`high` in 16-36 s each, and the agent bug-fix task in 9.9 s. `derail.py` found 0 of 20 rounds derailed, on 0.1.54 and
again on 0.2.10.

**Quirks.** Strata ends a reply after 256 repeats of one token: in the hard test it stopped Flash-Next writing zeros
while recalling a Sudoku puzzle. At the 24K budget it once wrote an endless number on the JSON task.

**UD-Q4_K_XL, measured and since deleted.** Unsloth's UD-Q4_K_XL (111 GB) ran on Strata as `switch-model.ps1
flash-next-q4`, with `--set STRATA_ROUTE_TAIL_SKIP=7`. Its files were deleted on 2026-11-03, so that target no longer
starts.

| Flash-Next UD-Q4_K_XL | Strata 0.1.54, as Strata sets it | 0.1.54, tail skip on | 0.2.10, DFlash2 |
|---|---|---|---|
| New prompts | 81 tok/s | 107 tok/s | 116 tok/s |
| 35K tokens deep | 97 tok/s | 123 tok/s | 129 tok/s |
| Same 4 chats 3 times | 101 tok/s | 134 tok/s | |
| Reading prompts | 2,260 tok/s | 2,270 tok/s | 2,330 tok/s |
| RAM | 105-113 GB | 105-113 GB | |

- Strata turns the route tail skip off for Unsloth quants once every expert fits in RAM, because it moves answers
  slightly. Left off, it costs a quarter of the speed.
- Only 31% of its experts fit in VRAM. strata-nvfp4's published figures, from a 9950X3D with DDR5-5600 and PCIe 5.0
  x16, are 228 tok/s on IQ3_S and 169 on UD-Q4_K_XL. This PC matches the IQ3_S figure; the Q4 gap fits the slower RAM
  and link.
- Hard test: 21 of 25 with room to think (19 at the 24K budget), in 92 minutes. Single tasks flipped between runs, so
  one run each reads as even with IQ3_S, not worse.

**Hard test** ([hard-test.md](hard-test.md)): IQ3_S 22 of 25 with room to think (21 at the 24K budget), coding 4 of 7,
51 minutes.

## 2. GLM-5.3-Flash (Project Maya)

A mixture-of-experts model with pictures as input. Strata doesn't run it; Project Maya, an MIT-licensed engine built on
Strata, does. The notes don't record its total and active parameter counts.

| | |
|---|---|
| Experts | 12,096 routed experts; each layer picks 8 per token |
| Context | 1,048,576 tokens in the model file; served at 256K |
| Input | Text and pictures |
| Quant | Maya-S24, Project Maya's own build of the weights, 94.7 GB. The original weights are FP8 |
| Engine | Project Maya v1.1.52, built from source with CUDA 13.4 and Visual Studio 2022 Build Tools |
| Start | `switch-model.ps1 glm` (config `maya-maya-s24.json`); ready in 75-95 s |
| API | `http://127.0.0.1:8081/v1`, model ID `glm-5.3-flash`, with a dashboard on the same port |
| VRAM, RAM | 29.4 GB, 74 GB |
| Experts in VRAM | 3,252, about 27% |
| Exact setup | `switch-model.ps1 glm-m`: Maya-M (116 GB) with no expert skips, config `maya-maya-m.json`, 29.6 GB of VRAM, 92-94 GB of RAM |

**Why Maya-S24 with skips.** Without skips, GLM is limited by RAM speed. Maya-S24 has smaller experts than Maya-M, so
more fit in VRAM, and two expert skips leave out the lightest experts the GPU doesn't hold. Maya's docs put Maya-S24's
next token at the original's 83% of the time, against 86% for Maya-M, and give the two skips below a KL of 0.052 from
the unskipped model. For scale, Maya-S itself is 0.43 from the original FP8.

| GLM-5.3-Flash, new prompts | Writing | 33K deep |
|---|---|---|
| Maya-M | 23 tok/s | 21 tok/s |
| Maya-M with the layer map | 25 tok/s | 24 tok/s |
| Maya-S24 | 27 tok/s | 25 tok/s |
| Maya-S24 with the layer map | 35 tok/s | 33 tok/s |
| Maya-S24 with both skips (in use) | 38 tok/s | 37 tok/s |
| Maya-S24 with MTP forced on | 26 tok/s | 24 tok/s |

With both skips, 78-82% of expert uses hit VRAM, against 57%, and the CPU's share of a token fell from 30 ms to 11 ms.

**Engine settings** (the `env` block of `maya-maya-s24.json`):

- `MAYA_GLM_TRUNK_KEEP_MAP=8888888888888888888888855555554444444444444448`. When the rest would come from RAM, layers
  23-29 keep their 5 best experts and layers 30-44 their 4 best. The early layers keep all 8, because they notice a
  missing expert far more.
- `MAYA_GLM_MISS_SKIP_W=0.14` with `MAYA_GLM_MISS_SKIP_FROM=23`. From layer 23 on, a missing expert carrying under 14% of
  its token's route weight is left out.
- **No MTP.** Maya's GLM engine drafts only with two or more GPUs. Forced on with one GPU, it was slower (table above).
- **CPU lane.** `START-MAYA.bat --calibrate --no-start` (about 10 minutes) raised Maya-M from 21.2 to 23.5 tok/s
  (`PCIE_SHARE=0.10`, `CPU_LANE=12`). On Maya-S24 with skips it kept the engine's own split (PCIe share about 0.31) at
  38.5 tok/s: PCIe shares of 0-0.75 measured 30-38 tok/s and CPU-lane counts of 4-23 measured 37-41, and none beat the
  engine's choice by the 3% calibration requires.
- Each of these moved Maya-M by under 1 tok/s: `RAM_RESIDENT`, `PROMOTE_MIN=6`, KDA graphs, pictures off, and a smaller
  VRAM reserve.
- v1.1.52 runs the DSA layers' feed-forward in CUDA graphs, an 8-9% faster decode in its release notes. That doesn't
  show here: the CPU lane takes about 10 of each token's 25 ms.
- All 12,096 experts fit in VRAM and RAM, with 0.00 SSD reads per token. The page-file warning at start is harmless.
- `START-MAYA.bat --setup` needs `--context 262144`, or it falls back to its menu, which defaults to 32K and stops at
  128K. A setup run rewrites the config, so the `env` block and `"port": 8081` must go back in afterwards.
- `switch-model.ps1` starts Maya's `server.py` directly, which compiles nothing. Update with `update-maya.ps1`, which
  builds the engines; a plain `git pull` would leave the new server running old engines.

**Calling it.**

- Thinking defaults to `max`, so set a level on every request. `"reasoning_effort": "low"` suits quick work. For
  careful work, use `high` and allow up to 15 minutes a task: one coding task at `high` took 20,000-35,000 tokens (14-25
  minutes) on Maya-M. `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off.
- **Thinking cap.** Maya closes the thinking at 32,768 tokens and makes it answer. `"thinking_budget"` in the config
  changes that, and 0 turns it off. The longest thinking in the tests was about 28,000 tokens.
- **Always send `"stop": ["<|assistant|>"]`.** GLM sometimes writes that token instead of ending its turn, then answers
  all over again, and Maya v1.1.52 doesn't stop on it. It showed on one prompt, in 3 of 4 tries; the stop string fixed
  it in 5 of 5.
- **JSON.** Since v1.1.50, `response_format` on `/v1/chat/completions` gives the model the schema, but nothing checks the
  answer. `/v1/responses` with `text.format` (a `json_schema` with `"strict": true`) checks it.
- Tools and pictures work. Going from thinking off to `max` mid-conversation re-reads the whole prompt; going from off to
  `low` doesn't.

**Speed.** Maya v1.1.46 against v1.1.52 (`ab.py`, greedy, thinking off):

| | v1.1.46 | v1.1.52 (in use) |
|---|---|---|
| Maya-S24, new prompts | 38.5 tok/s | 38.4 tok/s |
| Maya-S24, writing after a 33K-token prompt | 36.6 tok/s | 37.3 tok/s |
| Maya-S24, reading that prompt | 2,948 tok/s | 2,948 tok/s |
| Maya-M, new prompts | 21.8 tok/s | 22.8 tok/s |

- First token: about 1 s on a short prompt and 2.5-3 s on a few hundred tokens. A picture took 6.3 s.
- Long prompts (`deep-test.py`, Maya-S24): the first token after 44 s at about 125K tokens, 56 s at 150K, 75 s at 200K
  and 96 s at 250K. Writing held 36 tok/s at every size, and every hidden note was found. A follow-up reads only the new
  text, in about 1 s.

**Checks.** v1.1.52 gave the same greedy answers as v1.1.46, byte for byte. On the quality suite it passed tools 5/5,
pictures, facts 8/8, code 3/3 and the agent task, with 0 of 20 derailed chats. Reasoning was 7 of 8, from a typo in its
final line at temperature 0.6 after it had reversed both words correctly. The JSON task passes since v1.1.50. On the
tuned setup, the 3 coding tasks at `high` took 7.5-35 s and the agent task 29 s.

**Quirks.**

- On hard coding tasks it keeps re-deciding (`hmm, wait, actually`) and drafting code inside its thinking. A larger
  budget with no thinking cap didn't change that.
- Maya's Ollama-style API (`/api/tags`, `/api/pull`, `/api/delete` and others) is on by default, but switching models
  through it is refused here, because `switch-model.ps1` starts the server directly.

**Hard test** ([hard-test.md](hard-test.md)): 10 of 25 in the 24K-budget round at `high`. A rerun at its best settings
is in progress.

## 3. MiMo-V2.6-Flash-RL (strata-mimo)

A mixture-of-experts model with sliding-window attention and an MTP head; this is its RL checkpoint. Neither Strata nor
Project Maya runs it; strata-mimo, a third-party Strata fork, does. The notes don't record its total and active
parameter counts.

| | |
|---|---|
| Experts | 12,032 routed experts |
| Context | 262,144 tokens, the engine's limit; served at 256K |
| Input | Text only |
| Quant | The ggml project's MXFP4 GGUF of the RL checkpoint, 170 GB, at the revision the fork pins (`a5d1269`) |
| Engine | strata-mimo (MIT) at commit `268fdbc`, built from source with CUDA 13.4 and Visual Studio 2022 Build Tools, plus four local changes |
| Start | `switch-model.ps1 mimo`; ready in 10-30 s. The first answers run slow while the file cache warms up (9 tok/s, then 16) |
| API | `http://127.0.0.1:8082/v1`, model ID `mimo-v2.6-flash-rl`, with a web page on the same port |
| VRAM, RAM | 30.0 GB; RAM grows to about 157 GB as it runs |
| Experts in VRAM | About 1,500 at 256K, 12% |

**About the fork.** On 2026-11-01 it carried 57 commits by one author since leaving upstream Strata, measured only on a
96 GB card with 512 GB of DDR5-6400. Before building, its commits were compared with upstream: no binaries, and network
use limited to localhost, Hugging Face and pinned packages. The C++ wasn't reviewed line by line.

**Engine settings.**

- **`--ram-copy 0`** in `strata-mimo-mxfp4.json`. The engine asks for a pinned 150 GB copy of the experts, which this PC
  can't give: it stops with `pinned RAM for the experts: out of memory` about 150 s into loading. With `--ram-copy 0` it
  reads the experts through the file cache, and disk reads stay at 0 MB/s during answers once the cache is warm.
- **`"env": {"MIMO_TAIL_KEEP": "5"}`**, the tail skip below. MTP is on.
- The fork's setup rewrites `strata-mimo-mxfp4.json`, so both settings go back in after any setup run. Its `setup.py`
  also shares the installed Strata's `settings.json`, and given a new data folder it moves the other engines' `models`,
  `packs` and `mtp` folders into it. Run its setup with the settings folder pointed at a private directory.
- The engine uses one GPU only.

**Local changes**, saved as `strata-mimo-local.patch`:

- **A fix for derailed answers.** After a conversation longer than about 130 tokens, a new chat could reuse stale
  sliding-window keys, and the engine answered as if it couldn't see the prompt: gibberish, or a made-up task. The tool
  sequence that triggered it went wrong 2 times in 20 without any skip and 5 in 20 with one, then 0 in 40 after the fix.
  The engine now tracks how far it has written and reads the prompt again when the keys it needs are gone. The fork's
  own open fix (its PR #1) also turns off prefix reuse whenever MTP is on, which would re-read the whole conversation
  every turn. That part was left out, since stale MTP state only affects drafts, which the main model checks.
- **A bigger sliding-window ring**, 4,096 positions instead of 256, so ordinary turns keep their prefix reuse. It costs
  35 expert slots.
- **An expert tail skip**, `MIMO_TAIL_KEEP`: an expert missing from VRAM is left out when every token of the window
  ranks it 6th or lower. Unset, every expert is computed.
- **A looser start check.** The engine refused to start when free VRAM fell a few MiB under its 1,536 MiB margin. It now
  refuses below 1,280 MiB.

The tail skip's cost, as KL from the exact engine on 400 replayed tokens of three prompts (`mimo-kl.py`):

| `MIMO_TAIL_KEEP` | KL from the exact run | Same top token | Writing, new prompts |
|---|---|---|---|
| Off | 0 | 100% | 14.9 tok/s |
| 7 | 0.013 | 96.4% | 17.0 tok/s |
| 6 | 0.022 | 94.1% | 19.6 tok/s |
| 5 (in use) | 0.037 | 91.9% | 23.0 tok/s |

For scale, a community 3-bit MiMo quant sits about 0.17 from the original. MiMo's early layers turned out no more
sensitive than its late ones, unlike GLM's, so the skip applies to every layer.

**Calling it.**

- Thinking is on by default and has no levels. `"reasoning_effort": "low"` turns it off, as does
  `"chat_template_kwargs": {"enable_thinking": false}`; `medium`, `high` and `max` all mean on.
- **No thinking cap.** `max_tokens` is the whole budget: strata-mimo has Strata's wrap-up text but not its
  three-quarters rule.
- Thinking length varies a lot. The same clock puzzle took 1,106 tokens in one run (right) and 6,261 in another
  (wrong), and two more runs were still thinking at the test's 6,000-token limit. Allow about 5 minutes for a careful
  task.
- `response_format` works: the server asks for JSON and checks the result, but nothing constrains the writing.
  `tool_choice: "required"` isn't enforced: the model answered in text in two runs and called the tool in the third.
- No pictures.

**Speed.** With all four local changes, on fresh prompts at 256K: 25.5 tok/s on new prompts, 21 at 33K deep, and 1,030
tok/s reading a 35K-token prompt. Before the changes it wrote about 15 tok/s on new prompts and read about 950 tok/s.

- **First token.** Any prompt of 128 or more new tokens waits 5-13 s, so an agent turn costs about 10 s before it
  writes anything. Slower expert copies without the pinned copy are the likely cause, but the pinned mode can't run here
  to confirm it.
- **Long prompts** (`deep-test.py`): the first token after 171-259 s at about 125K tokens (the slow end on a cold file
  cache), 171 s at 150K, 232 s at 200K and 298 s at 250K, writing 26, 24, 24 and 24 tok/s. Every hidden note was found.
- **Context.** MiMo keeps its KV cache in VRAM. 256K costs 2.2 GB of VRAM and 10% of the expert slots against 64K, for
  5% less writing speed. A follow-up reads only the new text, in 1.5-2 s.
- In the hard test it wrote 26-48 tok/s, slowing to about 27 deep into an answer.
- **Settings tried** (800 tokens per run, thinking off, repeat runs): MTP on at 256K wrote 18.4 tok/s with 1,535 experts
  in VRAM, MTP off 18.1 (1,561), 64K context 19.4 (1,710), and a saved and reloaded learned profile 18.4 (1,535). First
  runs on a prompt were 15-16 tok/s in every case; repeats run faster because the experts in VRAM follow the work.
- Only about 1,500 of the 12,032 experts fit in VRAM at 256K, against about 6,600 on the fork author's 96 GB card. They
  start out covering about 40% of expert choices, and the CPU computes the rest from RAM, so RAM speed sets the pace.

**Checks.** With all four changes: facts 8/8, logic 8/8 (scored by hand: the script marked one answer wrong for stray
spaces), tool calls 5/5, the 3 coding tasks at `high` in 22-189 s, the agent bug-fix task in 85 s, and 0 of 20 derailed
rounds.

**Quirks.** A cold file cache costs speed: one baseline read 11.7 tok/s on its first run and 14.9 on the second. Disk
reads should stay at 0 MB/s during answers; if they don't, something else is taking the RAM.

**Hard test** ([hard-test.md](hard-test.md)): 12.9 of 25 in the 24K-budget round, with its last two coding tasks and
the repair job run at 128K.

## 4. DeepSeek-V4.1-Flash Coder (Project Maya's DeepSeek engine)

ISTA-DASLab's DeepSeek-V4.1-Flash-1017 GSQ-RCO Coder is DeepSeek's 552B model with half of each layer's 384 experts
removed. The kept experts were chosen for code, agent work and pictures. ISTA reports 94.8% of the original's SWE-bench
Verified score and 98.7% of its LiveCodeBench v6. It runs on the DeepSeek engine (`maya-ds4`) that ships with Project
Maya.

| | |
|---|---|
| Architecture | Mixture of experts: 192 of 384 routed experts kept per layer, 7,680 in all. Compressed attention keeps the KV cache small. An Engram n-gram table comes with the weights |
| Context | 1M (served) |
| Input | Text. The config runs with `--vision off` |
| Quant | ISTA-DASLab IQ2_XXS, revision `54229d3`: 72.1 GB of weights in five shards, plus the 110.6 GB Engram table as shard 6. `get-coder.py` downloads them and checks each SHA-256 |
| Engine | Project Maya v1.1.52, DeepSeek engine `maya-ds4`, with a local server change |
| Start | `switch-model.ps1 deepseek` (config `maya-ds4-coder.json`) |
| API | `http://127.0.0.1:8083/v1`, model ID `deepseek-v4.1-flash-coder` |
| VRAM | About 31 GB, flat across context sizes |
| RAM | About 43 GB of experts pinned at start; the KV cache also sits in RAM |
| Experts in VRAM | About 2,640 of 7,680 |
| Fallback | The llama.cpp router's `deepseek-v4.1-flash-coder` preset, 64K context |

**Setup on Maya.** Maya's setup offers only its own DeepSeek files, so this one was set up by hand. The engine was
compiled with Maya's own build commands for the `maya-ds4` target (`build-maya-ds4.bat`) and added to Maya's
`MAYA-BUILD.json`. Maya's `pack_gguf.py` indexed the files in place, Engram table included; it reads ISTA's types and the
pruned 192-expert layers as they are. `maya-ds4-coder.json` was written by hand from Maya's DeepSeek defaults:
`--tp 1 --threads on --vision off`, 1M context, reasoning effort `low` (50), port 8083. Maya's setup never rewrites it.

**The local server change** (`maya-text-think-end.patch`). Maya ends DeepSeek's thinking only at its special token, so a
model that quotes the tag isn't cut off. At effort `max`, this Coder writes `</think>` as plain text instead; Maya then
kept the whole answer as reasoning, and the first max-effort answers came back empty. With
`MAYA_DS4_TEXT_THINK_END=1`, which the `deepseek` target sets, the text tag ends the thinking too. Unset, Maya behaves as
shipped. `update-maya.ps1` carries the change to each new release and reports a conflict if it can't.

**Engine behaviour.**

- 2,600-2,700 of the 7,680 experts sit in VRAM; the other ~43 GB are copied into pinned RAM at start.
- The prompt path computes the experts that stay in RAM where they are, instead of copying them over PCIe. That's why it
  reads prompts 6-7 times as fast as llama.cpp. It also writes about 12% faster.
- The context costs almost nothing: Maya keeps DeepSeek's cache in RAM, and the compressed attention keeps it small.
- **No drafting.** Maya's 8 GB DSpark drafter is built for the full 384-expert model.
- **Restart once after each Maya update.** The pack's `lane-times.txt` is keyed by the engine's build time, so the first
  start after a new build re-times the PCIe link and starts on a decode path that streams some RAM experts over PCIe:
  1,345 experts in VRAM instead of 2,644, and 32.0 tok/s on new prompts and 35.6 on repeats. During that run the engine
  times both paths, finds streaming 3.1% slower and saves that, so the next start is back to normal.
- A start-up warning says 1.7 GB of VRAM is already in use, even with nothing else on the GPU. It's harmless. A healthy
  start logs about 2,640 of 7,680 experts in VRAM on its `ds4: CUDA0 layers` line.

**Calling it.**

- It thinks by default at effort 50 (`low`). `reasoning_effort` takes `low` (50), `high` (75), `max` (100), or a
  number from 1 to 100. `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off.
- **Thinking cap.** Maya's server budget applies unless the config turns it off, as the max-settings hard-test run did.
- Tools, `tool_choice: "required"` and `response_format` work.
- **Loops.** It can fall into endless repeats, such as a run of zeros, when an answer needs a long string of one
  character. Always set `max_tokens` and check `finish_reason`.

**Speed on Maya** (`ab.py` through `maya-ab.ps1`, greedy, thinking off):

| Context | New prompts | Writing after a 33K-token prompt | Reading that prompt | VRAM for the context |
|---|---|---|---|---|
| 64K | 33.4 tok/s | 31.2 tok/s | 2,922 tok/s (first token in 11 s) | |
| 256K | 34.8 tok/s | 30.9 tok/s | 3,357 tok/s | 0.24 GB |
| 512K | 35.1 tok/s | 28.8 tok/s | 3,361 tok/s | 0.49 GB |
| 1M (in use) | 33.5 tok/s | 30.6 tok/s | 3,328 tok/s | 0.97 GB |

- After the v1.1.52 update (restarted once): 33.2 and 34.3 tok/s on new prompts, 45.6-47.0 on a prompt asked again,
  and 3,199-3,230 tok/s reading the 33K prompt.
- Long prompts: 40-45 tok/s writing at 125K-250K deep, 3,300-4,300 tok/s reading, and every hidden note recalled up to
  250K.
- Strata 0.2.10 also runs this Coder (`--family deepseek41`, 256K). Over two runs it wrote 30.0-33.8 tok/s on new
  prompts and 27.8-29.0 after the 33K prompt, and read it at 1,526-2,483 tok/s, against Maya's 34.8, 30.9 and 3,357.
  Strata 0.1.54 couldn't run it, because its setup accepts only the files it pins.

**The llama.cpp fallback** (`ab.py` through `ds-ab.ps1`, build 12017):

| Placement | Context | New prompts | After a 33K prompt | Prompt reading |
|---|---|---|---|---|
| llama.cpp's own `--fit` | 16K | 21.7 tok/s | | |
| Every expert in RAM, 23.5 GB GPU cache of hot experts | 16K | 32.0 tok/s | | |
| Same, 22 GB cache, q8_0 KV cache (in use) | 64K | 29.8 tok/s | 29.7 tok/s | 472 tok/s |
| Same, 20 GB cache, f16 KV cache | 64K | 28.0 tok/s | 30.6 tok/s | 473 tok/s |
| Same, 19.5 GB cache, q8_0 KV cache | 128K | 26.8 tok/s | 28.7 tok/s | 447 tok/s |

- The preset: `n-gpu-layers = all`, `cpu-moe = true`, `moe-cache-mib = 22000`, 64K context, q8_0 K and V cache,
  `ubatch-size = 2048`, `parallel = 1`, `reasoning = on`, and `reasoning-effort = 50`, the effort ISTA's scores used.
- Each GB of GPU expert cache is worth about 1 tok/s, and each 64K of context takes about 2 GB of it. At 32.1-32.2 GB of
  VRAM the driver pages VRAM to RAM and prompts read at about 85 tok/s, which is why `-ub 4096` and a 21 GB cache at
  128K failed.
- Prompts read at about 470 tok/s because each batch copies experts over the x8 link. llama.cpp reads the Engram table
  from the SSD a row at a time.
- 12 to 24 CPU threads measured the same. `ngram-mod` stalled the server on a repeated prompt, so it's off. The first
  answer after loading runs at about 20 tok/s while the GPU expert cache fills, then 30-35.

**Checks.**

- On Maya v1.1.52: tools 5/5, JSON, the agent task in 60 s (77 s on llama.cpp), and 2 of 3 code tasks; the calculator
  ran out of the test's 8,000 tokens. Facts 7 of 8: it named neodymium for atomic number 74, as on llama.cpp. Reasoning
  8 of 8, though the checker missed one answer written as `\frac{1}{11}`. At effort `max`, 4 of 4 answers arrived in
  `content` (`max-effort-check.py`), so the local change works.
- Through the router: tools 5/5 including `tool_choice: "required"`, facts 7 of 8 (the eighth cut off at an 800-token
  cap), reasoning 8 of 8, long-prompt recall right at 8K and 32K, and all three thinking levels right.

**The loops are the model's own.** `loop-check.py` asks for "one followed by exactly 64 zeros". Every try wrote about
17,700 zeros until the cap, with the GPU expert cache and without it, at temperature 0.6 and 1.0. DRY sampling
(`dry_multiplier 0.8`) cut the runs to 28 zeros, but the answers still failed, so sampling stays at the defaults.

**Hard test** ([hard-test.md](hard-test.md)): 19 of 25 at max settings (effort `max`, a 384,000-token budget, Maya's
thinking cap off), coding 2 of 7, 298 minutes.

## 5. Qwen3.8-27B (llama.cpp router)

| | |
|---|---|
| Architecture | Dense, 27B. 64 layers, Gated DeltaNet and full attention at 3:1, so only 16 layers keep the whole context |
| Context | The card allows 262,144 tokens of thinking plus a 131,072-token answer; served at 208K |
| Input | Text and pictures |
| Quant | Unsloth UD-Q6_K, 22 GB, plus the model's MTP head at Q4_0, 1.4 GB |
| Engine | llama.cpp router, build 12017 |
| Start | `switch-model.ps1 router`; the router loads the model on its first request, in about 17 s |
| API | `http://127.0.0.1:9931/v1`, model ID `unsloth/Qwen3.8-27B-GGUF:Q6_K` |
| VRAM, RAM | 31.0 GB; a few GB of RAM |

Pull it with `pull-model.cmd unsloth/Qwen3.8-27B-GGUF:UD-Q6_K`. The MTP head comes from
`pull-model.cmd unsloth/Qwen3.8-27B-GGUF -hff MTP/mtp-Qwen3.8-27B-Q4_0.gguf --no-mmproj`, moved into the `draft`
folder.

**The preset** (`presets.ini`): device `CUDA0`, all layers on the GPU, `ctx-size = 212992`, q8_0 K and V cache,
`ubatch-size = 512`, `no-mmproj-offload = true`, `parallel = 1`, `spec-type = draft-mtp` with the MTP head and
`spec-draft-n-max = 2`, temperature 1.0, top-p 0.95, top-k 20, min-p 0.

- **MTP drafting.** The model's own head drafts 2 tokens a step. It needs `parallel = 1`, so concurrent requests queue.
- **208K context.** Only 16 of the 64 layers keep the whole context, about 4.3 GB per 128K at q8_0. Moving the picture
  encoder to the CPU and shrinking the prompt batch from 2048 to 512 freed about 1.7 GB, for 7% slower prompt reading.

**Speed.** Context and placement (`ab.py` through `qwen-ab.ps1`, greedy, thinking off, build 12017):

| Setting | VRAM | New prompts | After a 33K-token prompt | Reading prompts |
|---|---|---|---|---|
| 128K, pictures on the GPU, batch 2048 (before) | 29.9 GB | 128 tok/s | 101 tok/s | 2,830 tok/s |
| 192K, pictures on the CPU, batch 2048 | 32.0 GB, too close to paging | 128 tok/s | 103 tok/s | 2,810 tok/s |
| 208K, pictures on the CPU, batch 512 (in use) | 31.0 GB | 128 tok/s | 103 tok/s | 2,630 tok/s |
| 224K, pictures on the CPU, batch 512 | 31.8 GB, too close to paging | 128 tok/s | 101 tok/s | 2,615 tok/s |

- With thinking on at the card's sampling (`spec-sweep.py`): 64 tok/s without drafting, 89 drafting 1 token, 99
  drafting 2 or 3, and 98 drafting 4. The preset keeps 2, where half the drafts are accepted.
- On build 12000 at 128K: 65 tok/s plain and 107 with MTP on short prompts, 57 (at 32K) and 96 (at 43K) deep, and about
  3,400 tok/s reading prompts (2,500-2,700 at 32K-43K deep).
- In the hard test it wrote 788K tokens at about 99 tok/s.

**Calling it.**

- It thinks by default at `xhigh`. `reasoning_effort` takes `low`, `medium` or `xhigh`; `high` counts as `xhigh`, and
  any other value fails the request. `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off.
- The preset uses the card's thinking-mode sampling: temperature 1.0, top-p 0.95, top-k 20, min-p 0. Without thinking,
  the card suggests temperature 0.7, top-p 0.8 and presence penalty 1.5.
- Its template keeps all earlier thinking by default (`preserve_thinking`). Send `reasoning_content` back with earlier
  assistant messages, above all those with tool calls.
- Tools, `response_format` and pictures work. The picture encoder runs on the CPU.
- Hard tasks took it 60K-109K tokens of thinking at about 100 tok/s. Leave `max_tokens` out and allow 20 minutes or
  more a task.

**Quirks.** The whole model and its context sit in VRAM, so it's the strongest choice here when RAM has to stay free.

**Hard test** ([hard-test.md](hard-test.md)): 20 of 25, coding 4 of 7, 130 minutes.

## 6. Nanbeige4.3-33B (llama.cpp router)

| | |
|---|---|
| Architecture | Dense, 33B, Llama architecture. 52 layers, GQA 56/8, full attention in every layer. Vocabulary 166,912 |
| Context | The lab allows up to 262,144 output tokens. Served at 80K. Its hard-test runs with room used 96K (98,304), the most that fits beside the weights with a q8_0 cache |
| Input | Text only |
| Quant | bartowski Q4_K_M, 20.8 GB |
| Engine | llama.cpp router, build 12017 |
| Start | `switch-model.ps1 router`; the model loads in about 10 s |
| API | `http://127.0.0.1:9931/v1`, model ID `bartowski/Nanbeige_Nanbeige4.3-33B-GGUF:Q4_K_M` |
| VRAM | 29.6 GB at 80K; 31.4 GB at 96K |
| RAM | Little: the whole model sits in VRAM |

**The preset** (`presets.ini`): device `CUDA0`, all layers on the GPU, q8_0 K and V cache, `ubatch-size = 2048`,
`parallel = 1`, `spec-type = ngram-mod`, temperature 1.0, top-p 0.95.

**Speed** (`ab.py`, greedy, thinking off, through a standalone `llama-server` with the preset's settings,
`nanbeige-ab.ps1`):

| Setting | New prompts | Answer to a 39K-token prompt | Same answer asked again | Context that fits |
|---|---|---|---|---|
| q8_0 cache, no drafting | 69 tok/s | 38 tok/s | 69 tok/s | 80K (29.6 of 32.6 GB VRAM) |
| q8_0 cache + `ngram-mod` (in use) | 68 tok/s | 38 tok/s | 986 tok/s | 80K |
| f16 cache | 70 tok/s | 52 tok/s | 70 tok/s | about 48K |
| f16 K with q8_0 V, or the reverse | 69 tok/s | 36 tok/s | 69 tok/s | about 64K |

- Prompts read at about 2,050 tok/s at 39K tokens and 1,800 at 52K.
- 69 tok/s is about three quarters of what the 5090's memory speed allows for 20 GB of weights.
- `ngram-mod` drafts from text the server has seen before. It costs about 1% on new text, and wrote 14 times faster when
  an answer repeated earlier text. `ngram-cache` cost 3% on new text; `ngram-simple` and `ngram-map-k` gained nothing.
- The f16 cache is faster for long thinking on short prompts, but it halves the context. The preset keeps q8_0 because
  the hard test's longest prompt is 52.5K tokens.
- No draft model fits. Nanbeige4.2-3B has a different vocabulary (166,144 tokens against 166,912) and a looped
  architecture, and no DFlash, EAGLE or DSpark drafter existed for the 33B as of 2026-11-01. Nothing like Strata applies
  to a dense model that fits in VRAM.

**Calling it.**

- It always thinks. `reasoning_effort` changes nothing. `"chat_template_kwargs": {"enable_thinking": false}` turns
  thinking off.
- The lab suggests temperature 0.6 and top-p 0.95 for reasoning and chat, and 1.0 for agent work. The preset's 1.0 is
  the agent setting.
- It thinks much longer than the others: all 7 hard coding tasks ran out of a 24,000-token cap while it was still
  working. Allow 60,000 tokens or more for hard work.
- After a tool result, about 2 replies in 3 arrive with an empty `content` and the answer in `reasoning_content`: the
  model writes a second `<think>` where `</think>` belongs. When `content` is empty, read `reasoning_content`.
- `response_format` and `tool_choice: "required"` fail at once with HTTP 400 (`Failed to initialize samplers`) on build
  12017. Ask for JSON in the prompt instead.

**Checks.** With room to think (96K context), it ended three hard coding tasks by itself instead of running out of
budget, and solved one: it was still reasoning, not looping, when the 24K budget cut it off. In the 24K round it looped
once, repeating one line until its budget ran out.

**Hard test** ([hard-test.md](hard-test.md)): 18 of 25 at its best (19 counting a print slip), coding 3 of 7, 412
minutes, at the card's reasoning settings with top-k and min-p off and a 96K context. In the 24K-budget round it scored
9.8 (11.8 counting two print-format slips).

## 7. Gemma 4 26B-A4B and 31B (llama.cpp router)

Google's Gemma 4 (released 2026-04-02, Apache 2.0). Two of its five sizes fit the 5090. Both run in the router, each with
its own MTP drafter.

| | Gemma 4 26B-A4B | Gemma 4 31B |
|---|---|---|
| Architecture | Mixture of experts, 3.8B of 25.2B parameters active | Dense, 31B. 60 layers, 10 of which see the whole context |
| Input | Text and pictures | Text and pictures |
| Quant | The original weights at Q8_0, 26.9 GB | Google's quantization-aware (QAT) 4-bit, as Unsloth's UD-Q4_K_XL, 17.3 GB |
| Drafter | `mtp-gemma-4-26B-A4B-it.gguf`, 0.5 GB | `mtp-gemma-4-31B-it-qat-Q4_0.gguf`, 0.3 GB |
| Context | 224K (229,376), q8_0 cache | 224K (229,376), q8_0 cache |
| VRAM | 31.4 GB | 31.6 GB |
| Model ID | `unsloth/gemma-4-26B-A4B-it-GGUF:Q8_0` | `unsloth/gemma-4-31B-it-qat-GGUF:Q4_K_XL` |
| Pull tag | `Q8_0 --mtp` | `UD-Q4_K_XL --mtp` |

Both start with `switch-model.ps1 router`, answer on `http://127.0.0.1:9931/v1`, and use a few GB of RAM. Both presets
set device `CUDA0`, all layers on the GPU, q8_0 K and V cache, `no-mmproj-offload = true`, `parallel = 1`,
`spec-type = draft-mtp`, `spec-draft-n-max = 4`, `reasoning = on`, and Google's sampling.

**Why these files.** In Unsloth's own table, the 31B's QAT 4-bit has 96.7% top-1 agreement with its reference and a
mean KLD of 0.014, and Unsloth says more bits of these weights don't help. The 26B's QAT 4-bit agrees on only 85.6%, so
the 26B runs at Q8_0. That 4-bit file (15.7 GB) stays on disk without a preset: it's faster, 285 tok/s plain and 431
with its drafter (at 256K with an f16 cache and pictures on the GPU, 21.7 GB), but less faithful to the model.

**Speed** (`ab.py`, greedy, thinking off; `spec-sweep.py`, Google's sampling, thinking on):

| | 26B-A4B | 31B |
|---|---|---|
| Writing with thinking on, Google's sampling | 311 tok/s (202 without the drafter) | 170 tok/s (72 without the drafter) |
| New prompts, thinking off | 318 tok/s | 184 tok/s |
| Writing after a 38K-token prompt | 222 tok/s | 104 tok/s |
| Reading prompts | about 8,000 tok/s | about 2,550 tok/s |

Writing speed by drafted tokens per step, at Google's sampling with thinking on:

| Drafted tokens | 2 | 3 | 4 (in use) | 5 | 6 |
|---|---|---|---|---|---|
| 26B-A4B | 296 tok/s | 303 tok/s | 311 tok/s | 301 tok/s | 288 tok/s |
| 31B | 142 tok/s | 157 tok/s | 170 tok/s | 169 tok/s | 158 tok/s |

- At 4 drafted tokens, 58% (26B) and 60% (31B) of drafts are accepted. llama.cpp checks drafts token by token against
  the model's own sampling, so the answers stay the same. At temperature 0, more drafted tokens kept helping, up to 188
  tok/s at 5 on the 31B.
- A picture reached its first token in 2.1 s.

**VRAM sets the context.** Each 128K of context costs the 31B about 6.3 GB at q8_0, even though only 10 of its 60 layers
see the whole context, and the drafters take another 0.5-1.4 GB. At 256K, or with an f16 cache, both models pass about
32 GB and the driver pages VRAM into RAM: the 31B then wrote at 20 tok/s and read prompts at 780 tok/s. The picture
encoder runs on the CPU to make room.

**Calling it.**

- Google's sampling for every Gemma 4: temperature 1.0, top-p 0.95, top-k 64, min-p 0. llama.cpp's own min-p default of
  0.05 cuts further. Keep it for hard work: at 0.6, the 31B coded no better.
- Both think by default. `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off. Gemma 4 has no
  thinking levels, so `reasoning_effort` changes nothing.
- In a tool loop, send each assistant message that has tool calls back with its `reasoning_content`. The template
  replays that thinking until the next `user` turn, as Google asks.
- Tools, `tool_choice: "required"`, `response_format` and pictures work. Pictures are capped at 512 tokens of Gemma's
  1,120, because a picture has to fit one 512-token batch; a larger batch would cost context.
- The drafter needs a single slot, so requests queue.
- Leave `max_tokens` out for hard work. The longest hard-test answer was 59K tokens.

**Checks** (the quality suite at Google's sampling). The 31B passed everything: tools 5/5, JSON, pictures, facts 8/8,
reasoning 8/8, code 3/3, and the agent task in 12 s. The 26B did the same, except that one reasoning puzzle and the
calculator ran out of the test's 6K and 8K caps; its agent task took 18 s. Both had 0 of 20 derailed chats.

**Quirks.** Gemma solves puzzles with little thinking: the 31B did both routes and the knapsack in 10-14K tokens each.
Its coding failures are real bugs, not budget: every coding answer ended by itself after 1.6K-24K tokens.

**Hard test** ([hard-test.md](hard-test.md)): 26B-A4B 18 of 25, coding 2 of 7, 22 minutes. 31B 17 of 25 (18 counting a
print slip), coding 1 of 7, 18 minutes.

## 8. Inkling-Small (maya-ink)

Thinking Machines' Inkling-Small (released 2026-07-30, Apache 2.0). Mainline llama.cpp can't run it, and neither can
Strata or Project Maya. llama.cpp's open draft PR #25731 can, at about 9 tok/s. It runs here on maya-ink, a custom
engine in a Project Maya fork (branch `inkling`, `src/ink`, documented in the fork's `docs/INKLING.md`), behind Maya's
own server.

| | |
|---|---|
| Architecture | Mixture of experts, 276B total, 12B active. 42 layers: the first 2 with a dense SwiGLU FFN, the other 40 MoE with 256 routed experts (2,048 wide, top 6) and 2 shared experts. Width 4,096; 32 query heads and 8 KV heads of 128 |
| Attention | No RoPE. 35 sliding-window layers (window 512) and 7 global layers (every sixth), with a learned relative-position bias band. 4 short causal convolutions per layer |
| Context | 1M positions; served at 128K |
| Input | Text, pictures and audio. maya-ink reads text only so far |
| Vocabulary | 201,024 |
| Quant | Unsloth UD-Q4_K_XL, 163 GB: dense matrices at Q8_0 (6.5 GB), routed experts at Q4_K to Q6_K (150 GB) |
| Start | `switch-model.ps1 inkling`; ready in about 16 s |
| API | `http://127.0.0.1:8084/v1`, model ID `inkling-small` |
| VRAM | 29.9 GB |
| RAM | Working set about 164 GB; about 22 GB stays free |
| Expert slots in VRAM | 1,601: 1,104 of its own, 259 in the prompt path's buffers, 238 in the K/V caches' rows |

**The engine.**

- **Token path.** Everything but the routed experts runs on the GPU, with int8-activation GEMVs from Maya's DeepSeek
  kernels. The host picks the experts from the router's logits. Experts found in the VRAM cache run on the GPU while the
  CPU lane computes the rest straight from the GGUF's mapping. Each MoE layer runs as one CUDA graph between host syncs.
- **CPU lane.** One spinning thread pool per layer, at about 0.30 ms per expert: about 50 GB/s, what this RAM gives.
- **Prompt path.** Chunks of up to 8,192 tokens, a layer at a time on the GPU, so each expert crosses PCIe once a chunk
  instead of once a 512-token batch. Copies go through the engine's own pinned staging at 14 GB/s, where the driver's
  staging of pageable memory managed 4-15 GB/s. The link itself does 28.7 GB/s from pinned memory, but Windows won't
  pin 150 GB of the mapping. Prompts under 48 new tokens take the token path.
- **VRAM expert cache.** Slots are sized for each layer's experts. The prompt path's 3.9 GB of buffers and the global
  layers' unused K/V rows (3.6 GB at a conversation's start) are lent to the cache between prompts, and the embedding
  table stays in RAM. Scores decay 0.99 a token; every 2 tokens, up to 16 experts are swapped in, at a margin of 1.5.
  Swap copies go up while the CPU waits on the GPU.
- **Sampling** draws from the whole distribution in one pass, in 0.4 ms. Sorting the 200K-token vocabulary had cost
  13 ms a token.
- **RAM.** At start the engine touches all 157 GB of routed experts, in about 11 s, and keeps them in its working set.
  On Windows a page's first touch costs a fault even with the file cached, which had held prompt copies to 3.4 GB/s.

**Calling it.**

- `reasoning_effort` takes `none`, `low`, `medium`, `high` (the default, Inkling's own 0.9), `xhigh` and `max` (0.99,
  the level Thinking Machines evaluated at). Use `max` for hard work. `minimal` counts as off, and
  `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off.
- Sampling: the card's evaluation settings, temperature 1.0 and top-p 1.0. The run config sets them.
- **No thinking cap.** Inkling's thinking doesn't end at a special token, so Maya's server has nothing to close it with.
  Cap long work with `max_tokens`.
- Tools, a named `tool_choice`, `response_format` and streaming work. `tool_choice: "required"` can't be forced,
  because an Inkling tool call starts with the tool's name. No pictures or audio yet.

**Speed** (`ab.py`, greedy, thinking off, and `ink-sampling.py` at the card's sampling; maya-ink's runs each started
from the same expert profile by `ink-ab.ps1`, averaging 4 runs of the first version and 3 of the current one; llama.cpp
built at the PR's commit `48f7e71`):

| | llama.cpp PR, `--cpu-moe` | llama.cpp PR, 14 GB expert cache, `-ub 2048` | maya-ink, first version | maya-ink (in use) |
|---|---|---|---|---|
| Writing, new prompts | 9.1 tok/s | 9.0 tok/s | 14.5 tok/s | **30.2 tok/s** |
| The same at the card's sampling | | | 12.6 tok/s | **31.1 tok/s** |
| Writing after a 31.8K-token prompt | 9.6 tok/s | 9.7 tok/s | 15.6 tok/s | **26.3 tok/s** |
| Reading that prompt | 72 tok/s | 207 tok/s | 733 tok/s | **794 tok/s** |
| First token | 441 s | 154 s | 43.5 s | **40.4 s** |
| VRAM | 11.6 GB | 26.8 GB | 29.9 GB | 29.9 GB |

**Held-out check.** The cache settings were chosen by replaying `ab.py`'s routing offline with `ec-sim.py`, which
predicted the engine's CPU experts a token within 0.2%. `ab-heldout.py` is `ab.py` with other prompts: Go, SQL and Java
code, biology, French, German and Japanese, a C++ repeat, and 21.7K tokens of Python as the long prompt. Every run
started from a profile that had seen neither set. Writing speed in tok/s, `ab.py` / held-out, 3 runs each:

| | New prompts | After the long prompt | Repeated prompt, 3rd time |
|---|---|---|---|
| maya-ink, first version | 14.5 / 14.1 | 15.6 / 16.3 | 16.1 / 19.4 |
| Current build, first version's cache settings (decay 0.998, 4 swaps every 8 tokens) | 22.7 / 20.8 | 24.7 / 29.2 | 27.7 / 37.8 |
| Current build (in use) | **30.2 / 30.2** | 26.3 / 26.3 | **38.7 / 46.6** |

Right after a long prompt, the new settings gain nothing: on three long prompts (`ink-deep.py`) they wrote 24.2 tok/s
against 25.4 for the old ones, though the replay gives both 71% of the experts in VRAM there. Why is still open.

**What doubled the writing speed.** The cache grew from 871 to 1,601 slots and now follows what the conversation uses
right now. The attention no longer walks its keys one at a time, which had cost up to 8 ms a token. Each layer runs as
one CUDA graph, and sampling no longer sorts the vocabulary. A token now spends about 27 ms in the CPU lane (about 90
experts), about 6.5 ms on attention and routing and about 1.5 ms on the rest; the first version took 60-75 ms. About 70%
of a token is still the CPU's share of the experts, read at the RAM's full speed, so more speed needs more of the experts
in VRAM, or faster RAM.

Tried without gain: a `cudaStreamQuery` flush after the launches, fusing a layer graph's 22 nodes into 13 (about 1%),
and one block for all 4,096 channels' convolutions (slower). `cudaHostRegister` on the mapping "succeeds" without
speeding anything up. Under WDDM, a layer's graph takes about 115 µs on the GPU plus about 40 µs of submission and
wake-up.

In the hard test it wrote 671K tokens at 40 tok/s on average, above `ab.py`'s 30. That's likely the cache adapting
during long answers: a repeated prompt reaches 39-47 tok/s by its third run.

**Checks.**

- **Against llama.cpp** (`ink-ref.py`: the same token ids fed to both, greedy, each step's top 5 compared). The chat
  template's text and the tokenizer's ids matched exactly on 4 prompts of 27 to 3,054 tokens. The top token matched at
  87 of 88 positions; the other was a near tie (-0.68 against -0.69). The log-probabilities of llama.cpp's tokens were
  within 0.34, and 69 of 88 within 0.01.
- `ink-chat-test.py` through the server: thinking kept separate, thinking off, a tool call, the answer from its result,
  streaming and JSON, 6 of 6. `derail.py`: 0 of 20 rounds derailed, in 291 s against 406 on the first version.

**Quirks.**

- A/B runs must start from the same saved profile (`ink-ab.ps1` restores it); otherwise each run teaches the next.
- A new binary copied right after a stop can fail to replace the old one while the engine exits. Check its hash before
  measuring.
- The log is appended at each start. A healthy start logs `1104 expert slots`, `259 more in the prompt path's buffers`,
  `238 in the K / V caches' rows`, and `the routed experts' 156.8 GB touched` in about 11 s.
- After a change in the fork, rebuild with `build-maya-ink.bat`. `make_pack.py` rewrites the tokenizer folder from the
  GGUF.

**Hard test** ([hard-test.md](hard-test.md)): 21 of 25 at reasoning effort `max`, coding 3 of 7, 279 minutes.

## 9. Nemotron 3.5 Super VL (maya-nem)

NVIDIA's Nemotron 3.5 Super VL (released 2026-10-15, OpenMDW-1.1). Mainline llama.cpp (build 12017) runs it, pictures
included. maya-nem, a custom engine built on maya-ink's machinery, runs its text side about three times as fast. It
lives in the same Project Maya fork, on branch `nemotron` (`src/nem`, documented in the fork's `docs/NEMOTRON.md`).

| | |
|---|---|
| Architecture | Hybrid Mamba-2 / attention mixture of experts, 120B total, 12B active. 88 layers: 40 Mamba-2, 40 MoE and 8 attention, plus one NextN (MTP) block |
| MoE | 512 routed experts in a 1,024-wide latent space, top 22, relu², plus one shared expert |
| Mamba-2, attention | Mamba-2: 128 heads of 64, state size 128, 8 groups. Attention: 32 query heads and 2 KV heads of 128, no RoPE, 8 KiB of K/V a token |
| Context | 1M; served at 256K |
| Input | Text, pictures and video. maya-nem reads text only; the router preset reads pictures |
| Vocabulary | 131,072 |
| Quant, default | Unsloth UD-Q3_K_XL, 65.8 GB, with 55.6 GB of routed experts |
| Quant, alternative | Unsloth UD-Q4_K_XL, 86.9 GB, with 75.8 GB of routed experts. The picture encoder is 1.7 GB |
| Start | `switch-model.ps1 nemotron` (UD-Q3_K_XL) or `switch-model.ps1 nemotron-q4` (UD-Q4_K_XL); ready in about 10 s from a warm file cache |
| API | `http://127.0.0.1:8085/v1`, model ID `nemotron-3.5-super` for both quants |
| VRAM | 29.9 GB |
| RAM | Q3: about 62 GB, with about 113 GB free. Q4: working set about 84 GB, with about 95 GB free |
| Expert slots in VRAM | Q3: 8,526 of 20,480 (6,555 of its own, 1,283 in the prompt path's buffers, 688 in K/V rows). Q4: about 6,100 |

**Why these quants.** The routed experts are read from RAM for every token, so their size sets the speed. UD-Q4_K_XL
came first: its experts average about 5.4 bits (up rows Q5_K, Q4_K or Q6_K; down rows Q5_0 or IQ4_NL), close to the
original, and about 95 GB of RAM stays free. Q8_0 (131.6 GB) would read 1.6 times the bytes. UD-Q3_K_XL, despite its
name, is mostly 4-bit IQ formats: its experts are IQ4_NL (28.5 GB), IQ4_XS (15.7), IQ3_XXS (8.1), IQ2_XS (1.6) and Q2_0
(1.6). About 8,500 experts fit in VRAM instead of 6,100, and each one read from RAM is smaller. It writes 43% faster and
its top token is Q4's 97% of the time, so it became the default. `pdl.py` downloaded the files over 24 ranged
connections at 70-80 MB/s and checked each SHA-256.

**The engine.**

- **Token path.** Everything but the routed experts runs on the GPU. The Mamba-2 conv and selective scan are
  maya-nem's own kernels, and the attention uses maya-ink's split-key decode. Experts found in VRAM run on the GPU, in
  the latent space, while the CPU lane computes the rest; the shared expert runs on the GPU. Between the host's 40 syncs
  a token, the GPU's work runs as 41 CUDA graphs.
- **Prompt path.** Chunks of 16,384 tokens (`--chunk 16384` in the run config), a layer at a time on the GPU, so each
  expert crosses PCIe once a chunk through the engine's pinned staging. Prompts under 32 new tokens take the token path.
- **VRAM expert cache.** maya-ink's settings: decay 0.99 a token, up to 16 swaps every 2 tokens, margin 1.5.
- **Q2_0.** maya-nem's GPU kernels gained `Q2_0` (ggml's type 42) for UD-Q3_K_XL. With that build, Q4's output stays
  bit for bit the same.
- **AVX2 Q2_0 rows**, on since 2026-11-05 through the Q3 run config's `"env": {"DS4_Q2_0_AVX2": "1"}`. ggml's CPU dot
  has Q3's Q2_0 down rows only in scalar code; this AVX2 path, adapted from llama.cpp's x86 kernels (MIT), replaces it.
  Maya puts a config's `env` over the caller's, so a run without the rows needs that line out of the config.
- **Start.** The engine touches all routed experts: Q3's 55.6 GB in about 4 s, Q4's 75.8 GB in about 5-6 s, from a warm
  file cache. A cold cache takes minutes.

**Calling it.**

- It thinks by default. `reasoning_effort` `low` or `medium` (or `minimal`) turns on NVIDIA's lighter "efficient" mode:
  about 2.2 times fewer thinking tokens for about 9% lower accuracy, in NVIDIA's report. The other levels keep the full
  default. `"chat_template_kwargs": {"enable_thinking": false}` turns thinking off.
- Sampling: the card's text settings, temperature 1.0 and top-p 0.95. The run config sets them.
- **Thinking cap.** Maya's server closes the thinking at the first sentence or line end past 32,768 tokens, and the
  answer follows. `thinking_budget` in the run config changes the limit.
- Tools (Qwen3-Coder's XML format), `response_format` and streaming work.
- **Next turns.** The Mamba-2 state can't be rewound, so a prompt is read on from where the engine is only when it
  continues the tokens so far. A tool result does: `nem-turns.py` read on 473 of its 511 tokens. A new `user` message
  doesn't, because the template drops the earlier thinking, so the whole conversation is read again, about 20 s per 32K
  tokens. Keep long agent work in one tool loop where you can.

**Speed** (`ab.py`, greedy, thinking off, on a quiet machine; maya-nem's runs each started from the same expert profile
by `nem-ab.ps1`, 3 runs each at 16K chunks):

| | llama.cpp, `--cpu-moe` | llama.cpp's own fit, `-ub 2048` | maya-nem, UD-Q4_K_XL | maya-nem, UD-Q3_K_XL | Q3 with the AVX2 rows (in use) |
|---|---|---|---|---|---|
| Writing, new prompts | 11.2 tok/s | 13.0 tok/s | 26.3 tok/s | 37.7 tok/s | **41.3 tok/s** |
| Writing after a 34.2K-token prompt | 11.4 tok/s | 13.5 tok/s | 21.7 tok/s | 30.6 tok/s | **34.1 tok/s** |
| Reading that prompt | 107 tok/s | 474 tok/s | 1,607 tok/s | 1,981 tok/s | **1,973 tok/s** |
| First token | 319 s | 72 s | 21 s | 17 s | **17 s** |
| A repeated prompt, 1st and 3rd time | 11.4, 11.4 tok/s | 13.5, 13.5 tok/s | 23.9, 29.3 tok/s | 35.0, 41.1 tok/s | **38.1, 45.0 tok/s** |
| VRAM | 12.2 GB | 30.3 GB | 29.9 GB | 29.9 GB | 29.9 GB |

The llama.cpp columns are UD-Q4_K_XL. The AVX2 column was measured on a build with the same code before the change was
merged; the rebuilt serving binary wasn't measured again.

- The AVX2 rows wrote 41.2-41.4 tok/s (3 runs) against 37.6-37.7 (2 runs) on new prompts, and 33.5-34.5 against
  30.3-31.9 after the 34K prompt. With thinking on at the card's sampling (`spec-sweep.py`, 2 runs each), they wrote 41.7
  and 42.4 tok/s against 38.8 and 38.7. Prompt reading didn't change.
- In the hard test it wrote 472K tokens at 40 tok/s on average.

**Settings measured.**

- On UD-Q4_K_XL, maya-ink's tuned cache settings beat its earlier ones (decay 0.998, 4 swaps every 8 tokens, margin
  1.25): 26.6 against 22.1 tok/s on new prompts and 29.6 against 22.9 on the third repeat, 2 runs each. After the long
  prompt they were equal.
- 16K prompt chunks read prompts 42% faster than 8K (1,607 against 1,130 tok/s), for about 1% less writing speed (26.3
  against 26.6 tok/s on Q4).
- The CUDA graphs gained about 5%. About 80% of a token is the CPU's share of the experts, read at the RAM's speed of
  about 40 GB/s.
- **MTP is correct but slower.** On Q3 with the AVX2 rows, drafting with the GGUF's NextN head (`--mtp N`) passed every
  exactness check, but wrote 28.4 tok/s at 4 drafts, 32.0 at 2 and 32.3 at 1, against 41.3 without it. The head guesses
  well (194 of 205 single drafts accepted on a repeated prompt), but each drafted token still reads its own experts from
  RAM. The head also takes 3.7-4.2 GB of VRAM, so the expert slots drop from 6,555 to 4,911 at 4 drafts, and the drafts
  are checked one token at a time without CUDA graphs. Prompts read 10% slower with it on (1,785 against 1,973 tok/s).
  It isn't used.

**Checks.**

- **Q4 against llama.cpp** (`nem-ref.py`: the same token ids fed to both, greedy, each step's top 5 compared). The
  template's text and the tokenizer's ids matched exactly on 4 prompts of 28 to 3,621 tokens. The top token matched at
  88 of 88 positions through the prompt path and 47 of 48 through the token path alone; the miss was a near tie (-0.688
  against -0.703). The log-probabilities were within 0.067. The CUDA graphs give the same bits.
- **Q3 against llama.cpp on Q3:** the top token matched at 86 of 88 positions, the other two being near ties, with
  log-probabilities within 0.141.
- **Q3 against Q4** (`q3-agree.py`, fed Q4's greedy continuations of 3 prompts): Q3's top token was Q4's at 949 of 976
  positions (97.2%). Q4's token's log-probability was 0.026-0.030 apart on average and 1.07 at worst.
- **AVX2 rows:** the top token is the scalar build's at 88 of 88 positions and llama.cpp's at 86 of 88, as before. Long
  greedy runs part ways at some point, because the AVX2 sums round differently from the scalar path's. The row test
  (`ds4_q2_0_rows_test`) passes for the AVX2 path; the unchanged scalar path misses the test's tight tolerance in 15 of
  3,960 checks.
- `nem-chat-test.py` through the server: thinking kept separate, thinking off, a tool call, the answer from its result,
  streaming and JSON, 6 of 6. `derail.py`: 0 of 20 rounds derailed, on Q4, on Q3 and with the AVX2 rows.

**The router preset for pictures.** `nemotron-3.5-super-vl` in `presets.ini` runs UD-Q4_K_XL with the BF16 picture
encoder (`mmproj-BF16.gguf`) through llama.cpp: `no-mmproj-offload = true`, 256K context, batch and ubatch 2048,
`parallel = 1`, temperature 1.0, top-p 0.95, min-p 0. It writes 13.0 tok/s on new prompts and 13.5 after a 34K prompt,
reads prompts at about 474 tok/s, and uses 30.3 GB of VRAM at 256K. Leave its fit alone: given `-ngl 99` without
`--cpu-moe`, llama.cpp put the whole model "on" the GPU, filled 32.0 GB, spilled the rest to shared memory and wrote
3.0 tok/s.

**Hard test** ([hard-test.md](hard-test.md)): UD-Q3_K_XL 17 of 25 (18 counting an answer given outside its code block),
coding 0 of 7, 195 minutes. UD-Q4_K_XL hasn't run it.

## 10. Ling-3.1-flash (maya-ling)

inclusionAI's Ling-3.1-flash (`1019` build, on Hugging Face since 2026-10-13, MIT). Mainline llama.cpp (build 12017)
runs it as the `bailingmoe3` architecture, at about half maya-ling's writing speed; neither Strata nor Project Maya did.
maya-ling, a custom engine on maya-ink's machinery, lives in the same Project Maya fork on branch `ling` (`src/ling`,
documented in the fork's `docs/LING.md`). There's no router preset: llama.cpp reads text only too.

| | |
|---|---|
| Architecture | Mixture of experts, 564B total, 25B active. 72 layers: 63 with Kimi Delta Attention (linear attention with a fixed-size state) and 9 with gated MLA (layers 7, 15, ..., 71). The first 4 layers have a dense SwiGLU FFN; the other 68 are MoE with 512 routed experts (1,280 wide) and 1 shared. Plus one NextN (MTP) layer, not run |
| Routing | Group-limited: 8 groups of 64 experts, the top 4 groups, then the top 8 experts in them |
| Context | 262,144 native; served at 128K. The MLA cache takes 10.4 KB a token, and the KDA state 252 MiB a sequence |
| Input | Text only |
| Vocabulary | 157,184 |
| Quant | bartowski IQ2_XS, 177.4 GB |
| Start | `switch-model.ps1 ling`; ready in about 17 s from a warm file cache |
| API | `http://127.0.0.1:8086/v1`, model ID `ling-3.1-flash` |
| VRAM | 29.9 GB |
| RAM | Working set about 160 GB; only about 24 GB stays free |
| Expert slots in VRAM | 3,919 of 34,816: 1,350 of its own, 2,281 in the prompt path's buffers, 288 in the MLA caches' rows |

**Why IQ2_XS.** At 564B only 2-bit quants fit in 192 GB of RAM, and the experts have to stay there. bartowski's IQ2_XXS
is 170.8 GB, IQ2_XS 177.4, IQ2_S 188.5 and IQ2_M 213.0. IQ2_XS is the largest that leaves Windows some room: about 24 GB
stays free while it runs, where IQ2_S would leave about 13. IQ2_XXS would save only 6.6 GB, a few percent of speed, at a
cost in quality. The file holds the routed experts at IQ2_XS (160 GB; a few layers IQ3_XXS or IQ3_S), attention and the
dense layers mostly at Q5_K (10.2 GB), the shared experts at Q8_0 (1.1 GB), the output head and embedding at Q8_0 (0.7
GB each), and the MTP layer at Q4_0 (4.6 GB, not loaded).

**The engine.**

- **Token path.** maya-ink's design: everything but the routed experts on the GPU, the host routing, and cached experts
  on the GPU while the CPU lane computes the rest. The KDA recurrence follows ggml-cuda's layout, and the MLA decode
  attention is flash-decoding over the cache's splits. Each MoE layer's GPU work runs as one CUDA graph.
- **Prompt path.** Chunks of 16,384 tokens (`--chunk 16384`), a layer at a time, with each layer's attention in pieces
  of 2,048. Prompts under 256 new tokens take the token path, because a chunk of a few hundred tokens already touches
  nearly every expert.
- **Prefetch** (`LING_PF_PREFETCH=1`, set in the run config's `env`). A copier thread stages each MoE layer's experts
  into one of two 3.3 GB layer buffers while the GPU computes the layers before. The GEMMs read the same weights in the
  same order, so the output doesn't change.
- **VRAM expert cache.** maya-ink's settings (decay 0.99, up to 16 swaps every 2 tokens, margin 1.5), not yet re-tuned
  for Ling's 512 experts.
- **SwiGLU clamp.** The routed experts of layers 65-71 and the shared experts of layers 64-71 clamp their SwiGLU, as the
  GGUF specifies. Without the clamp, layer 64 blows up.
- **Next turns.** The KDA state can't be rewound, so the engine keeps a snapshot of it at the end of each prompt (283 MB
  of VRAM). A next turn then reads only the last answer and the new message, even when the template renders that answer
  differently (without its thinking, say). A prompt that changes earlier messages is read again from the start.
- **Start.** The engine touches all 160 GB of routed experts, in about 11 s from the file cache, and keeps them in its
  working set. Nothing else big should run beside it.

**Calling it.**

- It thinks by default, and thinking is only on or off: `reasoning_effort` `none` or `minimal`, or
  `"chat_template_kwargs": {"enable_thinking": false}`, turns it off. The other levels change nothing.
- Sampling: the recommended temperature 1.0, top-p 0.95 and top-k 20. The run config sets them.
- **Thinking cap.** Maya's server closes the thinking at the first sentence or line end past 32,768 tokens, and the
  answer follows. `thinking_budget` in the run config changes the limit.
- Tools (GLM's `<arg_key>` format), `response_format` and streaming work.

**Speed** (`ab.py`, greedy, thinking off, at 131K context on a quiet machine with the file cache warm; maya-ling's runs
each started from the same expert profile by `ling-ab.ps1`, 3 runs each):

| | llama.cpp, `--cpu-moe` | llama.cpp's own fit | maya-ling, 8K chunks | 16K chunks | 16K + prefetch (in use) |
|---|---|---|---|---|---|
| Writing, new prompts | 12.0 tok/s | 12.0 tok/s | 23.0 tok/s | 22.5 tok/s | **22.4 tok/s** |
| Writing after a 35.5K-token prompt | 12.5 tok/s | 12.8 tok/s | 16.6 tok/s | 16.2 tok/s | **16.2 tok/s** |
| Reading that prompt | 75.6 tok/s | 86.5 tok/s | 548 tok/s | 738 tok/s | **876 tok/s** |
| First token | 470 s | 410 s | 64.7 s | 48.1 s | **40.6 s** |
| A repeated prompt, 1st and 3rd time | 12.7, 12.9 tok/s | 12.9, 12.9 tok/s | 22.4, 26.0 tok/s | 20.5, 25.2 tok/s | **22.0, 25.4 tok/s** |
| VRAM | 15.8 GB | 30.7 GB | 29.9 GB | 29.9 GB | 29.9 GB |

- 16K chunks read prompts 35% faster than 8K, and the prefetch adds another 19%. Together they cost about 2% of the
  writing speed on new prompts.
- Adapting the cache every token instead of every 2 (`LING_CACHE_EVERY=1`, decay 0.97) wrote 5% faster on new prompts
  (24.1 tok/s), but 5% slower after the long prompt (15.8) and 8% slower on the repeats (23.9), so maya-ink's settings
  stay.
- **What limits it.** About half of a token's 544 routed experts are found in VRAM (49.9% in a replay of a run's
  routing). The CPU's share is 65-75% of a token, at about 0.11 ms per IQ2_XS expert: about 41 GB/s from RAM.
- **Open.** Writing after a long prompt drops to 16.2 tok/s from 22.4, more than on Nemotron or Inkling, while
  llama.cpp's speed doesn't drop at all. The cache is refilled from the prompt's experts and the MLA layers attend over
  the 35K positions, but the full cause is still open.
- Writing with thinking on wasn't measured separately. The hard test averaged 22 tok/s over 992K tokens.

**Checks.**

- **Against llama.cpp** (`ling-ref.py`). The chat template's text and the tokenizer's ids matched exactly on 5 prompts of
  33 to 3,771 tokens, with thinking on and off, a tool and a long ledger. The top token matched at 90 of 90 positions,
  and llama.cpp's tokens' log-probabilities were within 0.244. The longest prompt through the token path alone matched
  16 of 16, within 0.444.
- **Against an independent reference** (`ref-np.py`: numpy in float64, written from llama.cpp's graph and the model's
  Hugging Face code). On a 33-token prompt over layers 0-7, the prompt path agreed within 0.003% RMS (0.03% on single
  values), and the token path within 0.07% RMS (0.5% on single values).
- `--selftest`, 19 checks of the KDA recurrence, the MLA decode, the cache rows, RoPE and the routing against host
  references: all pass.
- `ling-chat-test.py` through the server: thinking kept separate, thinking off, a tool call, the answer from its result,
  a second turn that reused its prompt, streaming and JSON, 7 of 7. A thinking-on second turn read only 21 of its 363
  tokens. `derail.py`: 0 of 20 rounds derailed with the run config.

**Not done yet:** the cache settings confirmed on held-out prompts, the drop after long prompts, the MTP layer for
drafting, and 256K context. 256K would take about 1.4 GB more VRAM than 128K, and would have given the hard test's two
cut-off coding answers room.

**Hard test** ([hard-test.md](hard-test.md)): 21 of 25, coding 3 of 7, 744 minutes. Two more coding answers were still
thinking when they reached the 128K context.

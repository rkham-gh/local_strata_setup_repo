# Models

Per-model setup, settings, speeds, checks and quirks. All speeds were measured on this PC; figures from a model's lab, a
quant publisher or an engine's docs are marked as theirs. System specs and the baseline-vs-optimized summary:
[`../README.md`](../README.md). Methods: [measuring.md](measuring.md). Hard-test details: [hard-test.md](hard-test.md); misses by type: [failure-modes.md](failure-modes.md).

One engine runs at a time, since each needs the GPU and most of the RAM; `switch-model.ps1` stops the running server
before starting the next. Default: Qwen3.8-Flash-Next on Strata.

## Summary

Decode: `ab.py` on new prompts. Prefill: `ab.py`'s long prompt, 32-39K tokens depending on the tokenizer. "—": not
recorded.

| Model | Total / active parameters | Quant, size | Engine | Context | Decode, new prompts | Prefill | RAM in use |
|---|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next (default) | — | ISTA-DASLab IQ3_S, 84 GB | Strata 0.2.10 | 256K | 225-229 tok/s | ~5,700 tok/s | 85-88 GB |
| GLM-5.3-Flash | — | Maya-S24, 94.7 GB | Project Maya v1.1.52 | 256K | 38.4 tok/s | 2,948 tok/s | 74 GB |
| GLM-5.3-Flash, exact | — | Maya-M, 116 GB | Project Maya v1.1.52 | 256K | 22.8 tok/s | ~2,650 tok/s | 92-94 GB |
| MiMo-V2.6-Flash-MOPD (RL before 2026-11-08) | — | MXFP4, 170 GB | strata-mimo, patched | 256K | 25.7 tok/s (RL 25.5) | ~1,030 tok/s | grows to ~157 GB |
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

Flash-Next's UD-Q4_K_XL was measured, then deleted; its numbers are in section 1.

Everything in this file is on the RTX 5090 alone. Since 2026-11-11 an RTX 3090 sits beside it, and MiMo, Inkling,
Nemotron and Ling run with it as a helper card: 36.0, 36.6, 48.2 and 24.7 tok/s on new prompts. The one-card vs
two-card tables for every engine are in [two-gpu.md](two-gpu.md).

## Engines

| Engine | Port | Serves | Notes |
|---|---|---|---|
| llama.cpp router, build 12017 (CUDA 13.4) | 9931 | Qwen3.8-27B, Nanbeige4.3-33B, Gemma 4 26B-A4B and 31B; fallbacks for DeepSeek Coder and Nemotron VL | `llama-server` in router mode, one model loaded at a time (`--models-max 1`), per-model settings in `presets.ini` |
| Strata 0.2.10 (engine 0.1.56) | 8080 | Qwen3.8-Flash-Next | DFlash2 drafting |
| Project Maya v1.1.52 | 8081 | GLM-5.3-Flash | MIT-licensed Strata fork, built from source |
| strata-mimo, commit `268fdbc` | 8082 | MiMo-V2.6-Flash-RL | Third-party Strata fork, built from source, four local changes |
| Project Maya's DeepSeek engine (`maya-ds4`) | 8083 | DeepSeek-V4.1-Flash Coder | Same Maya install, with a local server change |
| maya-ink | 8084 | Inkling-Small | Custom engine in a Project Maya fork, branch `inkling` |
| maya-nem | 8085 | Nemotron 3.5 Super VL | Same fork, branch `nemotron` |
| maya-ling | 8086 | Ling-3.1-flash | Same fork, branch `ling` |

- `switch-model.ps1` targets: `flash-next`, `glm`, `glm-m`, `mimo`, `deepseek`, `inkling`, `nemotron`, `nemotron-q4`,
  `ling`, `router`, `stop`, `status`. It applies each setup's tuned settings and refuses to stop a server in use (exit
  codes 0 done, 1 failed, 2 refused). Don't pipe its output: the server it starts inherits the pipe, and the caller
  hangs until that server stops.
- The router drops Unsloth's `UD-` prefix from model IDs (pull with the full tag, select the ID without it). It starts
  in ~2 s, loads a model on its first request and unloads it when a request names another.
- maya-ink, maya-nem and maya-ling run behind Maya's server. Each has a run config (engine binary, arguments, tokenizer
  folder, model name, GPU, sampling, optional `env`, which Maya puts over the caller's environment), a tokenizer folder
  from its `make_pack.py`, and an `expert-profile.bin` of expert scores, saved after every request and loaded at the
  next start.
- Since 2026-11-11 `mimo`, `inkling`, `nemotron` and `ling` start two-card builds (fork branches `inkling-2gpu`,
  `nemotron-2gpu`, `ling-2gpu`; a local patch for strata-mimo). `"gpu": [0, 1]` in the run config turns the helper card
  on; `"gpu": [0]` runs the same binary on one card.
- `GET /health`: Strata's reports `"loaded": true`; Maya's is ready when `status` is `ok` and `max_context` is set.
- Above ~31.8-32 GB of VRAM in use the driver pages VRAM into system RAM and speed collapses. Every setup here stays
  below that.

| Script | Runs |
|---|---|
| `ab.py` | Greedy, thinking off: six new prompts, a 32-39K-token prompt read and answered, one prompt asked three times |
| `spec-sweep.py` | Decode with thinking on, at the model's own sampling |
| `match.py` | The same 4 chats 3 times, the workload of published Strata speed tables |
| `deep-test.py` | Prompts of 125K-250K tokens with three hidden notes |
| `glm-test.py` | The quality suite: tools, JSON, pictures, 8 facts, 8 reasoning puzzles, 3 coding tasks, an agent bug-fix task |
| `derail.py` | 20 rounds of a tool conversation, then a new chat; counts derailed answers |

Engines with a VRAM expert cache start each A/B run from the same saved expert profile.

## Calling the models

- OpenAI-style API at `http://127.0.0.1:<port>/v1`, no key. Thinking arrives in `reasoning_content`;
  `"chat_template_kwargs": {"enable_thinking": false}` turns it off on every model here.
- `max_tokens` limits the output, not the context: leave it out to allow the rest of the context, or allow at least
  40,000 tokens at `high`, and check `finish_reason` for `length`.
- Client timeouts of 30 minutes or more: a 250K-token prompt takes 44 s to first token on Flash-Next and up to 5 minutes
  on MiMo.
- Keep one thinking level through a conversation; changing it can re-read the whole prompt.
- A prompt longer than the context gets an HTTP 400 at once; nothing is cut.
- Maya's server gives any model with a special `</think>` token in its vocabulary a thinking budget: 32,768 tokens by
  default, closed at the first sentence or line end past it. `thinking_budget` in the config changes it, 0 turns it off.
  `"fit_max_tokens": true` trims a token budget larger than the room left instead of refusing the request.

| Engine | Thinking cap by default |
|---|---|
| Strata (Flash-Next) | Three quarters of `max_tokens` |
| Project Maya's server (GLM, DeepSeek Coder, Nemotron, Ling) | 32,768 tokens, set by `thinking_budget` |
| Project Maya's server with maya-ink (Inkling) | None: Inkling's thinking doesn't end at a special token |
| strata-mimo (MiMo) | None: `max_tokens` is the whole budget |

## 1. Qwen3.8-Flash-Next (Strata, default)

| | |
|---|---|
| Architecture | MoE with its own MTP head; text and pictures. Total and active parameters not recorded |
| Quant | ISTA-DASLab IQ3_S, 84 GB, checked against its pinned SHA-256 |
| Engine | Strata 0.2.10 (engine and terminal 0.1.56) |
| Launch | `switch-model.ps1 flash-next`, ready in 75-115 s. Port 8080, model ID `qwen3.8-flash-next-iq3_s` (Strata ignores the name, so older IDs work too) |
| Context | 256K, the model file's limit (262,144) |
| VRAM, RAM | 30.3-30.4 GB, 85-88 GB; ~11,900 experts (49%) in VRAM |
| Thinking | Default `xhigh`; `"reasoning_effort": "low"` for quick work. Hard test at temperature 0.6, top-p 0.95 |
| Works | Tools, `response_format`, pictures |

IQ3_S puts 49% of its experts in VRAM against UD-Q4_K_XL's 31%, so it decodes about twice as fast here. ISTA-DASLab's
benchmarks score it level with the full model (task average 93.26 vs 93.12); top-1 agreement with Q8_0 is 90%, vs 94%
for UD-Q4_K_XL. It passed every test here that Q4 did, and Q4 didn't score higher on the hard test.

**Settings.** `strata launch` rewrites its launch JSON on every start, so the flags live in `switch-model.ps1`, which
calls `strata.cmd` (PowerShell's script policy blocks `strata.ps1`):

```
strata.cmd launch --family qwen --model IQ3_S --model-path <IQ3_S folder> --context 262144 --vision cpu --parallel 1
  --set vram-reserve-mib=1200 --set adapt-every=1 --set adapt-swaps=80 --set adapt-decay=0.97 --yes --no-tui
```

| Setting | Notes |
|---|---|
| Drafting | 0.2.10 picks DFlash2 on this card (a 1.2 GB drafter, downloaded and packed on first start) and its own window, `--spec 6`. 0.1.54 drafted with the MTP head, best at a 5-token window; that setting went with the update |
| `adapt-every=1`, `adapt-swaps=80`, `adapt-decay=0.97` | Faster VRAM expert re-balancing than the defaults; the script restarts an IQ3_S engine running without them |
| `vram-reserve-mib=1200` | A plain `strata launch` drops it to 700 MiB, and the engine warns VRAM is nearly full. At 1,200 the log shows ~999 MiB free once; harmless |
| `--vision cpu` | ~3 s per chart, VRAM left to experts; with the smaller reserve, +4% |
| Route tail skip | On by default for IQ3_S: an expert the GPU doesn't hold is skipped when every token of a draft window ranks it 7th or lower |
| KV cache | Mostly in RAM, so `--kv-grow` turns itself off; each 128K costs ~1.5 GB of RAM and ~1% of the VRAM expert slots |
| Missed experts | Split between the CPU pool and the x8 link, ~29 GB/s of expert data each |

**Thinking cap.** With `max_tokens` of 1,024 or more and no `reasoning_budget_tokens`, Strata closes the thinking at
`max_tokens - max_tokens // 4` by inserting "I have thought about this long enough; time to give my answer." A
24,000-token cap thinks for 18,000. Without `max_tokens` the budget is the rest of the context, and thinking can run to
~196K. Only `"answer_room": false` in Strata's config turns the rule off, so leave `max_tokens` out for hard work. The
thinking level sits at the start of the prompt: turning thinking on mid-conversation re-reads the whole prompt, ~105 s
at 250K.

**Speed**, tok/s. The 0.1.54 rows are `flash-sweep.ps1`, one setting per restart; the plain start ranged 187-193 over
four runs, so under ~3% is noise, and 35K deep is one 400-token answer and varies more.

| IQ3_S | Decode, new prompts | Decode, 35K deep | `match.py` | Prefill |
|---|---|---|---|---|
| 0.1.54, plain start (mean of 4 runs) | 190 | 190 | 228 | 5,320 |
| 0.1.54, MTP window 5 + faster re-balancing (2 runs) | 202 | 192 | 240 | |
| 0.1.54, faster re-balancing alone (2 runs) | 194 | 195 | 238 | |
| 0.1.54, MTP window 5 alone (2 runs) | 195 | 184 | 235 | |
| 0.1.54, MTP window 6 | 195 | 191 | 230 | |
| 0.1.54, MTP window 3 | 177 | 177 | 220 | |
| 0.1.54, draft confidence 0.7, 0.4 or 0.3 (vs 0.5) | 182-188 | 181-186 | 221-224 | |
| 0.1.54, 64K of the KV cache in VRAM | 194 | 187 | 223 | |
| 0.1.54, 4-bit draft layer, larger prompt chunks, `adapt-decay 0.9` | 183-192 | 188-190 | 223-232 | |
| 0.2.10, DFlash2 + faster re-balancing (in use) | 225-229 | 201-217 | 274 | ~5,700 |

A PCIe share of 0.25 changed nothing. DFlash2 couldn't serve requests on 0.1.54; the 0.1.55 engine update changed CUDA
speed on Linux only. Thinking on, own sampling (`spec-sweep.py`): 208. `deep-test.py` on 0.1.54: TTFT 21, 26, 35 and
44 s at ~125K, 150K, 200K and 250K tokens, decode 173, 164, 173 and 186, every hidden note found. A follow-up reads only
the new text, in 0.5-2 s; going back a step at 125K took 1-2 s.

**Checks.** Quality suite: facts 8/8, logic 8/8, tools 5/5, a chart read correctly, the 3 coding tasks at `high` in
16-36 s each, the agent bug-fix task in 9.9 s. `derail.py`: 0 of 20 derailed on 0.1.54 and again on 0.2.10.

**Quirks.** Strata ends a reply after 256 repeats of one token; in the hard test that stopped Flash-Next writing zeros
while recalling a Sudoku puzzle. At the 24K budget it once wrote an endless number on the JSON task.

**UD-Q4_K_XL, measured and deleted.** Unsloth's UD-Q4_K_XL (111 GB) ran as `switch-model.ps1 flash-next-q4` with
`--set STRATA_ROUTE_TAIL_SKIP=7`; its files were deleted on 2026-11-03, so that target no longer starts. Only 31% of its
experts fit in VRAM.

| UD-Q4_K_XL, tok/s | llama.cpp router | Strata 0.1.54, as Strata sets it | 0.1.54, tail skip on | 0.2.10, DFlash2 |
|---|---|---|---|---|
| Decode, new prompts | 28 | 81 | 107 | 116 |
| Decode, 35K deep | | 97 | 123 | 129 |
| `match.py` | | 101 | 134 | |
| Prefill | ~480 | 2,260 | 2,270 | 2,330 |
| RAM | | 105-113 GB | 105-113 GB | |

The router gives it no drafting (only Strata does); don't run Flash-Next there. Strata turns the route tail skip off for
Unsloth quants once every expert fits in RAM, because it moves answers slightly; left off, it costs a quarter of the
speed. strata-nvfp4's published figures (9950X3D, DDR5-5600, PCIe 5.0 x16) are 228 tok/s on IQ3_S and 169 on UD-Q4_K_XL:
this PC matches the IQ3_S figure, and the Q4 gap fits the slower RAM and link. Hard test: 21 of 25 with room to think
(19 at the 24K budget), 92 minutes; single tasks flipped between runs, so one run each reads as even with IQ3_S, not
worse.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): IQ3_S 22 of 25 with room to think (21 at the 24K budget), coding 4 of 7,
51 minutes.

## 2. GLM-5.3-Flash (Project Maya)

Strata doesn't run GLM.

| | |
|---|---|
| Architecture | MoE, 12,096 routed experts, 8 per token in each layer; text and pictures. Total and active parameters not recorded |
| Quant | Maya-S24, Project Maya's own build of the FP8 original, 94.7 GB |
| Engine | Project Maya v1.1.52, built from source with CUDA 13.4 and Visual Studio 2022 Build Tools |
| Launch | `switch-model.ps1 glm` (config `maya-maya-s24.json`), ready in 75-95 s. Port 8081, with a dashboard on the same port; model ID `glm-5.3-flash` |
| Context | 256K; the model file allows 1,048,576 |
| VRAM, RAM | 29.4 GB, 74 GB; 3,252 experts (~27%) in VRAM. All 12,096 fit in VRAM and RAM: 0.00 SSD reads per token |
| Exact setup | `switch-model.ps1 glm-m`: Maya-M (116 GB), no expert skips, config `maya-maya-m.json`, 29.6 GB of VRAM, 92-94 GB of RAM |
| Thinking | Default `max`, so set a level on every request: `low` for quick work, `high` for careful work (allow up to 15 minutes a task) |
| Works | Tools, pictures; JSON under quirks |

Without skips GLM is RAM-bound. Maya-S24's smaller experts fit more in VRAM than Maya-M's, and two skips leave out the
lightest experts the GPU doesn't hold. Maya's docs put Maya-S24's next token at the original's 83% of the time (Maya-M
86%) and the two skips below at a KL of 0.052 from the unskipped model; Maya-S itself is 0.43 from the original FP8.

| Decode, tok/s | New prompts | 33K deep |
|---|---|---|
| Maya-M | 23 | 21 |
| Maya-M with the layer map | 25 | 24 |
| Maya-S24 | 27 | 25 |
| Maya-S24 with the layer map | 35 | 33 |
| Maya-S24 with both skips (in use) | 38 | 37 |
| Maya-S24 with MTP forced on | 26 | 24 |

With both skips 78-82% of expert uses hit VRAM, against 57%, and the CPU's share of a token fell from 30 ms to 11 ms.

**Settings**, the `env` block of `maya-maya-s24.json`:

```
MAYA_GLM_TRUNK_KEEP_MAP=8888888888888888888888855555554444444444444448
MAYA_GLM_MISS_SKIP_W=0.14
MAYA_GLM_MISS_SKIP_FROM=23
```

The keep map: where the rest would come from RAM, layers 23-29 keep their 5 best experts and layers 30-44 their 4 best;
the early layers keep all 8, since they notice a missing expert far more. The miss skip leaves out, from layer 23 on, a
missing expert carrying under 14% of its token's route weight.

| Also tried | Result |
|---|---|
| MTP forced on | Slower (table above); Maya's GLM engine drafts only with two or more GPUs |
| `START-MAYA.bat --calibrate --no-start` (~10 minutes) | Maya-M from 21.2 to 23.5 tok/s (`PCIE_SHARE=0.10`, `CPU_LANE=12`). On Maya-S24 with skips it kept the engine's own split (PCIe share ~0.31) at 38.5: PCIe shares of 0-0.75 measured 30-38 tok/s and CPU-lane counts of 4-23 measured 37-41, none beating the engine's choice by the 3% calibration requires |
| `RAM_RESIDENT`, `PROMOTE_MIN=6`, KDA graphs, pictures off, a smaller VRAM reserve | Each moved Maya-M by under 1 tok/s |
| v1.1.52's CUDA graphs for the DSA layers' feed-forward | 8-9% faster decode in its release notes; not here, where the CPU lane takes ~10 of each token's 25 ms |

`START-MAYA.bat --setup` needs `--context 262144`, or it falls back to its menu, which defaults to 32K and stops at
128K. A setup run rewrites the config: put the `env` block and `"port": 8081` back afterwards. `switch-model.ps1` starts
Maya's `server.py` directly, which compiles nothing; update with `update-maya.ps1`, which builds the engines (a plain
`git pull` would leave the new server running old engines).

**Speed**, v1.1.46 vs v1.1.52, tok/s:

| | v1.1.46 | v1.1.52 (in use) |
|---|---|---|
| Maya-S24, decode, new prompts | 38.5 | 38.4 |
| Maya-S24, decode after a 33K-token prompt | 36.6 | 37.3 |
| Maya-S24, prefill of that prompt | 2,948 | 2,948 |
| Maya-M, decode, new prompts | 21.8 | 22.8 |

TTFT ~1 s on a short prompt, 2.5-3 s on a few hundred tokens, 6.3 s with a picture. `deep-test.py` on Maya-S24: TTFT 44,
56, 75 and 96 s at ~125K, 150K, 200K and 250K tokens, decode 36 at every size, every hidden note found. A follow-up
reads only the new text, in ~1 s.

**Checks.** v1.1.52 gave the same greedy answers as v1.1.46, byte for byte. Quality suite: tools 5/5, pictures, facts
8/8, code 3/3, the agent task, 0 of 20 derailed; reasoning 7 of 8, from a typo in its final line at temperature 0.6
after it had reversed both words correctly. The JSON task passes since v1.1.50. On the tuned setup the 3 coding tasks at
`high` took 7.5-35 s, the agent task 29 s.

**Quirks.**

- Always send `"stop": ["<|assistant|>"]`: GLM sometimes writes that token instead of ending its turn, then answers all
  over again, and v1.1.52 doesn't stop on it. Seen on one prompt in 3 of 4 tries; the stop string fixed it in 5 of 5.
- Since v1.1.50 `response_format` on `/v1/chat/completions` gives the model the schema, but nothing checks the answer;
  `/v1/responses` with `text.format` (a `json_schema` with `"strict": true`) checks it.
- One coding task at `high` took 20,000-35,000 tokens (14-25 minutes) on Maya-M; the longest thinking in the tests was
  ~28,000 tokens, under Maya's 32,768 cap. On hard coding tasks it keeps re-deciding (`hmm, wait, actually`) and
  drafting code inside its thinking; a larger budget with no thinking cap didn't change that.
- Thinking off to `max` mid-conversation re-reads the whole prompt; off to `low` doesn't.
- Maya's Ollama-style API (`/api/tags`, `/api/pull`, `/api/delete` and others) is on by default, but switching models
  through it is refused here, because `switch-model.ps1` starts the server directly. The page-file warning at start is
  harmless.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 13 of 25 at its card's sampling, `max` effort and 131K (10 answers ran to
the budget, still thinking); 18.9 taking each task's best of re-runs at `high` and `low` with 262K. For hard work, send
`reasoning_effort: "high"`: the default `max` thinks past any budget. 10 of 25 in the 24K-budget round at `high`.

## 3. MiMo-V2.6-Flash-RL (strata-mimo)

Neither Strata nor Project Maya runs MiMo. On 2026-11-01 strata-mimo carried 57 commits by one author since leaving
upstream Strata, measured only on a 96 GB card with 512 GB of DDR5-6400. Its commits were compared with upstream before
building (no binaries; network use limited to localhost, Hugging Face and pinned packages); the C++ wasn't reviewed line
by line.

| | |
|---|---|
| Architecture | MoE with sliding-window attention and an MTP head, 12,032 routed experts; the RL checkpoint, text only. Total and active parameters not recorded |
| Quant | The ggml project's MXFP4 GGUF of the RL checkpoint, 170 GB, at the revision the fork pins (`a5d1269`) |
| Engine | strata-mimo (MIT) at commit `268fdbc`, built from source with CUDA 13.4 and Visual Studio 2022 Build Tools, plus four local changes. One GPU only |
| Launch | `switch-model.ps1 mimo`, ready in 10-30 s; the first answers run slow while the file cache warms up (9 tok/s, then 16). Port 8082, with a web page on the same port; model ID `mimo-v2.6-flash-rl` |
| Context | 256K, the engine's limit (262,144) |
| VRAM, RAM | 30.0 GB; RAM grows to ~157 GB as it runs. ~1,500 experts (12%) in VRAM at 256K |
| Thinking | On by default, no levels: `"reasoning_effort": "low"` turns it off; `medium`, `high` and `max` all mean on. No cap: strata-mimo has Strata's wrap-up text but not its three-quarters rule |

**Settings** (`strata-mimo-mxfp4.json`): `--ram-copy 0`, and `"env": {"MIMO_TAIL_KEEP": "5"}` for the tail skip below;
MTP on. The engine asks for a pinned 150 GB copy of the experts, which this PC can't give: it stops ~150 s into loading
with `pinned RAM for the experts: out of memory`. With `--ram-copy 0` it reads the experts through the file cache, and
disk reads stay at 0 MB/s during answers once the cache is warm. The fork's setup rewrites this config, so both settings
go back in after any setup run; its `setup.py` also shares the installed Strata's `settings.json`, and given a new data
folder it moves the other engines' `models`, `packs` and `mtp` folders into it. Run its setup with the settings folder
pointed at a private directory.

**Local changes** (`strata-mimo-local.patch`):

- Derail fix. After a conversation longer than ~130 tokens, a new chat could reuse stale sliding-window keys, and the
  engine answered as if it couldn't see the prompt: gibberish, or a made-up task. The triggering tool sequence went
  wrong 2 times in 20 without any skip, 5 in 20 with one, 0 in 40 after the fix. The engine now tracks how far it has
  written and re-reads the prompt when the keys it needs are gone. The fork's own open fix (its PR #1) also turns off
  prefix reuse whenever MTP is on, which would re-read the whole conversation every turn; that part was left out, since
  stale MTP state only affects drafts, which the main model checks.
- A sliding-window ring of 4,096 positions instead of 256, so ordinary turns keep their prefix reuse; it costs 35 expert
  slots.
- Expert tail skip, `MIMO_TAIL_KEEP`: an expert missing from VRAM is left out when every token of the window ranks it
  6th or lower. Unset, every expert is computed.
- A looser start check: the engine refused to start when free VRAM fell a few MiB under its 1,536 MiB margin; it now
  refuses below 1,280 MiB.

Tail-skip cost as KL from the exact engine, on 400 replayed tokens of three prompts (`mimo-kl.py`). A community 3-bit
MiMo quant sits ~0.17 from the original. Unlike GLM's, MiMo's early layers turned out no more sensitive than its late
ones, so the skip covers every layer.

| `MIMO_TAIL_KEEP` | KL from the exact run | Same top token | Decode, new prompts |
|---|---|---|---|
| Off | 0 | 100% | 14.9 tok/s |
| 7 | 0.013 | 96.4% | 17.0 tok/s |
| 6 | 0.022 | 94.1% | 19.6 tok/s |
| 5 (in use) | 0.037 | 91.9% | 23.0 tok/s |

**Speed.** With all four changes, on fresh prompts at 256K: decode 25.5 tok/s on new prompts and 21 at 33K deep, prefill
1,030 tok/s on a 35K-token prompt. Before the changes: ~15 tok/s decode, ~950 tok/s prefill.

- TTFT: any prompt of 128 or more new tokens waits 5-13 s, so an agent turn costs ~10 s before it writes anything.
  Slower expert copies without the pinned copy are the likely cause; the pinned mode can't run here to confirm it.
- `deep-test.py`: TTFT 171-259 s at ~125K tokens (the slow end on a cold file cache), 171 s at 150K, 232 s at 200K,
  298 s at 250K; decode 26, 24, 24 and 24; every hidden note found.
- The KV cache sits in VRAM: 256K costs 2.2 GB of VRAM and 10% of the expert slots against 64K, for 5% less decode
  speed. A follow-up reads only the new text, in 1.5-2 s.
- In the hard test it decoded at 26-48 tok/s, slowing to ~27 deep into an answer.
- ~1,500 of the 12,032 experts fit in VRAM at 256K, against ~6,600 on the fork author's 96 GB card. They start out
  covering ~40% of expert choices; the CPU computes the rest from RAM, so RAM speed sets the pace.

Settings tried at 800 tokens a run, thinking off, repeat runs. First runs on a prompt were 15-16 tok/s in every case;
repeats run faster because the experts in VRAM follow the work.

| Setting | Decode | Experts in VRAM |
|---|---|---|
| MTP on, 256K | 18.4 tok/s | 1,535 |
| MTP off | 18.1 tok/s | 1,561 |
| 64K context | 19.4 tok/s | 1,710 |
| A learned profile, saved and reloaded | 18.4 tok/s | 1,535 |

**Checks.** With all four changes: facts 8/8, logic 8/8 (scored by hand: the script marked one answer wrong for stray
spaces), tools 5/5, the 3 coding tasks at `high` in 22-189 s, the agent bug-fix task in 85 s, 0 of 20 derailed.

**Quirks.**

- Thinking length varies a lot. The same clock puzzle took 1,106 tokens in one run (right) and 6,261 in another (wrong),
  and two more runs were still thinking at the test's 6,000-token limit. Allow ~5 minutes for a careful task.
- `response_format`: the server asks for JSON and checks the result, but nothing constrains the decoding.
  `tool_choice: "required"` isn't enforced: the model answered in text in two runs and called the tool in the third. No
  pictures.
- A cold file cache costs speed: one baseline read 11.7 tok/s on its first run and 14.9 on the second. Disk reads should
  stay at 0 MB/s during answers; if they don't, something else is taking the RAM.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): RL 16 of 25 at its best (card sampling, thinking on, 131K), coding 1 of 7,
580 min; 12.9 in the 24K-budget round. 

**The MOPD checkpoint (in use since 2026-11-08).** Xiaomi's vLLM recipe says the RL checkpoint can get stuck
re-issuing tool calls and recommends MiMo-V2.6-Flash-MOPD instead. ggml-org's MOPD GGUF has the RL GGUF's exact layout
(MXFP4 167.4 GB + an MTP head) and a byte-identical chat template, so it drops into the same strata-mimo config. Only
the paths and the model ID (`mimo-v2.6-flash-mopd`) change. Its speed is RL's: `ab.py` 25.7 tok/s new prompts, 21.1
after 33K, prefill 1,034 tok/s. `derail.py` gave 0/20. Hard test at RL's settings: **16 of 25 (17 lenient), coding 3 of 7**
(regex, ordered, diff), in 465 min and 890K tokens, against RL's 1,098K. The RL checkpoint's files were deleted after
its run.

## 4. DeepSeek-V4.1-Flash Coder (Project Maya's DeepSeek engine)

ISTA-DASLab's DeepSeek-V4.1-Flash-1017 GSQ-RCO Coder: DeepSeek's 552B model with half of each layer's 384 experts
removed, keeping those chosen for code, agent work and pictures. ISTA reports 94.8% of the original's SWE-bench Verified
score and 98.7% of its LiveCodeBench v6.

| | |
|---|---|
| Architecture | MoE, 192 of 384 routed experts kept per layer, 7,680 in all. Compressed attention keeps the KV cache small. An Engram n-gram table comes with the weights. Text only: the config runs with `--vision off` |
| Quant | ISTA-DASLab IQ2_XXS, revision `54229d3`: 72.1 GB of weights in five shards, plus the 110.6 GB Engram table as shard 6. `get-coder.py` downloads them and checks each SHA-256 |
| Engine | Project Maya v1.1.52, DeepSeek engine `maya-ds4`, with a local server change |
| Launch | `switch-model.ps1 deepseek` (config `maya-ds4-coder.json`). Port 8083, model ID `deepseek-v4.1-flash-coder` |
| Context | 1M |
| VRAM, RAM | ~31 GB, flat across context sizes. The ~43 GB of experts not in VRAM are copied into pinned RAM at start; the KV cache also sits in RAM |
| Experts in VRAM | 2,600-2,700 of 7,680; a healthy start logs ~2,640 on its `ds4: CUDA0 layers` line |
| Thinking | Default effort 50 (`low`). `reasoning_effort` takes `low` (50), `high` (75), `max` (100), or a number from 1 to 100. Maya's server budget applies unless the config turns it off, as the max-settings hard-test run did |
| Works | Tools, `tool_choice: "required"`, `response_format` |
| Fallback | The llama.cpp router's `deepseek-v4.1-flash-coder` preset, 64K context |

**Setup on Maya.** Maya's setup offers only its own DeepSeek files, so this one was set up by hand: the engine built
with Maya's own build commands for the `maya-ds4` target (`build-maya-ds4.bat`) and added to `MAYA-BUILD.json`; the
files indexed in place by Maya's `pack_gguf.py`, Engram table included, which reads ISTA's types and the pruned
192-expert layers as they are; and `maya-ds4-coder.json` written by hand from Maya's DeepSeek defaults:
`--tp 1 --threads on --vision off`, 1M context, reasoning effort `low` (50), port 8083. Maya's setup never rewrites it.

**Local server change** (`maya-text-think-end.patch`). Maya ends DeepSeek's thinking only at its special token, so a
model that quotes the tag isn't cut off. At effort `max` this Coder writes `</think>` as plain text instead; Maya then
kept the whole answer as reasoning, and the first max-effort answers came back empty. `MAYA_DS4_TEXT_THINK_END=1`, set
by the `deepseek` target, makes the text tag end the thinking too; unset, Maya behaves as shipped. `update-maya.ps1`
carries the change to each new release and reports a conflict if it can't.

**Engine behaviour.**

- Prefill computes the RAM-resident experts in place instead of copying them over PCIe, hence 6-7 times llama.cpp's
  prefill speed; decode is ~12% faster too.
- The context costs almost nothing: Maya keeps DeepSeek's cache in RAM, and the compressed attention keeps it small.
- No drafting: Maya's 8 GB DSpark drafter is built for the full 384-expert model.
- Restart once after each Maya update. The pack's `lane-times.txt` is keyed by the engine's build time, so the first
  start after a new build re-times the PCIe link and starts on a decode path that streams some RAM experts over PCIe:
  1,345 experts in VRAM instead of 2,644, 32.0 tok/s on new prompts and 35.6 on repeats. That run times both paths,
  finds streaming 3.1% slower and saves the result, so the next start is back to normal.
- A start-up warning says 1.7 GB of VRAM is already in use, even with nothing else on the GPU; harmless.

**Speed**, tok/s (`ab.py`; Maya through `maya-ab.ps1`, llama.cpp build 12017 through `ds-ab.ps1`):

| Engine, placement | Context | Decode, new prompts | Decode after a 33K-token prompt | Prefill of that prompt | VRAM for the context |
|---|---|---|---|---|---|
| llama.cpp's own `--fit` | 16K | 21.7 | | | |
| llama.cpp, every expert in RAM, 23.5 GB GPU cache of hot experts | 16K | 32.0 | | | |
| llama.cpp, same, 22 GB cache, q8_0 KV cache (fallback preset) | 64K | 29.8 | 29.7 | 472 | |
| llama.cpp, same, 20 GB cache, f16 KV cache | 64K | 28.0 | 30.6 | 473 | |
| llama.cpp, same, 19.5 GB cache, q8_0 KV cache | 128K | 26.8 | 28.7 | 447 | |
| Strata 0.2.10, `--family deepseek41`, two runs | 256K | 30.0-33.8 | 27.8-29.0 | 1,526-2,483 | |
| Maya | 64K | 33.4 | 31.2 | 2,922 (TTFT 11 s) | |
| Maya | 256K | 34.8 | 30.9 | 3,357 | 0.24 GB |
| Maya | 512K | 35.1 | 28.8 | 3,361 | 0.49 GB |
| Maya (in use) | 1M | 33.5 | 30.6 | 3,328 | 0.97 GB |

After the v1.1.52 update (restarted once): 33.2 and 34.3 tok/s on new prompts, 45.6-47.0 on a prompt asked again,
prefill 3,199-3,230 tok/s on the 33K prompt. Long prompts: decode 40-45 tok/s at 125K-250K deep, prefill
3,300-4,300 tok/s, every hidden note recalled up to 250K. Strata 0.1.54 couldn't run the Coder: its setup accepts only
the files it pins.

**llama.cpp fallback preset:** `n-gpu-layers = all`, `cpu-moe = true`, `moe-cache-mib = 22000`, 64K context, q8_0 K and
V cache, `ubatch-size = 2048`, `parallel = 1`, `reasoning = on`, and `reasoning-effort = 50`, the effort ISTA's scores
used.

- Each GB of GPU expert cache is worth ~1 tok/s, and each 64K of context takes ~2 GB of it. At 32.1-32.2 GB of VRAM the
  driver pages VRAM to RAM and prefill drops to ~85 tok/s, which is why `-ub 4096` and a 21 GB cache at 128K failed.
- Prefill runs at ~470 tok/s because each batch copies experts over the x8 link. llama.cpp reads the Engram table from
  the SSD a row at a time.
- 12 to 24 CPU threads measured the same. `ngram-mod` stalled the server on a repeated prompt, so it's off. The first
  answer after loading runs at ~20 tok/s while the GPU expert cache fills, then 30-35.

**Checks.**

- On Maya v1.1.52: tools 5/5, JSON, the agent task in 60 s (77 s on llama.cpp), 2 of 3 code tasks (the calculator ran
  out of the test's 8,000 tokens). Facts 7 of 8: it named neodymium for atomic number 74, as on llama.cpp. Reasoning
  8 of 8, though the checker missed one answer written as `\frac{1}{11}`. At effort `max`, 4 of 4 answers arrived in
  `content` (`max-effort-check.py`), so the local change works.
- Through the router: tools 5/5 including `tool_choice: "required"`, facts 7 of 8 (the eighth cut off at an 800-token
  cap), reasoning 8 of 8, long-prompt recall right at 8K and 32K, all three thinking levels right.

**Quirks.** It can fall into endless repeats, such as a run of zeros, when an answer needs a long string of one
character: always set `max_tokens` and check `finish_reason`. The loops are the model's own. `loop-check.py` asks for
"one followed by exactly 64 zeros"; every try wrote ~17,700 zeros until the cap, with the GPU expert cache and without
it, at temperature 0.6 and 1.0. DRY sampling (`dry_multiplier 0.8`) cut the runs to 28 zeros, but the answers still
failed, so sampling stays at the defaults.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 19 of 25 at max settings (effort `max`, a 384,000-token budget, Maya's
thinking cap off), coding 2 of 7, 298 minutes.

## 5. Qwen3.8-27B (llama.cpp router)

| | |
|---|---|
| Architecture | Dense, 27B. 64 layers, Gated DeltaNet and full attention at 3:1, so only 16 layers keep the whole context. Text and pictures |
| Quant | Unsloth UD-Q6_K, 22 GB, plus the model's MTP head at Q4_0, 1.4 GB |
| Engine | llama.cpp router, build 12017 |
| Launch | `switch-model.ps1 router`; the router loads the model on its first request, in ~17 s. Port 9931, model ID `unsloth/Qwen3.8-27B-GGUF:Q6_K` |
| Context | 208K. The card allows 262,144 tokens of thinking plus a 131,072-token answer |
| VRAM, RAM | 31.0 GB; a few GB of RAM |
| Thinking | Default `xhigh`. `reasoning_effort` takes `low`, `medium` or `xhigh`; `high` counts as `xhigh`, and any other value fails the request |
| Works | Tools, `response_format`, pictures (encoder on the CPU) |

Pull with `pull-model.cmd unsloth/Qwen3.8-27B-GGUF:UD-Q6_K`; the MTP head comes from
`pull-model.cmd unsloth/Qwen3.8-27B-GGUF -hff MTP/mtp-Qwen3.8-27B-Q4_0.gguf --no-mmproj`, moved into the `draft` folder.

**Preset** (`presets.ini`): device `CUDA0`, all layers on the GPU, `ctx-size = 212992`, q8_0 K and V cache,
`ubatch-size = 512`, `no-mmproj-offload = true`, `parallel = 1`, `spec-type = draft-mtp` with the MTP head and
`spec-draft-n-max = 2`, and the card's thinking-mode sampling: temperature 1.0, top-p 0.95, top-k 20, min-p 0. Without
thinking, the card suggests temperature 0.7, top-p 0.8 and presence penalty 1.5. MTP needs `parallel = 1`, so concurrent
requests queue. Only 16 of the 64 layers keep the whole context, ~4.3 GB per 128K at q8_0; moving the picture encoder to
the CPU and shrinking the prompt batch from 2048 to 512 freed ~1.7 GB, for 7% slower prefill.

**Speed** (`ab.py` through `qwen-ab.ps1`, build 12017), tok/s:

| Setting | VRAM | Decode, new prompts | Decode after a 33K-token prompt | Prefill |
|---|---|---|---|---|
| 128K, pictures on the GPU, batch 2048 (before) | 29.9 GB | 128 | 101 | 2,830 |
| 192K, pictures on the CPU, batch 2048 | 32.0 GB, too close to paging | 128 | 103 | 2,810 |
| 208K, pictures on the CPU, batch 512 (in use) | 31.0 GB | 128 | 103 | 2,630 |
| 224K, pictures on the CPU, batch 512 | 31.8 GB, too close to paging | 128 | 101 | 2,615 |

With thinking on at the card's sampling (`spec-sweep.py`), by drafted tokens per step; at the preset's 2, half the
drafts are accepted:

| Drafted tokens | None | 1 | 2 (in use) | 3 | 4 |
|---|---|---|---|---|---|
| Decode, tok/s | 64 | 89 | 99 | 99 | 98 |

On build 12000 at 128K: 65 tok/s plain and 107 with MTP on short prompts, 57 (at 32K) and 96 (at 43K) deep, prefill
~3,400 tok/s (2,500-2,700 at 32K-43K deep). In the hard test it decoded 788K tokens at ~99 tok/s.

**Quirks.**

- Its template keeps all earlier thinking by default (`preserve_thinking`): send `reasoning_content` back with earlier
  assistant messages, above all those with tool calls.
- Hard tasks took it 60K-109K tokens of thinking at ~100 tok/s; allow 20 minutes or more a task.
- The whole model and its context sit in VRAM: the strongest choice here when RAM has to stay free.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 20 of 25, coding 4 of 7, 130 minutes.

## 6. Nanbeige4.3-33B (llama.cpp router)

| | |
|---|---|
| Architecture | Dense, 33B, Llama architecture. 52 layers, GQA 56/8, full attention in every layer. Vocabulary 166,912. Text only |
| Quant | bartowski Q4_K_M, 20.8 GB |
| Engine | llama.cpp router, build 12017 |
| Launch | `switch-model.ps1 router`; the model loads in ~10 s. Port 9931, model ID `bartowski/Nanbeige_Nanbeige4.3-33B-GGUF:Q4_K_M` |
| Context | 80K. Its hard-test runs with room used 96K (98,304), the most that fits beside the weights with a q8_0 cache. The lab allows up to 262,144 output tokens |
| VRAM, RAM | 29.6 GB at 80K, 31.4 GB at 96K. Little RAM: the whole model sits in VRAM |
| Thinking | Always on; `reasoning_effort` changes nothing |

**Preset** (`presets.ini`): device `CUDA0`, all layers on the GPU, q8_0 K and V cache, `ubatch-size = 2048`,
`parallel = 1`, `spec-type = ngram-mod`, temperature 1.0, top-p 0.95. The lab suggests temperature 0.6 and top-p 0.95
for reasoning and chat, and 1.0 for agent work; the preset's 1.0 is the agent setting.

**Speed** (`ab.py` through a standalone `llama-server` with the preset's settings, `nanbeige-ab.ps1`), tok/s:

| Setting | Decode, new prompts | Answer to a 39K-token prompt | Same answer asked again | Context that fits |
|---|---|---|---|---|
| q8_0 cache, no drafting | 69 | 38 | 69 | 80K (29.6 of 32.6 GB VRAM) |
| q8_0 cache + `ngram-mod` (in use) | 68 | 38 | 986 | 80K |
| f16 cache | 70 | 52 | 70 | ~48K |
| f16 K with q8_0 V, or the reverse | 69 | 36 | 69 | ~64K |

- Prefill ~2,050 tok/s at 39K tokens and 1,800 at 52K. 69 tok/s is about three quarters of what the 5090's memory speed
  allows for 20 GB of weights.
- `ngram-mod` costs ~1% on new text and decoded 14 times faster when an answer repeated earlier text. `ngram-cache` cost
  3% on new text; `ngram-simple` and `ngram-map-k` gained nothing.
- The f16 cache is faster for long thinking on short prompts but halves the context; q8_0 stays because the hard test's
  longest prompt is 52.5K tokens.
- No draft model fits: Nanbeige4.2-3B has a different vocabulary (166,144 tokens vs 166,912) and a looped architecture,
  and no DFlash, EAGLE or DSpark drafter existed for the 33B as of 2026-11-01.

**Checks.** With room to think (96K context) it ended three hard coding tasks by itself instead of running out of
budget, and solved one: it was still reasoning, not looping, when the 24K budget cut it off. In the 24K round it looped
once, repeating one line until its budget ran out.

**Quirks.**

- It thinks much longer than the others: all 7 hard coding tasks ran out of a 24,000-token cap while it was still
  working. Allow 60,000 tokens or more for hard work.
- After a tool result, ~2 replies in 3 arrive with an empty `content` and the answer in `reasoning_content`: the model
  writes a second `<think>` where `</think>` belongs. When `content` is empty, read `reasoning_content`.
- `response_format` and `tool_choice: "required"` fail at once with HTTP 400 (`Failed to initialize samplers`) on
  build 12017. Ask for JSON in the prompt instead.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 18 of 25 at its best (19 counting a print slip), coding 3 of 7,
412 minutes, at the card's reasoning settings with top-k and min-p off and a 96K context. In the 24K-budget round it
scored 9.8 (11.8 counting two print-format slips).

## 7. Gemma 4 26B-A4B and 31B (llama.cpp router)

Google's Gemma 4 (released 2026-04-02, Apache 2.0). Two of its five sizes fit the 5090, each with its own MTP drafter.

| | Gemma 4 26B-A4B | Gemma 4 31B |
|---|---|---|
| Architecture | MoE, 3.8B of 25.2B parameters active; text and pictures | Dense, 31B. 60 layers, 10 of which see the whole context; text and pictures |
| Quant | The original weights at Q8_0, 26.9 GB | Google's quantization-aware (QAT) 4-bit, as Unsloth's UD-Q4_K_XL, 17.3 GB |
| Drafter | `mtp-gemma-4-26B-A4B-it.gguf`, 0.5 GB | `mtp-gemma-4-31B-it-qat-Q4_0.gguf`, 0.3 GB |
| Context | 224K (229,376), q8_0 cache | 224K (229,376), q8_0 cache |
| VRAM | 31.4 GB | 31.6 GB |
| Model ID | `unsloth/gemma-4-26B-A4B-it-GGUF:Q8_0` | `unsloth/gemma-4-31B-it-qat-GGUF:Q4_K_XL` |
| Pull tag | `Q8_0 --mtp` | `UD-Q4_K_XL --mtp` |

Both start with `switch-model.ps1 router`, answer on port 9931 and use a few GB of RAM. Both presets set device `CUDA0`,
all layers on the GPU, q8_0 K and V cache, `no-mmproj-offload = true`, `parallel = 1`, `spec-type = draft-mtp`,
`spec-draft-n-max = 4`, `reasoning = on`, and Google's sampling for every Gemma 4: temperature 1.0, top-p 0.95, top-k
64, min-p 0. llama.cpp's own min-p default of 0.05 cuts further. Keep Google's sampling for hard work: at temperature
0.6 the 31B coded no better.

In Unsloth's own table, the 31B's QAT 4-bit has 96.7% top-1 agreement with its reference and a mean KLD of 0.014, and
Unsloth says more bits of these weights don't help. The 26B's QAT 4-bit agrees on only 85.6%, so the 26B runs at Q8_0.
That 4-bit file (15.7 GB) stays on disk without a preset: faster, 285 tok/s plain and 431 with its drafter (at 256K with
an f16 cache and pictures on the GPU, 21.7 GB), but less faithful to the model.

**Speed** (`ab.py`; `spec-sweep.py` at Google's sampling, thinking on), tok/s:

| | 26B-A4B | 31B |
|---|---|---|
| Decode with thinking on, Google's sampling | 311 (202 without the drafter) | 170 (72 without the drafter) |
| Decode, new prompts, thinking off | 318 | 184 |
| Decode after a 38K-token prompt | 222 | 104 |
| Prefill | ~8,000 | ~2,550 |

Decode by drafted tokens per step, Google's sampling, thinking on:

| Drafted tokens | 2 | 3 | 4 (in use) | 5 | 6 |
|---|---|---|---|---|---|
| 26B-A4B | 296 | 303 | 311 | 301 | 288 |
| 31B | 142 | 157 | 170 | 169 | 158 |

At 4 drafted tokens, 58% (26B) and 60% (31B) of drafts are accepted. At temperature 0 more drafted tokens kept helping,
up to 188 tok/s at 5 on the 31B. A picture reached its first token in 2.1 s.

VRAM sets the context. Each 128K costs the 31B ~6.3 GB at q8_0, even though only 10 of its 60 layers see the whole
context, and the drafters take another 0.5-1.4 GB. At 256K, or with an f16 cache, both models pass ~32 GB and the driver
pages VRAM into RAM: the 31B then decoded at 20 tok/s and prefilled at 780 tok/s. The picture encoder runs on the CPU to
make room.

**Checks** (the quality suite at Google's sampling). The 31B passed everything: tools 5/5, JSON, pictures, facts 8/8,
reasoning 8/8, code 3/3, the agent task in 12 s. The 26B did the same, except that one reasoning puzzle and the
calculator ran out of the test's 6K and 8K caps; its agent task took 18 s. Both had 0 of 20 derailed chats.

**Quirks.**

- Both think by default. Gemma 4 has no thinking levels, so `reasoning_effort` changes nothing.
- In a tool loop, send each assistant message that has tool calls back with its `reasoning_content`; the template
  replays that thinking until the next `user` turn, as Google asks.
- Tools, `tool_choice: "required"`, `response_format` and pictures work. Pictures are capped at 512 tokens of Gemma's
  1,120, because a picture has to fit one 512-token batch; a larger batch would cost context.
- The drafter needs a single slot, so requests queue. The longest hard-test answer was 59K tokens.
- Gemma solves puzzles with little thinking: the 31B did both routes and the knapsack in 10-14K tokens each. Its coding
  failures are real bugs, not budget: every coding answer ended by itself after 1.6K-24K tokens.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 26B-A4B 18 of 25, coding 2 of 7, 22 minutes. 31B 17 of 25 (18 counting a
print slip), coding 1 of 7, 18 minutes.

## 8. Inkling-Small (maya-ink)

Thinking Machines' Inkling-Small (released 2026-07-30, Apache 2.0). Mainline llama.cpp, Strata and Project Maya can't
run it; llama.cpp's open draft PR #25731 can, at ~9 tok/s. maya-ink is a custom engine in a Project Maya fork (branch
`inkling`, `src/ink`, documented in the fork's `docs/INKLING.md`), behind Maya's server.

| | |
|---|---|
| Architecture | MoE, 276B total, 12B active. 42 layers: the first 2 with a dense SwiGLU FFN, the other 40 MoE with 256 routed experts (2,048 wide, top 6) and 2 shared experts. Width 4,096; 32 query heads and 8 KV heads of 128. Vocabulary 201,024 |
| Attention | No RoPE. 35 sliding-window layers (window 512) and 7 global layers (every sixth), with a learned relative-position bias band. 4 short causal convolutions per layer |
| Input | Text, pictures and audio; maya-ink reads text only so far |
| Quant | Unsloth UD-Q4_K_XL, 163 GB: dense matrices at Q8_0 (6.5 GB), routed experts at Q4_K to Q6_K (150 GB) |
| Launch | `switch-model.ps1 inkling`, ready in ~16 s. Port 8084, model ID `inkling-small` |
| Context | 128K; the model has 1M positions |
| VRAM, RAM | 29.9 GB. RAM working set ~164 GB, with ~22 GB free |
| Expert slots in VRAM | 1,601: 1,104 of its own, 259 in the prompt path's buffers, 238 in the K/V caches' rows |
| Thinking | `reasoning_effort` takes `none`, `low`, `medium`, `high` (the default, Inkling's own 0.9), `xhigh` and `max` (0.99, the level Thinking Machines evaluated at; use it for hard work). `minimal` counts as off. No cap: Inkling's thinking doesn't end at a special token, so Maya's server has nothing to close it with; cap long work with `max_tokens` |
| Sampling | The card's evaluation settings, temperature 1.0 and top-p 1.0, set by the run config |
| Works | Tools, a named `tool_choice`, `response_format`, streaming. `tool_choice: "required"` can't be forced, because an Inkling tool call starts with the tool's name. No pictures or audio yet |

**Engine.**

- Token path: everything but the routed experts on the GPU, with int8-activation GEMVs from Maya's DeepSeek kernels. The
  host picks the experts from the router's logits; cached experts run on the GPU while the CPU lane computes the rest
  straight from the GGUF's mapping. Each MoE layer runs as one CUDA graph between host syncs.
- CPU lane: one spinning thread pool per layer, ~0.30 ms per expert: ~50 GB/s, what this RAM gives.
- Prompt path: chunks of up to 8,192 tokens, a layer at a time, so each expert crosses PCIe once a chunk instead of once
  a 512-token batch. Copies go through the engine's own pinned staging at 14 GB/s; the driver's staging of pageable
  memory managed 4-15 GB/s, and the link does 28.7 GB/s from pinned memory, but Windows won't pin 150 GB of the mapping.
  Prompts under 48 new tokens take the token path.
- VRAM expert cache: slots sized for each layer's experts. Between prompts it borrows the prompt path's 3.9 GB of
  buffers and the global layers' unused K/V rows (3.6 GB at a conversation's start); the embedding table stays in RAM.
  Scores decay 0.99 a token; every 2 tokens up to 16 experts are swapped in, at a margin of 1.5, the copies going up
  while the CPU waits on the GPU.
- Sampling draws from the whole distribution in one pass, in 0.4 ms; sorting the 200K-token vocabulary had cost 13 ms a
  token.
- At start the engine touches all 157 GB of routed experts, in ~11 s, and keeps them in its working set. On Windows a
  page's first touch costs a fault even with the file cached, which had held prompt copies to 3.4 GB/s.

**Speed** (`ab.py`, and `ink-sampling.py` at the card's sampling; maya-ink averaged over 4 runs of the first version and
3 of the current one, via `ink-ab.ps1`; llama.cpp built at the PR's commit `48f7e71`):

| | llama.cpp PR, `--cpu-moe` | llama.cpp PR, 14 GB expert cache, `-ub 2048` | maya-ink, first version | maya-ink (in use) |
|---|---|---|---|---|
| Decode, new prompts | 9.1 tok/s | 9.0 tok/s | 14.5 tok/s | 30.2 tok/s |
| The same at the card's sampling | | | 12.6 tok/s | 31.1 tok/s |
| Decode after a 31.8K-token prompt | 9.6 tok/s | 9.7 tok/s | 15.6 tok/s | 26.3 tok/s |
| Prefill of that prompt | 72 tok/s | 207 tok/s | 733 tok/s | 794 tok/s |
| TTFT | 441 s | 154 s | 43.5 s | 40.4 s |
| VRAM | 11.6 GB | 26.8 GB | 29.9 GB | 29.9 GB |

The cache settings were chosen by replaying `ab.py`'s routing offline with `ec-sim.py`, which predicted the engine's CPU
experts a token within 0.2%. As a held-out check, `ab-heldout.py` is `ab.py` with other prompts (Go, SQL and Java code,
biology, French, German and Japanese, a C++ repeat, and 21.7K tokens of Python as the long prompt), every run starting
from a profile that had seen neither set. Decode in tok/s, `ab.py` / held-out, 3 runs each:

| | New prompts | After the long prompt | Repeated prompt, 3rd time |
|---|---|---|---|
| maya-ink, first version | 14.5 / 14.1 | 15.6 / 16.3 | 16.1 / 19.4 |
| Current build, first version's cache settings (decay 0.998, 4 swaps every 8 tokens) | 22.7 / 20.8 | 24.7 / 29.2 | 27.7 / 37.8 |
| Current build (in use) | 30.2 / 30.2 | 26.3 / 26.3 | 38.7 / 46.6 |

- Right after a long prompt the new settings gain nothing: on three long prompts (`ink-deep.py`) they decoded 24.2 tok/s
  vs 25.4 for the old ones, though the replay gives both 71% of the experts in VRAM there. Why is still open.
- What doubled decode: the cache grew from 871 to 1,601 slots and now follows what the conversation uses right now; the
  attention no longer walks its keys one at a time, which had cost up to 8 ms a token; each layer runs as one CUDA
  graph; sampling no longer sorts the vocabulary. A token now spends ~27 ms in the CPU lane (~90 experts), ~6.5 ms on
  attention and routing and ~1.5 ms on the rest; the first version took 60-75 ms. About 70% of a token is still the
  CPU's share of the experts, read at the RAM's full speed.
- Tried without gain: a `cudaStreamQuery` flush after the launches, fusing a layer graph's 22 nodes into 13 (~1%), one
  block for all 4,096 channels' convolutions (slower). `cudaHostRegister` on the mapping "succeeds" without speeding
  anything up. Under WDDM a layer's graph takes ~115 µs on the GPU plus ~40 µs of submission and wake-up.
- In the hard test it decoded 671K tokens at 40 tok/s on average, above `ab.py`'s 30, likely from the cache adapting
  during long answers: a repeated prompt reaches 39-47 tok/s by its third run.

**Checks.** Against llama.cpp (`ink-ref.py`: the same token ids fed to both, greedy, each step's top 5 compared), the
chat template's text and the tokenizer's ids matched exactly on 4 prompts of 27 to 3,054 tokens. The top token matched
at 87 of 88 positions, the other a near tie (-0.68 vs -0.69), and the log-probabilities of llama.cpp's tokens were
within 0.34, 69 of 88 within 0.01. `ink-chat-test.py` through the server (thinking kept separate, thinking off, a tool
call, the answer from its result, streaming, JSON): 6 of 6. `derail.py`: 0 of 20 derailed, in 291 s vs 406 on the first
version.

**Quirks.**

- A/B runs must start from the same saved profile, which `ink-ab.ps1` restores; otherwise each run teaches the next.
- A new binary copied right after a stop can fail to replace the old one while the engine exits: check its hash before
  measuring.
- The log is appended at each start. A healthy start logs `1104 expert slots`, `259 more in the prompt path's buffers`,
  `238 in the K / V caches' rows`, and `the routed experts' 156.8 GB touched` in ~11 s.
- After a change in the fork, rebuild with `build-maya-ink.bat`; `make_pack.py` rewrites the tokenizer folder from the
  GGUF.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 21 of 25 at reasoning effort `max`, coding 3 of 7, 279 minutes.

## 9. Nemotron 3.5 Super VL (maya-nem)

NVIDIA's Nemotron 3.5 Super VL (released 2026-10-15, OpenMDW-1.1). Mainline llama.cpp (build 12017) runs it, pictures
included; maya-nem, on maya-ink's machinery, runs its text side about three times as fast. It lives in the same Project
Maya fork, on branch `nemotron` (`src/nem`, documented in the fork's `docs/NEMOTRON.md`).

| | |
|---|---|
| Architecture | Hybrid Mamba-2 / attention MoE, 120B total, 12B active. 88 layers: 40 Mamba-2, 40 MoE and 8 attention, plus one NextN (MTP) block. Vocabulary 131,072 |
| MoE | 512 routed experts in a 1,024-wide latent space, top 22, relu², plus one shared expert |
| Mamba-2, attention | Mamba-2: 128 heads of 64, state size 128, 8 groups. Attention: 32 query heads and 2 KV heads of 128, no RoPE, 8 KiB of K/V a token |
| Input | Text, pictures and video. maya-nem reads text only; the router preset reads pictures |
| Quant | Default: Unsloth UD-Q3_K_XL, 65.8 GB, with 55.6 GB of routed experts. Alternative: UD-Q4_K_XL, 86.9 GB, with 75.8 GB of routed experts. The picture encoder is 1.7 GB |
| Launch | `switch-model.ps1 nemotron` (UD-Q3_K_XL) or `switch-model.ps1 nemotron-q4` (UD-Q4_K_XL), ready in ~10 s from a warm file cache. Port 8085, model ID `nemotron-3.5-super` for both quants |
| Context | 256K; the model allows 1M |
| VRAM, RAM | 29.9 GB. Q3: ~62 GB of RAM, ~113 GB free. Q4: working set ~84 GB, ~95 GB free |
| Expert slots in VRAM | Q3: 8,526 of 20,480 (6,555 of its own, 1,283 in the prompt path's buffers, 688 in K/V rows). Q4: ~6,100 |
| Thinking | On by default. `reasoning_effort` `low` or `medium` (or `minimal`) turns on NVIDIA's lighter "efficient" mode: ~2.2 times fewer thinking tokens for ~9% lower accuracy, in NVIDIA's report. The other levels keep the full default. Maya's 32,768-token cap applies; `thinking_budget` in the run config changes it |
| Sampling | The card's text settings, temperature 1.0 and top-p 0.95, set by the run config |
| Works | Tools (Qwen3-Coder's XML format), `response_format`, streaming |

UD-Q4_K_XL came first: its experts average ~5.4 bits (up rows Q5_K, Q4_K or Q6_K; down rows Q5_0 or IQ4_NL), close to
the original, and ~95 GB of RAM stays free. Q8_0 (131.6 GB) would read 1.6 times the bytes. UD-Q3_K_XL, despite its
name, is mostly 4-bit IQ formats: its experts are IQ4_NL (28.5 GB), IQ4_XS (15.7), IQ3_XXS (8.1), IQ2_XS (1.6) and Q2_0
(1.6). ~8,500 experts fit in VRAM instead of 6,100; it decodes 43% faster and its top token is Q4's 97% of the time, so
it became the default. `pdl.py` downloaded the files over 24 ranged connections at 70-80 MB/s and checked each SHA-256.

**Engine.**

- Token path: maya-ink's, with the Mamba-2 conv and selective scan as maya-nem's own kernels and maya-ink's split-key
  decode for the attention. Cached experts run on the GPU in the latent space; the shared expert runs on the GPU.
  Between the host's 40 syncs a token, the GPU's work runs as 41 CUDA graphs.
- Prompt path: chunks of 16,384 tokens (`--chunk 16384` in the run config), a layer at a time through the engine's
  pinned staging. Prompts under 32 new tokens take the token path.
- VRAM expert cache: maya-ink's settings.
- maya-nem's GPU kernels gained `Q2_0` (ggml's type 42) for UD-Q3_K_XL; with that build, Q4's output stays bit for bit
  the same.
- AVX2 Q2_0 rows, on since 2026-11-05 through the Q3 run config's `"env": {"DS4_Q2_0_AVX2": "1"}`. ggml's CPU dot has
  Q3's Q2_0 down rows only in scalar code; this AVX2 path, adapted from llama.cpp's x86 kernels (MIT), replaces it.
  Since the config's `env` goes over the caller's, a run without the rows needs that line out of the config.
- At start the engine touches all routed experts, Q3's 55.6 GB in ~4 s and Q4's 75.8 GB in ~5-6 s, from a warm file
  cache; a cold cache takes minutes.

**Speed** (`ab.py`; maya-nem 3 runs each at 16K chunks via `nem-ab.ps1`). The llama.cpp columns are UD-Q4_K_XL. The AVX2
column was measured on a build with the same code before the change was merged; the rebuilt serving binary wasn't
measured again.

| | llama.cpp, `--cpu-moe` | llama.cpp's own fit, `-ub 2048` | maya-nem, UD-Q4_K_XL | maya-nem, UD-Q3_K_XL | Q3 with the AVX2 rows (in use) |
|---|---|---|---|---|---|
| Decode, new prompts | 11.2 tok/s | 13.0 tok/s | 26.3 tok/s | 37.7 tok/s | 41.3 tok/s |
| Decode after a 34.2K-token prompt | 11.4 tok/s | 13.5 tok/s | 21.7 tok/s | 30.6 tok/s | 34.1 tok/s |
| Prefill of that prompt | 107 tok/s | 474 tok/s | 1,607 tok/s | 1,981 tok/s | 1,973 tok/s |
| TTFT | 319 s | 72 s | 21 s | 17 s | 17 s |
| A repeated prompt, 1st and 3rd time | 11.4, 11.4 tok/s | 13.5, 13.5 tok/s | 23.9, 29.3 tok/s | 35.0, 41.1 tok/s | 38.1, 45.0 tok/s |
| VRAM | 12.2 GB | 30.3 GB | 29.9 GB | 29.9 GB | 29.9 GB |

- The AVX2 rows decoded 41.2-41.4 tok/s (3 runs) vs 37.6-37.7 (2 runs) on new prompts, and 33.5-34.5 vs 30.3-31.9 after
  the 34K prompt; with thinking on at the card's sampling (`spec-sweep.py`, 2 runs each), 41.7 and 42.4 vs 38.8 and
  38.7. Prefill didn't change.
- On UD-Q4_K_XL, maya-ink's tuned cache settings beat its earlier ones (decay 0.998, 4 swaps every 8 tokens, margin
  1.25): 26.6 vs 22.1 tok/s on new prompts and 29.6 vs 22.9 on the third repeat, 2 runs each; equal after the long
  prompt.
- 16K prompt chunks prefill 42% faster than 8K (1,607 vs 1,130 tok/s), for ~1% less decode speed (26.3 vs 26.6 tok/s on
  Q4).
- The CUDA graphs gained ~5%. About 80% of a token is the CPU's share of the experts, at the RAM's ~40 GB/s.
- In the hard test it decoded 472K tokens at 40 tok/s on average.

MTP is correct but slower. On Q3 with the AVX2 rows, drafting with the GGUF's NextN head (`--mtp N`) passed every
exactness check, but:

| `--mtp` | Off (in use) | 1 | 2 | 4 |
|---|---|---|---|---|
| Decode, tok/s | 41.3 | 32.3 | 32.0 | 28.4 |

The head guesses well (194 of 205 single drafts accepted on a repeated prompt), but each drafted token still reads its
own experts from RAM. The head also takes 3.7-4.2 GB of VRAM, so the expert slots drop from 6,555 to 4,911 at 4 drafts,
and the drafts are checked one token at a time without CUDA graphs. Prefill is 10% slower with it on (1,785 vs
1,973 tok/s).

**Checks.**

- Q4 against llama.cpp (`nem-ref.py`, as `ink-ref.py`): the template's text and the tokenizer's ids matched exactly on 4
  prompts of 28 to 3,621 tokens. The top token matched at 88 of 88 positions through the prompt path and 47 of 48
  through the token path alone, the miss a near tie (-0.688 vs -0.703); the log-probabilities were within 0.067. The
  CUDA graphs give the same bits.
- Q3 against llama.cpp on Q3: the top token matched at 86 of 88 positions, the other two near ties, with
  log-probabilities within 0.141.
- Q3 against Q4 (`q3-agree.py`, fed Q4's greedy continuations of 3 prompts): Q3's top token was Q4's at 949 of 976
  positions (97.2%). Q4's token's log-probability was 0.026-0.030 apart on average and 1.07 at worst.
- AVX2 rows: the top token is the scalar build's at 88 of 88 positions and llama.cpp's at 86 of 88, as before. Long
  greedy runs part ways at some point, because the AVX2 sums round differently from the scalar path's. The row test
  (`ds4_q2_0_rows_test`) passes for the AVX2 path; the unchanged scalar path misses the test's tight tolerance in
  15 of 3,960 checks.
- `nem-chat-test.py` through the server, `ink-chat-test.py`'s 6 checks: 6 of 6. `derail.py`: 0 of 20 derailed, on Q4, on
  Q3 and with the AVX2 rows.

**Quirks.** The Mamba-2 state can't be rewound, so a prompt is read on from where the engine is only when it continues
the tokens so far. A tool result does: `nem-turns.py` read on 473 of its 511 tokens. A new `user` message doesn't,
because the template drops the earlier thinking, so the whole conversation is read again, ~20 s per 32K tokens. Keep
long agent work in one tool loop where you can.

**Router preset for pictures.** `nemotron-3.5-super-vl` in `presets.ini` runs UD-Q4_K_XL with the BF16 picture encoder
(`mmproj-BF16.gguf`): `no-mmproj-offload = true`, 256K context, batch and ubatch 2048, `parallel = 1`, temperature 1.0,
top-p 0.95, min-p 0. It decodes 13.0 tok/s on new prompts and 13.5 after a 34K prompt, prefills at ~474 tok/s, and uses
30.3 GB of VRAM at 256K. Leave its fit alone: given `-ngl 99` without `--cpu-moe`, llama.cpp put the whole model "on"
the GPU, filled 32.0 GB, spilled the rest to shared memory and decoded 3.0 tok/s.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): UD-Q3_K_XL 17 of 25 (18 counting an answer given outside its code block),
coding 0 of 7, 195 minutes. UD-Q4_K_XL hasn't run it.

## 10. Ling-3.1-flash (maya-ling)

inclusionAI's Ling-3.1-flash (`1019` build, on Hugging Face since 2026-10-13, MIT). Mainline llama.cpp (build 12017)
runs it as the `bailingmoe3` architecture, at about half maya-ling's decode speed; neither Strata nor Project Maya did.
maya-ling, on maya-ink's machinery, lives in the same Project Maya fork on branch `ling` (`src/ling`, documented in the
fork's `docs/LING.md`). There's no router preset: llama.cpp reads text only too.

| | |
|---|---|
| Architecture | MoE, 564B total, 25B active. 72 layers: 63 with Kimi Delta Attention (linear attention with a fixed-size state) and 9 with gated MLA (layers 7, 15, ..., 71). The first 4 layers have a dense SwiGLU FFN; the other 68 are MoE with 512 routed experts (1,280 wide) and 1 shared. Plus one NextN (MTP) layer, not run. Vocabulary 157,184. Text only |
| Routing | Group-limited: 8 groups of 64 experts, the top 4 groups, then the top 8 experts in them |
| Quant | bartowski IQ2_XS, 177.4 GB |
| Launch | `switch-model.ps1 ling`, ready in ~17 s from a warm file cache. Port 8086, model ID `ling-3.1-flash` |
| Context | 128K; 262,144 native. The MLA cache takes 10.4 KB a token, the KDA state 252 MiB a sequence |
| VRAM, RAM | 29.9 GB. RAM working set ~160 GB; only ~24 GB stays free |
| Expert slots in VRAM | 3,919 of 34,816: 1,350 of its own, 2,281 in the prompt path's buffers, 288 in the MLA caches' rows |
| Thinking | On by default, and only on or off: `reasoning_effort` `none` or `minimal` turns it off; the other levels change nothing. Maya's 32,768-token cap applies; `thinking_budget` in the run config changes it |
| Sampling | The recommended temperature 1.0, top-p 0.95 and top-k 20, set by the run config |
| Works | Tools (GLM's `<arg_key>` format), `response_format`, streaming |

At 564B only 2-bit quants fit in 192 GB of RAM with the experts staying there. bartowski's IQ2_XXS is 170.8 GB, IQ2_XS
177.4, IQ2_S 188.5 and IQ2_M 213.0. IQ2_XS is the largest that leaves Windows some room: ~24 GB stays free while it
runs, where IQ2_S would leave ~13. IQ2_XXS would save only 6.6 GB, a few percent of speed, at a cost in quality. The
file holds the routed experts at IQ2_XS (160 GB; a few layers IQ3_XXS or IQ3_S), attention and the dense layers mostly
at Q5_K (10.2 GB), the shared experts at Q8_0 (1.1 GB), the output head and embedding at Q8_0 (0.7 GB each), and the MTP
layer at Q4_0 (4.6 GB, not loaded).

**Engine.**

- Token path: maya-ink's design. The KDA recurrence follows ggml-cuda's layout; the MLA decode attention is
  flash-decoding over the cache's splits; each MoE layer's GPU work runs as one CUDA graph.
- Prompt path: chunks of 16,384 tokens (`--chunk 16384`), a layer at a time, each layer's attention in pieces of 2,048.
  Prompts under 256 new tokens take the token path, because a chunk of a few hundred tokens already touches nearly every
  expert.
- Prefetch (`LING_PF_PREFETCH=1` in the run config's `env`): a copier thread stages each MoE layer's experts into one of
  two 3.3 GB layer buffers while the GPU computes the layers before. The GEMMs read the same weights in the same order,
  so the output doesn't change.
- VRAM expert cache: maya-ink's settings, not yet re-tuned for Ling's 512 experts.
- SwiGLU clamp: the routed experts of layers 65-71 and the shared experts of layers 64-71 clamp their SwiGLU, as the
  GGUF specifies. Without the clamp, layer 64 blows up.
- Next turns: the KDA state can't be rewound, so the engine keeps a snapshot of it at the end of each prompt (283 MB of
  VRAM). A next turn then reads only the last answer and the new message, even when the template renders that answer
  differently (without its thinking, say). A prompt that changes earlier messages is read again from the start.
- At start the engine touches all 160 GB of routed experts, in ~11 s from the file cache, and keeps them in its working
  set. Nothing else big should run beside it.

**Speed** (`ab.py` at 131K context, file cache warm; maya-ling 3 runs each via `ling-ab.ps1`):

| | llama.cpp, `--cpu-moe` | llama.cpp's own fit | maya-ling, 8K chunks | 16K chunks | 16K + prefetch (in use) |
|---|---|---|---|---|---|
| Decode, new prompts | 12.0 tok/s | 12.0 tok/s | 23.0 tok/s | 22.5 tok/s | 22.4 tok/s |
| Decode after a 35.5K-token prompt | 12.5 tok/s | 12.8 tok/s | 16.6 tok/s | 16.2 tok/s | 16.2 tok/s |
| Prefill of that prompt | 75.6 tok/s | 86.5 tok/s | 548 tok/s | 738 tok/s | 876 tok/s |
| TTFT | 470 s | 410 s | 64.7 s | 48.1 s | 40.6 s |
| A repeated prompt, 1st and 3rd time | 12.7, 12.9 tok/s | 12.9, 12.9 tok/s | 22.4, 26.0 tok/s | 20.5, 25.2 tok/s | 22.0, 25.4 tok/s |
| VRAM | 15.8 GB | 30.7 GB | 29.9 GB | 29.9 GB | 29.9 GB |

- 16K chunks prefill 35% faster than 8K, and the prefetch adds another 19%; together they cost ~2% of decode speed on
  new prompts.
- Adapting the cache every token instead of every 2 (`LING_CACHE_EVERY=1`, decay 0.97) decoded 5% faster on new prompts
  (24.1 tok/s), but 5% slower after the long prompt (15.8) and 8% slower on the repeats (23.9), so maya-ink's settings
  stay.
- About half of a token's 544 routed experts are found in VRAM (49.9% in a replay of a run's routing). The CPU's share
  is 65-75% of a token, at ~0.11 ms per IQ2_XS expert: ~41 GB/s from RAM.
- Open: decode after a long prompt drops to 16.2 tok/s from 22.4, more than on Nemotron or Inkling, while llama.cpp's
  speed doesn't drop at all. The cache is refilled from the prompt's experts and the MLA layers attend over the 35K
  positions, but the full cause is still open.
- Decode with thinking on wasn't measured separately; the hard test averaged 22 tok/s over 992K tokens.

**Checks.**

- Against llama.cpp (`ling-ref.py`): the chat template's text and the tokenizer's ids matched exactly on 5 prompts of 33
  to 3,771 tokens, with thinking on and off, a tool and a long ledger. The top token matched at 90 of 90 positions, and
  llama.cpp's tokens' log-probabilities were within 0.244. The longest prompt through the token path alone matched
  16 of 16, within 0.444.
- Against an independent reference (`ref-np.py`: numpy in float64, written from llama.cpp's graph and the model's
  Hugging Face code): on a 33-token prompt over layers 0-7, the prompt path agreed within 0.003% RMS (0.03% on single
  values), the token path within 0.07% RMS (0.5% on single values).
- `--selftest`: 19 checks of the KDA recurrence, the MLA decode, the cache rows, RoPE and the routing against host
  references, all passing.
- `ling-chat-test.py` through the server, `ink-chat-test.py`'s 6 checks plus a second turn that reused its prompt:
  7 of 7. A thinking-on second turn read only 21 of its 363 tokens. `derail.py`: 0 of 20 derailed with the run config.

**Not done yet:** the cache settings confirmed on held-out prompts, the drop after long prompts, the MTP layer for
drafting, and 256K context. 256K would take ~1.4 GB more VRAM than 128K, and would have given the hard test's two
cut-off coding answers room.

**Hard test** ([hard-test.md](hard-test.md); misses: [failure-modes.md](failure-modes.md)): 21 of 25, coding 3 of 7, 744 minutes. Two more coding answers were still
thinking when they reached the 128K context.

# Hard test

25 tasks generated from fixed seeds, so every model gets identical, unseen problems. Code scores every answer.
Per-task data for every run: [`../results/hard-test-tasks.csv`](../results/hard-test-tasks.csv).

| Group | Tasks | Check |
|---|---|---|
| Traces | trace-1, -2, -3 | Exact `print()` output of a short program, in the final code block |
| Long multiplication | multiply-8d, -14d, -20d | Exact product, by hand |
| Logic grids | grid-4x3, -5x4, -6x4 | Unique solution |
| Optimization | tour-7, tour-9, knapsack-16 | Against exact solvers |
| Constrained writing | follow-acrostic, -lipogram, -words | Mechanical checks |
| Long input | long-table, long-hops | A 200-row JSON transform; a clue chain across a 52K-token document |
| Coding | code-regex, -scheme, -ordered, -diff, -sudoku, -json, -sat | Hidden tests, most with time limits. The JSON parser must match `json.loads` exactly and reject its extensions (`NaN`, `Infinity`) |
| Repair | kitchen | A 7-file project fixed through tools (read, write, run tests, `finish`), 24 turns at most, 10 hidden tests |

**Scoring.**
- **Strict:** 1 point a task, and the repair job counts as its share of the 10 hidden tests.
- **Lenient:** additionally counts a trace whose values are all right but formatted unlike `print()` (`ok*`): commas
  between arguments, or `''` for an empty string.
- **`len`:** the budget or the context ran out.
- **`loop`:** `--loop-guard` ended the answer, when its last 20,000 characters compressed more than 15x with zlib.
  Observed loops compress 44-465x and real answers at most 7.7x.
- **Timed code answers keep their first verdict.** Too-slow answers were re-timed afterwards for information only. The
  Maya-based engines sit at 0% CPU between requests, so serving doesn't slow the test code.

## Settings

**24K round (2026-11-02):** a 24,000-token budget per request, temperature 0.6, top-p 0.95, and the engines' own
thinking caps on. Strata ends thinking at 3/4 of `max_tokens`, which cut Flash-Next at 18K on 10 tasks. Maya closes
thinking at 32,768 tokens.

**At their best (2026-11-03 to 11-13):**
- each model at its card's sampling and top reasoning effort, with as much output as fits here;
- engine thinking caps off: Maya run configs got `"thinking_budget": 0, "fit_max_tokens": true` for that start only;
- `--loop-guard` on, and `--tool-thinking` on wherever the template replays reasoning inside a tool loop;
- GLM's capped answers were re-run with 262K at `high`, then the ones that ran out again at `low`. They're reported separately below, and Flash-Next's 11 capped tasks the same way.

| Model | Sampling | Effort | Output room |
|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S (Strata) | 0.6 / top-p 0.95 | default | 262,144, trimmed to the context left. Only the 11 tasks the 24K round cut short were re-run |
| Inkling-Small (maya-ink) | 1.0 / 1.0 | `max` (0.99) | to the 128K context. maya-ink has no thinking cap |
| Inkling 2 Small own 2-bit (maya-ink's port, both cards, `INK_TAIL_KEEP=5`) | 1.0 / 1.0 | `max` (0.99) | to the 128K context |
| Ling-3.1-flash (maya-ling) | 1.0 / 0.95 / top-k 20 | always on | to the 128K context |
| Qwen3.8-27B (llama.cpp) | 1.0 / 0.95 / top-k 20 / min-p 0 | `xhigh` | to the 208K context |
| DeepSeek-V4.1-Flash Coder (Maya) | 0.6 / 0.95 | `max` | 384,000, its listed maximum, in 1M |
| Gemma 4 26B-A4B, 31B (llama.cpp) | 1.0 / 0.95 / top-k 64 | on | to the 224K context |
| Nemotron 3.5 Super UD-Q3_K_XL (maya-nem) | 1.0 / 0.95 | full mode | to the 256K context |
| Nanbeige4.3-33B (llama.cpp) | 0.6 / 0.95, top-k and min-p off (the card's reasoning row) | always on | to 96K: the most context that fits in VRAM with q8_0 KV |
| GLM-5.3-Flash Maya-S24 | 1.0 / 0.95, stop on `<\|assistant\|>` | `max`; capped tasks re-run at `high`, then `low` | 131,072 (the listed output) in 256K; re-runs 262,144 |
| MiMo-V2.6-Flash RL and MOPD (strata-mimo, `MIMO_TAIL_KEEP=5`) | 1.0 / 0.95 (the card; Xiaomi's API forces it with thinking on) | on (no levels) | 131,072 in 256K |
| Ripple-65B Preview AD-IQ4_XS (llama.cpp b12314 + #31904, both cards, DFlash2 drafter) | 0.6 / 0.95 (Ricursive publishes none) | the model decides when to think; no levels | to the 192K context |

## Results at their best

| Model | Strict | Lenient | Coding | Wall time | Tokens | Avg decode |
|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | **22** | 22 | 4 | 51 min | 711K | 233 tok/s |
| Inkling 2 Small own 2-bit (IQ2_XXS mix) | **22** | 22 | 4 | 184 min | 481K | 44 tok/s |
| Inkling-Small UD-Q4_K_XL | **21** | 21 | 3 | 279 min | 671K | 40 tok/s |
| Ling-3.1-flash IQ2_XS | **21** | 21 | 3 (+2 cut off at 128K) | 744 min | 992K | 22 tok/s |
| Qwen3.8-Flash-Next UD-Q4_K_XL | **21** | 21 | 3 | 92 min | 830K | 150 tok/s |
| Qwen3.8-27B Q6_K | **20** | 20 | 4 | 130 min | 788K | 101 tok/s |
| Ripple-65B Preview AD-IQ4_XS | **20** | 20 | 4 (+1 cut off at 192K) | 216 min | 657K | 51 tok/s |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | **19** | 19 | 2 | 298 min | 1,132K | 63 tok/s |
| Gemma 4 26B-A4B Q8_0 | **18** | 18 | 2 | 22 min | 458K | 349 tok/s |
| Nanbeige4.3-33B Q4_K_M | **18** | 19 | 3 | 412 min | 1,034K | 42 tok/s |
| Gemma 4 31B QAT | **17** | 18 | 1 | 18 min | 201K | 185 tok/s |
| Nemotron 3.5 Super UD-Q3_K_XL | **17** | 17 (18 with trace-3's line outside the block) | 0 | 195 min | 472K | 40 tok/s |
| MiMo-V2.6-Flash-MOPD MXFP4 | **16** | 17 | 3 | 465 min | 890K | 32 tok/s |
| MiMo-V2.6-Flash-RL MXFP4 (deleted after its run) | **16** | 16 | 1 | 580 min | 1,098K | 32 tok/s |
| GLM-5.3-Flash Maya-S24, `max`, 131K | **13** | 13 | 0 | 782 min | 1,840K | 39 tok/s |
| GLM-5.3-Flash, each task's best of `max` / `high` / `low` | 18.9 | | 2 | +17 h of re-runs | +2,230K | |

Average decode includes prefill time. It isn't the short-prompt decode in the README:
- **Expert-caching engines get faster on long single-domain answers**, as the cache converges: Inkling 40 tok/s here
  against 30 on `ab.py`, Inkling 2 44 against 32.5.
- **Dense models slow down with depth.** Nanbeige writes 69 tok/s on a new prompt and about 39 on its long coding
  answers.
- **Drafting goes faster on long chains of thought.** DeepSeek's Maya engine decoded 55-62 tok/s through the test,
  against 33-35 on new short prompts.

Per task. Each cell is the verdict and the tokens written:

| Task | Flash-Next IQ3_S | Inkling-Small | Ling-3.1-flash | Flash-Next Q4 | Qwen3.8-27B | DeepSeek Coder | Gemma 4 26B | Nanbeige4.3-33B | MiMo MOPD | MiMo RL | Gemma 4 31B | Nemotron Q3 | GLM (max, 131K) | Ripple-65B | Inkling 2 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| trace-1 | ok 1.7K | ok 1.8K | ok 4.2K | ok 2.0K | ok 2.0K | ok 2.2K | ok 6.7K | ok 2.8K | ok* 2.0K | x 2.3K | ok 4.5K | ok 2.9K | ok 2.5K | ok 2.5K | ok 1.4K |
| trace-2 | ok 6.4K | ok 11.1K | ok 10.9K | ok 5.5K | ok 7.4K | x 6.5K | ok 12.7K | ok* 7.6K | ok 3.6K | ok 4.5K | ok* 7.2K | ok 7.6K | ok 11.2K | x 8.6K | ok 7.8K |
| trace-3 | ok 5.7K | ok 17.5K | ok 17.9K | ok 6.8K | x 6.9K | ok 6.4K | ok 15.6K | ok 19.1K | x 13.7K | ok 6.5K | ok 8.1K | x 10.3K | ok 13.7K | ok 20.7K | ok 8.1K |
| multiply-8d | ok 4.7K | ok 9.0K | ok 35.2K | ok 9.9K | ok 7.8K | ok 2.2K | ok 21.1K | ok 19.3K | ok 10.2K | ok 10.1K | ok 12.3K | ok 6.3K | ok 22.7K | ok 0.5K | ok 2.6K |
| multiply-14d | ok 14.7K | ok 37.0K | ok 53.8K | ok 13.4K | ok 22.1K | ok 32.0K | ok 46.2K | ok 32.5K | ok 87.4K | ok 114.0K | ok 17.7K | ok 11.6K | len 131.1K | ok 8.8K | ok 21.2K |
| multiply-20d | ok 11.8K | ok 32.4K | ok 28.2K | ok 17.9K | ok 32.9K | ok 22.2K | x 59.3K | ok 97.1K | ok 73.8K | ok 66.7K | x 23.2K | ok 20.5K | len 131.1K | ok 15.4K | ok 40.5K |
| grid-4x3 | ok 1.3K | ok 3.5K | ok 1.1K | ok 1.4K | ok 1.5K | ok 1.2K | ok 5.9K | ok 1.4K | ok 1.8K | ok 1.5K | ok 2.3K | ok 2.4K | ok 4.1K | ok 1.4K | ok 1.4K |
| grid-5x4 | ok 2.2K | ok 5.4K | ok 2.3K | ok 5.0K | ok 2.8K | ok 3.2K | ok 7.4K | ok 3.9K | ok 3.0K | ok 3.3K | ok 5.4K | ok 5.8K | ok 5.2K | ok 3.0K | ok 3.7K |
| grid-6x4 | ok 15.9K | ok 22.3K | ok 16.0K | ok 17.9K | ok 12.8K | ok 13.7K | ok 15.4K | ok 19.5K | ok 34.1K | ok 18.5K | ok 8.9K | ok 25.4K | ok 39.5K | ok 45.2K | ok 28.8K |
| tour-7 | ok 50.8K | ok 46.7K | ok 14.5K | ok 52.7K | ok 33.7K | ok 52.8K | ok 29.9K | ok 42.1K | ok 23.8K | ok 25.5K | ok 13.1K | ok 24.7K | ok 48.4K | ok 20.4K | ok 31.7K |
| tour-9 | ok 62.8K | ok 62.8K | ok 95.8K | ok 68.9K | ok 60.3K | ok 120.6K | ok 34.3K | ok 70.1K | len 131.1K | len 131.1K | ok 14.0K | ok 53.7K | len 131.1K | ok 62.4K | ok 68.0K |
| knapsack-16 | ok 62.8K | ok 43.9K | ok 35.3K | ok 63.1K | ok 36.3K | ok 41.6K | ok 25.7K | ok 65.4K | ok 81.5K | len 131.1K | ok 10.2K | ok 71.5K | len 131.1K | ok 47.6K | ok 50.7K |
| follow-acrostic | ok 1.2K | ok 1.4K | ok 1.0K | ok 1.2K | ok 1.5K | ok 1.1K | ok 5.7K | ok 1.3K | x 0.8K | ok 1.0K | ok 2.1K | ok 1.4K | ok 1.4K | ok 1.2K | ok 1.0K |
| follow-lipogram | ok 3.0K | ok 2.6K | ok 2.0K | ok 1.9K | ok 1.8K | ok 3.8K | ok 16.2K | ok 5.7K | ok 12.7K | ok 3.8K | ok 5.3K | ok 2.3K | ok 1.3K | ok 6.3K | ok 4.3K |
| follow-words | ok 6.0K | ok 8.8K | ok 14.8K | ok 7.6K | loop 13.7K | ok 12.0K | x 8.8K | loop 15.3K | ok 19.7K | ok 9.2K | ok 3.7K | ok 6.7K | ok 4.0K | loop 21.3K | ok 5.0K |
| long-table | ok 23.3K | ok 21.5K | ok 39.1K | ok 25.0K | ok 30.9K | ok 14.2K | ok 11.8K | x 34.5K | x 28.4K | ok 29.4K | ok 9.5K | ok 30.4K | ok 20.3K | ok 13.5K | ok 15.5K |
| long-hops | ok 1.1K | ok 3.5K | ok 1.7K | ok 1.9K | ok 1.5K | ok 2.3K | ok 3.5K | ok 2.4K | ok 0.8K | ok 0.7K | ok 1.5K | ok 1.5K | ok 2.8K | ok 3.7K | ok 1.7K |
| code-regex | ok 99.5K | x 37.5K | x 29.3K | ok 69.3K | x 61.6K | len 384.0K | loop 31.1K | ok 87.1K | ok 34.0K | x 56.1K | x 8.1K | x 24.7K | len 131.1K | ok 20.9K | x 21.6K |
| code-scheme | x 102.0K | ok 57.3K | ok 55.8K | x 109.2K | x 79.6K | x 20.0K | x 18.7K | x 87.9K | x 47.3K | x 49.9K | x 9.0K | x 38.9K | len 131.1K | x 31.4K | x 52.5K |
| code-ordered | ok 35.1K | ok 30.1K | ok 85.0K | ok 102.5K | ok 60.4K | ok 10.7K | ok 8.6K | x 95.9K | ok 14.9K | ok 18.3K | ok 6.6K | x 9.0K | len 131.1K | ok 18.1K | ok 19.7K |
| code-diff | ok 77.2K | ok 50.3K | len 130.9K | ok 79.6K | ok 82.0K | ok 186.1K | x 24.3K | ok 76.0K | ok 103.1K | len 131.1K | x 10.7K | x 24.4K | len 131.1K | ok 28.2K | ok 9.0K |
| code-sudoku | len 2.2K | x 32.1K | len 131.0K | x 59.1K | ok 63.6K | x 58.3K | ok 8.6K | x 55.5K | x 31.6K | x 110.3K | x 1.6K | x 11.8K | len 131.1K | x 19.4K | x 6.3K |
| code-json | len 16.0K | x 51.0K | ok 34.9K | len 4.6K | loop 40.9K | x 28.0K | x 12.9K | ok 46.0K | loop 31.4K | x 26.4K | x 7.5K | x 23.6K | x 89.6K | ok 33.1K | ok 22.5K |
| code-sat | ok 86.5K | x 69.0K | x 125.9K | x 91.1K | ok 108.9K | x 96.6K | x 19.4K | x 80.3K | x 86.8K | x 130.6K | x 4.0K | x 48.1K | len 131.1K | len 196.5K | ok 48.2K |
| kitchen | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 0/10 | 10/10 | 10/10 |
| **Strict** | **22** | **21** | **21** | **21** | **20** | **19** | **18** | **18** | **16** | **16** | **17** | **17** | **13** | **20** | **22** |
| Lenient | 22 | 21 | 21 | 21 | 20 | 19 | 18 | 19 | 17 | 16 | 18 | 17 | 13 | 20 | 22 |
| Minutes | 51 | 279 | 744 | 92 | 130 | 298 | 22 | 412 | 465 | 580 | 18 | 195 | 782 | 216 | 184 |

GLM's attempts on the 11 tasks it re-ran (`-`: not re-run at that level):

| Task | max, 131K | high, 256K | low, 256K | Best |
|---|---|---|---|---|
| multiply-14d | len 131K | ok 38.4K | - | ok |
| multiply-20d | len 131K | ok 30.8K | - | ok |
| tour-9 | len 131K | len 262K | len 262K | x |
| knapsack-16 | len 131K | x 250.9K | ok 69.8K | ok |
| code-regex | len 131K | ok 209.8K | - | ok |
| code-scheme | len 131K | len 262K | ok 13.8K | ok |
| code-ordered | len 131K | len 262K | x 2.1K | x |
| code-diff | len 131K | x 154.3K | - | x |
| code-sudoku | len 131K | loop 172K | - | x |
| code-sat | len 131K | x 227.5K | - | x |
| kitchen | 0/10 (stopped) | 9/10 | - | 9/10 |
| **Score (best per task)** | **13** | | | **18.9** |

How each model lost its points, by type (loops, running out of room, format slips, bugs, slow code), with each miss's
cause, patterns across models, tool use in the repair job and behaviour under the 24K budget:
[failure-modes.md](failure-modes.md).

## 24K round

| Group | Flash-Next IQ3_S | Flash-Next UD-Q4_K_XL | GLM-5.3-Flash S24 `high` | Nanbeige4.3-33B | MiMo-V2.6-Flash |
|---|---|---|---|---|---|
| Traces | 3 | 3 | 2 | 1 (3 lenient) | 3 |
| Multiplication | 3 | 3 | 1, 2 `len` | 1, 2 `len` | 1, 2 `len` |
| Logic grids | 3 | 3 | 2 | 3 | 3 |
| Routes, knapsack | 2 | 2 | 0, 2 `len` | 1, 2 `len` | 0, all `len` |
| Constrained writing | 3 | 3 | 2 | 1, 1 loop | 2, 1 `len` |
| Long input | 2 | 1, table `len` | 2 | 2 | 2 |
| Coding | 4 | 3 | 0, all `len` | 0, all `len` | 1 (JSON at 128K); 4 `len`, Sudoku too slow |
| Repair | 10/10, 90 s | 10/10, 107 s | 10/10, 139 s | 8/10, 25 s | 9/10, 224 s |
| **Score** | **21** | **19** | **10** | **9.8 (11.8)** | **12.9** |
| Time | 24 min | 34 min | 145 min | 109 min | 259 min |

MiMo's last three tasks (JSON, SAT, repair) ran at 128K. Partial coding re-runs with room before the at-best round:
- **GLM, 128K, no thinking cap:** 0 of 3. It never converged on two of them (57-59 min each), re-deciding and drafting
  code inside its thinking.
- **Nanbeige, 96K:** 1 of 3 (the diff, 62K tokens).

| Task | Flash-Next IQ3_S | Flash-Next Q4 | MiMo | GLM S24 | Nanbeige |
|---|---|---|---|---|---|
| trace-1 | ok 1.7K | ok 2.0K | ok 1.8K | ok 2.0K | ok* 2.2K |
| trace-2 | ok 6.4K | ok 5.5K | ok 3.8K | x 3.1K | ok* 9.6K |
| trace-3 | ok 5.7K | ok 6.8K | ok 5.8K | ok 15.4K | ok 9.1K |
| multiply-8d | ok 4.7K | ok 9.9K | ok 10.0K | ok 3.0K | ok 13.1K |
| multiply-14d | ok 14.7K | ok 13.4K | len 24.0K | len 24.0K | len 24.0K |
| multiply-20d | ok 18.3K | ok 18.1K | len 24.0K | len 24.0K | len 24.0K |
| grid-4x3 | ok 1.3K | ok 1.4K | ok 2.0K | ok 3.3K | ok 1.6K |
| grid-5x4 | ok 2.2K | ok 5.0K | ok 3.5K | ok 7.7K | ok 4.7K |
| grid-6x4 | ok 15.9K | ok 17.9K | ok 18.0K | x 22.4K | ok 21.6K |
| tour-7 | ok 18.0K | x 18.1K | len 24.0K | len 24.0K | len 24.0K |
| tour-9 | x 18.0K | ok 18.1K | len 24.0K | x 5.5K | len 24.0K |
| knapsack-16 | ok 18.3K | ok 18.2K | len 24.0K | len 24.0K | ok 14.2K |
| follow-acrostic | ok 1.2K | ok 1.2K | ok 1.0K | ok 0.5K | ok 0.8K |
| follow-lipogram | ok 3.0K | ok 1.9K | ok 6.0K | ok 0.9K | x 3.0K |
| follow-words | ok 6.0K | ok 7.6K | len 24.0K | x 2.6K | len 24.0K |
| long-table | ok 23.3K | len 24.0K | ok 20.2K | ok 9.4K | ok 16.9K |
| long-hops | ok 1.1K | ok 1.9K | ok 0.8K | ok 0.7K | ok 0.8K |
| code-regex | x 20.3K | x 19.9K | len 24.0K | len 24.0K | len 24.0K |
| code-scheme | x 23.3K | x 23.1K | len 24.0K | len 24.0K | len 24.0K |
| code-ordered | ok 19.6K | ok 19.6K | len 24.0K | len 24.0K | len 24.0K |
| code-diff | ok 18.8K | x 20.7K | len 24.0K | len 24.0K | len 24.0K |
| code-sudoku | ok 19.5K | ok 20.2K | x 16.8K | len 24.0K | len 24.0K |
| code-json | len 16.0K | len 4.6K | ok 46.0K | len 24.0K | len 24.0K |
| code-sat | ok 20.8K | ok 21.2K | x 96.3K | len 24.0K | len 24.0K |
| kitchen | 10/10 | 10/10 | 9/10 | 10/10 | 8/10 |
| **Strict** | **21** | **19** | **12.9** | **10** | **9.8** |
| Lenient | 21 | 19 | 12.9 | 10 | 11.8 |
| Minutes | 24 | 34 | 259 | 145 | 109 |

## Raw data

`results/hard-test-tasks.csv` columns:

| Column | Holds |
|---|---|
| `run`, `task`, `group` | The run label, the task and its group |
| `verdict` | `ok` / `ok*` / `x` / `len` / `loop` / `n/10` |
| `strict_point`, `lenient_point` | The points on each scale |
| `finish` | The engine's finish reason |
| `tokens` | Tokens written |
| `seconds` | Wall time, prefill included |
| `rescored` | Set when the row came from a later checker fix |

There's one row per run and task: the last one, as the scorer reads them. Answer texts aren't included.

| Label | Run |
|---|---|
| `flash-next`, `flash-next-q4` | Flash-Next IQ3_S and UD-Q4_K_XL, 24K round |
| `flash-next-max`, `flash-next-q4-max` | Their 11 capped tasks re-run at 262K |
| `glm` | GLM S24 `high`, 24K round |
| `glm-max`, `glm-64k` | Coding re-runs: 3 tasks at 128K with no cap, 2 at 64K |
| `glm-best` | GLM at `max` with 131K |
| `glm-best-high256`, `glm-best-low256` | GLM's re-runs with 262K at `high` and `low` |
| `mimo-best` | MiMo RL at its best |
| `mimo-mopd-best` | MiMo MOPD at RL's settings |
| `nanbeige`, `nanbeige-max` | 24K round, and 3 coding re-runs at 96K |
| `nanbeige-best` | At its best |
| `nanbeige-best-llamacpp-defaults` | The first 4 tasks of that run with llama.cpp's default top-k 40 / min-p 0.05, stopped and restarted with both off. Same verdicts, with the print slip moved from trace-2 to trace-1 |
| `mimo`, `mimo-max` | 24K round, and the last 3 tasks at 128K |
| `deepseek-max`, `ds-coder-60k`, `deepseek-max-token-only` | At max settings, its coding at 60K, and a 2-task side check |
| `qwen27b-max`, `gemma26b-q8-max`, `gemma31b-max`, `nemotron-q3-max`, `inkling-max`, `ling-max` | At their best |
| `inkling2-max` | Inkling 2 Small at its best |
| `inkling2-max-v0` | Its first 5 tasks on the first, slower build of the same engine (all right), before the run was paused for the speedups and restarted from task 1 |
| `gemma31b-max-code-t06` | The 31B's coding at temperature 0.6 |

Checker fixes along the way:
- An import ban was applied to every coding task, not only where the prompt forbids a module.
- Code-block extraction kept an unclosed fence.
- The knapsack check refused a list inside a one-key object.

Re-scoring changed no earlier verdict, and every wrong answer was also checked by hand.

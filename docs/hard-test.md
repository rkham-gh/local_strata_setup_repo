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

**At their best (2026-11-03 to 11-07):**
- each model at its card's sampling and top reasoning effort, with as much output as fits here;
- engine thinking caps off: Maya run configs got `"thinking_budget": 0, "fit_max_tokens": true` for that start only;
- `--loop-guard` on, and `--tool-thinking` on wherever the template replays reasoning inside a tool loop;
- answers that hit the budget (not loops) on GLM and MiMo are re-run with 256K and reported separately, as Flash-Next's
  were.

| Model | Sampling | Effort | Output room |
|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S (Strata) | 0.6 / top-p 0.95 | default | 262,144, trimmed to the context left. Only the 11 tasks the 24K round cut short were re-run |
| Inkling-Small (maya-ink) | 1.0 / 1.0 | `max` (0.99) | to the 128K context. maya-ink has no thinking cap |
| Ling-3.1-flash (maya-ling) | 1.0 / 0.95 / top-k 20 | always on | to the 128K context |
| Qwen3.8-27B (llama.cpp) | 1.0 / 0.95 / top-k 20 / min-p 0 | `xhigh` | to the 208K context |
| DeepSeek-V4.1-Flash Coder (Maya) | 0.6 / 0.95 | `max` | 384,000, its listed maximum, in 1M |
| Gemma 4 26B-A4B, 31B (llama.cpp) | 1.0 / 0.95 / top-k 64 | on | to the 224K context |
| Nemotron 3.5 Super UD-Q3_K_XL (maya-nem) | 1.0 / 0.95 | full mode | to the 256K context |
| Nanbeige4.3-33B (llama.cpp) | 0.6 / 0.95, top-k and min-p off (the card's reasoning row) | always on | to 96K: the most context that fits in VRAM with q8_0 KV |
| GLM-5.3-Flash Maya-S24 | 1.0 / 0.95, stop on `<\|assistant\|>` | `max` | 131,072, the listed output, in 256K. Capped tasks re-run at 256K |
| MiMo-V2.6-Flash-RL (strata-mimo, `MIMO_TAIL_KEEP=5`) | 1.0 / 0.95 (the card) | always on | 131,072 in 256K. Capped tasks re-run at 256K |

## Results at their best

| Model | Strict | Lenient | Coding | Wall time | Tokens | Avg decode |
|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | **22** | 22 | 4 | 51 min | 711K | 233 tok/s |
| Inkling-Small UD-Q4_K_XL | **21** | 21 | 3 | 279 min | 671K | 40 tok/s |
| Ling-3.1-flash IQ2_XS | **21** | 21 | 3 (+2 cut off at 128K) | 744 min | 992K | 22 tok/s |
| Qwen3.8-Flash-Next UD-Q4_K_XL | **21** | 21 | 3 | 92 min | 830K | 150 tok/s |
| Qwen3.8-27B Q6_K | **20** | 20 | 4 | 130 min | 788K | 101 tok/s |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | **19** | 19 | 2 | 298 min | 1,132K | 63 tok/s |
| Gemma 4 26B-A4B Q8_0 | **18** | 18 | 2 | 22 min | 458K | 349 tok/s |
| Nanbeige4.3-33B Q4_K_M | **18** | 19 | 3 | 412 min | 1,034K | 42 tok/s |
| Gemma 4 31B QAT | **17** | 18 | 1 | 18 min | 201K | 185 tok/s |
| Nemotron 3.5 Super UD-Q3_K_XL | **17** | 17 (18 with trace-3's line outside the block) | 0 | 195 min | 472K | 40 tok/s |
| GLM-5.3-Flash Maya-S24 | running | | | | | |
| MiMo-V2.6-Flash-RL | queued | | | | | |

Average decode includes prefill time. It isn't the short-prompt decode in the README:
- **Expert-caching engines get faster on long single-domain answers**, as the cache converges: Inkling 40 tok/s here
  against 30 on `ab.py`.
- **Dense models slow down with depth.** Nanbeige writes 69 tok/s on a new prompt and about 39 on its long coding
  answers.
- **Drafting goes faster on long chains of thought.** DeepSeek's Maya engine decoded 55-62 tok/s through the test,
  against 33-35 on new short prompts.

Per task. Each cell is the verdict and the tokens written:

| Task | Flash-Next IQ3_S | Inkling-Small | Ling-3.1-flash | Flash-Next Q4 | Qwen3.8-27B | DeepSeek Coder | Gemma 4 26B | Nanbeige4.3-33B | Gemma 4 31B | Nemotron Q3 |
|---|---|---|---|---|---|---|---|---|---|---|
| trace-1 | ok 1.7K | ok 1.8K | ok 4.2K | ok 2.0K | ok 2.0K | ok 2.2K | ok 6.7K | ok 2.8K | ok 4.5K | ok 2.9K |
| trace-2 | ok 6.4K | ok 11.1K | ok 10.9K | ok 5.5K | ok 7.4K | x 6.5K | ok 12.7K | ok* 7.6K | ok* 7.2K | ok 7.6K |
| trace-3 | ok 5.7K | ok 17.5K | ok 17.9K | ok 6.8K | x 6.9K | ok 6.4K | ok 15.6K | ok 19.1K | ok 8.1K | x 10.3K |
| multiply-8d | ok 4.7K | ok 9.0K | ok 35.2K | ok 9.9K | ok 7.8K | ok 2.2K | ok 21.1K | ok 19.3K | ok 12.3K | ok 6.3K |
| multiply-14d | ok 14.7K | ok 37.0K | ok 53.8K | ok 13.4K | ok 22.1K | ok 32.0K | ok 46.2K | ok 32.5K | ok 17.7K | ok 11.6K |
| multiply-20d | ok 11.8K | ok 32.4K | ok 28.2K | ok 17.9K | ok 32.9K | ok 22.2K | x 59.3K | ok 97.1K | x 23.2K | ok 20.5K |
| grid-4x3 | ok 1.3K | ok 3.5K | ok 1.1K | ok 1.4K | ok 1.5K | ok 1.2K | ok 5.9K | ok 1.4K | ok 2.3K | ok 2.4K |
| grid-5x4 | ok 2.2K | ok 5.4K | ok 2.3K | ok 5.0K | ok 2.8K | ok 3.2K | ok 7.4K | ok 3.9K | ok 5.4K | ok 5.8K |
| grid-6x4 | ok 15.9K | ok 22.3K | ok 16.0K | ok 17.9K | ok 12.8K | ok 13.7K | ok 15.4K | ok 19.5K | ok 8.9K | ok 25.4K |
| tour-7 | ok 50.8K | ok 46.7K | ok 14.5K | ok 52.7K | ok 33.7K | ok 52.8K | ok 29.9K | ok 42.1K | ok 13.1K | ok 24.7K |
| tour-9 | ok 62.8K | ok 62.8K | ok 95.8K | ok 68.9K | ok 60.3K | ok 120.6K | ok 34.3K | ok 70.1K | ok 14.0K | ok 53.7K |
| knapsack-16 | ok 62.8K | ok 43.9K | ok 35.3K | ok 63.1K | ok 36.3K | ok 41.6K | ok 25.7K | ok 65.4K | ok 10.2K | ok 71.5K |
| follow-acrostic | ok 1.2K | ok 1.4K | ok 1.0K | ok 1.2K | ok 1.5K | ok 1.1K | ok 5.7K | ok 1.3K | ok 2.1K | ok 1.4K |
| follow-lipogram | ok 3.0K | ok 2.6K | ok 2.0K | ok 1.9K | ok 1.8K | ok 3.8K | ok 16.2K | ok 5.7K | ok 5.3K | ok 2.3K |
| follow-words | ok 6.0K | ok 8.8K | ok 14.8K | ok 7.6K | loop 13.7K | ok 12.0K | x 8.8K | loop 15.3K | ok 3.7K | ok 6.7K |
| long-table | ok 23.3K | ok 21.5K | ok 39.1K | ok 25.0K | ok 30.9K | ok 14.2K | ok 11.8K | x 34.5K | ok 9.5K | ok 30.4K |
| long-hops | ok 1.1K | ok 3.5K | ok 1.7K | ok 1.9K | ok 1.5K | ok 2.3K | ok 3.5K | ok 2.4K | ok 1.5K | ok 1.5K |
| code-regex | ok 99.5K | x 37.5K | x 29.3K | ok 69.3K | x 61.6K | len 384.0K | loop 31.1K | ok 87.1K | x 8.1K | x 24.7K |
| code-scheme | x 102.0K | ok 57.3K | ok 55.8K | x 109.2K | x 79.6K | x 20.0K | x 18.7K | x 87.9K | x 9.0K | x 38.9K |
| code-ordered | ok 35.1K | ok 30.1K | ok 85.0K | ok 102.5K | ok 60.4K | ok 10.7K | ok 8.6K | x 95.9K | ok 6.6K | x 9.0K |
| code-diff | ok 77.2K | ok 50.3K | len 130.9K | ok 79.6K | ok 82.0K | ok 186.1K | x 24.3K | ok 76.0K | x 10.7K | x 24.4K |
| code-sudoku | len 2.2K | x 32.1K | len 131.0K | x 59.1K | ok 63.6K | x 58.3K | ok 8.6K | x 55.5K | x 1.6K | x 11.8K |
| code-json | len 16.0K | x 51.0K | ok 34.9K | len 4.6K | loop 40.9K | x 28.0K | x 12.9K | ok 46.0K | x 7.5K | x 23.6K |
| code-sat | ok 86.5K | x 69.0K | x 125.9K | x 91.1K | ok 108.9K | x 96.6K | x 19.4K | x 80.3K | x 4.0K | x 48.1K |
| kitchen | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 |
| **Strict** | **22** | **21** | **21** | **21** | **20** | **19** | **18** | **18** | **17** | **17** |
| Lenient | 22 | 21 | 21 | 21 | 20 | 19 | 18 | 19 | 18 | 17 |
| Minutes | 51 | 279 | 744 | 92 | 130 | 298 | 22 | 412 | 18 | 195 |

Notes:
- **Flash-Next IQ3_S.**
  - With room it solved all three routes and knapsack, and the regex engine (99.5K tokens).
  - Sudoku looped: it recalled a puzzle and wrote zeros until Strata's repeat guard fired.
  - The Scheme interpreter timed out at 102K, and the JSON parser looped at 24K and wasn't re-run.
  - UD-Q4_K_XL is within one task at either budget; single tasks flip between runs.
- **Inkling-Small.** Its Scheme interpreter was the first to pass.
  - Regex: it calls its own helper with an extra argument. JSON: it rejects `1e400`, over-applying the no-`Infinity`
    rule to a valid literal that overflows.
  - Sudoku 19.6 s against a 10 s limit, and SAT past 90 s; both reproduce. The repair job passed 10/10 without ever
    calling `finish` (all 24 turns).
- **Ling-3.1-flash.**
  - It passed JSON, the second run to do so after MiMo, and Scheme.
  - Diff and Sudoku were still thinking at the 128K context (131K tokens, about 105 min each). It hasn't been run at
    256K: that would cost about 1.4 GB of VRAM, at 10.4 KB of MLA cache a token.
  - SAT ended at 126K saying a satisfiable formula is unsat, and regex rejects a valid `?`.
- **Qwen3.8-27B.** It solved ordered, diff, Sudoku and SAT at 60-109K tokens each.
  - Regex got 5 cases wrong, Scheme fails a plain `cond`, and JSON looped.
  - Trace-3 had two wrong values, and the word list looped.
- **DeepSeek Coder.** The 9-city route took 120K tokens, the diff 186K.
  - Regex looped to the full 384K before the guard was rebuilt. The first guard looked for exact repeats, and each
    paragraph carried a new number.
  - Scheme stopped mid-function, and Sudoku took 32.7 s against a 10 s limit.
- **Gemma 4.** It solves puzzles cheaply: the 31B did both routes and the knapsack in 10-14K tokens each.
  - Every coding answer ended on its own at 1.6-24K tokens, so its failures are plain bugs: lazy quantifiers, an
    undefined Scheme symbol, surrogate pairs, "unsatisfiable" SAT, diffs that don't rebuild their inputs.
  - The 31B's coding re-run at temperature 0.6 gave 2/7 against 1/7: noise, not the sampling.
- **Nanbeige4.3-33B.** With room it gained 8 tasks, all of which had run out of budget at 24K.
  - It solved regex (the second, after Flash-Next with room), JSON and diff.
  - Scheme raises a `TypeError`, and Sudoku an `IndexError`. SAT ended with a stray `</code>` instead of the closing
    fence, and still times out without it.
  - The word list looped again, and long-table had 5 wrong rows (right at 24K).
  - The 20-digit product took 97K tokens, just inside the context.
- **Nemotron 3.5 Super.** All non-coding tasks were right, counting trace-3, whose correct line came outside the code
  block. Every coding answer ended on its own at 9-48K tokens with a bug. One Q3 run can't separate the model from the
  quant: Q3's top-1 matches Q4's 97.2% of the time.

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
| `glm-best` | GLM at its best (running) |
| `nanbeige`, `nanbeige-max` | 24K round, and 3 coding re-runs at 96K |
| `nanbeige-best` | At its best |
| `nanbeige-best-llamacpp-defaults` | The first 4 tasks of that run with llama.cpp's default top-k 40 / min-p 0.05, stopped and restarted with both off. Same verdicts, with the print slip moved from trace-2 to trace-1 |
| `mimo`, `mimo-max` | 24K round, and the last 3 tasks at 128K |
| `deepseek-max`, `ds-coder-60k`, `deepseek-max-token-only` | At max settings, its coding at 60K, and a 2-task side check |
| `qwen27b-max`, `gemma26b-q8-max`, `gemma31b-max`, `nemotron-q3-max`, `inkling-max`, `ling-max` | At their best |
| `gemma31b-max-code-t06` | The 31B's coding at temperature 0.6 |

Checker fixes along the way:
- An import ban was applied to every coding task, not only where the prompt forbids a module.
- Code-block extraction kept an unclosed fence.
- The knapsack check refused a list inside a one-key object.

Re-scoring changed no earlier verdict, and every wrong answer was also checked by hand.

# The hard test

Most quick checks (tool calls, JSON, short reasoning) are passed by every model here, so they don't separate them. The hard
test climbs in difficulty until models fail. It has 25 tasks, each generated from a fixed seed, so every model gets
exactly the same problems, and none could have seen them before. Every answer is scored by code, not by eye.

| Group | Tasks | How it's checked |
|---|---|---|
| Traced programs | trace-1, trace-2, trace-3 | The model works out by hand what a short Python program prints. The line must match exactly, in the final code block |
| Long multiplication | multiply-8d, -14d, -20d | The exact product of two 8-, 14- and 20-digit numbers, by hand |
| Logic grids | grid-4x3, -5x4, -6x4 | Puzzles with exactly one solution |
| Optimization | tour-7, tour-9, knapsack-16 | Shortest 7- and 9-city routes and a 16-item knapsack, compared with exact solvers |
| Following instructions | follow-acrostic, -lipogram, -words | Constrained writing, checked mechanically |
| Long input | long-table, long-hops | A 200-row JSON table to transform, and a chain of clues hidden in a 52K-token document |
| Coding | code-regex, -scheme, -ordered, -diff, -sudoku, -json, -sat | A regex engine, a Scheme interpreter, an ordered multiset, a shortest-edit-script diff, a Sudoku solver, a JSON parser and a SAT solver. Each runs against hidden tests, most with time limits |
| Repair job (`kitchen`) | 1 | Fix a 7-file project through tool calls (read, write, run tests, finish). Scored by 10 hidden tests |

**Scoring.**
- **Strict:** one point a task, with the repair job counting as its share of the 10 hidden tests (8 of 10 is 0.8).
- **Lenient:** also counts a traced program whose values are all right but printed differently from `print()`, such as
  commas between arguments or `''` for an empty string. Tables mark those `ok*`.
- **Markers:** `len` means the answer ran out of the token budget or the context. `loop` means the loop guard ended an
  answer that kept repeating itself.
- Neither `len` nor `loop` scores. A `len` shows the budget was too small, not that the model can't do the task.

## Two rounds

1. **The 24K round, 2026-11-02.** Every model had a 24,000-token budget per request, at temperature 0.6 and top-p 0.95.
   - The engines' own thinking caps applied. Strata ends Flash-Next's thinking at three quarters of `max_tokens`, so its
     thinking stopped at 18K on 10 tasks. Project Maya closes thinking at 32,768 tokens.
   - About half of the slower models' tasks hit the 24K cap while still thinking.
2. **At their best, 2026-11-03 to 2026-11-06.** Each model ran at its model card's sampling and highest reasoning
   effort, with as much output room as it can use on this PC.
   - Engine thinking caps were off. Maya's run configs had `"thinking_budget": 0` and `"fit_max_tokens": true` for that
     start only.
   - `--loop-guard` was on.
   - `--tool-thinking` was on wherever the chat template keeps earlier thinking in a tool loop. With it, the repair
     job's assistant turns go back with their `reasoning_content`.

| Model (at its best) | Sampling | Reasoning effort | Output room per answer |
|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S, Strata | temperature 0.6, top-p 0.95 | default | 262,144-token budget, trimmed to the context left. Only the 11 tasks Strata's cap cut short were re-run with it; the other 14 never reached the cap |
| Inkling-Small, maya-ink | 1.0, top-p 1.0 | `max` (0.99, the level its lab evaluated at) | 262,144 budget, stops at the 128K context |
| Ling-3.1-flash, maya-ling | 1.0, top-p 0.95, top-k 20 | thinks always (no levels) | 262,144 budget, stops at the 128K context |
| Qwen3.8-27B, llama.cpp | 1.0, top-p 0.95, top-k 20, min-p 0 | `xhigh` (its highest) | 262,144 budget, stops at the 208K context |
| DeepSeek-V4.1-Flash Coder, Maya | 0.6, top-p 0.95 | `max` | 384,000 (its listed maximum output), 1M context |
| Gemma 4 26B-A4B and 31B, llama.cpp | 1.0, top-p 0.95, top-k 64 | thinking on (no levels) | 262,144 budget, stops at the 224K context |
| Nemotron 3.5 Super UD-Q3_K_XL, maya-nem | 1.0, top-p 0.95 | thinking on, full mode | 262,144 budget, stops at the 256K context |
| Nanbeige4.3-33B, llama.cpp | 0.6, top-p 0.95, top-k and min-p off (the card's reasoning row) | always thinks | 262,144 budget, stops at the 96K context, the most that fits in VRAM with a q8_0 cache |
| GLM-5.3-Flash Maya-S24 | 1.0, top-p 0.95 | `max` | 131,072 (its listed output limit) in a 256K context |

The Nanbeige and GLM runs at their best were still in progress when this was written. Their 24K results are below.

## Results at their best

| Model | Score of 25 (strict) | Lenient | Coding, of 7 | Whole test | Tokens written | Average writing speed |
|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | **22** | 22 | 4 | 51 min | 711K | 233 tok/s |
| Inkling-Small UD-Q4_K_XL | **21** | 21 | 3 | 279 min | 671K | 40 tok/s |
| Ling-3.1-flash IQ2_XS | **21** | 21 | 3, plus 2 cut off by the 128K context | 744 min | 992K | 22 tok/s |
| Qwen3.8-Flash-Next UD-Q4_K_XL | **21** | 21 | 3 | 92 min | 830K | 150 tok/s |
| Qwen3.8-27B Q6_K | **20** | 20 | 4 | 130 min | 788K | 101 tok/s |
| DeepSeek-V4.1-Flash Coder IQ2_XXS | **19** | 19 | 2 | 298 min | 1,132K | 63 tok/s |
| Gemma 4 26B-A4B Q8_0 | **18** | 18 | 2 | 22 min | 458K | 349 tok/s |
| Gemma 4 31B QAT 4-bit | **17** | 18 | 1 | 18 min | 201K | 185 tok/s |
| Nemotron 3.5 Super UD-Q3_K_XL | **17** | 17 (18 counting an answer outside its code block) | 0 | 195 min | 472K | 40 tok/s |

The writing speeds are averages over whole answers, including the time spent reading prompts. They can differ from the
short-prompt speeds in `models.md`. Long answers on engines with an adaptive VRAM expert cache run faster than new short
prompts do (Inkling 40 against 30 tok/s), and dense models slow down as an answer grows (Nanbeige 69 tok/s on a new
prompt, about 39 on its long coding answers).

Per task, at their best. Each cell is the verdict and the tokens written:

| Task | Flash-Next IQ3_S | Inkling-Small | Ling-3.1-flash | Flash-Next Q4 | Qwen3.8-27B | DeepSeek Coder | Gemma 4 26B | Gemma 4 31B | Nemotron Q3 |
|---|---|---|---|---|---|---|---|---|---|
| trace-1 | ok 1.7K | ok 1.8K | ok 4.2K | ok 2.0K | ok 2.0K | ok 2.2K | ok 6.7K | ok 4.5K | ok 2.9K |
| trace-2 | ok 6.4K | ok 11.1K | ok 10.9K | ok 5.5K | ok 7.4K | x 6.5K | ok 12.7K | ok* 7.2K | ok 7.6K |
| trace-3 | ok 5.7K | ok 17.5K | ok 17.9K | ok 6.8K | x 6.9K | ok 6.4K | ok 15.6K | ok 8.1K | x 10.3K |
| multiply-8d | ok 4.7K | ok 9.0K | ok 35.2K | ok 9.9K | ok 7.8K | ok 2.2K | ok 21.1K | ok 12.3K | ok 6.3K |
| multiply-14d | ok 14.7K | ok 37.0K | ok 53.8K | ok 13.4K | ok 22.1K | ok 32.0K | ok 46.2K | ok 17.7K | ok 11.6K |
| multiply-20d | ok 11.8K | ok 32.4K | ok 28.2K | ok 17.9K | ok 32.9K | ok 22.2K | x 59.3K | x 23.2K | ok 20.5K |
| grid-4x3 | ok 1.3K | ok 3.5K | ok 1.1K | ok 1.4K | ok 1.5K | ok 1.2K | ok 5.9K | ok 2.3K | ok 2.4K |
| grid-5x4 | ok 2.2K | ok 5.4K | ok 2.3K | ok 5.0K | ok 2.8K | ok 3.2K | ok 7.4K | ok 5.4K | ok 5.8K |
| grid-6x4 | ok 15.9K | ok 22.3K | ok 16.0K | ok 17.9K | ok 12.8K | ok 13.7K | ok 15.4K | ok 8.9K | ok 25.4K |
| tour-7 | ok 50.8K | ok 46.7K | ok 14.5K | ok 52.7K | ok 33.7K | ok 52.8K | ok 29.9K | ok 13.1K | ok 24.7K |
| tour-9 | ok 62.8K | ok 62.8K | ok 95.8K | ok 68.9K | ok 60.3K | ok 120.6K | ok 34.3K | ok 14.0K | ok 53.7K |
| knapsack-16 | ok 62.8K | ok 43.9K | ok 35.3K | ok 63.1K | ok 36.3K | ok 41.6K | ok 25.7K | ok 10.2K | ok 71.5K |
| follow-acrostic | ok 1.2K | ok 1.4K | ok 1.0K | ok 1.2K | ok 1.5K | ok 1.1K | ok 5.7K | ok 2.1K | ok 1.4K |
| follow-lipogram | ok 3.0K | ok 2.6K | ok 2.0K | ok 1.9K | ok 1.8K | ok 3.8K | ok 16.2K | ok 5.3K | ok 2.3K |
| follow-words | ok 6.0K | ok 8.8K | ok 14.8K | ok 7.6K | loop 13.7K | ok 12.0K | x 8.8K | ok 3.7K | ok 6.7K |
| long-table | ok 23.3K | ok 21.5K | ok 39.1K | ok 25.0K | ok 30.9K | ok 14.2K | ok 11.8K | ok 9.5K | ok 30.4K |
| long-hops | ok 1.1K | ok 3.5K | ok 1.7K | ok 1.9K | ok 1.5K | ok 2.3K | ok 3.5K | ok 1.5K | ok 1.5K |
| code-regex | ok 99.5K | x 37.5K | x 29.3K | ok 69.3K | x 61.6K | len 384.0K | loop 31.1K | x 8.1K | x 24.7K |
| code-scheme | x 102.0K | ok 57.3K | ok 55.8K | x 109.2K | x 79.6K | x 20.0K | x 18.7K | x 9.0K | x 38.9K |
| code-ordered | ok 35.1K | ok 30.1K | ok 85.0K | ok 102.5K | ok 60.4K | ok 10.7K | ok 8.6K | ok 6.6K | x 9.0K |
| code-diff | ok 77.2K | ok 50.3K | len 130.9K | ok 79.6K | ok 82.0K | ok 186.1K | x 24.3K | x 10.7K | x 24.4K |
| code-sudoku | len 2.2K | x 32.1K | len 131.0K | x 59.1K | ok 63.6K | x 58.3K | ok 8.6K | x 1.6K | x 11.8K |
| code-json | len 16.0K | x 51.0K | ok 34.9K | len 4.6K | loop 40.9K | x 28.0K | x 12.9K | x 7.5K | x 23.6K |
| code-sat | ok 86.5K | x 69.0K | x 125.9K | x 91.1K | ok 108.9K | x 96.6K | x 19.4K | x 4.0K | x 48.1K |
| kitchen | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 | 10/10 |
| **Strict** | **22** | **21** | **21** | **21** | **20** | **19** | **18** | **17** | **17** |
| Lenient | 22 | 21 | 21 | 21 | 20 | 19 | 18 | 18 | 17 |
| Minutes | 51 | 279 | 744 | 92 | 130 | 298 | 22 | 18 | 195 |

### Notes per model

- **Flash-Next IQ3_S, with room to think.** All three routes and the knapsack were right, and so was the regex engine.
  - It lost the Sudoku solver to a loop: recalling a puzzle, it wrote zeros until Strata's guard ended the reply.
  - Its Scheme interpreter timed out after 102K tokens.
  - The JSON parser looped at 24K and wasn't re-run.
  - UD-Q4_K_XL scored 21 with room, against 22. Single tasks flip between runs, so the two quants read as even. IQ3_S
    stays the default because it writes twice as fast.
- **Inkling-Small.** Every task outside coding was right. Its Scheme interpreter was the first of any run to pass.
  - The regex engine crashed: it calls one of its own helpers with five arguments where it takes four.
  - The JSON parser refused `1e400`, a valid number that overflows to infinity, where `json.loads` returns `inf`.
  - The Sudoku solver took 19.6 s against a 10 s limit, and the SAT solver ran past its 90 s. Both did the same when re-run
    with nothing else on the machine. The engine uses no CPU between requests, so the code itself is slow.
  - In the repair job it passed 10 of 10 but never called `finish`, using all 24 turns.
- **Ling-3.1-flash.** Every task outside coding was right. Its JSON parser is the second of any run to pass, after
  MiMo's. The Scheme interpreter passed too.
  - The diff and the Sudoku solver were still thinking when they reached the 128K context, after about 131K tokens and
    105 minutes each. The model supports 256K, which costs about 1.4 GB more VRAM but wasn't tried.
  - The SAT solver ended by itself at 126K tokens but calls a satisfiable formula unsatisfiable.
  - The regex engine rejects a valid `?`.
  - It thinks long: 35K tokens on the 8-digit product, where Inkling used 9K.
- **Qwen3.8-27B.** Everything but coding was right except trace-3 (two wrong values) and the word list, which looped
  until the guard ended it.
  - Coding was 4 of 7: the ordered multiset, diff, Sudoku and SAT solver, each after 60K-109K tokens.
  - The regex engine got 5 cases wrong, the Scheme interpreter fails a plain `cond`, and the JSON parser looped.
- **DeepSeek-V4.1-Flash Coder.** It solved the 9-city route in 120K tokens.
  - Coding was 2 of 7: the ordered multiset, and the diff after 186K tokens.
  - The regex answer used all 384K tokens in a loop. The loop guard was rebuilt after this run, because the first
    version looked for exact repeats and each paragraph carried a new number.
  - The Scheme interpreter stopped mid-function. The Sudoku solver took 32.7 s against a 10 s limit. The JSON parser and
    the SAT solver had bugs.
  - Trace-2 had a wrong value.
- **Gemma 4.** Both solve puzzles with little thinking: the 31B did both routes and the knapsack in 10-14K tokens each.
  - Coding is where they fall behind. Every coding answer ended by itself at 1.6K-24K tokens, so none was cut short, and
    the failures were real bugs: lazy quantifiers in the regex engine, an undefined symbol in the Scheme interpreter,
    surrogate pairs in the JSON parser, SAT solvers that say "unsatisfiable", and diffs that don't rebuild their inputs.
  - The 26B's regex engine looped.
  - Google's temperature 1.0 isn't what holds them back. The 31B's coding tasks re-run at 0.6 solved 2 of 7 against 1,
    within the noise.
- **Nemotron 3.5 Super.** Outside coding, every answer was right, counting trace-3, whose correct line was given outside
  the code block the prompt asks for.
  - Coding was 0 of 7. Every coding answer ended by itself at 9K-48K tokens with a bug in its own code.
  - One run can't tell the model's coding from the quant's. Q3's top token matches Q4's 97% of the time.

## The 24K round

Every model at temperature 0.6, top-p 0.95 and a 24,000-token budget per request:

| Group | Flash-Next IQ3_S | Flash-Next UD-Q4_K_XL | GLM-5.3-Flash S24, `high` | Nanbeige4.3-33B | MiMo-V2.6-Flash |
|---|---|---|---|---|---|
| Traced programs (3) | 3 | 3 | 2 | 1 (3 counting two print-format slips) | 3 |
| Multiplying 8, 14 and 20 digits | 3 | 3 | 1, two out of budget | 1, two out of budget | 1, two out of budget |
| Logic grids | 3 | 3 | 2 | 3 | 3 |
| Routes and knapsack | 2 | 2 | 0, two out of budget | 1, two out of budget | 0, all out of budget |
| Following instructions | 3 | 3 | 2 | 1, one looped | 2, one out of budget |
| Table and clue chain | 2 | 1, the table out of budget | 2 | 2 | 2 |
| Coding (7) | 4 | 3 | 0, all out of budget | 0, all out of budget | 1 (of its last 2, run at 128K); 4 out of budget, Sudoku too slow |
| Repair job | 10 of 10 in 90 s | 10 of 10 in 107 s | 10 of 10 in 139 s | 8 of 10 in 25 s | 9 of 10 in 224 s |
| **Score of 25** | **21** | **19** | **10** | **9.8 (11.8)** | **12.9** |
| Time | 24 min | 34 min | 145 min | 109 min | 259 min |

Per task, 24K round:

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

- GLM and Nanbeige ran out of budget on every coding task. Three of those tasks re-ran with as much room as each could
  get:
  - GLM, with no thinking cap and 128K of output, solved none of the three. It never converged on two of them (57-59
    minutes each): it keeps re-deciding and drafting code inside its thinking.
  - Nanbeige, at 96K, ended all three by itself and solved the diff, after 62K tokens.
  - MiMo's last three tasks ran at 128K. It solved the JSON parser in 46K tokens, and its SAT solver returned a wrong
    assignment after 96K.

## Method notes

These rules each come from a wrong result during this testing:

- **A budget cut isn't a failure.** A low cap once made GLM look as if it couldn't solve a coding task it could. That's
  why the at-their-best round exists.
- **Loop guard.** `--loop-guard` ends an answer when its last 20,000 characters compress more than 15 times with zlib.
  Every loop seen compressed 44-465 times, every real answer at most 7.7. Without it, one looping answer ran to 384K
  tokens.
- **Timed code answers keep their first verdict**, because a second run is a different measurement. Too-slow answers were
  re-timed for information only. The VRAM-expert-cache engines use no CPU between requests (measured), so they don't
  slow the tests' code.
- **Check the checker.** Three checker bugs were found and fixed along the way:
  - An import ban applied to every coding task instead of only where the prompt rules a module out.
  - The code-block extraction kept an unclosed fence.
  - The knapsack check rejected a list inside a one-key object.
  - Re-scoring the saved answers changed none of the earlier verdicts. Every wrong answer was also checked by hand.
- **Treat one run as an example.** Single tasks flip between runs, as the two routes did for Flash-Next's two quants at
  24K. Differences of one task are within the noise.

## Raw data

`../results/hard-test-tasks.csv` has one row per run and task, holding the last row per task as the scorer reads it:

| Column | What it holds |
|---|---|
| `verdict` | `ok`, `ok*` (print-format slip), `x`, `len`, `loop`, or `n/10` for the repair job |
| `strict_point`, `lenient_point` | The points scored on each scale |
| `finish` | The engine's finish reason, or `loop` |
| `tokens` | Tokens written |
| `seconds` | Wall time, including prompt reading |
| `rescored` | Whether the row came from a later checker fix |

Answer texts and error output aren't included. Run labels:

| Label | Run |
|---|---|
| `flash-next`, `flash-next-q4` | Flash-Next IQ3_S and UD-Q4_K_XL, 24K round |
| `flash-next-max`, `flash-next-q4-max` | The 11 tasks Strata's cap cut short, re-run at 262K |
| `glm` | GLM-5.3-Flash S24 at `high`, 24K round |
| `glm-max`, `glm-64k` | GLM coding re-runs: 3 tasks at 128K with no thinking cap, 2 at 64K |
| `nanbeige` | Nanbeige, 24K round |
| `nanbeige-max` | Nanbeige's 3 coding re-runs at 96K |
| `nanbeige-best` | Nanbeige at its best (in progress) |
| `nanbeige-best-llamacpp-defaults` | The first 4 tasks of that run with llama.cpp's default top-k 40 / min-p 0.05, stopped and restarted with them off |
| `mimo` | MiMo, 24K round |
| `mimo-max` | MiMo's last 3 tasks at 128K |
| `deepseek-max` | DeepSeek Coder at max settings |
| `ds-coder-60k` | Its coding at 60K |
| `deepseek-max-token-only` | A 2-task side check |
| `qwen27b-max` | Qwen3.8-27B at its best |
| `gemma26b-q8-max`, `gemma31b-max` | Gemma 4 at its best |
| `gemma31b-max-code-t06` | The 31B's coding at temperature 0.6 |
| `nemotron-q3-max`, `inkling-max`, `ling-max` | Those models at their best |

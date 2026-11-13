# Failure modes

Every point each model lost in the hard test at its best, sorted by how it was lost. The sorting comes from reading
the saved answers, not just the verdicts in [hard-test.md](hard-test.md). Each task ran once per model, and single
tasks flip between runs: Flash-Next IQ3_S's 262K re-runs gained two tasks over its 24K run and lost Sudoku, which the
24K run had passed. The mix of miss types says more about a model than which tasks it missed.

## Miss types

| Type | What it covers | Verdicts |
|---|---|---|
| Format | Right content in the wrong shape: `''` printed for an empty string, a quoted string, the line outside its code block, the list inside an object with extra keys, `</code>` instead of the closing fence | `ok*`, `x` |
| Slip | Execution errors in otherwise sound work: a value off by one, a dropped digit, a 6-letter word in a 7-letter list, a 10-word sentence, binary strings drifting | `x` |
| Wrong | The method failed: a mis-traced program, estimating instead of multiplying | `x` |
| Bug | Code that crashes or answers wrong on the hidden tests, including code stricter than `json.loads` | `x` |
| Slow | Code over the task's time limit | `x` |
| Loop | Degenerate repetition, ended by `--loop-guard`, by Strata's repeat guard (256 repeats of a token, `finish=length` a few K tokens in) or by the cap | `loop`, `len` |
| Out of room | Still working, not repeating, when the budget or the context ran out | `len` |
| Cut off | The answer stops mid-code on its own | `x` |
| Stall | Repair job: whole turns of thinking without a tool call | `n/10` |

## Counts at their best

"On misses" is the share of the run's output tokens spent on answers that scored nothing.

| Model | Score | Format | Slip | Wrong | Bug | Slow | Loop | Out of room | Other | On misses |
|---|---|---|---|---|---|---|---|---|---|---|
| Qwen3.8-Flash-Next IQ3_S | 22 | | | | | 1 | 2 | | | 17% |
| Inkling 2 Small | 22 | | | | 2 | 1 | | | | 17% |
| Inkling-Small | 21 | | | | 2 | 2 | | | | 28% |
| Ling-3.1-flash | 21 | | | | 2 | | | 2 (128K context) | | 42% |
| Qwen3.8-Flash-Next UD-Q4_K_XL | 21 | | | | 2 | 1 | 1 | | | 32% |
| Qwen3.8-27B | 20 | | 1 | | 2 | | 2 | | | 26% |
| Ripple-65B Preview | 20 | | 1 | | 1 | 1 | 1 | 1 (192K context) | | 42% |
| DeepSeek-V4.1-Flash Coder | 19 | | 1 | | 2 | 1 | 1 | | 1 cut off | 52% |
| Gemma 4 26B-A4B | 18 | | 2 | | 4 | | 1 | | | 38% |
| Nanbeige4.3-33B | 18 | 2 | 1 | | 3 | | 1 | | | 36% |
| Gemma 4 31B | 17 | 1 | | 1 | 5 | 1 | | | | 35% |
| Nemotron 3.5 Super UD-Q3_K_XL | 17 | 1 | | | 5 | 2 | | | | 40% |
| MiMo-V2.6-Flash-MOPD | 16 | 1 | 2 | 1 | 3 | | 1 | 1 | | 42% |
| MiMo-V2.6-Flash-RL | 16 | | 1 | | 4 | 1 | | 3 | | 70% |
| GLM-5.3-Flash, `max`, 131K | 13 | | | | 1 | | | 10 | 1 stall | 90% |

## By model

### Qwen3.8-Flash-Next IQ3_S

It got every non-coding task, plus regex (99.5K tokens) and SAT (86.5K). Only it and Qwen3.8-27B solved SAT. The 17%
of its tokens spent on misses is the lowest share of any run.
- **Loop, Sudoku.** It recalled a "hardest puzzle" as a digit string and kept writing zeros. Strata's repeat guard ended
  it at 2.2K tokens, so it's scored `len`. The 24K run had solved Sudoku.
- **Loop, JSON.** Listing number edge cases, it wrote `1E+1000…` until the same guard fired at 16K, in the 24K round.
  It wasn't re-run with room.
- **Slow, Scheme.** It timed out on the hidden tests after 102K tokens.

### Inkling 2 Small

It got every non-coding task, plus JSON and SAT: the only run to pass both. All three misses are code, and the 17%
of its tokens spent on them ties Flash-Next IQ3_S for the lowest. Its coding answers are short (median 21.6K tokens),
and four still passed.
- **Bug, regex.** Lazy quantifiers. Its thinking weighed `a*?` and ruled it out ("not required"), reading the
  prompt's "the operators are greedy" as excluding them, so the parser rejects a second quantifier and `match`
  returns False. 3 of 400 cases; with a `?` after a quantifier accepted and ignored, it passes all 400.
- **Bug, Scheme.** `cond`'s tail call: a matching clause sets the expression to its last form and says `continue`,
  but that `continue` is inside the loop over the clauses, so evaluation moves on to the next clause. A `cond` that
  runs out of clauses returns nothing, displayed `()`. Its plan in the thinking had the same `expr = body[-1];
  continue`. Fixing that one line passes the whole test.
- **Slow, Sudoku.** After 6.3K tokens: bitmasks, naked and hidden singles, fewest candidates first. It solves 7 of
  the 8 puzzles in 0.1 s together, but the sixth (`.....6....59.....82....8....45...`, the worst case in Norvig's
  sudoku essay for this same method) ran past 20 s alone, and the 60 s run limit ended it.
- **Repair.** 10/10 in 3.9 min, 8 turns, 17 tool calls, and it called `finish`, which Inkling-Small never did.

### Qwen3.8-Flash-Next UD-Q4_K_XL

It also got every non-coding task, and one coding task fewer. Its 262K re-runs gained four tasks and lost Sudoku and SAT,
which its 24K run had solved, so it reads as even with IQ3_S.
- **Bugs.** Scheme raises its own `not a procedure`. SAT calls a satisfiable formula unsatisfiable.
- **Slow, Sudoku.** It timed out.
- **Loop, JSON.** `1e1000…` zeros, stopped by Strata at 4.6K tokens in the 24K round.

### Inkling-Small

It got every non-coding task, and its Scheme interpreter was the first to pass. Every miss is code, and two are speed.
- **Slow.** Sudoku took about 20 s against a 10 s limit, and SAT went past 90 s. Both reproduce on a re-time.
- **Bug, regex.** It calls its own helper with one argument too many (`TypeError`).
- **Bug, JSON.** It rejects `1e400`. The prompt's no-`Infinity` rule covers the literals; `json.loads` turns an
  overflowing number into `inf`.
- **Repair.** 10/10, but it never called `finish` and spent all 24 turns, one tool call each.

### Ling-3.1-flash

It got every non-coding task and was the only run to pass both JSON and Scheme. Its weak point is length: it was the
slowest run (744 min), spent 42% of its tokens on misses, and left two coding answers still thinking at the 128K
context.
- **Out of room (context), diff and Sudoku.** Each ran about 131K tokens and 105 min. The diff was still restructuring its
  plan. Sudoku had a finished exact-cover solver and was auditing it line by line ("let's think about if there is any
  issue with…").
  - It hasn't been run at 256K. That costs ~1.4 GB more VRAM, at 10.4 KB of MLA cache a token.
- **Bug, SAT.** At 126K tokens it called a satisfiable formula unsatisfiable.
- **Bug, regex.** Its parser rejects stacked quantifiers such as `+?` with its own error ("unexpected character").

### Qwen3.8-27B

With Flash-Next IQ3_S, it's the strongest coder here: it solved ordered, diff, Sudoku and SAT at 60-109K tokens each. It
was also one of two runs to pass Sudoku. Two of its five misses are loops. llama.cpp ran both until the test's guard
stopped them.
- **Loop, word list.** `"mal?"`, thousands of times; the last 20,000 characters compress 179x.
- **Loop, JSON.** A run of zeros, stopped at 41K tokens.
- **Slip, trace-3.** Two of its five values were wrong.
- **Bugs.** Regex got 5 cases wrong, among them plain `[a-c]+d?` on `abcd`. Scheme fails a plain `cond`.

### Ripple-65B Preview

A dense 65B that decides for itself when to think. It solved regex (21K tokens) and JSON (33K) in the same run, a pair
only Nanbeige also managed (87K and 46K), plus ordered and diff. Most of the 42% spent on misses is one answer: SAT's
196K tokens.
- **Out of room (context), SAT.** It thought for 90 min and 196K tokens, still weighing how to move watched literals
  ("Standard: swap with the watch that is `p`? Actually, ..."), and reached the 192K context with no answer. The
  context was set by VRAM: the model, its drafter and 192K of q8_0 cache fill both cards.
- **Loop, word list.** `"hajj?":`, thousands of times, ended by `--loop-guard` at 21K tokens. Qwen3.8-27B (`"mal?"`)
  and Nanbeige looped on the same task the same way.
- **Bug, Scheme.** Its subtraction builtin gets something that isn't a number (`TypeError` in `res -= a`): the evaluator
  hands an unevaluated form down.
- **Slow, Sudoku.** A bitmask solver with a popcount table, over the 10 s limit.
- **Slip, trace-2.** `6 [93, 81, 91] []` against `6 [81, 91, 11] []`: the count right, the list one step off.
- **Repair.** 10/10 in 11 turns and 25 tool calls, and it called `finish`.

### DeepSeek-V4.1-Flash Coder

It's patient with long problems: tour-9 took 120K tokens and the diff 186K, inside its 384K budget. Half its tokens (52%)
went to misses, mostly to one loop.
- **Loop, regex.** It tested its engine on `abab…` strings one character longer each paragraph, all the way to the 384K
  cap. This was before the loop guard was rebuilt: the first guard looked for exact repeats, and each paragraph carried
  a new number. It's scored `len`.
- **Cut off, Scheme.** The answer stops inside a function, mid-comment, at 20K tokens (`IndentationError`).
- **Bug, JSON.** It never consumes a closing bracket, so it rejects every array and object, `[]` included.
- **Bug, SAT.** An `UnboundLocalError`: a counter assigned in a nested function without `nonlocal`.
- **Slow, Sudoku.** 32.7 s.
- **Slip, trace-2.** It printed 23 where 6 was right.

### Gemma 4 26B-A4B

It's fast and cheap: 22 minutes for the whole test, and it was one of two runs to pass Sudoku. Its coding answers are
short, with a median of 19K tokens, and they fail on plain bugs. Its non-coding misses are slips in long output.
- **Slips.**
  - The 20-digit product is right except for one dropped digit: 39 digits instead of 40.
  - The word list includes `jacket`, 6 letters, among 7-letter words. Every other rule was met.
- **Loop, regex.** It was writing a test pattern, `*a*a*a…`, until the guard stopped it at 31K tokens.
- **Bugs.**
  - Scheme looks `define` up as a variable (`KeyError`).
  - The diff's edit script doesn't rebuild its inputs.
  - JSON decodes `"é"` wrong.
  - SAT calls a satisfiable formula unsatisfiable.

### Nanbeige4.3-33B

With room, it solved regex, JSON and diff, three of the hardest tasks. It needs that room: it scored 9.8 at a 24K budget
against 18 at 96K. Its median non-coding answer, 19K tokens, is the longest of any run. Its misses come from every
direction.
- **Loop, word list.** It cycles through the same candidates ("consider? no. contain? no. …"). At 24K it ran out of budget
  doing the same.
- **Slip, 200-row table.** 35 rows are wrong: rows 38-47, and every row from 99 to 127. Every error is in the binary
  string; the other fields are right. Its 24K-round table had none wrong.
- **Format.**
  - Trace-2 prints `''` for the empty string. With llama.cpp's default top-k 40 and min-p 0.05, the slip moved to
    trace-1.
  - SAT ends its code with `</code>` instead of the closing fence. Without that line, it still times out.
- **Bugs.**
  - Scheme raises a `TypeError` walking past its global environment.
  - Ordered gets `range_count` wrong.
  - Sudoku raises an `IndexError` building its exact-cover masks.
- **Repair.** 10/10 in 6 turns, but it took 65K tokens (the most of any passing run) and 15 minutes.
- **Near miss.** The 20-digit product took 97K tokens, just inside the 98,304-token context.

### Gemma 4 31B

It's the cheapest run: 201K tokens in 18 minutes, with both routes and the knapsack at 10-14K tokens each. Coding is its
weak point: 1 of 7, at a median of 7.5K tokens per coding answer. A coding re-run at temperature 0.6 scored 2 of 7: that
is noise, not the sampling.
- **Wrong, 20-digit product.** It switched to estimates and ended on a 40-digit number padded with zeros. Its own answer
  marks it "(This is wrong)".
- **Format, trace-2.** It prints `''` for the empty string.
- **Bugs.**
  - Regex fails on stacked quantifiers.
  - Scheme hits an unbound variable.
  - The diff raises an `IndexError`.
  - SAT calls a satisfiable formula unsatisfiable.
  - JSON rejects a lone surrogate (`"\ud83d"`), which `json.loads` accepts.
- **Slow, Sudoku.** 13.9 s.

### Nemotron 3.5 Super UD-Q3_K_XL

Every non-coding answer was right in substance, at a median of 7.6K tokens, and it finished the repair job in 7 turns.
Coding scored 0 of 7: every answer ended on its own at 9-48K tokens with a bug or a slow algorithm. One Q3 run can't
separate the model from the quant, though Q3's top-1 token matches Q4's 97.2% of the time.
- **Format, trace-3.** The right line, but outside any code block.
- **Bugs.**
  - Regex crashes at the end of the pattern (`None in '*+?'`).
  - Scheme fails `(display (+ 1 2))`.
  - Ordered's `select` computes an index of -2 and raises.
  - The diff's edit script doesn't rebuild its inputs.
  - JSON rejects a lone surrogate.
- **Slow.** Sudoku took 17.7 s, and SAT timed out.

### MiMo-V2.6-Flash-MOPD

It solved three coding tasks (regex, ordered, diff) at a median of 34K tokens per coding answer. That's the leanest of
the runs that solved three or more. Five of its nine misses are non-coding tasks that most models pass, and four of
those came from checking too little. Compared with RL, it wrote 19% fewer tokens and finished almost 2 hours sooner, at the same decode speed.
- **Slips.**
  - One acrostic sentence has 10 words where each needed 9, after only 845 tokens.
  - In the 200-row table, 199 is marked not prime.
- **Format, trace-1.** It prints `'qb'` with the quotes.
- **Wrong, trace-3.** Every value is wrong.
- **Out of room, tour-9.** It was enumerating cycle covers by hand at 131K. Its 256K re-run was cancelled.
- **Loop, JSON.** A run of zeros, stopped at 31K tokens.
- **Bugs.**
  - Scheme raises an `AttributeError` walking a body list.
  - Sudoku returns an invalid grid.
  - SAT calls a satisfiable formula unsatisfiable.

### MiMo-V2.6-Flash-RL (deleted after its run)

It got the 200-row table, the acrostic and trace-3, which MOPD missed. It lost points by thinking exhaustively: three
answers ran to 131K, and 70% of its tokens went to misses.
- **Out of room.**
  - Tour-9 and the knapsack: it was enumerating cycle covers and subsets by hand.
  - The diff: it was checking, example after example, that trimming a common prefix keeps the LCS.
- **Slip, trace-1.** It printed 89 for 88.
- **Bugs.**
  - Regex mishandles backreferences to a group set in an earlier iteration, and empty iterations.
  - Scheme gets `set!` wrong on a closed-over variable.
  - JSON doesn't combine a surrogate pair.
  - SAT raises an `IndexError` in conflict analysis.
- **Slow, Sudoku.** 18.8 s.
- **Repair.** 10/10, without getting stuck re-issuing tool calls, despite Xiaomi's warning that this checkpoint can.

### GLM-5.3-Flash Maya-S24

At `max` it rarely converges: 10 of its 25 answers ran to the 131K budget, and 90% of its tokens went to misses. At a
level where it finishes, it solves hard tasks:
- at `high` with 262K: both big products, regex (210K tokens) and the repair job (9/10 in 7 turns);
- at `low`: the knapsack (70K) and Scheme (14K).

Each task's best comes to 18.9. Its attempts per task are tabled in [hard-test.md](hard-test.md).

**At `max`, 131K:**
- **Out of room (10 tasks).** It ran out in three ways:
  - re-verifying partial products it had already confirmed (both big products);
  - enumerating by hand (tour-9, knapsack);
  - designing, then drafting, whole programs inside its thinking (6 coding tasks). Scheme's builtins were being written
    out when the cap hit.
- **Stall, repair job.** Two turns each ran to 131K without a tool call. It was stopped and scored as the untouched
  project, 0/10.
- **Bug, JSON.** This was its one coding answer that ended, and it calls a helper, `_hex4`, that it never defined.

**At `high`, 262K:**
- **Out of room.**
  - Tour-9: enumerating by hand.
  - Scheme: tracing its own trampoline.
  - Ordered: switching between designs ("FINAL ✓✓✓ … hmm, hold on").
- **Loop, Sudoku.** A cycle of the same phrases at 172K.
- **Format, knapsack.** It found the optimum (584) but returned it in an object with extra keys.
- **Bugs.** The diff's edit script doesn't rebuild its inputs, and SAT calls a satisfiable formula unsatisfiable.
- **Repair.** 9/10: one cost was rounded differently (4,286 for 4,300).

**At `low`:**
- Tour-9 ran to 262K again.
- Ordered crashed with an `AttributeError` after 2.1K tokens.

Planned `max` re-runs of the diff, Sudoku and SAT, which had failed at `high` with room left, were skipped.

## Across models

**How long coding answers run.** The runs that end their coding answers early submit bugs, and GLM at `max` never
ends them:

| Run | Coding passed | Median tokens per coding answer |
|---|---|---|
| Gemma 4 31B | 1 | 7.5K |
| Gemma 4 26B-A4B | 2 | 18.7K |
| Inkling 2 Small | 4 | 21.6K |
| Nemotron Q3 | 0 | 24.4K |
| MiMo MOPD | 3 | 34.0K |
| Inkling-Small | 3 | 50.3K |
| MiMo RL | 1 | 56.1K |
| DeepSeek Coder | 2 | 58.3K |
| Qwen3.8-27B | 4 | 63.6K |
| Flash-Next IQ3_S | 4 | 77.2K |
| Flash-Next Q4 | 3 | 79.6K |
| Nanbeige4.3-33B | 3 | 80.3K |
| Ling-3.1-flash | 3 | 85.0K |
| GLM, `max` | 0 | 131.1K, the cap |

Inkling 2 Small is the exception: four passes at a 21.6K median.

**Loops.** Nine answers looped at their best, plus GLM's Sudoku at `high`.
- **Runs of zeros (five).** Each came while the model pictured a huge number or a puzzle string: the JSON parser for
  both Flash-Next quants, Qwen3.8-27B and MOPD, and Flash-Next IQ3_S's Sudoku. The JSON prompt invites this by asking
  for exponents and integers of any size. Strata's repeat guard ended its three at 2-16K tokens; the test's guard ended
  the other two at 31K and 41K.
- **The rest.** A word list cycling (Qwen3.8-27B, Nanbeige), a test pattern growing (Gemma 26B), a counted test running to
  384K (DeepSeek), and GLM's phrase cycle.

**SAT.** Calling a satisfiable formula unsatisfiable was the most common single coding failure: Flash-Next Q4, Ling,
both Gemmas, MOPD and GLM at `high` did it. Only Flash-Next IQ3_S, Qwen3.8-27B and Inkling 2 passed.

**Sudoku.** Most failures were correct but slow. Five solvers took 14-33 s against a 10 s limit: Inkling, DeepSeek,
Gemma 31B, MiMo RL and Nemotron. Flash-Next Q4's timed out. Only Qwen3.8-27B and Gemma 26B passed.
Inkling 2's solved 7 of 8 in 0.1 s and stalled on the sixth, the classic worst case for propagation with
fewest-candidates search.

**JSON.** Only Ling, Nanbeige, Ripple-65B and Inkling 2 passed. Four parsers broke on `\u` escapes:
- Gemma 31B and Nemotron reject a lone surrogate, which `json.loads` accepts.
- MiMo RL doesn't combine a surrogate pair.
- Gemma 26B decodes `é` wrong.

Inkling rejected `1e400`.

**Regex.** The hidden cases include stacked quantifiers such as `+?` and `*?`, which Python reads as lazy. For a
whole-match yes/no, they act as `+` and `*`. Ling's parser rejects them, and both of Gemma 31B's wrong cases are of
this kind. Inkling 2's rejects them by decision: its thinking took the prompt's "greedy" to rule them out, and they
were its only 3 wrong cases of 400.

**Format slips.** These are rare, and mostly in the traces:
- `''` for an empty string (Gemma 31B, Nanbeige);
- a quoted string (MOPD);
- the line outside its code block (Nemotron).

Outside the traces there were two: GLM's knapsack wrapped in an object, and Nanbeige's `</code>`.

**Running out of room.** Only GLM, both MiMo checkpoints and Ling (at its 128K context) ran out of room. With the room
given here, no other run reached its cap except through a loop.

## Repair job

Every run at its best passed it 10/10 except GLM at `max`. They differ in how they got there:

| Run | Passed | Turns | Tool calls | Called `finish` | Tokens | Time |
|---|---|---|---|---|---|---|
| Flash-Next IQ3_S (24K round) | 10/10 | 22 | 29 | yes | 17.1K | 90 s |
| Inkling-Small | 10/10 | 24 | 24 | no | 12.4K | 408 s |
| Inkling 2 Small | 10/10 | 8 | 17 | yes | 7.8K | 234 s |
| Ling-3.1-flash | 10/10 | 6 | 20 | yes | 25.3K | 1,217 s |
| Flash-Next Q4 (24K round) | 10/10 | 14 | 19 | yes | 12.8K | 107 s |
| Qwen3.8-27B | 10/10 | 10 | 21 | yes | 15.1K | 131 s |
| DeepSeek Coder | 10/10 | 5 | 15 | yes | 10.8K | 259 s |
| Gemma 4 26B-A4B | 10/10 | 17 | 17 | yes | 8.4K | 28 s |
| Nanbeige4.3-33B | 10/10 | 6 | 15 | yes | 65.4K | 909 s |
| Gemma 4 31B | 10/10 | 15 | 15 | yes | 4.5K | 30 s |
| Nemotron Q3 | 10/10 | 7 | 17 | yes | 6.0K | 186 s |
| MiMo MOPD | 10/10 | 13 | 25 | yes | 12.5K | 603 s |
| MiMo RL | 10/10 | 10 | 30 | yes | 15.7K | 710 s |
| GLM, `max` | 0/10 | 2 | 0 | no | 262K | 6,982 s |
| GLM, `high` re-run | 9/10 | 7 | 17 | yes | 13.3K | 402 s |

- **Calls per turn.** The Gemmas and Inkling-Small make one tool call per turn, Flash-Next about 1.3, and the
  rest 2-3.3 (Inkling 2: 2.1).
- **Inkling-Small** never calls `finish`; Inkling 2 does.
- **Nanbeige** thinks the most per turn.

## Under a 24K budget

The 24K round (2026-11-02), for the five models in it:

| Model | Out of budget | Other misses |
|---|---|---|
| Flash-Next IQ3_S | JSON (a loop Strata stopped) | Tour-9 not a tour; regex 19 cases wrong; Scheme with its own bug |
| Flash-Next Q4 | The 200-row table; JSON (a loop) | Tour-7 at cost 341 against 280; regex 2 cases wrong; Scheme with an unterminated string; the diff doesn't rebuild its inputs |
| GLM `high` | 11: both big products, tour-7, the knapsack, all 7 coding | Trace-2; the 6x4 grid; tour-9 at 5.5K tokens (276 against 241); the word list at 2.6K |
| Nanbeige4.3-33B | 12: both big products, both routes, the word list (looping), all 7 coding | The lipogram at 3K tokens; two print slips; repair 8/10 in 25 s |
| MiMo RL | 10: both big products, both routes, the knapsack, the word list, 4 coding | Sudoku too slow; at 128K, its SAT solver returned an assignment that breaks a clause; repair 9/10 |

- **Strata makes Flash-Next answer.** It closes the thinking at 18K, 3/4 of `max_tokens`, so Flash-Next answers where
  the others run out. Only the Q4's 200-row table, written after the thinking, ran past 24K. Its budget misses become
  wrong answers instead: the invalid and overpriced tours, and coding bugs. The 262K re-runs of the tasks the cap cut
  gained 2 and lost 1 (IQ3_S), and gained 4 and lost 2 (Q4).
- **GLM, Nanbeige and MiMo run out instead.** GLM and Nanbeige ran out on every coding task. GLM also committed early
  and wrongly on tour-9 and the word list.

# Measurement

All numbers go through each engine's own OpenAI-compatible server with streamed requests. Decode is generated tokens
over decode time, prefill is prompt tokens over prompt time, both from the server's timings where it reports them.

## Throughput: `ab.py`

`ab.py` runs greedy (temperature 0, seed 7) with thinking off, after one untimed warm-up request.

| Measure | Workload |
|---|---|
| `fresh` | Six unseen prompts, once each: code in Rust, TypeScript and Python, two explanations, an essay. Total tokens over total decode time. This is the "new prompts" figure |
| `deep` | An unseen ~32-35K-token prompt (source code), read and then answered: prefill, TTFT and decode after a long prompt |
| `repeat` | One prompt three times. The first-to-third gain shows how far an expert cache adapts to a workload it has seen |

- Every A/B compares 3+ runs per setup, with a restart between setups.
- Engines that persist expert scores (Strata, maya-ink, maya-nem, maya-ling) start each run from the same saved profile,
  so one run can't warm the cache for the next.

## Thinking-on decode: `spec-sweep.py`

`spec-sweep.py` asks four unseen prompts at the model's own sampling, seed 7, 1,500 tokens each, with thinking on.
Drafting gains (MTP, DFlash2, n-gram) shrink at temperature 1, so `ab.py`'s greedy numbers overstate them. The rows
marked † in the README come from this script.

## Correctness

- **Against llama.cpp:** for maya-ink, maya-nem and maya-ling, the same token ids went to both, with greedy top-1
  compared at every position, through the prompt path and token by token. Log-probability gaps for llama.cpp's tokens
  were checked as well. Template text and tokenizer ids match llama.cpp exactly.
- **Chat checks:** separate `reasoning_content`, thinking off, a tool call and the answer from its result, streaming, and
  JSON.
- **`derail.py`:** 20 rounds of a tool conversation followed by a fresh short chat. A wrong word or a `length` finish on
  the short chat counts as derailed. This catches stale-state bugs that short isolated chats and KL checks miss: it
  found a strata-mimo bug that broke about 1 new chat in 10 after a long conversation, which a good KL score and 120
  clean short chats had both passed. It's run after any engine or setting change that alters outputs.

## Pitfalls that produced wrong numbers here

- **Repeated prompts flatter an expert cache.** Flash-Next on Q4 decoded 80 tok/s on new prompts and 118 on repeats.
- **The first baseline after other work runs on a cold file cache.** One read 11.7 tok/s, then 14.9 on the next run.
- **Without a unique prefix**, such as a timestamp, the prompt cache skips the prefill you meant to measure. Check
  `prompt_n`.
- **Size long prompts from `usage.prompt_tokens`, not characters.** Tokenizers here range from about 3.5 to 3.8
  characters a token.
- **Time with elapsed time, not wall clocks.** A DST change moved the clock back an hour during one run.
- **A low `max_tokens` cap isn't a failure to solve.** One cap made GLM look as if it couldn't do a coding task it could.
- **Whole-task benchmarks such as generating a game vary too much** for single runs to rank anything.
- **Check the PCIe link width before comparing.** After a card was reseated, one linked at x1 and decode halved.
- **Don't compile while a CPU-lane engine is being timed.** The lane is the bottleneck and the build takes its cores.
- **A 160 GB+ model needs a second run after other models ran.** MiMo's first `ab.py` pass read 19.6 tok/s, the second
  24.5; compare second runs.
- **Environment settings set by a test script leak into the next run in the same shell.** A "background swaps off"
  switch stayed on for two later runs and made two settings look identical.
- **Extra CUDA devices visible to a process can change timings on the first one.** maya-ling's prefetch copier ran 38%
  slower with a second card merely visible. For a one-card baseline set `CUDA_VISIBLE_DEVICES`.

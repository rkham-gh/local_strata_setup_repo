# How the numbers were measured

Every number here was measured through each engine's own OpenAI-compatible server, with streamed requests, the way a
client would use it. Writing speed is tokens generated divided by decode time, from the server's own timings where it
reports them. Prompt reading is prompt tokens divided by prompt time.

## Speed: `ab.py`

`ab.py` compares server setups on prompts the engine hasn't seen. It runs greedy (temperature 0, seed 7) with thinking
off, after one short untimed warm-up request. It reports three numbers:

| Measure | What it does | Why |
|---|---|---|
| `fresh` | Six different prompts, asked once each: Rust, TypeScript and Python code, two explanations and an essay. Averaged as total tokens over total decode time | What a new chat gets. **This is the number quoted as "new prompts"** |
| `deep` | A new long prompt (about 32-35K tokens of source code, depending on the tokenizer) is read, then answered | Prompt-reading speed, time to the first token, and writing speed after a long prompt |
| `repeat` | One prompt asked three times; the first and third speeds are reported | Shows how far an engine gains by adapting to a workload it has seen |

Each comparison is at least 3 runs per setup. Engines that keep a VRAM expert cache (Strata's Flash-Next, maya-ink,
maya-nem, maya-ling) adapt that cache to what they see and save their expert scores, so each run starts from the same
saved expert profile. That way, one run can't teach the next.

## Speed with thinking on: `spec-sweep.py`

Greedy speed isn't what a thinking model at its own sampling gets, above all on engines that draft tokens (MTP, DFlash2,
n-gram drafting), because fewer drafts are accepted at temperature 1. `spec-sweep.py` asks four new prompts with thinking
on, at the model's own sampling and seed 7, 1,500 tokens each, and reports tokens over decode time.

## Correctness

- **Against llama.cpp.** For the engines written for this PC (maya-ink, maya-nem, maya-ling), the same token ids were fed
  to llama.cpp and the engine, and the top token was compared at every greedy position, both through the prompt path and
  token by token. Log-probability gaps for llama.cpp's tokens were checked too, and the chat template's text and the
  tokenizer's ids were compared exactly.
- **Chat checks.** Thinking kept separate, thinking off, a tool call and the answer from its result, streaming, and JSON
  output.
- **`derail.py`.** Twenty rounds of a tool conversation followed by a new short chat. A wrong word, or a `length` finish on
  the short chat, counts as derailed. This sequence once made one engine write gibberish in about 1 new chat in 10,
  while a good KL score and 120 clean short chats both missed it. It's run after any change to an engine or setting that
  changes answers.

## Rules, each from a wrong result

- **Use prompts the engine hasn't seen, and restart between setups.** Repeating a prompt lets the VRAM expert cache
  adapt: Flash-Next on Q4 read 80 tok/s on new prompts and 118 on repeats, and only the first is what a new chat gets.
- **Run a baseline twice after other work.** On a cold file cache, the first run read 11.7 tok/s where the second read
  14.9.
- **Don't time the first request on a new prompt.** The scripts do a warm-up first.
- **Start each test prompt with something unique**, such as a timestamp. Otherwise the prompt cache skips the reading
  you meant to measure. `prompt_n` in the server's timings shows how many tokens were actually read.
- **Size long prompts from `usage.prompt_tokens`**, not from characters. A token is about 3.5 characters on one tokenizer
  and 3.8 on another.
- **Run nothing else on the GPU during speed tests.**
- **Use elapsed time, not clock times.** Daylight saving moved the clock back an hour during one run.
- **Use at least 3 runs before calling a setup better.** Single results, especially on whole tasks like generating a
  game, varied too much to rank anything.
- **Don't score an answer cut off by a low cap as a model failure** (see `hard-test.md`).
- **Check answer keys by hand, and strip formatting before comparing.** A wrong key and a LaTeX-formatted answer both
  gave false results once.

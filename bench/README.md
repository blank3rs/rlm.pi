# bench/ — RLM engine benchmarks

These benchmarks run the real engine (`pi-plugin/rlm/src/core/engine.ts`) with the real
Python sandbox and real provider calls.

## Panel

The models under test are in [`panel.json`](panel.json). It is the only source of the model
list: `run.ts` and `hero.py` both read it.

| Model | Key |
|---|---|
| `zai/glm-5.3` | `ZAI_API_KEY` (Z.ai coding-plan endpoint) |
| `b-ai/qwen3.8-27b` | `BAI_API_KEY` (`https://api.b.ai/v1`) |
| `openrouter/openai/gpt-oss-120b` | `OPENROUTER_API_KEY` |

To change the panel, edit `panel.json`. Do not add model names to the code.

## Suite

There is one suite: `oolong` (`oolongbench/oolong-synth`). Each task is an aggregation
question over synthetic records. The grader gives a score from 0 to 1. `correct` is true
only for a full score. The datasets download from HuggingFace on first use into
`bench/data/` (gitignored).

## Run

Keys go in env vars only. Do not write keys in files.

```bash
export ZAI_API_KEY=... BAI_API_KEY=... OPENROUTER_API_KEY=...
bun run bench --oolong-max-cl 65536 --oolong-limit 24   # canonical run, all panel models
bun run bench --model zai/glm-5.3 --oolong-max-cl 65536 --oolong-limit 24   # one model
bun run bench --limit 1                                  # smoke test: first task only
bun run bench --list                                     # show tasks, no model calls
bun run bench --model zai/glm-5.3 --journal <file> --resume   # continue a stopped run
bun run bench --task-delay-ms 60000                      # pause between tasks (rate limits)
bun run bench --runs 3                                   # pooled repeats
```

`--resume` skips tasks that are already in the journal. It removes rows that ended with an
upstream `error`, and runs those tasks again.

Optional env var: `RLM_BENCH_CONTEXT_WINDOW` (default `1000000`).

## Results

- `bench/runs/<model>/<timestamp>.jsonl` — one row per task: `suite, taskId, model, run,
  correct, recall, answer, gold, error, latencyMs, inputTokens, outputTokens, iterations,
  costUsd`.
- `<journal>.config.json` — the run settings (model, git rev, limits, temperature, pricing).
- `bun run bench/report.ts <journal>...` — summary table and per-task matrix.
- `python3 bench/hero.py` — chart of the newest full journal (≥ 6 rows) per panel model.
  Needs `matplotlib`.

Commit full journals. Do not commit smoke tests or journals of models outside the panel.

## Recall eval (no API key)

`bun run bench/recall-eval.ts` replays the native context transform over scripted
transcripts. It is offline and deterministic. It checks that the elision window stays
bounded and that needle facts stay reachable through `ctx/session-log/*`.

## Engine settings (`engine.ts`)

`makeRun()` builds the engine like the test fixtures in `pi-plugin/rlm/test/helpers.ts`,
but with a real provider. Main settings: `maxIterations 200`, `autoSeedCwd false`,
`enableTokenBudget false`, 5 retries and 30 rate-limit retries. On `zai`, concurrency is 1
and retry delays are longer.

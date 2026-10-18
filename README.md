<div align="center">

# rlm.pi

**Recursive Language Models for Pi.**

Use a cheap model to read very large contexts with high accuracy.

<p>
<a href="https://www.npmjs.com/package/@hicaru/pi-rlm"><img src="https://img.shields.io/npm/v/@hicaru/pi-rlm?color=cb3837&label=npm" alt="npm" /></a>
<a href="https://arxiv.org/abs/2512.24601"><img src="https://img.shields.io/badge/arXiv-2512.24601-b31b1b" alt="RLM paper" /></a>
<img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT" />
</p>

<img src="https://github.com/openzebra/rlm.pi/blob/master/assets/hero.png" width="92%" alt="rlm.pi — OOLONG benchmark results" />

<sub><b>English</b> · <a href="README.zh-CN.md">中文</a> · <a href="README.ru.md">Русский</a></sub>
</div>

## Install

| | Install | Upgrade | Remove |
|---|---|---|---|
| **Pi** | `pi install npm:@hicaru/pi-rlm` | add `--force` | `pi uninstall npm:@hicaru/pi-rlm` |
| **oh-my-pi** | `omp plugin install @hicaru/pi-rlm` | add `--force` | `omp plugin uninstall @hicaru/pi-rlm` |

1. Install the plugin.
2. Type `/reload`, or restart Pi.
3. Make sure that `/rlm`, `/rlm-config` and `/rlm-stop` show under **[Extensions]**.
4. Push `Ctrl+Shift+R` or type `/rlm`. Your prompts now go to the RLM engine.

## How it works

```
 you ──► prompt ──► RLM engine ──► main model (Python in repl(), one turn at a time)
                        │              ├─► search / grep / outline   (no tokens)
                        │              ├─► llm_query ──► cheap workers
                        │              └─► rlm_query ──► child RLMs (depth limit)
                        └──► state Σ + validation + limits (tokens, time, errors)
```

- The context stays in a Python sandbox. The model sees only an index.
- The main model writes Python code. The code finds the data that the model needs.
- Cheap worker models read and summarize. Your main model does not read the full text.
- A child RLM solves a difficult sub-task. It gets the full sandbox at no token cost.
- The engine keeps the run state `Σ` and validates each change to it.
- `/compact` makes a digest of the session. It does not use an LLM.

## Benchmarks

The benchmark is **OOLONG** (oolong-synth): 24 tasks, context up to 65,536 tokens, temperature 0.

| Model | OOLONG | Tokens/task | Cost/task |
|---|---|---|---|
| `zai/glm-5.3` | **95.8%** (23/24) | **~28k** | ~$0.00 \* |
| `b-ai/qwen3.8-27b` | **91.7%** (22/24) | ~37k | ~$0.00 \* |
| `openrouter/google/gemma-4-31b-it:free` | pending | — | — |

The model list is in [`bench/panel.json`](bench/panel.json). Each model has its own
results folder: `bench/runs/<model>/`. To run all models:

```bash
export ZAI_API_KEY=... BAI_API_KEY=... OPENROUTER_API_KEY=...
bun run bench --oolong-max-cl 65536 --oolong-limit 24
```

For more data, see [`bench/README.md`](bench/README.md).

## Commands

| Command | Shortcut | Function |
|---|---|---|
| `/rlm` | `Ctrl+Shift+R` | Set RLM mode on or off |
| `/rlm-stop` | | Stop the current run |
| `/rlm-config` | | Set sampling, budgets and limits |

## Sandbox API

The model can use only these retrieval and delegation functions:

```python
search("needle", k=10)        # BM25 hits in the context
grep_context(pattern, k=50)   # regex hits with line numbers
outline(path)                 # definitions in a file
add_context("report.pdf")     # add a PDF, DOCX, XLSX, EPUB, CSV or HTML file
llm_query(prompt)             # send text to a cheap worker
map_files(paths, prompt)      # ask one question about many files
rlm_query(task, paths=...)    # start a child RLM
answer["content"] = "..."     # write the final answer
answer["ready"] = True        # end the run
```

## Security

- The engine does not write to the disk. The answer is the only output of a run.
- API keys do not go into the sandbox. The host sends all sub-LLM calls.
- Python runs in a guarded subprocess with a list of reserved names.

## License

[MIT](pi-plugin/rlm/LICENSE) © hicaru contributors

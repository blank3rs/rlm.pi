# rlm-pi — Project Instructions

## Rule zero: less code

**The smaller the code, the better.** Every line is a liability: it must be read, typed,
tested and kept in sync. Before writing anything, ask "can this be deleted or reused instead?"

- **Delete before you add.** A fix that removes lines beats one that adds them. Dead code,
  stale comments, unused exports and one-caller helpers go.
- **No speculative code.** No knobs, aliases, env-var twins of CLI flags, fallbacks or
  options nobody asked for. One way to do each thing.
- **No abstraction for a single use.** Inline it. Extract only on the second real caller.
- **No duplication.** If the logic exists, call it; if it almost exists, extend its seam.
- **No error handling for impossible states.** Narrow types so the state can't exist.
- **Comments say *why*, briefly.** No changelog prose, no amendment history, no restating
  the code. History lives in git and `CHANGELOG.md`.
- **Surgical diffs.** Touch only what the task needs; every changed line traces to it.
- **Model-visible surface stays tiny**: two Pi tools (`repl`, `rlm`) and a REPL namespace of
  retrieval + delegation only. No `todo` / `save_artifact` / `advance_phase` — task tracking
  belongs to Pi. A new sandbox function needs proven profit.

Test: *would a senior engineer call this overcomplicated?* If yes, cut it.

## Commands

- Runtime: **bun** only — `bun install`, `bun run index.ts`, `bun run test` (never npm/pnpm/yarn).
- Typecheck: `bunx tsc --noEmit` (root and `pi-plugin/rlm/`); lint: `bun run lint`.
- Never commit secrets, API keys or session files. Bench keys travel via env vars only.

## Architecture

A **Recursive Language Model (RLM) plugin** for the Pi coding agent. The engine drives a
model turn-by-turn over Python `repl()` blocks run in a persistent subprocess sandbox
(`sandbox/py/worker.py`). Sub-LLM calls (`llm_query`, `rlm_query`) are served in-process by
bridges that hold the API keys — the sandbox never sees them. The engine does **no disk
I/O**: a run lives in memory and its answer is its only output.

```
pi-plugin/rlm/src/
├── core/      headless RLM loop, limits, compaction, Σ state
├── bridge/    sub-LLM/rlm handlers (bridge/handlers/ is the single impl)
├── sandbox/   Python worker (py/), JSONL protocol, interrupts, manager
├── tool/      repl() / rlm() Pi tools + event emitter
├── config/    rlm.json, defaults, model resolution
├── prompts/   glossary (single wording source) → system + native prompts
├── context/   native walker, document conversion, add_context
├── ui/        config panel, picker, status line, stage cards, tree
├── text/      REPL block parsing, token estimates, previews
├── mode/      RlmController, native-mode guards
├── util/      Result, errors, concurrency, retry/throttle, merge, bm25
├── commands/  /rlm, /rlm-stop, /rlm-config
└── index.ts   extension entry (`rlmExtension()`)
```

Entry points: root `index.ts` (harness), `src/index.ts`, `core/engine.ts` `createEngine()`,
`sandbox/py/worker.py` (siblings `guards.py` / `retrieval.py` / `tasks.py` load via
`sys.path[0]`). Key types: `core/types.ts`.

## Single seams — one implementation each

| Concern | The one place |
|---|---|
| Model completion | `bridge/model.ts` — only caller of pi-ai `completeSimple` |
| Leaf completion (`complete1`, grounding via `groundLeaf?`) | `bridge/handlers/` `createSubcallHandlers` |
| Recursion (depth cap → guard → spawn → debit) | `childRun` in `bridge/handlers/` — the only builder of a child `RlmInput` |
| Child context inheritance | `SubcallHandlerDeps.getChildContext` |
| Subcall emit pattern | `emitting()` (leaves); `childRun` emits its own node — never wrap it |
| Display model / leaf summary | `displayModel()`, `summarizeLeaf()` in `bridge/handlers/` |
| Concurrency | `util/concurrency.ts` — one session `Semaphore` for leaves, per-depth `DepthGates` for children (one shared gate deadlocks) |
| Retry / rate-limit cooldown | `util/retry.ts` + `util/throttle.ts` |
| State merge (Σ and skill notes) | `util/state-merge.ts` `deepMergeWithNullDeletion` |
| Wording (digest labels, recall line, elision stub) | `prompts/glossary.ts` |
| REPL block extraction | `text/parsing.ts` `findReplBlocks` |
| Config validation | `config/settings.ts` `validate*` (accept `unknown`) |
| Errors | `formatError` / `isErrorText` — never throw strings |
| BM25 | `sandbox/py/retrieval.py` and `util/bm25.ts` — same constants, two runtimes |

Callers differ only in `resolve`: the engine binds one `Invocation` per run; `repl()` swaps
one per turn and routes `spawn()`ed work to `BackgroundTasks`. Need a new variation? Add a
field to `SubcallHandlerDeps` — never a second construction path.

## TypeScript

- `strict`, `noUnusedLocals`, `noUnusedParameters` on in both tsconfigs.
- **No `any`** (use `unknown` + `is*` guards), **no `!`** (use `?.`, `??`, guards), no casts
  where a guard works.
- Optional props use `prop?: T`, never `prop: T | undefined`.
- Discriminated unions over boolean flags; `Result<T, E>` from `util/errors.ts` for fallible ops.
- `Object.freeze` constant arrays/sets/configs; `readonly` on DTOs and option bags.
- Pre-size arrays when the length is known; build big strings with `join`, not `+=` in loops.
- Fail soft: writers return `boolean` and warn; the turn loop never throws. Never
  `console.log` on a TUI path.
- No per-session state at module load or in module-level consts — it lives in the
  `rlmExtension` closure or enters via function arguments.

## Adding a sandbox function

1. Interrupt type in `sandbox/protocol.ts` (`WorkerInterrupt`).
2. `sandbox/interrupts.ts`: `SubLlmHandlers` entry, `REJECT` default, `serviceInterrupt` dispatch.
3. `py/worker.py` method + RPC; add the name to `RESERVED` (`guards.py`/`worker.py`) and `_restore_scaffold`.
4. Wire it in a bridge file reusing `emitting()`; add one glossary line.

## SKILL.state (Σ) invariants

Notation in comments: `P` spec, `Σ_t` state, `ΔΣ_t` patch, `⊕` merge (`null` = delete),
`V(ΔΣ_t, Σ_t)` validator, `Ξ` skill block, `κ_Σ` byte cap; cite paper sections (§3, §5.7).

- Patches arrive as `unknown`, are validated runtime-side and return `Result`; implicit key
  drops are `implicit-drop` errors, not silent keeps.
- Trackers are `active | degraded` unions; on retry exhaustion fall back to append-only +
  `compactHistory`. Degrade, never crash.
- Σ serializes with compact `JSON.stringify`; `RUN_STATE_LIMITS` is frozen and enforced
  before every persist.
- Persistence follows `config/settings.ts`: `join(getAgentDir(), name)`, fail-soft reader →
  frozen empty value, fail-soft boolean writer. Pins: `undefined` = from disk, `null` = clear.
- Paradigm flags (`enableRunState`, `enableSkillState`, `enableSkillStateDistill`,
  `enableRootContextTransform`, `enableRootStateFences`, `enableRootDigestCompaction`) are
  always on via `validateEnforcedOn`; an explicit `false` is traced and ignored.
  `RLM_BENCH_NO_ROOTCONTEXT=1` is a dev-only measurement hatch.

## Root Σ (native harness) invariants

- `core/root-state.ts` `RootStateTracker` is runtime-derived (tool outcomes,
  `EngineDeps.onRunState`, latest prompt); fences are the only model input and ride the same
  validator. Idle-degrade after `ROOT_IDLE_DEGRADE_TURNS` (6) fence-eligible turns with no
  accepted delta.
- `session_before_compact` (`core/root-digest.ts`): deterministic no-LLM digest; cut points
  only tighten Pi's boundary; every displaced message folds into the digest; never return
  `cancel: true`.
- `context` handlers mutate the `structuredClone` in place, return `{ messages }`, never
  touch the disk transcript, and return the identity on any throw. The Σ snapshot
  (`customType: "rlm-sigma"`) is immune to elision.
- Stage transitions post `rlm.stage` cards (`ui/stage-cards.ts`), collapsed by default; the
  digest card posts at the next `turn_start`, never re-entrantly mid-compaction. Identical
  sibling `llm` leaves group into one `×N` row; `rlm` nodes never group.

## Testing

- Tests live in `pi-plugin/rlm/test/`; `bun run test` (`test/smoke.ts`) runs every suite
  and boots a real sandbox (needs `python3`).
- Fix → reproduce it in a test first, then make it pass. Keep tests as small as the code.
- Benchmarks: the model panel lives only in `bench/panel.json`; journals go to
  `bench/runs/<model>/`. Never commit probes or off-panel journals.

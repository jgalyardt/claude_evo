# Evo

A self-evolving Elixir application that uses Claude's API to iteratively improve its own code. Runs on a Raspberry Pi 5 with Phoenix 1.8, SQLite3, and a real-time LiveView dashboard.

## How It Works

Evo runs a continuous evolution loop that proposes, validates, and applies code changes to itself:

1. **Benchmark** — Measure the current performance of a target module (execution time, memory, code size)
2. **Propose** — Send the module source + benchmarks to Claude, requesting one focused improvement
3. **Validate** — Run a 5-gate safety pipeline: size limits, AST allowlist, side-effect detection, compilation check, and test suite
4. **Apply** — Write the new code to disk and hot-swap the module in the running VM (no restart)
5. **Evaluate** — Re-benchmark and compare fitness (60% time, 30% memory, 10% code size). Rollback if regressed
6. **Record** — Persist the generation to SQLite and git commit the change

## The Self-Modifying Mechanism

The system uses the BEAM's native hot code loading to replace a live module while the application keeps running:

```elixir
:code.purge(module)      # unload the old bytecode
:code.delete(module)     # remove it from the code server
Code.compile_file(path)  # compile the new .ex file and load it
```

The sequence in `Evo.Applier` is:

1. Claude returns a proposed rewrite of a module
2. `Applier` writes the new source to disk (whitelisted paths only)
3. The old module is purged from the BEAM's code server
4. The new source is compiled in-process and loaded — immediately active for all subsequent calls
5. If post-change benchmarks show a regression, `Applier.rollback/1` restores the original source and repeats the same hot-swap with the old code

Because Erlang/OTP supports running two versions of a module simultaneously during a transition, in-flight calls to the old version complete normally while new calls dispatch to the replacement.

## The Evolvable Surface

Only four modules are allowed to be modified — the code that controls how the system evolves:

| Module | Role |
|---|---|
| `Evo.Evolvable.PromptBuilder` | Constructs the prompt sent to Claude |
| `Evo.Evolvable.Fitness` | Evaluates before/after benchmark fitness |
| `Evo.Evolvable.Strategy` | Selects which module to evolve next (round-robin) |
| `Evo.Evolvable.CreativeDisplay` | Generates the animated SVG visualization on the dashboard |

This is the recursive part: by targeting `PromptBuilder`, the system can rewrite the instructions it gives Claude; by targeting `Fitness`, it can redefine what counts as an improvement; by targeting `Strategy`, it can change which module it works on next.

`Evo.Applier` enforces a hardcoded module→path whitelist and `Evo.Validator` enforces an allowlist-based AST walk — no other modules or paths can be written to regardless of what Claude proposes.

## Architecture

```
Evo.Evolver          GenServer orchestrating the evolution loop
Evo.Benchmarker      Microbenchmark runner (:timer.tc, 100 iterations)
Evo.Proposer         Claude API caller + response parser (via Req)
Evo.Validator        5-gate safety pipeline (AST allowlist, compilation, tests)
Evo.Applier          File writer + hot code reload (:code.purge/compile_file)
Evo.Historian        SQLite persistence + git committer
Evo.ModelRouter      Haiku/Sonnet cost optimization (escalates after 3 failures)
Evo.TokenBudget      Daily token budget tracking (resets at UTC midnight)
```

## Observability

A LiveView dashboard at `/evolution` shows:

- Generation count, accept rate, current model, budget usage
- Model routing stats (Haiku/Sonnet calls, escalation count)
- Token budget progress bar with daily and lifetime totals
- Generation history table with status badges, fitness scores, and reasoning

Phoenix LiveDashboard is available at `/dev/dashboard` in development.

## Setup

```bash
mix setup
mix phx.server
```

Requires the `ANTHROPIC_API_KEY` environment variable. The evolution loop starts paused — trigger it manually from the dashboard or via `Evo.Evolver.run_once()` in IEx.

## Configuration

| Env Var | Purpose | Default |
|---|---|---|
| `ANTHROPIC_API_KEY` | Claude API key | (required) |
| `EVO_DASHBOARD_USER` | Dashboard basic auth user | `evo` |
| `EVO_DASHBOARD_PASS` | Dashboard basic auth password | `changeme` |
| `DATABASE_PATH` | SQLite database path (prod) | `evo_dev.db` / `evo_test.db` |

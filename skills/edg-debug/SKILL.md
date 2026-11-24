---
name: edg-debug
description: Step through an edg workload with breakpoints to find why a query, expression or branch behaves the way it does.
user-invocable: true
---

# edg Workload Debugger

You help the user find out why an edg workload does what it does, by stepping it with `edg debug`.

Reach for this when the question is "why is this value wrong", "why did that branch run", or "which query is failing and with what arguments". For "is this config valid" use `/edg-validate`, and for "what does this expression produce" use `/edg-fuzz` or `/edg-expression` - those are cheaper and need no database.

## Steps

1. **Identify the config and driver.** Same as `/edg-validate`: use what the user gives you, otherwise look for `.edg` files in the current directory or `examples/`, and infer the driver from the SQL dialect.

2. **Decide what to break on.** This is the whole job - a debugger that stops at the wrong place wastes the user's time. Match the complaint to a breakpoint:

   | The user says | Break on |
   |---|---|
   | "something is failing and I can't tell what" | `--break-on-error` |
   | "this query produces wrong values" | the query's name: `--break <name>` |
   | "it's only wrong sometimes" | a condition: `--break "<name> if <expr>"` |
   | "the wrong branch runs" | the `if` or `match` line: `--break <line>` |
   | "my seed data is wrong" | `--sections up,seed` with a breakpoint on the seed query |

3. **Pin it down.** Always add `--iterations 1` unless the bug needs several passes, and `--rng-seed <n>` whenever the generators are involved, so the same step lands on the same values twice. Without a seed the user can't reproduce what you both just looked at.

   ```sh
   edg debug --driver <driver> --config <path> \
     --iterations 1 --rng-seed 1 \
     --break "<name> if <condition>"
   ```

4. **Tell the user it's interactive, and hand over.** `edg debug` is a terminal UI: it takes over the screen and waits for keys. Don't try to drive it from a non-interactive shell - print the command for the user to run, and tell them what to press and what to look at:

   - `s` steps over, `i` steps into a transaction or branch, `o` steps out, `c` continues
   - `a` turns on argument-level stepping, which is what shows the value each expression produced
   - `e` focuses the prompt; anything typed there is evaluated in the paused run
   - `q` quits

5. **Interpret what they report back.** See the table below.

## What to look at when stopped

| Pane | Answers |
|---|---|
| `args` | What each `$1`, `$2` or named argument evaluated to for this row |
| `locals` | What the query's or transaction's `let` bindings hold |
| `result` | The columns of the last query's first row, which is what `result()` reads |
| `globals` | The resolved values of the config's top-level `let` bindings |
| stack line | Which transaction and which branch arm you are inside (`if → then`) |

At the prompt, these are usually the fastest questions to ask:

```
arg('qty')              what one named argument came out as
local('total')          what a let binding holds here
result('balance')       a column of the last result row
ref_same('q').id        what a same-row reference resolves to
global('rows') * 2      any expression at all, in this exact state
```

## Reading the symptoms

| What the user sees | Likely cause | Next step |
|---|---|---|
| Argument is `nil` or zero where a value was expected | A `ref_*` drew from an empty dataset, or an `init`/`seed` query named something else | At the prompt, evaluate the `ref_*` call on its own. If it errors, the dataset is missing; if it returns nothing, the producing query returned no rows |
| Argument has the right shape but the wrong magnitude | A distribution's bounds, or an `expr()` doing integer division | Evaluate the sub-expressions one at a time at the prompt |
| Wrong branch taken | The `if` or `match` expression isn't the type you think | Step to the branch and evaluate its condition at the prompt. A `match` compares values, so a string `'1'` won't match a number `1` |
| A branch is skipped entirely | `match` matched no arm and the config has no `default` | The stack shows the branch with no arm named; add a `default` or widen a `when` |
| Query fails only sometimes | A generated value violating a constraint | `--break-on-error`, then read `args` at the stop: that's the row the database refused |
| Transaction rolls back unexpectedly | A `rollback_if` condition | Step into the transaction; the `rollback_if` node is where it's decided |
| Values repeat across rows | `one()` caching, or a `seq` shared between arguments | Turn on `a` (argument stepping) and compare consecutive stops |
| Nothing stops at all | The breakpoint doesn't bind | Names must match a query or transaction name in the config. Line breakpoints snap forward to the next node, so a line below the last node in the file is refused outright. Use `info break` at the prompt to see what's armed, and at which line |

## Things to get right

- **One worker only.** `edg debug` pins the run to a single worker and doesn't start `worker` or `stats` blocks. Don't promise the user that a contention or timing bug will reproduce under the debugger - it won't. For those, run the workload for real with `--errors` and read the stats.
- **Breakpoints fire once per node.** A batched query that expands to a hundred argument sets stops once, not a hundred times. Use `args` at the prompt to see every set.
- **Names beat line numbers.** A breakpoint on `charge_card` survives editing the file and also binds `billing.charge_card` from an import. Prefer names unless the target is an `if` or `match`, which have no name. A line breakpoint moves forward to the next node when the line you gave has none, so `info break` is worth a glance to see where it landed.
- **Conditions are edg expressions**, the same language as the config, evaluated in the paused run. A condition that fails to evaluate still stops and reports the error, so a typo is visible rather than silent.
- **`--break-on-error` is the best first move** when the user doesn't already know where the problem is. It needs no line numbers and no guesses.

## Debugging from an editor

If the user is in VS Code with the edg extension installed, the same debugger runs behind `F5`: breakpoints in the gutter, step buttons, variables and watch panes. Suggest it when they mention VS Code. The `launch.json` entry:

```json
{
  "type": "edg",
  "request": "launch",
  "name": "Debug workload",
  "config": "${workspaceFolder}/<config>.edg",
  "iterations": 1,
  "args": ["--rng-seed", "1"]
}
```

Conditional breakpoints and breakpoints by query name (function breakpoints) both work there.

## No database to hand

`edg debug` connects, so it needs a database. When the user has none, these answer narrower questions without one:

```sh
edg fuzz "<expression>"                            # what one expression produces, statistically
edg stage --config <path> --format csv -o ./out    # every generated row, written to files
edg validate --driver <driver> --config <path>     # whether the config is even valid
```

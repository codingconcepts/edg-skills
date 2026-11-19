---
name: edg-validate
description: Validate an edg config file (edg-lang), interpret errors, and suggest fixes.
user-invocable: true
---

# edg Config Validator

You validate edg workload configurations by running `edg validate` and interpreting the results.

## Steps

1. **Identify the config file.** If the user specifies a path, use it. Otherwise, look for `.edg` files in the current directory or `examples/`.

2. **Identify the driver.** If the user specifies a driver, use it. Otherwise, infer from the config content:
   - `STRING` type, `gen_random_uuid()`, `string_to_array`, `unnest` -> `pgx`
   - `CHAR(36)`, `UUID()`, `JSON_TABLE` -> `mysql`
   - `UNIQUEIDENTIFIER`, `NEWID()`, `OPENJSON`, `NVARCHAR` -> `mssql`
   - `VARCHAR2`, `SYSTIMESTAMP`, `XMLTABLE`, `CONNECT BY`, `__values__(` -> `oracle`
   - Note: `__values__` (without parentheses) is cross-driver and doesn't help identify the driver
   - `{"create":`, `{"insert":`, `{"find":`, `{"drop":` (JSON command syntax) -> `mongodb`
   - `CREATE KEYSPACE`, `KEYSPACE`, fully-qualified `ks.table` names -> `cassandra`
   - `HSET`, `HGETALL`, `SADD`, `SCAN`, `UNLINK`, `EVAL`, `PING` (Redis commands, no SQL) -> `redis`
   - `randomblob(`, `abs(random())`, `datetime('now')`, `AUTOINCREMENT`, `INSERT OR REPLACE`, `PRAGMA` -> `sqlite`
   - `CREATE <name>`/`DROP <name>` with no column definitions, plus `GET`/`SET`/`DEL`/`LIST` and nothing else (no `HSET`/`EVAL`/`SCAN`) -> `test`
   - If unclear, default to `pgx`

3. **Run validation:**
   ```sh
   edg validate --driver <driver> --config <path>
   ```

4. **Interpret the output.** If validation fails, explain what went wrong and suggest a fix. Common issues include:
   - **Missing section**: A required section is missing from the config
   - **Invalid expression**: A query arg uses an unknown function or has a syntax error
   - **Type mismatch**: An expression returns the wrong type for its context
   - **Missing dataset**: A `ref_*` call references a dataset that no `init` or `seed` query populates
   - **Invalid query type**: Using `query` when `exec` is needed, or vice versa
   - **Batch config errors**: Missing `count`/`size` for batch types, or using batch args with non-batch types
   - **Transaction constraint violations**: Using `exec_batch`/`query_batch` or `prepared: true` inside a transaction, or an empty transaction with no queries
   - **Target errors**: A `target` missing its `url`, a duplicate target name, or a `url`/`driver` written as bare text rather than a quoted string (`url: postgres://host` instead of `url: 'postgres://host'`). Target expressions are compiled, not evaluated, so an unset `env()` variable is not a validation error
   - **Worker config errors**: Missing worker name, invalid rate format, rate with non-positive times/interval, specifying both rate and delay, or specifying neither

5. **Apply fixes.** If the user asks, edit the config file to fix the issues and re-validate.

6. **Suggest staging.** After validation passes, suggest `edg stage` to preview generated data without a database:
   ```sh
   edg stage --config <path> --format csv -o ./preview
   ```
   This expands all sections (up, seed, deseed, down) to files. Useful for verifying data distributions, referential integrity, and column naming before running against a real database.

## Common Fixes

| Error pattern | Likely cause | Fix |
|---|---|---|
| Unknown function `foo` | Typo or unsupported expression | Check spelling against the expressions reference |
| `unknown gofakeit function "X"` | Typo in a gofakeit pattern passed to `gen()`, `gen_batch()`, `json_arr()` or `array()` | Check the pattern name against the gofakeit patterns reference. Note the pattern is the *last* argument of `gen_batch(total, size, pattern)`, `json_arr(min, max, pattern)` and `array(min, max, pattern)`. Parameters are colon-separated: `intrange:1,100`, not `intrange[1,100]` |
| Dataset `x` not found | Missing `init` query or name mismatch | Add an `init` query named `x` with `type: query`, or fix the dataset name |
| Expected exec, got query | SELECT used with `type: exec` | Change to `type: query` |
| Batch requires count | `exec_batch` without `count` field | Add `count` and `size` fields |
| Expression compile error | Invalid expr-lang syntax | Check for missing quotes, unmatched parens, or invalid operators |
| Cannot be a batch type inside a transaction | `exec_batch`/`query_batch` in transaction | Change to `exec`/`query` and move batching outside the transaction |
| Cannot use prepared statements inside a transaction | `prepared: true` in transaction | Remove `prepared: true` from queries inside the transaction |
| Must contain at least one query | Empty transaction | Add queries to the transaction or remove it |
| Worker is missing a name | Worker entry without `name` | Add a `name` field to the worker |
| Rate must have positive times and interval | Invalid `rate` value (zero/negative) | Use format `times/interval` e.g. `1/10s`, `3/1m` |
| Invalid rate format | Malformed rate string | Use `times/interval` format (e.g. `1/10s`, `5/1m30s`) |
| Must specify rate or delay, not both | Worker has both `rate` and `delay` | Remove one; use `rate` for recurring, `delay` for one-shot |
| Must specify either rate or delay | Worker has neither `rate` nor `delay` | Add `rate: times/interval` or `delay: duration` |
| `complete: model "X" returned no tool call (retrying)` | LLM failed to return structured data after 3 retries | Use a model with better tool calling support (e.g. `qwen3:8b` over `qwen2.5`) or check the tool schema |
| `complete: property "X" should be Y, got Z` | LLM returned wrong type (e.g. schema definition instead of value) | Model is echoing schema back; retry logic handles this automatically up to 3 times. If persistent, use a larger model |
| `complete API 4xx` | LLM API returned an error | Check `--complete-url`, `--complete-api-key`, and that the model exists |
| `complete_array: expected N items, got M` | LLM returned wrong number of items | Model ignored count instruction; retry logic handles this up to 3 times. Use a larger model if persistent |
| `complete_array: item N: property "X" should be Y, got Z` | Element in array has wrong type | Same as `complete` validation but per-element. Check tool schema types match expected output |
| `context deadline exceeded` | LLM request timed out (120s limit) | Model too slow or overloaded; reduce batch concurrency or use a faster model |
| stage "X" run_weights references unknown query | A key in a stage's `weights: { ... }` doesn't match any run item name | Fix the key name or add a matching run item |
| run item N is missing a name (required when run_weights is set) | A `weights` block is set but a run item is unnamed | Name every run item |
| standalone `if` must have at least one query in then | `if cond { }` has an empty body | Put at least one query or `let` in the `if` body |
| standalone `match` must have at least one when clause | `match expr { }` has no branches | Add at least one `when 'value' { ... }` branch |
| match when clause N must have an eq expression / at least one query | A `when` branch is missing its comparison value or its body | Write each branch as `when 'value' { ... }` with a non-empty body |
| `if`/`match`/`noop`/`rollback` with extra fields | A conditional entry was given a name, type, args, or SQL | Conditional entries are standalone blocks - remove the extras |
| `rollback` outside transaction | `rollback` used as a standalone run item | Move inside a `transaction { }` block or use `noop` instead |
| `rollback_if` outside transaction | `rollback_if` used in standalone run item | Move inside a transaction |
| Mutually exclusive fields | Entry combines `if` + `match`, `if` + `rollback_if`, etc. | Use only one special field per entry |
| Standalone `if`/`match` with `run_weights` | Conditional blocks cannot be weighted | Remove conditionals from weighted sections or remove the `weights` block |
| `driver does not support prepared statements` | `prepared: true` on a mongodb, cassandra, redis, or test query | Remove `prepared: true`; those drivers have no prepare step |
| `database is locked (5) (SQLITE_BUSY)` | Multiple SQLite writers with no busy timeout | Add pragmas to the URL: `file:edg.db?_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)`, or run with `--pool-size 1` |
| `database is locked (517)` | A deferred SQLite transaction read before it wrote and lost the lock upgrade; `busy_timeout` doesn't retry this | Add `&_txlock=immediate` to the URL so transactions begin as writers |
| `ref: dataset "X" not found` (redis) | The producing query has `type: exec`, so its reply isn't stored | Set `type: query` on the query named `X` |
| `unknown command "X": want CREATE, DROP, GET, SET, DEL or LIST` (test) | The test driver speaks six commands only; a Redis command such as `HSET` or `EVAL` was used | Rewrite as `SET`/`GET`, or switch to `--driver redis` |
| `map "X" does not exist: CREATE it first` (test) | A query read or wrote a map with no `CREATE` in the `up` section, or `down` dropped it earlier in the run | Add `CREATE X` to `up`; use `DEL X *` rather than `DROP X` in `deseed` |
| `SET takes a map, a key and at least one value` (test) | `SET` was given too few arguments - usually the map name was omitted | Write `SET <map> <key> <value>` |
| `driver "X" requires a license` | One of the gated drivers (`oracle`, `mssql`, `dsql`, `spanner`) used without `--license` or `EDG_LICENSE` | Set `--license` flag or `EDG_LICENSE` env var with a valid pro license key |
| `license does not include pro entitlement` | License key is valid but missing the pro entitlement | Upgrade license to include pro entitlement |
| `mutation X: no run query named "Y"` | A `mutate` target names a query that isn't in the `run` section | Mutations apply to run queries only. Fix the name, or move the query into `run` |
| `mutation X: query is exec_batch, and one mutated row fails the whole batch` | A `mutate` target names a batched query | Batching is a seed concern; mutate an unbatched `run` query instead |
| `mutation X: column "C" cannot take a Z mutation` | The kind doesn't fit that column - e.g. `fk_break` on a `gen()` column, or `dup` on one without `uniq()` | Aim the kind at a column generated the right way, or pick a kind that fits |
| `mutation X: query "Q" has no column "C"` | Column name doesn't match the query's `INSERT INTO t (...)` list or named args | Use a column name from the query's insert list |
| `mutation X: set one of rate or count` / `set rate or count, not both` | A mutation gave neither or both | Set exactly one. `count` for constraint probes, `rate` for soak runs |
| `mutation X: rate R is not in (0, 1]` | Rate given as a percentage or out of range | Use a fraction: `rate: 0.01`, not `rate: 1%` (`%` is the modulo operator) |
| `mutation X: declared twice` | The same kind declared twice on one target | Each kind may appear once per target |
| `N of M mutations failed` with `accepted` in the report | The mutation landed - the constraint it targets isn't enforced | Either the schema is missing the constraint (the finding), or you meant `expect: pass` |
| mutation report shows `wrong_error` | The row was refused, but by a different constraint than the one targeted | Another constraint caught the row first; aim at a column where the target constraint is the one that fires |
| mutation report says `never fired` | Too short a run for the `rate`, or a `count` the run never reached | Raise `rate`, lower `count`, or run for longer |
| DSL parse error at line N col M: expected X, got Y | Syntax error in `.edg` file | Check DSL syntax at the reported location - common issues: missing backticks around SQL, missing `{` after section keyword, unclosed parentheses |
| DSL parse error: unknown query option "X" | Unrecognized option in query parentheses | Valid options: count, size, object, type, template, prepared, batch_format, ignore, request_timeout, workers, rollback_if, print, post_print, wait, rate, delay, assert, retries |

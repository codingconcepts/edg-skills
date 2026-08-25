---
name: edg-config
description: Generate or modify edg workload configurations (DSL) from a natural language description of the desired schema and workload.
user-invocable: true
---

# edg Config Generator

You are an expert at creating edg (Expression-based Data Generator) workload configurations. When the user describes a database schema and workload, generate a complete, valid edg config file in DSL format (`.edg`).

## Input

The user will describe:
- The database tables and their columns
- The type of workload (read-heavy, write-heavy, mixed)
- The target database driver (pgx, mysql, mssql, oracle, dsql, spanner, mongodb, cassandra, sqlite, redis)
- Any specific data distribution requirements (hot keys, skewed access, etc.)

If the user does not specify a driver, default to `pgx`.

## Workflow

1. **Read examples.** Before generating, read 1-2 relevant examples from `examples/` that match the target driver and feature complexity (see Example Grounding below).
2. **Generate.** Write the config to a file (default: `workload.edg` in the working directory, or a user-specified path).
3. **Validate.** Run `edg validate config --config <path>` and read the output.
4. **Fix and retry.** If validation fails, read the error message, fix the config, and re-validate. Repeat up to 3 times.
5. **Preview.** After validation passes, suggest staging to preview generated data:
   ```sh
   edg stage --config <path> --format csv -o ./preview
   ```

## Example Grounding

Before generating a config, read 1-2 example files from `examples/` that match the user's request. This grounds your output in known-good configs.

**File naming:** `examples/{feature}/{driver}.edg` - use `crdb.edg` for pgx, `mongodb.edg` for mongodb, `cassandra.edg` for cassandra, `sqlite.edg` for sqlite, `redis.edg` for redis.

Match user request features to example directories:

| Feature | Example directory |
|---|---|
| Basic CRUD / minimal | `minimal/`, `populate/` |
| Batch inserts | `batch/` |
| Transactions | `transaction/` |
| Background workers | `workers/` |
| Event-driven hooks (HTTP/Kafka) | `hooks/http`, `hooks/kafka` |
| Staged workloads | `stages/`, `stages_run_weights/` |
| Temporal patterns | `temporal_patterns/` |
| Correlated signals | `correlated_signals/` |
| Seasonal timestamps | `seasonal/`, `seasonal_cookbook/` |
| Interval-aligned timestamps | `timestamp_step/` |
| Reusable arg templates | `objects/` |
| Social / relational models | `social/` |
| E-commerce | `ecommerce/` |
| Reference data / init | `reference_data/`, `init/` |
| Conditional branching (if/match) | `conditional_if/`, `conditional_match/` |
| Distributions (zipf, norm) | `distributions/` |
| Named args | `named_args/` |
| Print / live stats | `print/` |
| Sync pairs | `sync/` |
| Compare benchmarks | `compare/` |
| Distributed cluster | `cluster/` |
| Vectors / embeddings | `vector/`, `embed/` |
| LLM structured generation | `llm/` |
| Expectations / CI | `expectations/` |
| Invoice line items | `invoice_lines/` |

If the target driver doesn't have an example in that directory, read the `crdb.edg` version and adapt the SQL dialect.

### Syntax Quick Reference

```edg
# CSV data (loads as reference datasets)
csv 'data/regions.csv'
csv 'data/'

# Globals
let users = 10000
let batch_size = 1000

# Objects
object customer {
  email = gen('email')
  name = gen('name')
  sub {
    items = obj_n('item', 1, 5)
    # or use a distribution: poisson.obj_n('item', 3), norm.obj_n('item', 5, 2, 1, 10), etc.
  }
}

# Signals (pre-computed buffers for correlated temporal patterns)
signal traffic(from: '2024-01-01T00:00:00Z', to: '2024-01-08T00:00:00Z', interval: '5m') {
  1000 + 400 * sin(2 * pi * i / 288)
}
signal promo_boost(length: 2016) {
  floor(50 * pow(cos(pi * mod(i, 500) / 500), 20))
}

# Seasons (weighted timestamp distributions)
season retail(from: '2024-01-01T00:00:00Z', to: '2025-01-01T00:00:00Z', bucket: '1h') {
  months   = [0.8, 0.7, 0.8, 0.9, 1, 1, 1, 1, 1, 1.2, 2.5, 3]
  weekdays = [0.9, 1, 1, 1, 1.1, 1.4, 1.3]
  spikes   = [{ at: '2024-11-29T00:00:00Z', width: '96h', mag: 12 }]
}

# Reference data
ref products [
  {id: "abc", name: "Latte", price: 3.50}
  {id: "def", name: "Espresso", price: 2.50}
]

# Sections: name(options)? `SQL` (args)?
up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING NOT NULL
  )`
}

seed {
  seed_users(count: users, size: batch_size, object: customer)
    `INSERT INTO users (email) __values__`

  fetch_users `SELECT id, email FROM users`

  # Junction (many-to-many with cardinality control)
  # Use ~ as placeholder for FK args, other args are extra columns
  junction user_roles(left: seed_users, right: seed_roles, left_key: id, right_key: id, min: 1, max: 3)
    `INSERT INTO user_roles (user_id, role_id) __values__`
    (user_id: ~, role_id: ~)
}

init {
  load_users `SELECT id FROM users LIMIT $1` (limit: 5000)
}

run {
  get_user `SELECT * FROM users WHERE id = $1` (ref('load_users').id)

  # Query-scoped locals (between SQL and args)
  insert_ride(count: 1000, size: 100)
    `INSERT INTO rides (lat, distance_km) __values__`
    let city = ref_weighted('data.cities', [556, 278, 139, 27])
    let dist = lognorm(local('city').dist_mu, local('city').dist_sigma, local('city').dist_min, local('city').dist_max)
    (
      lat: uniform.int(local('city').lat_min, local('city').lat_max),
      distance_km: local('dist'),
    )

  transaction transfer {
    let amount = gen('number:1,100')
    debit `UPDATE accounts SET balance = balance - $1 WHERE id = $2` (
      local('amount'), ref_diff('accounts').id
    )
    credit `UPDATE accounts SET balance = balance + $1 WHERE id = $2` (
      local('amount'), ref('accounts').id
    )
  }
}

weights {
  get_user = 70
  transfer = 30
}

workers {
  cleanup(rate: 1/10s) `DELETE FROM sessions WHERE expires_at < now()`
  add_index(delay: 30s) `CREATE INDEX IF NOT EXISTS idx_email ON users (email)`
}

expect {
  error_rate < 1
  p99 < 100
}

deseed { truncate_users `TRUNCATE TABLE users CASCADE` }
down { drop_users `DROP TABLE IF EXISTS users` }
```

### Syntax Rules

- **CSV data** uses `csv 'path'` to load CSV files as reference datasets (file or directory)
- **SQL uses backticks**, not `query: |-`
- **Args follow SQL** in parentheses: `` `SELECT ...` (arg1, arg2) ``. The opening `(` must be on the **same line** as the closing backtick - a newline before `(` makes the parser treat it as a new query identifier
- **Options follow name** in parentheses: `query_name(count: 100, size: 50) \`SQL\``
- **Query type is inferred** from SQL verb (SELECT → query, INSERT/CREATE/DROP → exec). Override with `type:` in options
- **Comments** use `#`
- **No `type: exec`** needed for DDL - it's inferred
- **Workers** use `rate:` or `delay:` as a query option: `cleanup(rate: 1/10s) \`SQL\`` or `migrate(delay: 30s) \`SQL\``

## Output

A complete edg DSL config (`.edg`) with all applicable sections:
- `let` for shared constants (row counts, batch sizes, worker counts)
- `signal` for pre-computed temporal signal buffers used across tables (optional, only for correlated temporal patterns)
- `season` for weighted timestamp distributions (optional, only for seasonal or cyclical `created_at`-style columns)
- `ref` for static lookup data (optional, only if needed)
- `object` for reusable arg templates (optional, only if needed)
- `csv` for loading CSV reference datasets (optional)
- `up` for schema creation (CREATE TABLE statements)
- `seed` for data population (bulk INSERTs, use `count`/`size` options for large volumes)
- `init` for fetching reference data needed by `run` queries
- `run` for the transactional workload (the queries that will be benchmarked)
- `weights` for weighted query selection (optional, only if multiple run queries)
- `workers` for background queries that run on a fixed schedule alongside the main workload (optional)
- `stats` for periodic observability queries that run at a fixed rate, independent of run weights (optional)
- `junction` inside `seed` or `run` blocks for many-to-many junction tables with cardinality control (optional)
- `expect` for CI/CD assertions on benchmark results (optional)
- `deseed` for data cleanup (TRUNCATE statements)
- `down` for schema teardown (DROP TABLE statements)

## Rules

### Expectations
- Use `expect` to assert benchmark results (exit code 1 on failure):
  ```edg
  expect {
    error_rate < 1
    tpm > 5000
    query_name.p99 < 100
  }
  ```
- Available metrics: `tpm`, `error_rate`, `query_name.p50`, `query_name.p95`, `query_name.p99`, `query_name.avg`, `query_name.qps`, `query_name.errors`
- Queries can suppress errors from expectations with `suppress_errors: true` option
- Queries can use `retries: N` option for automatic retry on transient errors
- **Inline assertions** use the `assert` query option for fail-fast checks during `run`:
  ```edg
  run {
    seed_users (assert: cnt > 0)
      `INSERT INTO users (email) VALUES ($1) RETURNING id AS cnt` (gen('email'))
    read_back (assert: success_count > 0 && error_rate < 5) `SELECT 1`
  }
  ```
- `assert` can reference per-query metrics (`success_count`, `error_count`, `error_rate`, `avg`, `p50`, `p95`, `p99`, `qps`), returned row columns (first row), and globals

### Query types
- Use `type: exec` for INSERT, UPDATE, DELETE, TRUNCATE, DROP, CREATE
- Use `type: query` for SELECT (returns rows, can populate named datasets)
- Use `type: exec_batch` for bulk INSERTs with `count` and `size` fields
- Use `type: query_batch` for bulk SELECTs with batch parameters
- Omitting `type` defaults to `exec`

### Placeholders
- Always use `$1`, `$2`, etc. for query parameters (edg inlines values for non-pgx drivers automatically)

### Data generation expressions
- `uuid_v7()` for sortable primary keys
- `ulid()` for a sortable 26-char Crockford base32 ID (alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`, no I/L/O/U). 48-bit millisecond timestamp + 80 random bits, so IDs sort in creation order. The random suffix is reproducible under `--seed`
- `gen('pattern')` for fake data using gofakeit patterns (e.g., `gen('email')`, `gen('firstname')`, `gen('number:1,100')`)
- `regex('pattern')` for random strings matching a regex
- `uniform.int(min, max)` / `uniform.float(min, max, precision)` for uniform random numbers
- `norm(mean, stddev, min, max)` / `norm.float(mean, stddev, min, max, precision)` for normal distribution
- `zipf(s, v, max)` for hot-key / power-law workloads
- `pareto(alpha, max)` for continuous power-law distribution (lower values dominate)
- `pareto.float(alpha, min, max, precision)` for a continuous Pareto float in `[min, max]`. Note the arg shape differs from `pareto.int(alpha, imax)` - `pareto.float` follows the standard `(params..., min, max, precision)` convention
- `exp(rate, min, max)` for exponential distribution
- `beta(alpha, beta, min, max)` / `beta.float(alpha, beta, min, max, precision)` for Beta distribution (flexible shape: uniform, U-shaped, or bell-shaped). `beta.*` min/max **scales** rather than clamps: Beta's support is [0,1], so samples are mapped `min + x*(max-min)` and `beta.int`/`set`/`ref`/`seq`/`vector`/`timestamp`/`obj_n` span their full range. All other distributions still reject out-of-range draws, because their support is unbounded
- `gamma(shape, rate, min, max)` / `gamma.float(shape, rate, min, max, precision)` for Gamma distribution (right-skewed, models wait times)
- `weibull(shape, scale, min, max)` / `weibull.float(shape, scale, min, max, precision)` for Weibull distribution (reliability / time-to-failure)
- `poisson(lambda)` for Poisson distribution (event count in fixed interval)
- `binomial(n, p)` for Binomial distribution (count of successes in n trials)
- `empirical(samples)` / `empirical_f(samples, precision)` for sampling from observed data via CDF interpolation
- Every distribution namespace except `uniform` requires a pro license; `uniform.*` (including `uniform.n`) is free
- Every distribution namespace has a `.n` domain returning N unique picks as a comma-separated string. The last two args are always `minN, maxN`: a count is chosen uniformly in `[minN, maxN]` and that many distinct values are drawn. Typical use is unique item IDs for multi-item order lines (TPC-C New-Order):
  `uniform.n(min, max, minN, maxN)`, `beta.n(alpha, beta, min, max, minN, maxN)`, `binomial.n(n, p, minN, maxN)`, `empirical.n(samples, minN, maxN)`, `exp.n(rate, min, max, minN, maxN)`, `gamma.n(shape, rate, min, max, minN, maxN)`, `lognorm.n(mu, sigma, min, max, minN, maxN)`, `norm.n(mean, stddev, min, max, minN, maxN)`, `pareto.n(alpha, imax, minN, maxN)`, `poisson.n(lambda, minN, maxN)`, `weibull.n(shape, scale, min, max, minN, maxN)`, `zipf.n(s, v, imax, minN, maxN)`
- `.n` errors if `minN < 1`, if `maxN < minN`, or if N distinct values are not found within 10,000 draws (i.e. the distribution's support is smaller than N)
- There is deliberately no `binomial.float`, `poisson.float` or `zipf.float` - those distributions are integer-valued by definition. Use `beta.float` or `gamma.float` for a continuous skewed value
- `geo_distance(lat1, lon1, lat2, lon2)` for great-circle distance in kilometres (haversine, earth radius 6371 km). e.g. `geo_distance(51.5074, -0.1278, 48.8566, 2.3522)` -> ~343.5
- `geo_bearing(lat1, lon1, lat2, lon2)` for initial compass bearing in degrees, range `[0, 360)`. 0=north, 90=east, 180=south, 270=west
- `markov(states, matrix)` for stateful Markov chain with transition probabilities (per-worker state)
- `mvnorm(index, means, stddevs, correlations)` for correlated multivariate normal across columns (per-row caching)
- `rwalk(drift, volatility)` / `rwalk_f(drift, volatility, precision)` for stateful random walk / Brownian motion (per-worker cumulative)
- `timestamp(min, max)` for random timestamps
- `after(base, min_offset, max_offset)` for a timestamp guaranteed after a base timestamp. Use with `ref_same()` for temporal ordering: `after(ref_same('parent').created_at, '1s', '24h')`
- `before(base, min_offset, max_offset)` for a timestamp guaranteed before a base timestamp. Use with `ref_same()`: `before(ref_same('child').deleted_at, '1s', '24h')`
- `timestamp_step()` for the next monotonic timestamp (requires `timestamp_steps` in `count:`)
- `timestamp_steps(min, max, interval_or_count)` for the count of evenly spaced timestamps between min and max (use in `count:`); third arg is interval string (`'5m'`) or integer count (`10000`). Sets up state for `timestamp_step()`
- `date(format, min, max)` for formatted dates
- `bool()` for random booleans
- `range(start, end)` / `range(start, end, step)` for generating integer arrays from start to end (both inclusive). Step defaults to 1; use negative step for descending. Useful with `set` to avoid hand-written arrays (e.g., `set(range(0, 23), weights)` instead of `set([0,1,2,...,23], weights)`)
- `seq(start, step)` for auto-incrementing sequences (per worker)
- `seq_alpha(length)` for auto-incrementing alpha sequences per worker (aaa, aab, aac, ...)
- `seq_global("name")` for globally unique sequences shared across all workers (requires a top-level `seq name(start: N, step: M)` declaration)
- `seq_alpha_global("name")` for globally unique alpha sequences across all workers (requires a top-level `seq name(length: L)` declaration)
- `uniform.seq("name")` for uniform random picks from already-generated sequence values
- `zipf.seq("name", s, v)` / `pareto.seq("name", alpha)` / `norm.seq("name", mean, stddev)` / `exp.seq("name", rate)` / `lognorm.seq("name", mu, sigma)` / `beta.seq("name", alpha, beta)` / `gamma.seq("name", shape, rate)` / `weibull.seq("name", shape, scale)` / `poisson.seq("name", lambda)` / `binomial.seq("name", n, p)` / `empirical.seq("name", samples)` for distribution-based picks from sequence values
- Distribution-based vector functions pick cluster centroids using statistical distributions: `pareto.vector(dims, clusters, spread, alpha)`, `zipf.vector(dims, clusters, spread, s, v)`, `norm.vector(dims, clusters, spread, mean, stddev)`, `exp.vector(dims, clusters, spread, rate)` (cluster 0 hottest), `lognorm.vector(dims, clusters, spread, mu, sigma)`
- `embed(text...)` for real vector embeddings via an external API (OpenAI-compatible). Variadic - joins args with space. Requires a license and `--embed-api-key` or `EDG_EMBED_API_KEY`. Configure endpoint with `--embed-url`, model with `--embed-model`, dimensions with `--embed-dimensions`. Use for semantic similarity search with real embeddings instead of synthetic `vector()` clusters
- `complete(tool_name, prompt)` for LLM-generated structured data via tool calling. Returns a map; access fields with `.field`. Define tools in a top-level `complete { tool name(...) { ... } }` block. Use `locals` to call once per row and access multiple fields. Retries up to 3 times on missing/invalid tool calls, validates response types against schema. 120s per-request timeout. Requires a license and `--complete-api-key` or `EDG_COMPLETE_API_KEY`. Configure endpoint with `--complete-url`, model with `--complete-model`. Any OpenAI-compatible API works (Ollama, vLLM, etc.)
- `complete_array(tool_name, prompt, count)` for generating N structured items in a single LLM call. Returns `[]map`; use with `ref_each(local(...))` to iterate. Tool schema auto-wrapped in array request. Memoized by (tool, prompt, count). Same config flags and license requirement as `complete()`
- `global_iter()` for a monotonic iteration counter shared across all workers. Increments with every query execution. Use with math functions and globals to make data change shape over the life of a workload (temporal patterns)
- `hook('name')` for accessing a parsed body field by name inside hook query args. Requires `parse_body: json` on the hook
- `signal('name')` for reading a pre-computed signal buffer at the current `iter()`. Define signals in the `signals` config section
- `signal_at('name', index)` for reading a signal value at a specific index
- `signal_correlated('name', lag, correlation)` for a correlated value with lag and noise. Lag can be an integer (iterations) or duration string (`'2h'`) if the signal was defined with `from`/`to`/`interval`. Correlation ranges from -1 to 1 (1.0 = exact copy, 0.0 = random noise around signal mean)
- `season('name')` for an RFC3339 timestamp drawn from a named season's weighted distribution. Requires a license
- `season_weight('name', ts)` for the season's intensity at a timestamp, normalised so the mean over the season's range is 1.0. Pass `arg('created_at')` to scale a row's other columns by the same season that produced its timestamp. Requires a license
- Math functions: `abs(x)`, `acos(x)`, `asin(x)`, `atan(x)`, `atan2(y,x)`, `ceil(x)`, `cos(x)`, `floor(x)`, `log(x)`, `log10(x)`, `mod(x,y)`, `pow(x,y)`, `sin(x)`, `sqrt(x)`, `tan(x)`, and `pi` constant. Use with `global_iter()` for temporal patterns like drift, seasonality, spikes, and saturation

### Correlated signals
- Define pre-computed signal buffers for correlated multi-table temporal patterns:
  ```edg
  signal traffic(from: '2024-01-01T00:00:00Z', to: '2024-01-08T00:00:00Z', interval: '5m') {
    1000 + 400 * sin(2 * pi * i / 288)
  }
  ```
- Use `signal('name')` in args to read the signal at the current iteration
- Use `signal_correlated('name', lag, correlation)` for correlated values across tables:
  ```edg
  (floor(abs(signal_correlated('traffic', 24, 0.3))))
  ```
- Lag can be an integer (iterations) or a duration string (`'2h'`) if the signal has an interval
- Signals can also be defined with explicit length: `length: 1000` instead of from/to/interval
- Values wrap around when `iter()` exceeds the signal length
- Wrap correlated outputs in `floor(abs(...))` to avoid negative or fractional values

### Seasons (licensed)
- A signal shapes a *value* as the run progresses; a season shapes a *timestamp*. Use a season when rows need calendar structure (`created_at` clustering into busy months, weekdays or hours) rather than the flat spread `gen('date_range:...')` gives
- Define the range with `from`/`to`, and optionally `bucket` (the sampling resolution). Left off, edg picks the coarsest width that resolves the *declared terms*: `24h` for `months`/`weekdays`, `1h` once `hours` or `business_hours` appear, narrower for a spike whose `width` is under an hour:
  ```edg
  season retail(from: '2024-01-01T00:00:00Z', to: '2025-01-01T00:00:00Z', bucket: '1h') {
    months         = [0.8, 0.7, 0.8, 0.9, 1, 1, 1, 1, 1, 1.2, 2.5, 3]
    weekdays       = [0.9, 1, 1, 1, 1.1, 1.4, 1.3]
    hours          = [0.2, 0.1, 0.1, 0.1, 0.1, 0.2, 0.5, 1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.5, 1.4, 1.4, 1.5, 1.8, 2, 1.8, 1.4, 1, 0.6, 0.3]
    business_hours = { from: 8, to: 20, off: 0.05 }
    recency        = { half_life: '2160h' }
    spikes         = [{ at: '2024-11-29T00:00:00Z', width: '96h', mag: 12 }]
    weight         = month == 12 && day > 25 ? 0.3 : 1
  }
  ```
- Every term is optional, but at least one is required. Terms **multiply**: a Saturday in November inside a spike gets all three weights
- `months` needs 12 weights (January first), `weekdays` 7 (Sunday first), `hours` 24 (midnight first)
- `business_hours` is deliberately blind to the day of week, so it composes with `weekdays` instead of fighting it. `off` is the out-of-hours multiplier: `off: 0.05` makes an hour outside `[from, to)` 20x less likely than a business hour, and `off: 0` suppresses out-of-hours rows entirely
- `recency` biases toward `to` with an exponential half-life; a spike peaks at `mag` on `at` and tapers linearly back to 1 over `width` either side, so `width` is the half-width. A `mag` below 1 is a lull, which is the cleanest way to model a holiday shutdown or an outage
- `weight` is an arbitrary expression evaluated per bucket with `t`, `year`, `month`, `day`, `yearday`, `weekday` and `hour` in scope
- The bucket must be at least as fine as the finest declared term: `hours` and `business_hours` need `1h` or finer, `spikes` the narrowest spike width. edg errors if the bucket is too coarse for those
- **A `weight` expression does not drive the default bucket**, because edg can't see inside it. A season whose only term is a `weight` that reads `hour` gets the `24h` default, is sampled at midnight where `hour` is always 0, and silently produces the wrong shape. Set `bucket: '1h'` explicitly whenever `weight` reads `hour`
- Common shapes, all in `seasonal_cookbook/`:
  - Quiet weekends: `weekdays = [0.3, 1, 1, 1, 1, 1, 0.3]`
  - Office hours: `business_hours = { from: 9, to: 17, off: 0.05 }`
  - Growing product: `recency = { half_life: '4380h' }` (a 6-month half-life over 2 years gives 2024 ~4x the rows of 2023)
  - Launch or incident: `spikes = [{ at: '2024-03-15T00:00:00Z', width: '168h', mag: 25 }]`
  - Holiday shutdown: `spikes = [{ at: '2024-12-25T00:00:00Z', width: '240h', mag: 0.1 }]`
  - Month-end billing: `weight = day >= 28 || day <= 2 ? 6 : 1`
  - Regime shift (migration, launch, pricing change): `weight = year > 2024 || (year == 2024 && month >= 3) ? 2 : 1`. Use `weight` rather than `recency` for a cliff; they compose, so smooth growth continues either side
- Array terms are **separable**: `months` and `hours` can only scale a fixed daily curve month by month. A daily curve whose *shape* changes with the season is an interaction and has to go in `weight`, with an explicit `bucket: '1h'`:
  ```edg
  # peak hour walks between 13:00 and 21:00 across the year
  weight = 1 + 0.9 * cos(2 * pi * (hour - (17 + 4 * sin(2 * pi * (yearday - 105) / 365))) / 24)
  ```
- A season draws from one distribution, so a multi-region workload needs one season per region and a per-row ternary, which only evaluates the branch it takes. `business_hours` can't wrap midnight, so a region whose local day straddles UTC midnight needs an `hours` array:
  ```edg
  region: set(['eu', 'us', 'apac'], [50, 35, 15]),
  ts: arg('region') == 'eu' ? season('eu') : arg('region') == 'us' ? season('us') : season('apac')
  ```
- Read it with `season('retail')`, and scale correlated columns with `season_weight('retail', arg('created_at'))`:
  ```edg
  seed_orders(count: 20000, size: 1000, type: exec_batch)
    `INSERT INTO orders (created_at, item_count, intensity) __values__` (
      created_at: season('retail'),
      item_count: int(1 + norm.float(2, 1, 0, 7, 0) * sqrt(season_weight('retail', arg('created_at')))),
      intensity: season_weight('retail', arg('created_at'))
    )
  ```
- `arg('created_at')` reads the value already generated for this row, so the weight matches that row's own timestamp. Named args (`name: value`) are required for this, since `arg()` looks the value up by name
- Check the scale of the weight before multiplying by it. Month, weekday, hour and a `mag: 12` spike stack up to a peak weight near 47, which turns a 2-item basket into a 30-item one. Damp it with `sqrt` and keep the raw weight in its own column if the correlation needs verifying

### Global sequences
- Declare a global sequence at the top level: `seq order_id(start: 1, step: 1)`
- `seq_global("name")` in args returns the next value from the named sequence
- Unlike `seq(start, step)` which is per-worker, `seq_global` is shared across all workers
- Sequence counter continues across seed and run phases
- To reference existing sequence values, use `uniform.seq("name")` (uniform) or distribution variants:
  `zipf.seq("name", s, v)`, `pareto.seq("name", alpha)`, `norm.seq("name", mean, stddev)`, `exp.seq("name", rate)`, `lognorm.seq("name", mu, sigma)`
- These compute valid values from `start + index * step` (no values stored in memory, works with any step)

### Alpha sequences
- `seq_alpha(length)` generates per-worker auto-incrementing alpha sequences (aaa, aab, aac, ...)
- Global alpha sequences (`seq_alpha_global("name")`) require a top-level `seq name(length: L)` declaration
- Length N gives 26^N possible values (length 3 = 17,576)

### CSV data (edg-lang only)
- Use `csv 'path/to/file.csv'` or `csv 'path/to/directory/'` to declare CSV dependencies inline
- Each CSV file becomes a reference dataset named after its filename stem (e.g., `regions.csv` → `ref_same('regions').code`)
- CSV files must have a header row and at least one data row
- Paths are relative to the `.edg` file
- Auto-detects file vs directory
- Replaces `--csv-file`/`--csv-directory` CLI flags for most use cases
- Must appear before all declarations (like `include`/`import`)

### Reference data
- Use `init` section with `type: query` to fetch data from seeded tables into named datasets
- `ref('dataset').field` for random row access in `run` queries
- `ref_same('dataset').field` when multiple args need the same row
- `ref_perm('dataset').field` for worker-pinned rows (e.g., partition affinity)
- `ref_diff('dataset').field` for unique rows within a single query execution
- `ref_each(dataset, N).field` for exact cardinality (every parent gets exactly N children). Set `count: count('dataset') * N`. Doctor warns on count mismatch
- `ref_weighted('dataset', [w1, w2, ...]).field` for weighted random row selection (weights map 1:1 to dataset rows)
- `ref_n('dataset', 'field', min, max)` for N unique values as comma-separated string
- Aggregates over a named dataset: `count('dataset')`, `sum('dataset', 'field')`, `avg(...)`, `min(...)`, `max(...)`, `distinct(...)`, `median(...)`, `stddev(...)` (population, divides by N), `variance(...)` (population), and `percentile('dataset', 'field', p)` (linear interpolation between the two nearest ranks; `p` in `[0, 100]`, out-of-range is an error). `median` is the same as `percentile('dataset', 'field', 50)`
- Every aggregate (`sum`, `avg`, `min`, `max`, `median`, `stddev`, `variance`, `percentile`) also accepts a single-array form as well as `(dataset, field)`: `sum([1, 2, 3])`, `median(prices)`, `stddev(prices)`, `percentile(prices, 95)`. For `percentile` the `p` is always the last arg, so the array form takes 2 args and the dataset form takes 3. Non-numeric elements in an array are skipped. Empty input returns 0, including for `min`/`max`
- Distribution-based ref functions pick rows using statistical distributions: `zipf.ref(name, s, v)`, `pareto.ref(name, alpha)`, `norm.ref(name, mean, stddev)`, `exp.ref(name, rate)`, `lognorm.ref(name, mu, sigma)`, `beta.ref(name, alpha, beta)`, `gamma.ref(name, shape, rate)`, `weibull.ref(name, shape, scale)`, `poisson.ref(name, lambda)`, `binomial.ref(name, n, p)`, `empirical.ref(name, samples)`
- `each(values)` for deterministic cycling through an inline list (no randomness). Row 0 → first item, row 1 → second, wraps via modulo. Use with `count: len(values)` for exact enum seeding: `each(['admin', 'user', 'guest'])`
- Distribution-based set functions pick from a predefined array: `set(values, weights)`, `zipf.set(values, s, v)`, `pareto.set(values, alpha)`, `norm.set(values, mean, stddev)`, `exp.set(values, rate)`, `lognorm.set(values, mu, sigma)`, `beta.set(values, alpha, beta)`, `gamma.set(values, shape, rate)`, `weibull.set(values, shape, scale)`, `poisson.set(values, lambda)`, `binomial.set(values, n, p)`, `empirical.set(values, samples)`

### Correlated totals
- `distribute_sum(total, minN, maxN, precision)` partitions a total into N random parts (comma-separated) that sum exactly to it. Use SQL `unnest`/`string_to_array` (pgx) or `JSON_TABLE` (MySQL) to expand into rows
- `distribute_weighted(total, weights, noise, precision)` splits a total by proportional weights with controlled noise (0=exact, 1=fully random). Returns comma-separated values; use `split_part` (pgx) or `SUBSTRING_INDEX` (MySQL) to extract individual parts
- These are useful for invoice/line-item patterns, budget breakdowns, and tax allocations

### PII & masking
- `gen_locale('first_name', 'ja_JP')` for locale-aware names, cities, streets, phones, zips, addresses
- `gen_locale('name', 'de_DE')` for full name in locale order (eastern = last+first, western = first last)
- Supported locales: `en_US`, `ja_JP`, `de_DE`, `fr_FR`, `es_ES`, `pt_BR`, `zh_CN`, `ko_KR` (aliases: `ja`, `de`, etc.)
- `mask(key, value)` for deterministic HMAC-keyed hex pseudonymization (16 chars default). The `key` argument is required - source it from `env('MASK_KEY')` so the mapping is stable across runs
- `mask(key, value, length)` for custom-length hex token
- `mask(key, value, 'base64')` / `mask(key, value, 'base32')` for alternative encodings
- `mask(key, value, 'asterisk')` for `****************` (length configurable)
- `mask(key, value, 'redact')` for fixed `[REDACTED]` output
- `mask(key, value, 'email')` to preserve `@domain` and mask local part: `mask(env('MASK_KEY'), arg('email'), 'email', 4)` -> `****@example.com`
- `hash(value, algo)` for a deterministic unkeyed hex digest, where `algo` is `md5` | `sha1` | `sha256` | `crc32` (lowercase hex, 32/40/64/8 chars respectively). Unlike `mask()` (HMAC-keyed pseudonymization), `hash()` takes no key, so the same input always yields the same digest across runs and machines. Use for dedup keys and shard selection, not for pseudonymization: `hash('user@example.com', 'sha256')`

### Dependent columns
- `arg(index)` to reference a previously evaluated arg by zero-based index
- `arg('name')` to reference by name when using named args (map-style `args:`)
- Bare sibling names: with named args, previously evaluated args can be referenced directly by name without `arg()`:
  ```edg
  insert_user `INSERT INTO users (first_name, last_name, full_name) VALUES ($1, $2, $3)`
    (
      first_name: gen('firstname'),
      last_name: gen('lastname'),
      full_name: first_name + " " + last_name,
    )
  ```
- If a bare name collides with a built-in function, use `arg('name')` as fallback
- `cond(predicate, trueVal, falseVal)` for conditional values
- `nullable(expr, probability)` for nullable columns
- `bool()` + `arg()` + `cond()` for mutually exclusive columns

### Named args
- Args can be a map instead of a list, giving each arg a name:
  ```edg
  insert_user `INSERT INTO users (email, region, amount, label) VALUES ($1, $2, $3, $4)`
    (
      email: gen('email'),
      region: ref_same('regions').name,
      amount: uniform.int(1, 500),
      label: email + " (" + region + ")",
    )
  ```
- Named args bind to `$1`, `$2`, etc. in declaration order
- Bare sibling names work directly: `email` instead of `arg('email')`
- Index-based `arg(0)` and explicit `arg('name')` still work
- Named and positional forms are mutually exclusive per query

### Objects (reusable arg templates)
- Define named arg templates with `object`:
  ```edg
  object order {
    email = gen('email')
    product = gen('productname')
    quantity = int(uniform.int(1, 100))
    ordered_at = timestamp('2024-01-01T00:00:00Z', '2025-01-01T00:00:00Z')
  }
  ```
- **`object:` option** expands all fields as positional args in declaration order:
  ```edg
  insert_order(object: order)
    `INSERT INTO "order" (email, product, quantity, ordered_at) VALUES ($1, $2, $3, $4)`
  ```
- Bare field names work inside objects: `total = quantity * price` instead of `field('quantity') * field('price')`
- **`field('name')`** cherry-picks fields when `object:` is set (mixable with other expressions):
  ```edg
  insert_order(object: order)
    `INSERT INTO "order" (email, product) VALUES ($1, $2)`
    (field('email'), field('product'))
  ```
- **`obj('name', 'field')`** accesses a specific field without `object:`:
  ```edg
  insert_order `INSERT INTO "order" (email, product) VALUES ($1, $2)`
    (obj('order', 'email'), obj('order', 'product'))
  ```
- **`obj('name').field`** evaluates all fields, accesses via dot notation (cached per query execution):
  ```edg
  insert_order `INSERT INTO "order" (email, product) VALUES ($1, $2)`
    (obj('order').email, obj('order').product)
  ```
- `object:` works with batch inserts + `__values__` for bulk inserts using an object template

### Print (live aggregated stats)
- The `print` option evaluates expressions each iteration and displays aggregated values:
  ```edg
  get_user(print: ref_same('regions').name)
    `SELECT * FROM users WHERE region = $1`
    (ref_same('regions').name)
  ```
- Print expressions have access to the same context as args: `ref_same`, `ref`, `arg()`, `global()`, `local()`
- The compound form takes `{key: K, value: E, agg: A, series: value, window: D}` (use `expr` instead of `value` when there's no `key`)
- `agg` is an expr expression over `count`, `freq`, `min`, `max`, `avg`, `sum` (e.g. `agg: string(int(min)) + '..' + string(int(max))`)
- `window` aggregates over a sliding window instead of the whole run (e.g. `window: 30s`). The window is a ring of buckets one `--print-interval` wide (default `1s`), so it rounds to the nearest multiple of that interval, minimum one, maximum 3600
- A query's `rate` and its `window` are independent: `rate` sets how many samples land in each bucket (line smoothness), `window` sets the time span. Neither is derived from the `stats` block rate, which only applies to queries that don't set their own `rate`
- `series: value` plots one chart line per distinct value returned, at that value's per-second rate. Only meaningful on a `stats` query with `type: line`. Values stay categorical even when numeric, so the table shows a frequency breakdown
- A value that stops appearing is plotted as zero on the chart (lifetime values decide which series exist) but drops out of the stats table entirely once it ages out of the window. Frequency rows list at most 10 values
- Only applies to `run` section queries
- The `post_print` option works like `print` but evaluates **after** query execution, giving access to `result()`
- `result()` returns the first row of a SELECT result as a map (e.g. `result().column_name`)
- Use `post_print` when you need to observe query output (balances, counts, totals) in progress output

### Batch operations
- For seed operations with large row counts, use `exec_batch` with `count` (total rows) and `size` (rows per batch)
- Use `gen_batch(total, batchSize, pattern)` for generating batched values
- Use `batch(n)` for sequential indices
- Use `ref_cursor(query, size, cursor_col, repeat?)` for keyset-paginated iteration over large tables. Pages through results using `WHERE cursor_col > last_value ORDER BY cursor_col LIMIT size`, fetching one page at a time with constant memory. Works with `__values__` for multi-row INSERTs. Ideal when `ref_each()` would load too many rows into memory (e.g. 10M+ rows). Optional `repeat` count gives exact cardinality: each row is returned N times before advancing. Examples: `ref_cursor('SELECT id FROM customer ORDER BY id', 1000, 'id')`, `ref_cursor('SELECT id FROM customer ORDER BY id', 1000, 'id', 3)` (3 children per customer)
- Use `iter()` for a 1-based row counter within batch queries (resets per query)
- Use `uniq("expression")` to retry a generator until a unique value is produced (e.g., `uniq("gen('airlineairportiata')")` for unique IATA codes). Defaults to 100 retries; override with `uniq("expression", 500)`
- For composite uniqueness across columns, pass multiple expressions: `uniq("gen('firstname')", "gen('lastname')")[0]` and `...[1]`. Returns `[]any`; same-row calls with identical expressions return cached tuple
- Use `uniq_across('pool_name', "expression")` for cross-query uniqueness. Unlike `uniq()` which resets between queries, `uniq_across` shares a named pool that persists across all queries. Use when values must be globally unique across tables (e.g., `uniq_across('emails', "gen('email')")` in both `users` and `admins` seeds). Supports composite tuples: `uniq_across('names', "gen('first')", "gen('last')")`
- **`__values__` token (recommended)**: Use `__values__` in the query to generate a multi-row `VALUES` clause instead of driver-specific batch expansion (`unnest`/`JSON_TABLE`/`OPENJSON`). Produces `VALUES (v1, v2), (v3, v4), ...` - one INSERT per batch. Works with `exec_batch`/`query_batch` and also with `type: exec`/`query` when using batch-expanding args (`gen_batch()`, `batch()`, `ref_each()`, `ref_cursor()`). Works with pgx, mysql, mssql, spanner, sqlite, dsql. For Oracle, use `__values__(table(col1, col2))` to generate `INSERT ALL INTO table (cols) VALUES (...) ... SELECT 1 FROM DUAL`. Does not work with MongoDB, Cassandra, or Redis. Also supports upsert (`ON CONFLICT`/`ON DUPLICATE KEY`/`MERGE`) and update via CTE

### Transactions
- Group related `run` queries into an explicit `BEGIN/COMMIT` block using `transaction`:
  ```edg
  run {
    transaction transfer {
      let amount = gen('number:1,100')
      read_source `SELECT id, balance FROM account WHERE id = $1`
        (ref_diff('fetch_accounts').id)
      debit_source `UPDATE accounts SET balance = balance - $2 WHERE id = $1`
        (ref_same('read_source').id, local('amount'))
      credit_target `UPDATE accounts SET balance = balance + $2 WHERE id = $1`
        (ref_same('fetch_accounts').id, local('amount'))
    }
  }
  ```
- Use `let` inside transactions for scoped variables evaluated once at transaction start, accessible via `local('name')`
- Use `rollback_if` between queries for conditional early rollback:
  ```edg
  rollback_if ref_same('read_source').balance < local('amount')
  ```
- `rollback_if` must evaluate to a boolean
- Local names must not collide with query names in the same transaction

### Query-scoped locals
- `let` bindings can appear on standalone queries (outside transactions), placed between the SQL template and args
- Evaluated per row before args; available to subsequent bindings and args via `local('name')`
- Useful when a `ref`/`ref_weighted` result is needed in multiple args - bind once, access fields:
  ```edg
  populate_rides(count: 1000, size: 100)
    `INSERT INTO rides (pickup_lat, distance_km) __values__`
    let city = ref_weighted('data.cities', [556, 278, 139, 27])
    let dist = lognorm(local('city').dist_mu, local('city').dist_sigma, local('city').dist_min, local('city').dist_max)
    (
      pickup_lat: uniform.int(local('city').pickup_lat_min, local('city').pickup_lat_max),
      distance_km: local('dist'),
    )
  ```
- Bindings evaluate in declaration order, so later bindings can reference earlier ones
- Each row gets its own evaluation - `ref`/`ref_weighted` returns a fresh row per data row
- Ref objects with array properties (e.g. `hour_weights: [2, 1, 1, ...]`) can be accessed via dot notation: `local('city').hour_weights`
- Conditional rollbacks are not errors, the worker continues to the next iteration
- Multiple `rollback_if` elements can be placed at different points in the transaction
- Batch types (`exec_batch`, `query_batch`) cannot be used inside a transaction
- `prepared: true` cannot be used inside a transaction
- Transactions appear in a separate TRANSACTION stats section (with COMMITS, ROLLBACKS, ERRORS columns)
- Use `weights` to weight transactions against standalone queries (reference by transaction name)

### Conditionals (if/then/else and match/when/default)
- `if/then/else` for binary branching based on a boolean expression
- `match/when/default` for multi-way dispatch
- **Let in branches** - use `let` inside conditional branches to set variables that persist for the rest of the transaction. DSL syntax:
  ```edg
  if ref_same('read_buyer').market == 'uk' {
    let tax_rate = 0.20
    let currency = 'GBP'
  } else {
    let tax_rate = 0.10
    let currency = 'USD'
  }
  insert_order `INSERT INTO order_log (customer_id, tax, currency)
    VALUES ($1::UUID, $2::FLOAT, $3::STRING)` (local('tax_rate'), local('currency'))
  ```
- Branch locals are cleared at the end of the transaction or run iteration
- Both work inside transactions and as standalone run items
- `else` and `default` are optional
- Special naked entries: `noop` (do nothing) and `rollback` (roll back transaction, only inside transactions)
- Conditionals can be nested
- The `if` condition must evaluate to boolean; `match` and `eq` values are compared as strings
- Standalone conditionals cannot be used with `weights`
- Example files: `examples/conditional_if/`, `examples/conditional_match/`

### Workers
- Use the `workers` section for background maintenance queries that run on a fixed schedule alongside the main workload
- Each worker is a regular query with either a `rate` option (recurring) or a `delay` option (one-shot)
- Rate format is `times/interval` (e.g. `1/10s` = once every 10 seconds, `3/1m` = 3 times per minute)
- Executions are evenly spaced: `3/1m` fires every 20 seconds
- A worker with `delay` instead of `rate` executes once after the specified duration, then stops. Useful for mid-run schema changes or one-shot maintenance
- A worker must specify either `rate` or `delay`, not both
- Workers support all query options: `type`, `args`, `prepared`, `object`, `ignore`, `request_timeout`, etc.
- Each worker runs in its own goroutine with its own environment
- Worker results appear in stats, Prometheus metrics, and expectations (unless `ignore: true`)
- In staged mode, workers run for the entire duration across all stages
- Example use cases: lease reapers, stats refreshers, cache warmers, periodic cleanup, mid-run schema changes
  ```edg
  workers {
    reap_expired_leases(rate: 1/5s) `UPDATE runs
      SET status = 'pending', worker_id = NULL
      WHERE status = 'claimed' AND lease_expires_at < now()`

    refresh_counts(rate: 3/1m) `SELECT count(*) AS total FROM events`

    add_index(delay: 30s) `CREATE INDEX IF NOT EXISTS idx_status ON orders (status)`
  }
  ```

### Stats
- Use the `stats` section for periodic observability queries that run at a fixed rate alongside the main workload
- The block takes a required `rate` option: `stats (rate: 1/5s) { ... }`
- Queries sharing the block rate run **sequentially** in a single goroutine (one tick = all queries in order)
- Individual queries can override with their own `rate`: `slow_check(rate: 1/30s) \`SQL\``
- Queries with a per-query rate get their own goroutine and ticker
- By default stats queries are hidden from the progress table and TUI table/graph (`include: false`). Set `include: true` to show them. Print Values (from `post_print`) always appear regardless.
- Stats queries support all query options including `post_print`, `print`, `type`, `ignore`, etc.
- In staged mode, stats run for the entire duration across all stages
- Use stats instead of putting observability queries in `run` with weights - stats don't compete with workload queries
- `type: bar` on a stats query renders the print values as a live frequency distribution in the TUI; `type: line` (default) plots them over time
- To chart how a categorical mix shifts over time (rolling upgrades, node drains), use `type: line` with `series: value` and poll fast:
  ```edg
  stats (rate: 20 / 1s) {
    node (type: line) (
      post_print: { key: 'node', value: result().node_id, series: value, window: 5s }
    ) `SHOW node_id`
  }
  ```
  ```edg
  stats (rate: 1/5s) {
    read_stats(
      post_print: { key: 'total_mb',    value: to_mb(result().total_mb) },
      post_print: { key: 'old_mvcc_mb', value: to_mb(result().old_mvcc_mb) }
    ) `SELECT
        round(sum(range_size_mb)::DECIMAL, 2) AS total_mb,
        round(sum(range_size_mb - ((span_stats->>'live_bytes')::DECIMAL / 1048576))::DECIMAL, 2) AS old_mvcc_mb
      FROM [SHOW RANGES WITH TABLES, DETAILS]
      GROUP BY table_name
      ORDER BY old_mvcc_mb DESC`
  }
  ```

### Hooks
- Use the `hooks` section for event-driven listeners that run alongside the main workload
- Each hook is a named handler that listens for events (HTTP requests or Kafka messages) and runs queries when events arrive
- Two hook types: `kafka` (consumer) and `http` (endpoint)
- Each hook gets its own goroutine and environment (like workers)
- Hook results appear in stats, Prometheus metrics, and expectations
- In staged mode, hooks run for the entire duration across all stages

**Body parsing:**
- `parse_body: "json"` auto-parses the JSON body into a map. Parsed fields are accessible via `hook('field_name')` in query args
- `__meta__` is available in query arg expressions for transport metadata:
  - **Kafka**: `key`, `topic`, `partition`, `offset`, `headers` (map)
  - **HTTP**: `method`, `path`, `headers` (map)

**Key function:**
- `hook('name')` - access a parsed body field by name in query args

**Example:**
```edg
hooks {
  orders(type: kafka, brokers: "127.0.0.1:9092", topic: "orders", group: "edg-orders", parse_body: "json") {
    insert_order `INSERT INTO orders (id, amount) VALUES ($1, $2)`
      (hook('order_id'), hook('amount'))
  }

  payments(type: http, addr: "0.0.0.0:3030", method: "POST", path: "/payments", parse_body: "json") {
    insert_payment `INSERT INTO payments (id, amount) VALUES ($1, $2)`
      (hook('payment_id'), hook('amount'))
  }
}
```

**Validation requirements:**
- `type` must be `kafka` or `http`
- `parse_body` is required (currently only `json` is supported)
- Kafka hooks require `brokers`, `topic`, and `group`
- HTTP hooks require `addr`, `method`, and `path`
- All hooks require at least one query
- Multiple HTTP hooks each get their own server on different `addr` values

### Stages
- Define sequential workload phases with different worker counts and durations:
  ```edg
  stages {
    ramp(workers: 1, duration: 10s, weights: { check_balance: 90, credit_account: 5, make_transfer: 5 })
    steady(workers: 10, duration: 30s, qps: ramp(10, 100, 10s))
  }
  ```
- Each stage is named by its identifier and takes `workers` and `duration`, plus optional `ramp_duration`, `qps`, and `weights`
- `qps` accepts a static integer (e.g. `qps: 1000`) or `ramp(start, end, duration)` to linearly increase QPS over time (e.g. `qps: ramp(10, 1000, 60s)` ramps from 10 to 1,000 QPS over 60 seconds)
- `ramp_duration` controls worker staggering: for stages without `qps`, workers are spawned incrementally over the ramp period; for stages with a static `qps`, all workers start immediately and QPS increases linearly. `ramp_duration` is independent of `qps: ramp()`. Must be less than `duration`
- When a stage defines `weights`, workers in that stage use those weights instead of the top-level `weights` block
- Workers (background queries) run for the entire duration across all stages, unaffected by per-stage weights
- When `stages` is defined, the `-w` and `-d` CLI flags are ignored

### Temporal patterns
- Use `global_iter()` with math functions and globals to make generated data change shape over a workload's lifetime
- Estimate total iterations: `total_iterations = workers * duration_seconds / avg_latency_seconds`
- Define `total_iters` (or similar) as a global so expressions can normalize `global_iter()` to a 0–1 progress ratio
- Common patterns:
  - **Zipf skew drift**: `zipf(initial_skew + (final_skew - initial_skew) * global_iter() / total_iters, 1, max)`
  - **Logarithmic growth**: `floor(base * (1.0 + log(1.0 + global_iter() / 1000.0)) * 100.0) / 100.0`
  - **Sine wave seasonality**: `floor(abs(base + 0.5 * sqrt(global_iter()) + amplitude * sin(2.0 * pi * global_iter() / period)))`
  - **Periodic spikes**: `pow(cos(pi * mod(global_iter(), interval) / interval), 2.0) * scale`
  - **Bounded drift (arctan saturation)**: `base + (2.0 * atan(sqrt(global_iter()) / 100.0) / pi) * max_drift + noise`
- For periodic patterns, set the period relative to estimated total iterations (e.g., `period = total_iterations / 2` for 2 visible cycles)

### Ignore
- Setting `ignore: true` on a query, transaction, or worker hides it from progress output, summary table, Prometheus metrics, and expectations
- The query still executes normally; only stats collection is suppressed
- When a transaction is ignored, all its inner queries are also ignored
- Individual queries inside a non-ignored transaction can be ignored independently
- Use for helper queries whose latency is not meaningful to the benchmark (e.g. setup reads, cache refreshes)
  ```edg
  run {
    refresh_cache(ignore: true) `SELECT id, name FROM product`
  }
  ```

### Request Timeout
- Setting `request_timeout` on a query applies a per-execution timeout
- If the query exceeds the deadline, it is cancelled and counted as an error
- Overrides the `--request-timeout` CLI flag passed to the run command
  ```edg
  run {
    fast_lookup(request_timeout: 500ms)
      `SELECT * FROM users WHERE id = $1`
      (ref('fetch_users').id)
  }
  ```

### Formatting
- SQL goes in backticks - no indentation concerns
- Single-line sections are fine: `deseed { truncate_users \`TRUNCATE TABLE users CASCADE\` }`
- Use `#` for comments
- Name every query descriptively

## Example

```edg
let users = 10000
let orders = 50000
let batch_size = 1000

up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
  )`

  create_orders `CREATE TABLE IF NOT EXISTS orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
  )`
}

seed {
  seed_users(count: users, size: batch_size)
    `INSERT INTO users (id, email, name) __values__`
    (uuid_v7(), gen('email'), gen('firstname') + ' ' + gen('lastname'))

  seed_orders(count: orders, size: batch_size)
    `INSERT INTO orders (id, user_id, total, status) __values__`
    (
      uuid_v7(),
      ref('fetch_users').id,
      uniform.float(5.00, 500.00, 2),
      set(['pending', 'shipped', 'delivered', 'cancelled'], [40, 30, 25, 5])
    )
}

init {
  fetch_users `SELECT id, email FROM users`
}

run {
  get_user_orders
    `SELECT id, total, status, created_at
     FROM orders
     WHERE user_id = $1
     ORDER BY created_at DESC
     LIMIT 10`
    (ref('fetch_users').id)

  place_order
    `INSERT INTO orders (id, user_id, total, status)
     VALUES ($1, $2, $3, 'pending')`
    (uuid_v7(), ref('fetch_users').id, uniform.float(5.00, 500.00, 2))
}

weights {
  get_user_orders = 70
  place_order = 30
}

deseed {
  truncate_orders `TRUNCATE TABLE orders`
  truncate_users `TRUNCATE TABLE users`
}

down {
  drop_orders `DROP TABLE IF EXISTS orders`
  drop_users `DROP TABLE IF EXISTS users`
}
```

## Database-Specific Patterns

Apply these patterns based on the target driver.

### pgx (PostgreSQL / CockroachDB)

- **UUIDs**: Native `UUID` type with `DEFAULT gen_random_uuid()`
- **Strings**: Use `STRING` (CockroachDB) or `VARCHAR(n)` (PostgreSQL)
- **Timestamps**: `DEFAULT now()`
- **Row generation in seed**: Use `generate_series(1, $1)` for bulk generation inside SQL
- **Array columns**: Use `ARRAY[...]` type and `array(minN, maxN, pattern)` expression
- **Vector columns**: Use `VECTOR(n)` type and `vector(dims, clusters, spread)` expression for synthetic clustered vectors, or `embed(text...)` for real embeddings from an external API. `embed()` requires a license and `--embed-api-key`; dimensions must match the `VECTOR(n)` column type and `--embed-dimensions` flag. Use `--embed-max-batch` to limit texts per API call in batch queries
- **Batch expansion (unnest)**: Use `unnest(string_to_array('$1', __sep__))` to expand batch args into rows. `__sep__` is a query-text token that emits the correct SQL separator function for the target driver (`chr(31)` for pgx, `CHAR(31)` for MySQL/MSSQL, `codepoints-to-string(31)` for Oracle, `CODE_POINTS_TO_STRING([31])` for Spanner)
- **Batch expansion (__values__)**: Use `__values__` to generate a multi-row VALUES clause. Simpler than unnest and produces one INSERT per batch:
  ```sql
  INSERT INTO t (name, email) __values__
  ```
- **Batch upsert (__values__)**: Combine `__values__` with `ON CONFLICT`:
  ```sql
  INSERT INTO t (name, price) __values__
  ON CONFLICT (name) DO UPDATE SET price = EXCLUDED.price
  ```
- **Batch update (__values__)**: Use a CTE with `__values__`:
  ```sql
  UPDATE t SET price = v.price
  FROM (__values__) AS v(id, price)
  WHERE t.id = v.id::UUID
  ```
- **Upsert**: `ON CONFLICT (col) DO UPDATE SET ...`
- **Pagination**: `LIMIT $1 OFFSET $2`
- **Random ordering**: `ORDER BY random()`
- **Cleanup**: `TRUNCATE TABLE ... CASCADE`
- **DDL safety**: `CREATE TABLE IF NOT EXISTS`, `DROP TABLE IF EXISTS`

### mysql

- **UUIDs**: Use `CHAR(36)` with `DEFAULT (UUID())`
- **Strings**: `VARCHAR(n)` - always specify length
- **Timestamps**: `DEFAULT CURRENT_TIMESTAMP`
- **Row generation in seed**: Use a recursive CTE:
  ```sql
  WITH RECURSIVE seq AS (
    SELECT 1 AS s UNION ALL SELECT s + 1 FROM seq WHERE s < $1
  ) SELECT * FROM seq
  ```
- **Batch expansion (JSON_TABLE)**: Use `JSON_TABLE` to convert batch args into rows. `__sep__` emits the driver-aware separator:
  ```sql
  SELECT j.val FROM JSON_TABLE(
    CONCAT('["', REPLACE('$1', __sep__, '","'), '"]'),
    '$[*]' COLUMNS(val VARCHAR(255) PATH '$')
  ) j
  ```
- **Batch expansion (__values__)**: Use `__values__` for simpler multi-row VALUES:
  ```sql
  INSERT INTO t (name, email) __values__
  ```
- **Upsert**: `ON DUPLICATE KEY UPDATE col = VALUES(col)`
- **Categorical selection**: Use `ELT(index, 'val1', 'val2', ...)` instead of array indexing
- **Random ordering**: `ORDER BY RAND()`
- **Cleanup**: `DELETE FROM table` (preferred over TRUNCATE for FK-safe cleanup)

### mssql (SQL Server)

- **UUIDs**: `UNIQUEIDENTIFIER` with `DEFAULT NEWID()`
- **Strings**: `NVARCHAR(n)` for Unicode support, `NVARCHAR(MAX)` for unlimited
- **Timestamps**: `DATETIME2` with `DEFAULT GETDATE()`
- **Row generation in seed**: Recursive CTE with `OPTION (MAXRECURSION 0)`:
  ```sql
  WITH seq AS (
    SELECT 1 AS s UNION ALL SELECT s + 1 FROM seq WHERE s < $1
  ) SELECT * FROM seq OPTION (MAXRECURSION 0)
  ```
- **Batch expansion (OPENJSON)**: Use `batch_format: json` and `OPENJSON`:
  ```sql
  SELECT value FROM OPENJSON('$1')
  ```
- **Batch expansion (__values__)**: Use `__values__` for simpler multi-row VALUES (max 1000 rows per INSERT):
  ```sql
  INSERT INTO t (name, email) __values__
  ```
- **Upsert**: Use `MERGE INTO ... USING ... ON ... WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`
- **DDL safety**: Wrap in existence check:
  ```sql
  IF OBJECT_ID('table_name', 'U') IS NULL CREATE TABLE table_name (...)
  ```
- **Pagination**: `OFFSET @p1 ROWS FETCH NEXT @p2 ROWS ONLY`
- **Random ordering**: `ORDER BY NEWID()`
- **Cleanup**: `DELETE FROM table` (preferred)

### oracle

- **Identifiers**: `NUMBER GENERATED ALWAYS AS IDENTITY` for auto-increment, or explicit `NUMBER` type (UUID is uncommon)
- **Strings**: `VARCHAR2(n)` - Oracle-specific type
- **Timestamps**: `DEFAULT SYSTIMESTAMP`
- **Row generation in seed**: Use `CONNECT BY`:
  ```sql
  SELECT LEVEL FROM DUAL CONNECT BY LEVEL <= $1
  ```
- **Batch expansion (XMLTABLE)**: Use `XMLTABLE` to expand batch args. `__sep__` emits the driver-aware separator:
  ```sql
  SELECT column_value FROM XMLTABLE(('"' || REPLACE('$1', __sep__, '","') || '"'))
  ```
- **Batch expansion (__values__)**: Use `__values__(table(col1, col2))` for Oracle `INSERT ALL`:
  ```sql
  INSERT ALL __values__(product(name, price))
  ```
  Generates: `INTO product (name, price) VALUES (...)\nINTO product ... \nSELECT 1 FROM DUAL`
- **Upsert**: Use `MERGE INTO ... USING (SELECT :1 AS col FROM DUAL) src ON ... WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`
- **DDL safety**: Wrap in PL/SQL with exception handling:
  ```sql
  BEGIN
    EXECUTE IMMEDIATE 'CREATE TABLE ...';
  EXCEPTION WHEN OTHERS THEN
    IF SQLCODE != -955 THEN RAISE; END IF;
  END;
  ```
- **Drop safety**: `DROP TABLE ... CASCADE CONSTRAINTS PURGE`
- **Categorical selection**: Use `DECODE(index, 1, 'val1', 2, 'val2', ...)` instead of array indexing
- **Random functions**: `DBMS_RANDOM.VALUE()` for floats, `DBMS_RANDOM.STRING()` for strings
- **Pagination**: `FETCH FIRST :1 ROWS ONLY`
- **Random ordering**: `ORDER BY DBMS_RANDOM.VALUE`

### spanner (Google Cloud Spanner)

- **Types**: `INT64`, `FLOAT64`, `NUMERIC`, `STRING(n)`, `BOOL`, `TIMESTAMP`, `BYTES(n)`
- **UUIDs**: Use `STRING(36)` with `DEFAULT (GENERATE_UUID())`
- **Strings**: `STRING(n)` - always specify max length
- **Timestamps**: `TIMESTAMP` with `DEFAULT (CURRENT_TIMESTAMP())`
- **No `RAND()`**: Use `MOD(ABS(FARM_FINGERPRINT(GENERATE_UUID())), N)` for random integers
- **No `CHR()`**: Use `CODE_POINTS_TO_STRING([code_point])` instead
- **No `TRUNCATE`**: Use `DELETE FROM table WHERE TRUE` for deseed
- **No `UNNEST(...) AS v(col1, col2)`**: Column aliasing on UNNEST is unsupported. Use `__values__` instead
- **Drop indexes before tables**: Spanner requires `DROP INDEX IF EXISTS idx` before `DROP TABLE IF EXISTS t` in the `down` section
- **Strict typing with bind params**: `gen('number:...')` returns float64, which Spanner rejects for INT64 columns when using native bind params (`@pN`). Wrap in `int()`: `int(gen('number:1,100'))`
- **String bind params**: If a ref value needs STRING type for Spanner bind params, use `template('%v', value)` or inlined `'$1'` placeholders instead of `@pN`
- **Batch expansion**: Use `__values__` (recommended) or `UNNEST(SPLIT('$1', CODE_POINTS_TO_STRING([31]))) AS val`
- **Random ordering**: `TABLESAMPLE RESERVOIR (N ROWS)` or `ORDER BY FARM_FINGERPRINT(GENERATE_UUID())`
- **Upsert**: `INSERT OR UPDATE INTO t (...) VALUES (...)`
- **Ignore duplicates**: `INSERT OR IGNORE INTO t (...) VALUES (...)`
- **Pagination**: `LIMIT @p1 OFFSET @p2`
- **DDL safety**: `CREATE TABLE IF NOT EXISTS`, `DROP TABLE IF EXISTS`

### dsql (Aurora DSQL)

- Follows the same patterns as `pgx` (uses PostgreSQL wire protocol)
- Note: Some CockroachDB-specific SQL (e.g., `STRING` type) may not be available; prefer standard PostgreSQL types

### mongodb

MongoDB uses BSON/JSON command syntax instead of SQL. Queries are JSON objects specifying the command and its parameters.

- **Collections (not tables)**: Use `{"create": "name"}` to create, `{"drop": "name"}` to drop
- **Inserts**: `{"insert": "collection", "documents": [{"_id": $1, "field": $2}]}`
- **Reads**: `{"find": "collection", "filter": {}}` or `{"find": "collection", "filter": {"field": $1}}`
- **Deletes**: `{"delete": "collection", "deletes": [{"q": {}, "limit": 0}]}`
- **Updates**: `{"update": "collection", "updates": [{"q": {"_id": $1}, "u": {"$set": {"field": $2}}}]}`
- **Placeholders**: `$1`, `$2`, etc. are inlined directly into the JSON command text
- **No DDL types**: MongoDB is schemaless; `up` creates collections, `down` drops them
- **Batch inserts**: Use `exec_batch` with `count`/`size`; each batch inserts one document per execution
- **Reference data**: Use `{"find": "collection", "filter": {}}` in `seed` or `init` with `type: query` to populate datasets
- **ObjectIDs**: Use `objectid()` to generate MongoDB ObjectIDs. Format as `{"$oid": "$1"}` in JSON commands
- **Transactions**: edg supports `transaction` blocks for MongoDB using multi-document sessions. Commands run within a session context and are committed or rolled back atomically. Use the same `transaction` / `let` / `rollback_if` syntax as SQL drivers
- **Transaction-safe counting**: The `count` command and `$count` aggregation stage cannot be used inside multi-document transactions. Use `$group` with `$cond` instead - it always returns a document even when no rows match:
  ```json
  {"aggregate": "coll", "pipeline": [{"$group": {"_id": null, "n": {"$sum": {"$cond": [{"$eq": ["$field", true]}, 1, 0]}}}}], "cursor": {}}
  ```
- **Consistency tuning**: MongoDB tuning is done via URI parameters in `--url`, not dedicated CLI flags. Append `?w=majority` for write concern, `?readConcernLevel=majority` for read concern, and `?readPreference=secondaryPreferred` for read routing. Use `--retries 3` to handle transient `WriteConflict` errors under contention

Example:
```edg
up {
  create_users `{"create": "users"}`
}

seed {
  insert_users(count: 1000)
    `{"insert": "users", "documents": [{"_id": $1, "email": $2}]}`
    (gen('uuid'), gen('email'))

  fetch_users `{"find": "users", "filter": {}}`
}

init {
  load_users `{"find": "users", "filter": {}}`
}

run {
  get_user `{"find": "users", "filter": {"_id": $1}}`
    (ref('load_users')._id)
}

deseed {
  delete_users `{"delete": "users", "deletes": [{"q": {}, "limit": 0}]}`
}

down {
  drop_users `{"drop": "users"}`
}
```

### cassandra

Cassandra uses CQL (Cassandra Query Language). Tables must live inside a keyspace.

- **Keyspaces**: `CREATE KEYSPACE IF NOT EXISTS ks WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1}`
- **Tables**: `CREATE TABLE IF NOT EXISTS ks.table (id UUID PRIMARY KEY, ...)`
- **Column types**: `UUID`, `TEXT`, `INT`, `DOUBLE`, `TIMESTAMP`, `BOOLEAN`, `BLOB`
- **No `DEFAULT` values**: Generate all values in args (e.g., `gen('uuid')` for UUIDs)
- **Inserts**: Standard `INSERT INTO ks.table (col1, col2) VALUES ($1, $2)`
- **Reads**: `SELECT col1, col2 FROM ks.table` or `SELECT ... WHERE partition_key = $1`
- **Batch inserts**: Use `exec_batch` with `count`/`size`; edg uses Cassandra's unlogged batch internally
- **Cleanup**: `TRUNCATE ks.table` for deseed
- **Teardown**: `DROP TABLE IF EXISTS ks.table` then `DROP KEYSPACE IF EXISTS ks`
- **Placeholders**: Use `$1`, `$2`, etc.; edg converts to `?` automatically
- **Transactions**: edg supports `transaction` blocks for Cassandra using logged batches. Reads execute immediately; writes are buffered and committed atomically. Use the same `transaction` / `let` / `rollback_if` syntax as SQL drivers
- **Multi-host URLs**: Comma-separated hosts in the URL: `cassandra://user:pass@host1,host2,host3:9042/keyspace`. Port and auth apply to all hosts
- **Consistency tuning**: Use `--cassandra-default-consistency` to set read/write consistency (`one`, `quorum`, `all`, `local_quorum`, etc.). Default is `quorum`
- **Serial consistency**: Use `--cassandra-serial-consistency` for lightweight transactions (LWT). Values: `serial` (global Paxos) or `local_serial` (local DC only, default)
- **Idempotent queries**: Use `--cassandra-idempotent` to enable speculative execution and retry on other nodes. Safe for reads and repeatable writes
- **No discovery**: Use `--cassandra-no-discovery` to skip `system.peers` lookup and connect only to seed hosts. Useful behind load balancers or for faster startup
- **LWT workloads**: Require `--no-atomic-tx` since Cassandra does not support `IF` conditions inside batches

Example:
```edg
up {
  create_keyspace `CREATE KEYSPACE IF NOT EXISTS edg
    WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1}`

  create_users `CREATE TABLE IF NOT EXISTS edg.users (
    id UUID PRIMARY KEY,
    email TEXT
  )`
}

seed {
  insert_users(count: 1000)
    `INSERT INTO edg.users (id, email) VALUES ($1, $2)`
    (gen('uuid'), gen('email'))

  fetch_users `SELECT id, email FROM edg.users`
}

init {
  load_users `SELECT id FROM edg.users`
}

run {
  get_user `SELECT id, email FROM edg.users WHERE id = $1`
    (ref('load_users').id)
}

deseed { truncate_users `TRUNCATE edg.users` }

down {
  drop_users `DROP TABLE IF EXISTS edg.users`
  drop_keyspace `DROP KEYSPACE IF EXISTS edg`
}
```

### sqlite

SQLite is embedded, so the `--url` is a file path or `file:` URI rather than a host. edg passes it to the driver unchanged.

- **Placeholders**: `?` (positional). Note `?` cannot be repeated; if a value is needed twice, add it twice to the args
- **Batch inserts**: `__columns__`/`__values__` work as they do on Postgres
- **No `TRUNCATE`**: use `DELETE FROM t` for deseed
- **No inline indexes**: `CREATE INDEX IF NOT EXISTS idx ON t (col)` as its own `up` query
- **Type affinity**: `TEXT` for UUIDs, `INTEGER`, `REAL`, `BLOB`, `TIMESTAMP`. Any type name is accepted; affinity is derived from it
- **Single writer**: SQLite serialises writers. Multi-worker write workloads need `?_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)&_txlock=immediate` in the URL, or they fail with `database is locked (5) (SQLITE_BUSY)`. `_txlock=immediate` matters for any transaction that reads before it writes: a deferred transaction fails its lock upgrade with `database is locked (517)`, which `busy_timeout` does not retry. `--pool-size 1` also works, at the cost of concurrency
- **`:memory:`**: each connection gets its own private database, so pair it with `--pool-size 1`
- **Function equivalents**: `UUID()` -> `lower(hex(randomblob(16)))`, `RAND()` -> `abs(random()) % 1000000 / 1000000.0`, `FLOOR(RAND()*N)` -> `abs(random()) % N`, `MD5(RAND())` -> `hex(randomblob(16))`, `CONCAT(a,b)` -> `a || b`, `SUBSTRING` -> `substr`, `IF(c,a,b)` -> `iif(c,a,b)`, `ELT` -> `CASE`, `NOW()` -> `datetime('now')`, `MOD(a,b)` -> `a % b`, `DATE_ADD(d, INTERVAL n DAY)` -> `date(d, '+n days')`, `ORDER BY RAND()` -> `ORDER BY random()`, `CAST(x AS SIGNED)` -> `CAST(x AS INTEGER)`
- **Upsert**: `INSERT ... ON CONFLICT(col) DO UPDATE SET x = excluded.x`; `INSERT OR IGNORE` to skip duplicates
- **Integer division**: `/` truncates on integers. Divide by `100.0` rather than `100` when a fraction is wanted
- **No `FOR UPDATE`**: writers already serialise, so drop it

Example:
```edg
up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    email TEXT NOT NULL
  )`

  create_users_email_idx `CREATE INDEX IF NOT EXISTS idx_users_email ON users (email)`
}

seed {
  insert_users(type: exec_batch, count: 1000, size: 100)
    `INSERT INTO users (id, email) __values__` (gen('uuid'), gen('email'))
}

init {
  load_users `SELECT id FROM users`
}

run {
  get_user `SELECT id, email FROM users WHERE id = ?` (ref('load_users').id)
}

deseed { clean_users `DELETE FROM users` }

down { drop_users `DROP TABLE IF EXISTS users` }
```

### redis

Redis queries are commands, written as you'd type them into `redis-cli`. There is no schema and no SQL.

- **Placeholders**: `$1`, `$2`, ... inlined into the command text, so the same placeholder can be repeated
- **Quoting**: arguments containing spaces can be wrapped in `"` or `'`; quotes are stripped
- **Type inference**: no Redis command matches edg's verb inference, so commands default to `query`. Mark writes `type: exec` to skip reading a reply
- **Replies as rows**: a hash is one row of fields, an array is one row per element under a `value` column, a scalar is a single `value`. A missing key yields no rows rather than an error
- **`up`**: `PING` - there is no DDL, but it keeps the lifecycle uniform
- **Reads for `init`**: no `SELECT`, so use a cursored `SCAN` wrapped in `EVAL`. Never `KEYS`, which blocks the server for the whole sweep. Refer to the results as `ref('fetch_x').value`
- **Cleanup**: `SCAN` + `UNLINK` in a Lua script for deseed/down. `UNLINK` frees keys on a background thread; `DEL` blocks
- **Batches**: `exec_batch` sends commands in a single non-transactional pipeline, applied independently
- **Transactions**: `transaction` blocks use `MULTI`/`EXEC`. Writes queue until commit, reads run immediately, so a transaction cannot read its own uncommitted writes. `rollback_if` issues `DISCARD`, which cannot undo reads that already ran
- **No prepared statements**: `prepared: true` errors with `driver does not support prepared statements`
- **Numbers**: `uniform.int` returns a float. Wrap in `int()` where Redis requires an integer (`HINCRBY`, `EXPIRE`, `SETRANGE`)
- **URLs**: `redis://user:pass@host:6379/0`. A bare `host:port` is accepted and prefixed automatically
- **Built-in workloads**: only `kv` and `ycsb` ship a Redis config

Example:
```edg
up {
  create(type: exec) `PING`
}

seed {
  insert_users(type: exec_batch, count: 1000, size: 100)
    `HSET user:$1 email $2 visits $3` (seq(1, 1), gen('email'), int(uniform.int(1, 100)))
}

init {
  load_users(type: query)
    `EVAL "local c=0 local out={} repeat local r=redis.call('SCAN',c,'MATCH',ARGV[1],'COUNT',1000) c=tonumber(r[1]) for _,k in ipairs(r[2]) do out[#out+1]=k end until c==0 return out" 0 user:*`
}

run {
  get_user(type: query) `HGETALL ${ref('load_users').value}`

  bump_visits(type: exec) `HINCRBY user:$1 visits 1` (int(uniform.int(1, 1000)))
}

deseed {
  delete_users(type: exec)
    `EVAL "local c=0 repeat local r=redis.call('SCAN',c,'MATCH',ARGV[1],'COUNT',1000) c=tonumber(r[1]) for _,k in ipairs(r[2]) do redis.call('UNLINK',k) end until c==0 return 1" 0 user:*`
}
```

## Generating from an existing database

If the user already has a database with tables, suggest `edg init` to generate a starting config:

```sh
# PostgreSQL / CockroachDB
edg init --driver pgx --url "postgres://..." --schema public > workload.edg

# MySQL (use --database for the database name)
edg init --driver mysql --url "root:pass@tcp(localhost:3306)/dbname?parseTime=true" --database dbname > workload.edg

# MSSQL
edg init --driver mssql --url "sqlserver://..." --schema dbo > workload.edg

# Oracle
edg init --driver oracle --url "oracle://..." --schema SYSTEM > workload.edg

# Aurora DSQL
edg init --driver dsql --url "clusterid.dsql.us-east-1.on.aws" --schema public > workload.edg

# SQLite (the attached schema is "main" unless the caller has ATTACHed others)
edg init --driver sqlite --url "file:edg.db" --database main > workload.edg
```

The `--schema` and `--database` flags are interchangeable. Use `--schema` for drivers where the value is a schema name (pgx, dsql, mssql, oracle) and `--database` for drivers where it's a database name (mysql, sqlite).

Redis has no schema to inspect, so `edg init` doesn't support it. Use `edg scaffold` instead, which generates a config from a table/column description rather than a live database.

The generated config is a starting point, seed expressions will match column types and constraints but won't produce realistic data. The user should refine the config after generation.

## Capturing from real query statistics

If the user wants to generate a workload config that approximates a real production workload, suggest `edg capture`. It reads query statistics from built-in stats views (`pg_stat_statements` for PostgreSQL, `crdb_internal.statement_statistics` for CockroachDB, `performance_schema.events_statements_summary_by_digest` for MySQL) and produces a config with `run`, `weights`, and `stages` sections. Requires a pro license.

```sh
# CockroachDB
edg capture \
  --driver pgx \
  --flavour cockroachdb \
  --url "postgres://root@localhost:26257/movr?sslmode=disable" \
  --name workload

# PostgreSQL (requires pg_stat_statements extension)
edg capture \
  --driver pgx \
  --flavour postgres \
  --url "postgres://user:pass@localhost:5432/mydb?sslmode=disable" \
  --name workload
```

The `--name` flag is required and specifies the output file name without extension. Capture writes `<name>.edg`.

### Capture flags

| Flag | Required | Default | Description |
|---|---|---|---|
| `--flavour` | Yes | | Database flavour: `postgres`, `cockroachdb`, or `mysql` |
| `--name` | Yes | | Output file name without extension (writes `.edg`) |
| `--min-calls` | No | `10` | Minimum call count to include a query |
| `--top` | No | `50` | Maximum number of queries to include |
| `--duration` | No | `1h` | Assumed observation window for think time calculation |
| `--workers` | No | `10` | Number of workers in the generated config |
| `--schema` | No | | Schema or database name to introspect for `up`/`down` DDL |
| `--database` | No | | Alias for `--schema` |

### Schema introspection with capture

When `--schema` or `--database` is provided, capture also inspects the target schema and adds `up` (CREATE TABLE) and `down` (DROP TABLE IF EXISTS) sections to the output. This uses the same introspection logic as `edg init`, reading the database's own DDL and sorting tables by foreign key dependencies. This produces a self-contained config that can create the schema, run the workload, and tear it down.

```sh
edg capture \
  --driver pgx \
  --flavour cockroachdb \
  --url "postgres://root@localhost:26257/movr?sslmode=disable" \
  --schema public \
  --name workload
```

Without `--schema`, the output contains only `stages`, `weights`, and `run` - suitable for replaying against an existing database.

### Captured config notes

- Query parameters are set to `TODO` placeholders. The user must replace these with appropriate expressions (e.g., `ref('fetch_users').id`).
- `weights` are proportional to observed call counts.
- Per-query `wait` durations are derived from inter-arrival times.
- A single `stages` entry is generated matching the observation window and worker count.

## Common Mistakes

Do NOT generate configs with these errors:

| Mistake | Why it breaks | Fix |
|---|---|---|
| SELECT in `init` without explicit type | Type inferred as `exec` from context, won't populate named dataset | Ensure init SELECT queries populate datasets correctly |
| `ref('x')` before dataset `x` is populated | No `init` or `seed` query named `x` exists | Add an `init` or `seed` query that populates `x` |
| Batch types inside `transaction` | Not supported - batch types cannot be used in transactions | Use regular queries inside transactions |
| `prepared: true` inside `transaction` | Not supported | Remove `prepared: true` from transaction queries |
| `gen('number:1,100')` for Spanner INT64 columns | Returns float64, Spanner rejects for INT64 bind params | Wrap in `int()`: `int(gen('number:1,100'))` |
| Mixing named and positional args in one query | Mutually exclusive - use map-style OR list-style, not both | Pick one form per query |
| `seq_global("name")` without a `seq` declaration | Sequence doesn't exist at runtime | Add a top-level `seq name(start: 1, step: 1)` declaration |
| `count`/`size` on non-batch query types | Only valid for batch operations | Add batch count/size as query options |
| `__values__` with MongoDB, Cassandra, or Redis | Only works with SQL drivers (pgx, mysql, mssql, oracle, spanner, sqlite, dsql) | Use single-document inserts for MongoDB, standard CQL for Cassandra, one command per row for Redis |
| `weights` referencing nonexistent run item | Key must match a query name in `run` section | Fix name or add matching run query |
| `rollback` outside a transaction | `rollback` only valid inside transactions | Use `noop` or remove the entry |
| Standalone `if`/`match` with `weights` | Conditional blocks can't be weighted | Remove conditionals from weighted run or remove `weights` |
| Missing backticks around SQL | Parser expects backtick-delimited SQL | Wrap SQL in backticks |
| Args `(...)` on a new line after SQL backtick | Parser expects identifier, sees `(` - treats args as a new (invalid) query | Put opening `(` on the same line as the closing backtick: `` \`SQL\` ( `` then args can wrap to next lines |

## Staging (file output without a database)

The `edg stage` command generates data to files instead of executing against a database. No `--url` or database connection is required. This is useful for previewing generated data, loading into external tools, or generating migration scripts.

```sh
edg stage --config <path> --format <format> --output-dir <dir>
```

| Flag | Short | Default | Description |
|---|---|---|---|
| `--format` | `-f` | `sql` | Output format: `sql`, `json`, `csv`, `parquet`, or `stdout` |
| `--output-dir` | `-o` | `.` | Directory for output files (created if it doesn't exist) |

### Output formats

| Format | File naming | Description |
|---|---|---|
| `sql` | `{section}.sql` | Executable SQL statements (DDL + one resolved statement per generated row) |
| `json` | `{section}.json` | Objects keyed by query name (data-generating queries only) |
| `csv` | `{section}_{query}.csv` | CSV with headers per data-generating query |
| `parquet` | `{section}_{query}.parquet` | Apache Parquet per data-generating query (all columns as optional byte arrays) |
| `stdout` | *(none)* | Streams resolved SQL to stdout (no files written, log output suppressed) |

### Key behaviours

- Batch queries (`exec_batch`) are expanded into individual rows. The batch CSV-joining logic is bypassed
- Referential integrity is preserved: generated data from earlier queries is stored in memory for `ref`, `ref_each`, `ref_cursor`, `seq_rand`, etc.
- The `--driver` flag still controls SQL value formatting (quote style, hex literals)
- The `--rng-seed` flag produces deterministic, reproducible output
- Column names come from: named args > INSERT column list > fallback `col_1`, `col_2`, etc.
- DDL/DML without args (CREATE, DROP, DELETE, TRUNCATE) is included in SQL output but skipped in JSON/CSV/Parquet

## Sync Configs (Cross-Database Consistency)

When the user wants to test dual-write consistency, CDC pipelines, or cross-database replication, generate **paired configs** - one per database driver - for use with `edg sync run`. Note: `edg sync` commands (run, down, verify) require a license.

### Requirements for sync-compatible configs

- **Explicit IDs**: Use `INT PRIMARY KEY` with `seq(1, 1)` instead of auto-generated IDs (`SERIAL`, `AUTO_INCREMENT`, `UUID`). Both databases must produce identical IDs.
- **Matching schemas**: Same table names, column names, and logical types. SQL dialect differs (e.g., `DEFAULT NOW()` vs `DEFAULT CURRENT_TIMESTAMP`).
- **Matching seed args**: Both configs must use identical `args` expressions (same `gen()`, `ref()`, `uniform.int()`, `set()`, etc.). The `--rng-seed` flag + PRNG re-seeding ensures identical values. Fetched datasets (`ref`) are sorted deterministically before use, so different database row orderings won't cause divergence.
- **Use `exec_batch`**: Sync configs should use `type: exec_batch` with `count` and `size` for efficient bulk inserts.
- **Batch SQL**: Use `__values__` for a cross-driver multi-row VALUES clause (works with pgx, mysql, mssql, spanner, dsql, and oracle via `__values__(table(cols))`). Also works with `type: exec` + `gen_batch()`/`batch()`/`ref_each()`.
- **Precision for floats**: When comparing across SQL and NoSQL databases, use `uniform.float(min, max, precision)` to generate floats with fixed decimal places. This avoids false mismatches from floating-point representation differences (e.g. `364.8` vs `364.80`).
- **No `run` section**: Sync configs only need `up`, `seed`, `deseed`, and `down`. The benchmark workload is separate.
- **Shared globals**: Both configs should use the same `let` values (row counts, batch sizes).
- **Cassandra batch size**: Cassandra rejects large batches. Use `size: 50` or similar in the config to keep batches small.

### Example

For a CockroachDB + MySQL sync pair, generate two files:

**crdb.edg:**
```edg
let user_count = 1000
let batch_size = 100

up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
  )`
}

seed {
  seed_users(count: user_count, size: batch_size)
    `INSERT INTO users (id, email, name) __values__`
    (seq(1, 1), gen('email'), gen('name'))
}
```

**mysql.edg:**
```edg
let user_count = 1000
let batch_size = 100

up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
  )`
}

seed {
  seed_users(count: user_count, size: batch_size)
    `INSERT INTO users (id, email, name) __values__`
    (seq(1, 1), gen('email'), gen('name'))
}
```

### Usage

After generating paired configs, show the user how to run them:

```sh
edg sync run \
  --source-driver pgx --source-url "postgres://..." --source-config crdb.edg \
  --target-driver mysql --target-url "root:pass@tcp(...)/?parseTime=true" --target-config mysql.edg \
  --rng-seed 42

edg sync verify \
  --source-driver pgx --source-url "postgres://..." \
  --target-driver mysql --target-url "root:pass@tcp(...)/?parseTime=true" \
  --tables users --order-by id --ignore-columns created_at
```

Add `--verbose` to `sync verify` to print individual row-level mismatches. Without it, only the per-table summary is shown.

`sync verify` supports all drivers except Redis, which has no ordered scan to page through. For Cassandra, use `127.0.0.1` instead of `localhost` in the URL to avoid IPv6 connection warnings.

For CDC mode (source only, replication handles target), omit `--target-config` from `sync run`.

## Compare Configs (Cross-Database Performance)

When the user wants to benchmark the same workload against two databases side-by-side, generate **paired configs** - one per database driver - for use with `edg compare run`. Note: `edg compare` commands require a license.

### Requirements for compare-compatible configs

- **Matching schemas**: Same table names, column names, and logical types. SQL dialect differs per driver.
- **Matching seed args**: Both configs should use identical `args` expressions for deterministic seeding with `--rng-seed`.
- **Include a `run` section**: Unlike sync configs, compare configs must include `run` queries that define the workload to benchmark. Both configs should have the same logical queries adapted for each driver's SQL dialect.
- **Same query names**: Use identical query names in both configs so the TUI can show side-by-side comparisons per query.
- **Weight parity**: If using `weights`, use the same weights in both configs so the workload mix is comparable.
- **Use `exec_batch`**: Seed sections should use `type: exec_batch` with `count` and `size` for efficient bulk inserts.
- **Batch SQL**: Use `__values__` for cross-driver multi-row VALUES clauses.

### Example

For a Postgres + MySQL comparison, generate two files:

**postgres.edg:**
```edg
let user_count = 1000
let batch_size = 100

up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
  )`
}

seed {
  seed_users(count: user_count, size: batch_size)
    `INSERT INTO users (id, email, name) __values__`
    (seq(1, 1), gen('email'), gen('name'))
}

run {
  lookup_user
    `SELECT id, email, name FROM users WHERE id = $1`
    (uniform_i(1, user_count))

  insert_user
    `INSERT INTO users (id, email, name) VALUES ($1, $2, $3)
     ON CONFLICT (id) DO NOTHING`
    (gen('number:100000,999999'), gen('email'), gen('name'))
}

weights {
  lookup_user = 80
  insert_user = 20
}
```

**mysql.edg:**
```edg
let user_count = 1000
let batch_size = 100

up {
  create_users `CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
  )`
}

seed {
  seed_users(count: user_count, size: batch_size)
    `INSERT INTO users (id, email, name) __values__`
    (seq(1, 1), gen('email'), gen('name'))
}

run {
  lookup_user
    `SELECT id, email, name FROM users WHERE id = $1`
    (uniform_i(1, user_count))

  insert_user
    `INSERT INTO users (id, email, name) VALUES ($1, $2, $3)
     ON DUPLICATE KEY UPDATE email = VALUES(email)`
    (gen('number:100000,999999'), gen('email'), gen('name'))
}

weights {
  lookup_user = 80
  insert_user = 20
}
```

### Usage

After generating paired configs, show the user how to run them:

```sh
edg compare all \
  --a-driver pgx --a-url "postgres://root:password@localhost:5432/defaultdb?sslmode=disable" --a-config postgres.edg \
  --b-driver mysql --b-url "root:password@tcp(localhost:3306)/defaultdb?parseTime=true" --b-config mysql.edg \
  --rng-seed 42 -w 4 -d 30s --tui
```

Or run each phase separately with `edg compare up`, `edg compare seed`, `edg compare run`, `edg compare deseed`, `edg compare down`.

Add `--tui` for an interactive side-by-side TUI with merged tables and overlaid latency graphs. Without `--tui`, results print to stdout with delta percentages at the end.

## Cluster Configs (Distributed Runners)

When the user wants to run workloads across multiple agents (pods, nodes, clouds, or regions) with coordinated stages and global QPS control, generate a config for use with `edg cluster`. Requires a pro license and a CockroachDB instance for coordination.

### How cluster configs differ from regular configs

- **Same config format**: Cluster configs use the same edg-lang format as regular configs. The coordinator distributes the config blob to agents.
- **Stages drive coordination**: When stages are defined, the coordinator orchestrates stage transitions across all agents.
- **QPS is global**: A stage's `qps` value is the total across all agents. The coordinator divides it evenly: `per_agent_qps = ceil(total_qps / active_agent_count)`.
- **Workers are per-agent**: The `workers` value in each stage is the number of concurrent workers *on each agent*, not the global total.
- **Region labels**: Each agent has a `--region` flag. All Prometheus metrics include a `region` label for per-region dashboards.

### Usage

```sh
# Start coordinator
edg cluster coordinator --crdb-url "postgres://root@localhost:26257/edg_cluster?sslmode=disable"

# Start agents in different regions
edg cluster agent --crdb-url "..." --region us-east
edg cluster agent --crdb-url "..." --region eu-west

# Submit workload
edg cluster submit \
  --target-url "postgres://root@localhost:26257?sslmode=disable" \
  --config workload.edg \
  --duration 5m \
  --workers 10 \
  --regions us-east,eu-west \
  --stream
```

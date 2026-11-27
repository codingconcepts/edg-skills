---
name: edg-expression
description: Help compose edg expressions. Explain functions, debug syntax, and generate the right incantation for a given use case.
user-invocable: true
---

# edg Expression Helper

You help users compose, debug, and understand edg expressions. edg uses [expr-lang/expr](https://github.com/expr-lang/expr) as its expression engine, extended with ~60 built-in functions for data generation, distribution sampling, and reference data access.

## What you do

- Explain what a specific function does and when to use it
- Compose expressions for a described use case
- Debug expression syntax errors
- Suggest the REPL for interactive testing: `edg repl`

## Where expressions live

edg-lang (`.edg`) is the only config format. Expressions appear as query args, either positional or named:

**Positional:**
```edg
query_name `SELECT ...` (ref('fetch_users').id, gen('email'))
```

**Named:**
```edg
query_name `SELECT ...` (user_id: ref('fetch_users').id, email: gen('email'))
```

Named args are required when another expression in the same query reads the value back with `arg('name')`. Pick one form per query - mixing named and positional args in a single query is an error.

## Quick Reference

### Identifiers
| Function | Description |
|---|---|
| `uuid_v1()` | Version 1 UUID (timestamp + node ID) |
| `uuid_v4()` | Random UUID |
| `uuid_v6()` | Version 6 UUID (reordered timestamp) |
| `uuid_v7()` | Sortable UUID (preferred for PKs) |
| `ulid()` | Sortable 26-char Crockford base32 ID (48-bit ms timestamp + 80 random bits) |
| `seq(start, step)` | Auto-incrementing counter per worker |
| `seq_alpha(length)` | Auto-incrementing alpha sequence per worker (aaa, aab, aac, ...) |
| `seq_global(name)` | Shared auto-incrementing counter across all workers (requires a top-level `seq name(start: N, step: M)`) |
| `seq_alpha_global(name)` | Shared auto-incrementing alpha sequence across all workers (requires a top-level `seq name(length: L)`) |
| `uniform.seq(name)` | Uniform random from generated sequence values |
| `zipf.seq(name, s, v)` | Zipfian-distributed pick from sequence (hot early values) |
| `norm.seq(name, mean, stddev)` | Normal-distributed pick from sequence |
| `pareto.seq(name, alpha)` | Pareto-distributed pick from sequence (continuous power-law) |
| `exp.seq(name, rate)` | Exponential-distributed pick from sequence |
| `lognorm.seq(name, mu, sigma)` | Log-normal-distributed pick from sequence |
| `objectid()` | MongoDB ObjectID (24-char hex string) |

### Random Data (gofakeit)
| Function | Description |
|---|---|
| `gen('email')` | Random email |
| `gen('firstname')` | Random first name |
| `gen('lastname')` | Random last name |
| `gen('number:1,100')` | Random int in range |
| `gen('sentence:5')` | Random sentence |
| `regex('[A-Z]{3}-[0-9]{4}')` | String matching regex |
| `regex('[a-z]{5,12}')` | Regex with an explicit length range |
| `bool()` | Random true/false |
| `markov_string('person.first')` | New string from a Markov chain trained on a list, a `dataset.field` from an init query, or a gofakeit `category.key` list |
| `markov_string(source, order, min_len, max_len)` | Same, with order (default 2) and length bounds (default 3-12). Never returns a training word; trained once per run, so keep the arguments constant |
| | Raising order shrinks the chain's vocabulary, since more output is a training word and those are rejected. `edg doctor` counts it; too narrow a chain under `uniq()`/`uniq_across()` is a config error |

`regex()` takes RE2, so no lookarounds or backreferences. `*`, `+` and `{n,}` generate at most 10 repetitions, while an explicit `{n,m}` is honoured as written - use `{n,m}` when the length matters. Anchors (`^`, `$`, `\b`) are ignored. `.` and negated classes draw from printable ASCII. Under `(?i)` each letter is drawn from its Unicode case variants, so `(?i)s` can yield `ſ` - spell the cases out as `[sS]` when the column must stay ASCII. An unparseable pattern is a validation error, not a generated value.

### Numeric Distributions
| Function | Description |
|---|---|
| `uniform.int(min, max)` | Flat/even distribution |
| `uniform.float(min, max, precision)` | Uniform float with decimal places |
| `norm(mean, stddev, min, max)` | Bell curve |
| `norm.float(mean, stddev, min, max, precision)` | Bell curve float |
| `exp(rate, min, max)` | Exponential decay |
| `lognorm(mu, sigma, min, max)` | Right-skewed long tail |
| `pareto(alpha, max)` | Continuous power-law, lower values dominate |
| `pareto.float(alpha, min, max, precision)` | Continuous Pareto float in [min, max] |
| `zipf(s, v, max)` | Power-law hot keys |
| `nurand(A, x, y)` | TPC-C non-uniform random |
| `rwalk(drift, volatility)` | Stateful random walk per worker (cumulative steps) |
| `rwalk_f(drift, volatility, precision)` | Random walk with decimal precision |
| `mvnorm(index, means, stddevs, correlations)` | Correlated multivariate normal (cached per row) |

All distributions also support `.obj_n` to generate N object instances with distribution-controlled count (e.g. `uniform.obj_n(...)`, `zipf.obj_n(...)`).

Every distribution namespace is free to use.

There is deliberately no `binomial.float`, `poisson.float` or `zipf.float` - those distributions are integer-valued by definition. Use `beta.float` or `gamma.float` for a continuous skewed value.

`beta.*` min/max **scales** rather than clamps. Beta's support is [0, 1], so samples are mapped `min + x * (max - min)`, letting `beta.int`/`set`/`ref`/`seq`/`vector`/`timestamp`/`obj_n` span their full range. All other distributions still reject out-of-range draws, because their support is unbounded.

### Set Distributions
| Function | Description |
|---|---|
| `set(values[, weights])` | Pick from a set, uniform without `weights`, weighted with them |
| `norm.set(values, mean, stddev)` | Normal distribution over set indices |
| `exp.set(values, rate)` | Exponential over set indices |
| `pareto.set(values, alpha)` | Pareto over set indices (continuous power-law) |
| `zipf.set(values, s, v)` | Zipfian over set indices |
| `lognorm.set(values, mu, sigma)` | Log-normal over set indices |

### Dates & Times
| Function | Description |
|---|---|
| `after(base, min_offset, max_offset)` | Random timestamp between min_offset and max_offset after base (Go durations) |
| `before(base, min_offset, max_offset)` | Random timestamp between min_offset and max_offset before base (Go durations) |
| `timestamp(min, max)` | Random RFC3339 timestamp |
| `timestamp_step()` | Next monotonic timestamp (requires `timestamp_steps` in `count:`) |
| `timestamp_steps(min, max, interval_or_count)` | Count of steps between min and max; third arg is interval string (`'5m'`) or integer count (`10000`). Sets up `timestamp_step()` |
| `date(format, min, max)` | Random formatted date |
| `date_offset(duration)` | Now +/- duration |
| `duration(min, max)` | Random Go duration string |
| `time(min, max)` | Random HH:MM:SS |
| `timez(min, max)` | Random time of day with +00:00 timezone suffix |
| `sim_time()` | Current simulated instant as RFC3339Nano (needs a `sim` block, run phase only) |
| `sim_progress()` | How far through the run the simulated clock is, 0 to 1 (needs a `sim` block, run phase only) |

### Strings & Formatting
| Function | Description |
|---|---|
| `humanize(value, format)` | Format numbers for humans: bytes, ibytes, comma, ordinal |
| `to_kb(bytes)` | Convert bytes to kilobytes (floor) |
| `to_mb(bytes)` | Convert bytes to megabytes (floor) |
| `to_gb(bytes)` | Convert bytes to gigabytes (floor) |
| `to_tb(bytes)` | Convert bytes to terabytes (floor) |
| `template(format, args...)` | Go fmt.Sprintf |
| `json_obj(k1, v1, ...)` | Build JSON object |
| `json_arr(minN, maxN, pattern)` | Build JSON array |
| `array(minN, maxN, pattern)` | PostgreSQL array literal |
| `vector(dims, clusters, spread)` | vector literal with clustered unit vectors (uniform) |
| `pareto.vector(dims, clusters, spread, alpha)` | vector literal with Pareto centroid selection |
| `zipf.vector(dims, clusters, spread, s, v)` | vector literal with Zipfian centroid selection |
| `norm.vector(dims, clusters, spread, mean, stddev)` | vector literal with normal centroid selection |
| `exp.vector(dims, clusters, spread, rate)` | vector literal with exponential centroid selection (cluster 0 hottest) |
| `lognorm.vector(dims, clusters, spread, mu, sigma)` | vector literal with log-normal centroid selection |
| `embed(text...)` | Real vector embedding via external API (OpenAI-compatible). Variadic, joins args with space. Requires `--embed-api-key` |

### LLM / Structured Generation
| Function | Description |
|---|---|
| `complete(tool_name, prompt)` | Generate structured data via LLM tool calling. Returns a map; access fields with dot notation. Retries up to 3 times on missing/invalid tool calls. Validates response types against tool schema. 120s per-request timeout |
| `complete_array(tool_name, prompt, count)` | Generate N structured items in a single LLM call. Returns `[]map`; use with `ref_each()` to iterate. Schema auto-wrapped in array. Memoized by (tool, prompt, count). Same retry/validation as `complete()` |

Declare a query-level `let` to call once per row and access multiple fields. `let` bindings go between the SQL backtick and the args:

```edg
# Single item
insert_review(type: exec) `INSERT INTO review (text, sentiment, rating) VALUES ($1, $2, $3)`
  let review = complete('review', 'Review: ' + ref_same('products').name)
  (
    local('review').review_text,
    local('review').sentiment,
    local('review').rating
  )

# Batch (N items in one call)
insert_reviews(type: exec_batch, count: 5, size: 5) `INSERT INTO review (text, sentiment, rating) __values__`
  let reviews = complete_array('review', 'Generate 5 reviews', 5)
  (
    ref_each(local('reviews')).review_text,
    ref_each(local('reviews')).sentiment,
    ref_each(local('reviews')).rating
  )
```

Tool schemas are defined in a top-level `complete` block:

```edg
complete {
  tool review(system: 'You are a product review generator.') {
    rating(description: 'Star rating from 1 to 5', type: 'integer', required: true)
    review_text(description: 'A 2-3 sentence review', type: 'string', required: true)
    sentiment(description: 'Overall sentiment', enum: '[positive negative neutral]', type: 'string', required: true)
  }
}
```

Requires `--complete-api-key` (or `EDG_COMPLETE_API_KEY`). Configure endpoint with `--complete-url`, model with `--complete-model`. Any OpenAI-compatible API works (Ollama, vLLM, etc.).

### Reference Data
| Function | Scope | Description |
|---|---|---|
| `ref(name)` | None | Fresh random row every call |
| `ref_same(name)` | Per-query | Same row within one query execution |
| `ref_diff(name)` | Per-query | Unique row per call within one query |
| `ref_perm(name)` | Per-worker | Fixed row for worker lifetime |
| `ref_each(query_or_dataset)` | Expansion / Per-query | SQL string: one execution per returned row. Named dataset (unquoted): sequential iteration with same-row caching |
| `ref_n(name, field, min, max)` | None | N unique values as a list: index it (`[0]`) for one value, or pass it whole for a comma-separated value |
| `norm.ref(name, mean, stddev)` | None | Row picked via normal distribution |
| `exp.ref(name, rate)` | None | Row picked via exponential distribution |
| `lognorm.ref(name, mu, sigma)` | None | Row picked via log-normal distribution |
| `pareto.ref(name, alpha)` | None | Row picked via Pareto distribution |
| `zipf.ref(name, s, v)` | None | Row picked via Zipfian distribution |

### Aggregation (over named datasets)
| Function | Description |
|---|---|
| `count(name)` | Row count |
| `sum(name, field)` | Sum of field |
| `avg(name, field)` | Average of field |
| `min(name, field)` | Minimum of field |
| `max(name, field)` | Maximum of field |
| `median(name, field)` | Median of field (same as `percentile(name, field, 50)`) |
| `percentile(name, field, p)` | Pth percentile of field, linear interpolation between nearest ranks. `p` in [0, 100] |
| `stddev(name, field)` | Population standard deviation of field (divides by N) |
| `variance(name, field)` | Population variance of field |
| `distinct(name, field)` | Count of distinct values |

Every aggregate (`sum`, `avg`, `min`, `max`, `median`, `stddev`, `variance`, `percentile`) also accepts a single-array form as well as `(dataset, field)`: `sum([1, 2, 3])`, `median(prices)`, `stddev(prices)`, `percentile(prices, 95)`. For `percentile` the `p` is always the last arg, so the array form takes 2 args and the dataset form takes 3. Non-numeric elements in an array are skipped. Empty input returns 0, including for `min`/`max`.

### Conditionals & Dependencies
| Function | Description |
|---|---|
| `arg(index)` | Reference earlier arg by zero-based index |
| `arg('name')` | Reference earlier arg by name (named args only) |
| `env(name)` | Job environment variable (`--env`/`--env-file`), falling back to the OS environment (error if missing) |
| `env_nil(name)` | Job environment variable (`--env`/`--env-file`), falling back to the OS environment (nil if missing) |
| `result()` | First row of current query's SELECT result (post_print only) |
| `results()` | All rows of current query's SELECT result as []map (post_print only) |
| `affected()` | Rows the current query touched: affected for `exec`, returned for `query`, summed across its arg sets and batches. Post-execution only (`until`, `post_print`). `-1` where the driver reports no count (Cassandra, Redis, a transaction's queued statements). Drives drain loops: `until: affected() < 1000` |
| `cond(pred, trueVal, falseVal)` | Ternary conditional (inline, for args) |
| `nullable(expr, probability)` | NULL with given probability |
| `coalesce(v1, v2, ...)` | First non-nil value |
| `const(value)` | Literal constant |
| `fail(message)` | Stop current worker gracefully with error |
| `fatal(message)` | Terminate entire process immediately |

### Control Flow (config-level, not expression functions)
Conditional branching uses the same expr-lang expressions but at the config level, not inside args:
- `if:` condition must evaluate to **boolean** (e.g., `"ref_same('x').balance >= local('amount')"`)
- `match:` expression can be any type; compared to each `eq:` value as strings
- `eq:` expressions can be any type (string literals need inner quotes: `"'savings'"`)
- All conditional expressions have access to `ref_same()`, `ref()`, `local()`, `global()`, etc.

### Correlated Totals
| Function | Description |
|---|---|
| `distribute_sum(total, minN, maxN, precision)` | N random parts summing exactly to total, as a list (renders comma-separated) |
| `distribute_weighted(total, weights, noise, precision)` | Split total by proportional weights with controlled noise (0=exact, 1=random) |

### Multi-Value
| Function | Description |
|---|---|
| `nurand_n(A, x, y, minN, maxN)` | N unique NURand values as a list (renders comma-separated) |
| `weighted_sample_n(name, field, weightField, minN, maxN)` | N weighted unique rows from a dataset, as a list (renders comma-separated) |

Every distribution namespace has a `.n` domain returning N unique picks as a list. The last two args are always `minN, maxN`: a count is chosen uniformly in [minN, maxN] and that many distinct values are drawn. Typical use is unique item IDs for multi-item order lines (TPC-C New-Order).

A list used whole reaches the database comma-separated, so the SQL patterns that split it (`unnest(string_to_array(...))`, `JSON_TABLE`, `STRING_SPLIT`, `XMLTABLE`) still apply. To use single elements, index the list in the expression instead of splitting the string in SQL: `uniform.n(1, 1000, 2, 2)[0]`. `len(list)` gives the count and `string(list)` the comma-separated form. The same applies to `ref_n`, `weighted_sample_n`, `nurand_n`, `distribute_sum` and `distribute_weighted`:

```edg
transaction transfer {
  let pair   = ref_n('fetch_accounts', 'id', 2, 2)
  let source = local('pair')[0]
  let target = local('pair')[1]
}
```

| Function | Description |
|---|---|
| `uniform.n(min, max, minN, maxN)` | N unique uniform values |
| `beta.n(alpha, beta, min, max, minN, maxN)` | N unique Beta-distributed values |
| `binomial.n(n, p, minN, maxN)` | N unique Binomial-distributed values |
| `empirical.n(samples, minN, maxN)` | N unique values sampled from an empirical CDF |
| `exp.n(rate, min, max, minN, maxN)` | N unique exponentially-distributed values |
| `gamma.n(shape, rate, min, max, minN, maxN)` | N unique Gamma-distributed values |
| `lognorm.n(mu, sigma, min, max, minN, maxN)` | N unique log-normally-distributed values |
| `norm.n(mean, stddev, min, max, minN, maxN)` | N unique normally-distributed values |
| `pareto.n(alpha, imax, minN, maxN)` | N unique Pareto-distributed values |
| `poisson.n(lambda, minN, maxN)` | N unique Poisson-distributed values |
| `weibull.n(shape, scale, min, max, minN, maxN)` | N unique Weibull-distributed values |
| `zipf.n(s, v, imax, minN, maxN)` | N unique Zipfian-distributed values |

Errors if `minN < 1`, if `maxN < minN`, or if N distinct values aren't found within 10,000 draws (i.e. the distribution's support is smaller than N).

### Batch
| Function | Description |
|---|---|
| `batch(n)` | Sequential indices [0, n) |
| `gen_batch(total, size, pattern)` | Batched gofakeit values |
| `iter()` | 1-based row counter for batch queries (resets per query) |

### Uniqueness
| Function | Description |
|---|---|
| `uniq(expression)` | Retry expression until unique value found (100 attempts default) |
| `uniq(expression, maxRetries)` | Same, with custom retry limit |
| `uniq(expr1, expr2 [, ...])` | Composite uniqueness - retries until the tuple is unique. Returns `[]any`; index to pick column. Same-row calls with identical expressions return cached tuple |
| `uniq(expr1, expr2, maxRetries)` | Composite with custom retry limit |
| `uniq_across(pool, expr...)` | Cross-query uniqueness pool; values unique across all queries sharing same pool name. Pool persists for environment lifetime |
| `uniq_across(pool, expr..., maxRetries)` | Cross-query pool with custom retry limit |
| `uniq(array)` | De-duplicates a collection, keeping first-seen order; no retrying. An expression that produces a collection takes this form too, so `uniq(concat(tags, ['books']))` de-duplicates rather than redrawing |

Quoted and unquoted spellings behave the same: `uniq("gen('email')")` and `uniq(gen('email'))` both redraw. A plain call would normally be evaluated before `uniq()` saw it, leaving nothing to redraw from, so the compiler quotes call-shaped arguments for you. Quoting explicitly is still the clearer spelling.

What the compiler can't fix: a bare literal (`uniq(500)`) or an identifier holding a non-expression value. An identifier is left alone deliberately - it may already hold the expression as a string, as in `uniq(email_expr)` where a global holds `"gen('email')"`. `edg doctor` reports the cases that remain.

`uniq()`'s seen set covers one pass of arg evaluation. A batch query generates all its rows in one pass, so `uniq()` de-duplicates across the batch. A non-batch `run` query generates one row per iteration, so a lone `uniq()` call there compares against an empty set and can never reject a duplicate - use `uniq_across('pool', "expr")` for uniqueness across iterations. `edg doctor` reports this too.

### PII & Masking
| Function | Description |
|---|---|
| `gen_locale('ja_JP', 'first_name')` | Locale-aware name/address/phone generation |
| `gen_locale('de_DE', 'name')` | Full name in locale order (eastern = last+first, western = first last) |
| `mask(key, value)` | Deterministic 16-char hex token, HMAC-keyed by `key` (same key + input -> same output) |
| `mask(key, value, length)` | Hex token truncated to `length` chars |
| `mask(key, value, 'base64')` | Base64-encoded token |
| `mask(key, value, 'base32')` | Base32-encoded token |
| `mask(key, value, 'asterisk')` | Repeated `*` characters (default 16) |
| `mask(key, value, 'asterisk', 4)` | 4 asterisks |
| `mask(key, value, 'redact')` | Fixed string `[REDACTED]`, length ignored |
| `mask(key, value, 'email')` | Preserves `@domain`, masks local part with `*` |
| `mask(key, value, 'email', 4)` | `****@domain.com` |
| `hash(value, algo)` | Deterministic unkeyed hex digest; `algo` is `md5`, `sha1`, `sha256` or `crc32` |

`hash()` returns lowercase hex of 32, 40, 64 or 8 chars respectively. Unlike `mask()` (HMAC-keyed pseudonymization), `hash()` takes no key, so the same input always yields the same digest across runs and machines. Use it for dedup keys and shard selection, not for pseudonymization.

Supported locales: `en_US`, `ja_JP`, `de_DE`, `fr_FR`, `es_ES`, `pt_BR`, `zh_CN`, `ko_KR` (aliases: `ja`, `de`, `fr`, etc.)

### Iteration Counter
| Function | Description |
|---|---|
| `global_iter()` | Monotonic iteration counter shared across all workers. Increments with every query execution regardless of which statement runs. Use with math functions to make data change shape over the life of a workload |

### Math
| Function | Description |
|---|---|
| `abs(x)` | Absolute value |
| `acos(x)` | Arc cosine (radians) |
| `asin(x)` | Arc sine (radians) |
| `atan(x)` | Arc tangent (radians) |
| `atan2(y, x)` | Two-argument arc tangent (radians) |
| `ceil(x)` | Smallest integer >= x |
| `cos(x)` | Cosine (radians) |
| `e` | Euler's number (2.71828...). Bare variable, not a call. Use `pow(e, x)` for exponentials |
| `floor(x)` | Largest integer <= x |
| `log(x)` | Natural logarithm |
| `log10(x)` | Base-10 logarithm |
| `mod(x, y)` | Floating-point remainder |
| `pi` | Pi constant (3.14159...) |
| `pow(x, y)` | x raised to the power y |
| `round(x, places)` | Round to N decimal places |
| `sin(x)` | Sine (radians) |
| `sqrt(x)` | Square root |
| `tan(x)` | Tangent (radians) |

### Other
| Function | Description |
|---|---|
| `blob(n)` | Random n bytes as raw binary (cross-database) |
| `bytes(n)` | Hex-encoded random bytes (pgx only) |
| `bit(n)` | Fixed-length bit string |
| `varbit(n)` | Variable-length bit string |
| `inet(cidr)` | Random IP in CIDR block |
| `ltree(parts...)` | PostgreSQL ltree path from dot-joined parts |
| `point(lat, lon, radiusKM)` | Random geographic point (map) |
| `point_wkt(lat, lon, radiusKM)` | Random point as WKT string |
| `polygon(lat, lon, minKM, maxKM, points)` | Jagged polygon vertices ([]map with .lat/.lon) |
| `polygon_wkt(lat, lon, minKM, maxKM, points)` | Jagged polygon as WKT POLYGON string |
| `geo_distance(lat1, lon1, lat2, lon2)` | Great-circle distance in km (haversine, earth radius 6371 km) |
| `geo_bearing(lat1, lon1, lat2, lon2)` | Initial compass bearing in degrees, [0, 360). 0=north, 90=east, 180=south, 270=west |

## Common Patterns

### Mutually exclusive columns
```edg
insert_contact(type: exec) `INSERT INTO contact (is_email, email, phone) VALUES ($1, $2, $3)` (
  bool(),                            # coin flip
  cond(arg(0), gen('email'), nil),   # email if true
  cond(!arg(0), gen('phone'), nil)   # phone if false
)
```

### Polymorphic associations (STI / MTI)

A type column decides which parent table the foreign key points at. Generate the type first, then branch every dependent column on it with `cond()` and `arg()`, so type and id can never disagree. Weights on the type set the parent mix independently of how many rows each parent table holds.

```edg
# STI: one id column, no foreign key possible
populate_comment(type: exec_batch, count: comments) `INSERT INTO comment (commentable_type, commentable_id, body)
  __values__` (
  commentable_type: uniform.set(['post', 'video'], [70, 30]),
  commentable_id: cond(arg('commentable_type') == 'post', ref('populate_post').id, ref('populate_video').id),
  body: gen('loremipsumsentence:12')
)

# MTI: one nullable FK per parent, exclusive arc enforced by a CHECK constraint
populate_reaction(type: exec_batch, count: reactions) `INSERT INTO reaction (target_type, post_id, video_id)
  __values__` (
  target_type: uniform.set(['post', 'video'], [70, 30]),
  post_id: cond(arg('target_type') == 'post', ref('populate_post').id, nil),
  video_id: cond(arg('target_type') == 'video', ref('populate_video').id, nil)
)
```

`ref('populate_post')` reads rows an earlier seed query generated, so no `SELECT` round-trip is needed. Never call `ref()` twice for the same parent in one row: each call picks a fresh row, so the columns disagree. Reuse the value with `arg()` instead.

### Computed total from earlier args
```edg
# Positional
insert_line(type: exec) `INSERT INTO line (price, qty, total) VALUES ($1, $2, $3)` (
  uniform.float(1.00, 99.99, 2),   # price
  gen('number:1,10'),              # quantity
  arg(0) * float(arg(1))           # total = price * qty
)

# Named (equivalent)
insert_line(type: exec) `INSERT INTO line (price, qty, total) VALUES ($1, $2, $3)` (
  price:    uniform.float(1.00, 99.99, 2),
  quantity: gen('number:1,10'),
  total:    arg('price') * float(arg('quantity'))
)
```

### Full name from parts
```edg
# Positional
insert_person(type: exec) `INSERT INTO person (first, last, full) VALUES ($1, $2, $3)` (
  gen('firstname'),
  gen('lastname'),
  arg(0) + ' ' + arg(1)       # "Alice Smith"
)

# Named (equivalent)
insert_person(type: exec) `INSERT INTO person (first, last, full) VALUES ($1, $2, $3)` (
  first: gen('firstname'),
  last:  gen('lastname'),
  full:  arg('first') + ' ' + arg('last')
)
```

### Weighted category selection
```edg
set(['electronics', 'clothing', 'books', 'food'], [40, 30, 20, 10])
```

### Hot-key access (Zipfian)
```edg
zipf(2.0, 1.0, 999)  # value 0 is most frequent
```

### Hotspot rows (distribution-based ref)
```edg
zipf.ref('fetch_ids', 2.0, 1.0).id      # heavy skew toward early rows
pareto.ref('fetch_ids', 2.0).id         # continuous power-law, first rows dominate
norm.ref('fetch_ids', 0.5, 0.2).id      # bell curve centered at midpoint
exp.ref('fetch_ids', 1.5).id            # exponential decay from first row
lognorm.ref('fetch_ids', 0.0, 0.5).id   # right-skewed access pattern
```

### Worker-pinned partition
```edg
ref_perm('fetch_warehouses').w_id  # same warehouse for entire worker lifetime
```

### Invoice line items (distribute_sum)
```edg
insert_invoice(type: exec) `INSERT INTO invoice (id, total, amounts)
  SELECT $1::UUID, $2::NUMERIC, unnest(string_to_array($3, ',')::NUMERIC[])` (
    uuid_v4(),                       # invoice id
    uniform.float(100, 10000, 2),    # total
    distribute_sum(arg(1), 3, 7, 2)  # 3-7 amounts summing to total
)
```

### Subtotal/tax/shipping breakdown (distribute_weighted)
```edg
insert_order(type: exec) `INSERT INTO order_total (total, subtotal, tax, shipping)
  VALUES (
    $1::NUMERIC,
    split_part($2, ',', 1)::NUMERIC,
    split_part($2, ',', 2)::NUMERIC,
    split_part($2, ',', 3)::NUMERIC
  )` (
    uniform.float(100, 10000, 2),                      # total
    distribute_weighted(arg(0), [85, 10, 5], 0.1, 2)   # ~85/10/5 split with 10% noise
)
```

### Masking PII
```edg
insert_customer(type: exec) `INSERT INTO customer (...) VALUES (...)` (
  first_name:     gen_locale('ja_JP', 'first_name'),
  last_name:      gen_locale('ja_JP', 'last_name'),
  full_name:      arg('last_name') + arg('first_name'),
  email:          gen('email'),
  masked_name:    mask(env('MASK_KEY'), arg('full_name')),           # hex token
  masked_email:   mask(env('MASK_KEY'), arg('email'), 'email', 4),   # ****@example.com
  redacted_phone: mask(env('MASK_KEY'), arg('phone'), 'redact')      # [REDACTED]
)
```

### Unique codes in batch inserts
```edg
# 200 airports each with a unique 3-char IATA code
populate_airport(type: exec_batch, count: 200, size: 100) `INSERT INTO airport (id, code)
  __values__` (
    iter(),
    uniq("gen('airlineairportiata')")
)
```

### Composite uniqueness across columns
```edg
# 500 people with unique (first_name, last_name) pairs
populate_people(type: exec_batch, count: 500, size: 100) `INSERT INTO people (first_name, last_name, email)
  __values__` (
    uniq("gen('firstname')", "gen('lastname')")[0],
    uniq("gen('firstname')", "gen('lastname')")[1],
    gen('email')
)
```

### Real embeddings
```edg
# With generated data
populate_product(type: exec) `INSERT INTO product (name, description, embedding)
  VALUES ($1, $2, $3::VECTOR)` (
    gen('productname'),
    gen('sentence:3'),
    embed(arg(0), arg(1))            # embed name + description
)

# With reference dataset (each product exactly once)
populate_product(type: exec) `INSERT INTO product (name, description, embedding)
  VALUES ($1, $2, $3::VECTOR)` (
    ref_each(product_catalog).name,
    ref_each(product_catalog).description,
    embed(ref_each(product_catalog).name, ref_each(product_catalog).description)
)
```

Requires `--embed-api-key` (or `EDG_EMBED_API_KEY`). Configure endpoint with `--embed-url`, model with `--embed-model`, dimensions with `--embed-dimensions`, max batch size with `--embed-max-batch`. Any OpenAI-compatible API works (Ollama, vLLM, etc.).

### Error handling with fail/fatal
```edg
# Stop worker if map lookup misses
{'us': 'us-east-1', 'eu': 'eu-west-1'}[env('REGION')] ?? fail('unknown REGION')

# Kill entire process on critical misconfiguration
fatal('missing required config')
```

`fail()` stops only the current worker; `fatal()` terminates the entire process.

### Temporal patterns (global_iter + math)

Combine `global_iter()` with math functions to make generated data change shape over the life of a workload. Define `total_iters` as a global and estimate it from: `workers * duration_seconds / avg_latency_seconds`.

```edg
# Zipf skew drift: popularity concentrates over time
zipf(initial_skew + (final_skew - initial_skew) * global_iter() / total_iters, 1, products - 1)

# Logarithmic growth: steep rise then plateau (inflation, adoption)
floor(base_price * (1.0 + log(1.0 + global_iter() / 1000.0)) * 100.0) / 100.0

# Sine wave seasonality: oscillation on a growing baseline
floor(abs(base_traffic + 0.5 * sqrt(global_iter()) + amplitude * sin(2.0 * pi * global_iter() / period)))

# Periodic spikes: cos² pulse at regular intervals
pow(cos(pi * mod(global_iter(), interval) / interval), 2.0) * 10.0

# Bounded drift: arctangent saturation (sensor calibration)
100.0 + (2.0 * atan(sqrt(global_iter()) / 100.0) / pi) * 15.0 + norm.float(0, 0.5, -2, 2, 2)
```

### Time-keyed patterns (sim_time + sim_progress)

`global_iter()` is keyed on how many iterations you happen to complete, so the shape drifts with database speed and `total_iters` has to be estimated. With a `sim` block the clock is keyed on elapsed time instead, and no estimate is needed:

```edg
sim(period: '24h', in: '20m', from: '2024-01-01T00:00:00Z')
```

```edg
# Timestamp a row with the simulated instant, so the data spans the full period
sim_time()

# Progress-driven drift, the direct replacement for global_iter() / total_iters
zipf(initial_skew + (final_skew - initial_skew) * sim_progress(), 1, products - 1)

# Ramp a value linearly over the simulated run
base_price * (1.0 + 0.4 * sim_progress())
```

Both are run-phase only and error without a `sim` block. `sim_time()` is RFC3339Nano rather than RFC3339 because at a high scale factor second precision would collapse a burst of concurrent inserts onto one timestamp.

## Debugging Tips

- Use `edg repl` to test any expression interactively without a database
- Expressions are compiled at startup; syntax errors will be caught immediately
- `arg(index)` is zero-based and only works within the same query's args; `arg('name')` requires named args (`name: expr`)
- Named and positional arg forms are mutually exclusive per query
- `ref_same` resets between queries; `ref_perm` never resets
- Set distributions accept expr-lang array literals: `['a', 'b', 'c']`
- Weights in `set` are relative, not percentages (`[40, 30, 20, 10]` and `[4, 3, 2, 1]` behave identically)

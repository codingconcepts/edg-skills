---
name: edg-fuzz
description: Fuzz test edg expressions to inspect output distributions, null rates, type breakdowns, and catch surprises before seeding data.
user-invocable: true
---

# edg Fuzz

Help users fuzz test their edg expressions using `edg fuzz`. This command evaluates an expression thousands of times and reports statistics about the output.

## When to Use

- User wants to verify an expression's output distribution
- User wants to check null rates from `nullable()`
- User wants to compare distribution shapes (uniform vs zipf vs normal)
- User suspects an expression always returns the same value
- User wants to find panics or errors in custom expressions

## Usage

```sh
edg fuzz "<expression>" [-n iterations] [--config workload.edg]
```

### Flags

| Flag | Default | Description |
|---|---|---|
| `-n` / `--iterations` | `10000` | Number of evaluations |
| `--config` | | Optional config for globals, refs, user-defined expressions |

## Output Interpretation

The output includes:

- **Type breakdown**: What Go types the expression returns and their percentages
- **Nulls**: Count and percentage of nil values
- **Panics**: Count of panics during evaluation (should be 0)
- **Errors**: Count of errors during evaluation
- **Unique**: Number of distinct values produced
- **Min/Max/Mean/Median/Stddev**: Numeric statistics (only for numeric output)
- **Distribution**: ASCII histogram showing value distribution across 10 buckets

## Common Patterns

### Verify Uniform Distribution
```sh
edg fuzz "uniform(1, 100)"
```
Expect: roughly equal bucket counts (~10% each).

### Check Null Rate
```sh
edg fuzz "nullable(0.1, gen('name'))"
```
Expect: `Nulls: ~1000 (10.0%)` with 10000 iterations.

### Compare Distributions
```sh
edg fuzz "zipf(1000, 2.0, 1.0)"    # heavy left skew
edg fuzz "norm(0, 100, 50, 10)"     # bell curve
edg fuzz "uniform(0, 100)"          # flat
```

### Test with Config Context
```sh
edg fuzz "norm(0, warehouses * 10, warehouses * 5, warehouses)" --config workload.edg
```

## Workflow

1. Run `edg fuzz` with the user's expression
2. Read the output and explain what it means
3. If the distribution looks wrong, suggest alternative expressions
4. If panics or errors appear, help debug the expression

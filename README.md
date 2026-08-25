# edg skills

Agent skills for [edg](https://github.com/codingconcepts/edg). Slash commands that help you generate, validate, and debug edg-lang workload configs without leaving your terminal.

## Install

Install all skills into your project:

```sh
npx skills add codingconcepts/edg-skills --agent claude-code --skill '*' --yes
```

Or install a specific skill:

```sh
npx skills add codingconcepts/edg-skills --agent claude-code --skill edg-config --yes
```

## Available Skills

| Skill | Description |
|---|---|
| [`/edg-config`](skills/edg-config/) | Generate a complete edg config from a natural language description of your schema and workload |
| [`/edg-validate`](skills/edg-validate/) | Run `edg validate` on a config file, interpret errors, and suggest fixes |
| [`/edg-expression`](skills/edg-expression/) | Find the right edg expression for a use case, explain functions, and debug syntax |
| [`/edg-fuzz`](skills/edg-fuzz/) | Fuzz an expression to inspect its output distribution, null rate, and type breakdown |

## Usage

### Generate a config

```
/edg-config

I have a users table and an orders table on CockroachDB.
Users have an email, name, and created_at timestamp.
Orders reference a user and have a total, status, and created_at.
Seed with 10k users and 50k orders, then benchmark order lookups.
```

### Validate and fix

```
/edg-validate

Check my config at workloads/bench.edg for the pgx driver.
```

### Compose expressions

```
/edg-expression

I need a price between $1 and $500 with a log-normal distribution
skewed toward cheaper items, rounded to 2 decimal places.
```

### Fuzz an expression

```
/edg-fuzz

Check the distribution of zipf(1.2, 1, 10000) over 50k iterations.
```

## Tips

- **Combine skills**: Generate with `/edg-config`, sanity-check expressions with `/edg-fuzz`, then validate with `/edg-validate`
- **Use the REPL**: `edg repl` lets you test expressions interactively with tab completion
- **Discover functions**: `edg functions [search]` lists available functions by name

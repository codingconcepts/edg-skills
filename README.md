# edg skills

Agent skills for [edg](https://github.com/codingconcepts/edg). Slash commands that help you generate, validate, debug, and migrate workload configs (YAML and DSL) without leaving your terminal.

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
| [`/edg-config`](skills/edg-config/) | Generate a complete edg config (YAML or DSL) from a natural language description of your schema and workload |
| [`/edg-validate`](skills/edg-validate/) | Run `edg validate` on a config file (YAML or DSL), interpret errors, and suggest fixes |
| [`/edg-expression`](skills/edg-expression/) | Find the right edg expression for a use case, explain functions, and debug syntax |
| [`/edg-migrate`](skills/edg-migrate/) | Convert an edg config between database drivers or between formats (YAML ↔ DSL) |

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

Check my config at workloads/bench.yaml for the pgx driver.
```

### Compose expressions

```
/edg-expression

I need a price between $1 and $500 with a log-normal distribution
skewed toward cheaper items, rounded to 2 decimal places.
```

### Migrate between drivers

```
/edg-migrate

Convert workloads/bench.yaml from pgx to mysql.
```

### Convert YAML to DSL

```
/edg-migrate

Convert workloads/bench.yaml to DSL format.
```

## Tips

- **Combine skills**: Generate with `/edg-config`, validate with `/edg-validate`, port with `/edg-migrate`
- **YAML vs DSL**: Use YAML for complex configs (stages, conditionals, LLM). Use DSL for compact, query-heavy workloads (~60% smaller)
- **Use the REPL**: `edg repl` lets you test expressions interactively with tab completion
- **Discover functions**: `edg functions [search]` lists available functions by name

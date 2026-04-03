# Environment Strategy

Use this file whenever the migration includes `.env`, `env_file`, Compose interpolation, or inline `-e` flags.

## Goals

- preserve source-of-truth for variables
- avoid leaking secrets into generated Quadlet files by default
- keep resulting units readable
- report missing variables explicitly
- reduce large upstream env templates into a small set of user decisions

## Default rules

The agent should actively interpret the env template and its comments. Do not offload the entire env review back to the user.

### Prefer `Environment=` when

- there are only a few variables
- values are stable and not sensitive
- keeping everything in one file materially improves readability

### Prefer `EnvironmentFile=` when

- the source already uses `.env` or `env_file`
- variables are numerous
- variables contain secrets or deployment-specific values
- the same env file is shared by multiple services

## Sensitive-value heuristic

Treat names containing these substrings as sensitive unless the user tells you otherwise:

- `PASSWORD`
- `TOKEN`
- `SECRET`
- `KEY`
- `PRIVATE`
- `PASS`

Default behavior:

- do not inline them in `Environment=`
- keep them in `EnvironmentFile=` or generate a placeholder sample file instead

## Compose interpolation

Common forms:

- `${VAR}`
- `${VAR:-default}`
- `${VAR-default}`

Strategy:

- if the actual value source is present, resolve it and document where it came from
- if only a default is available, note that the value is default-derived
- if the variable is missing, list it as unresolved

Do not fabricate values.

## Agent responsibility

When the source project ships a large `.env.example` or multiple env templates:

- read the comments and deployment docs yourself
- determine which values can safely stay at documented defaults
- determine which values are true deployment decisions and must be confirmed with the user
- prepare a candidate `.env` or env delta instead of asking the user to read the whole template manually

The user should only need to answer the small number of high-impact questions that cannot be discovered locally.

## Minimal examples

### Inline stable values

Source intent:

```yaml
environment:
  APP_ENV: production
  APP_PORT: "8080"
```

Reasonable result shape:

```ini
[Container]
Environment=APP_ENV=production
Environment=APP_PORT=8080
```

Use this when there are only a few non-sensitive structural values.

### Preserve env-file workflow

Source intent:

```yaml
env_file:
  - .env
```

Reasonable result shape:

```ini
[Container]
EnvironmentFile=/opt/myapp/.env
```

Use this when the source already relies on an env file, especially for secrets or many variables.

### Large template reduced to a candidate env

Source intent:

- upstream ships `.env.example`
- only a few values are true deployment decisions

Recommended behavior:

- keep documented defaults that are safe to preserve
- ask the user only for high-impact values such as domain, storage path, or database password
- generate a candidate `.env` or env delta with clear placeholders for the unresolved items

## Output patterns

### Small inline set

```ini
[Container]
Environment=APP_ENV=production
Environment=APP_PORT=8080
```

### External env file

```ini
[Container]
EnvironmentFile=/opt/myapp/myapp.env
```

### Mixed pattern

Use this when a few values are structural and the rest are secret or deployment-specific.

```ini
[Container]
Environment=APP_ENV=production
EnvironmentFile=/opt/myapp/myapp.env
```

## Missing-variable reporting

When variables cannot be resolved, report them as a concrete checklist.

Example:

- missing `DB_PASSWORD`
- missing `IMMICH_VERSION`
- missing `UPLOAD_LOCATION`

If the user asks for scaffolding, generate a sample env file with obvious placeholders.

Even when the user does not explicitly ask for scaffolding, produce a candidate env result when that materially advances the migration.

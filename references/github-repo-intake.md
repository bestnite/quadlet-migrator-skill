# GitHub Repository Intake

Use this file when the user provides a GitHub repository URL and expects you to find the deployment inputs yourself.

## Goal

Discover the canonical self-hosting assets before attempting any Quadlet conversion.

## Discovery order

1. Read the repository `README.md`.
2. Look for self-hosting instructions and explicit references to Compose files.
3. If documentation names a path, follow that path first.
4. Search the repository tree for likely deployment files:
   - `docker-compose.yaml`
   - `docker-compose.yml`
   - `compose.yaml`
   - `compose.yml`
   - `.env.example`
   - `.env.sample`
   - `env.example`
   - `middleware.env.example`
5. Inspect likely deployment directories when needed:
   - `docker/`
   - `deploy/`
   - `ops/`
   - `infra/`
   - `examples/`
   - `.devcontainer/`
6. Read deployment-specific README files in those directories.
7. Identify helper scripts that generate or sync compose files.

## What to extract

- canonical compose file path
- companion env template path
- additional compose files used for middleware or optional services
- whether the compose file is generated
- whether the source relies on profiles
- whether startup requires preparatory steps such as copying `.env.example` to `.env`

## Heuristics

- Prefer the path explicitly linked from the main README over a randomly discovered file.
- Do not hardcode assumptions like "the deployment entry point is always under `docker/`".
- If the repo has both a template and a generated compose file, treat the generated file as the runnable source and the template as explanatory context.
- If profiles control optional databases or vector stores, decide which profile set the user actually wants before generating Quadlet.
- If env management is mandatory, preserve that pattern rather than flattening hundreds of variables into inline `Environment=` values.
- If several candidate compose files exist, explain which one you selected and why.

## Migration posture for GitHub-sourced projects

When converting a GitHub-hosted project, report:

- which files you chose as source of truth
- which optional files or profiles you ignored
- which variables still require user decisions

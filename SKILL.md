---
name: quadlet-migrator
description: Convert docker run commands or Docker Compose configurations into maintainable Podman Quadlet files, help users map env/env_file/.env usage into Environment or EnvironmentFile, and explain rootless or rootful deployment details and migration risks.
---

# Quadlet Migrator

Use this skill when the user wants to migrate `docker run`, `docker compose`, or Compose-like service definitions into Podman Quadlet units, especially when environment variables, `env_file`, `.env`, rootless deployment, or systemd layout need to be planned or generated.

Do not rely on `podlet` as an execution dependency. You may use it only as prior-art knowledge, not as part of the runtime workflow.

## What this skill does

This skill helps you:

- discover Docker deployment entry points from a GitHub repository URL
- translate `docker run` flags and Compose service fields into Quadlet units
- choose between `.container`, `.pod`, `.network`, `.volume`, and `.build`
- decide whether values belong in `Environment=` or `EnvironmentFile=`
- work closely with the user to confirm deployment-specific environment values and operational choices
- identify missing variables, unsafe inline secrets, and unsupported Compose semantics
- produce deployment notes for rootless or rootful systemd setups

## Operating modes

Choose the lightest mode that satisfies the user's request.

- `advice`: explain mappings, review source inputs, answer targeted migration questions, or sketch a recommended structure without writing final artifacts
- `design`: perform `Planning` and `Finalize`, produce `QUADLET-FINALIZE.md`, but stop before writing runnable artifacts
- `generate`: perform `Planning`, `Finalize`, and `Execution`, then write the approved runnable artifacts

Do not force `generate` mode when the user only wants explanation, review, or a partial conversion.

## Workflow

The full workflow has three explicit phases: `Planning`, `Finalize`, and `Execution`.

- `advice` usually stays in `Planning` or gives a targeted answer without entering all phases
- `design` uses `Planning` and `Finalize`
- `generate` uses all three phases

Do not skip phase boundaries when you are using them. The skill should not jump directly from discovery into writing files.

### Planning phase

Goal: gather inputs, understand the project, and work with the user to make the key decisions.

Tasks in this phase:

1. Classify the input.
   - `docker run` or `podman run` style command
   - single-file Compose configuration
   - Compose plus `.env` and optional `env_file`
   - GitHub repository URL that likely contains self-hosting assets
   - already partially converted Quadlet files that need cleanup

2. If the input is a GitHub repository, discover the deployment entry points first.
   - start from the repository README and deployment subdirectory READMEs
   - follow explicit links or references from documentation before making assumptions about file locations
   - search the repository tree for `docker-compose.yaml`, `docker-compose.yml`, `compose.yaml`, `compose.yml`
   - inspect common deployment-oriented subdirectories such as `docker/`, `deploy/`, `ops/`, `infra/`, `.devcontainer/`, and `examples/`, but do not assume the right entry point must live there
   - look for `.env.example`, `.env.sample`, `env.example`, `middleware.env.example`, or similar templates
   - inspect whether the project uses Compose profiles, multiple compose files, generated compose files, or helper scripts
   - identify the canonical self-hosting entry point rather than assuming the repo root file is authoritative

3. Build a semantic model before writing Quadlet.
   - services
   - images or builds
   - ports
   - volumes and bind mounts
   - networks
   - environment variables and env files
   - restart policy
   - health checks
   - startup dependencies

4. Identify which values and decisions must be confirmed with the user.
   - external URLs, domains, and ports
   - database hostnames, passwords, and database names
   - storage backend selection and credentials
   - profile selection or optional service selection
   - pod grouping when the project has many services or optional containers
   - volume mode selection: named volume, bind mount, or anonymous volume
   - rootless vs rootful deployment mode
   - whether secrets should stay in env files or be moved elsewhere

Do not silently invent deployment-specific values. If the repository or compose file provides placeholders, defaults, or examples, read the surrounding documentation and comments yourself, infer the intended meaning, and only ask the user to confirm the values that materially affect deployment.

When many variables exist, do not hand the raw `.env.example` back to the user for manual review. Your job is to digest it, reduce it, and produce a concise checklist of high-impact decisions. Prioritize the variables that are required to produce a safe and runnable output.

At the end of planning, summarize what you learned and what you intend to generate, then explicitly ask the user whether anything should be changed or added before you move to the next phase.

Planning is also the phase where you must actively ask the user for the unresolved high-impact decisions you identified.

Do not defer first-time decision gathering into `QUADLET-FINALIZE.md`.

If decisions are still unresolved, stop in planning and ask the user directly. Do not write `QUADLET-FINALIZE.md` yet.

### Finalize phase

Goal: consolidate the decisions already made in planning into one internally consistent design snapshot and ask the user to review it.

The output of this phase must be written to a Markdown file named `QUADLET-FINALIZE.md`.

This filename is fixed. Do not rename it per project.

This phase starts only after planning-phase questions have been asked and the user has had a chance to answer or explicitly say there is nothing more to add.

Finalize is not a second discovery pass. Do not use it to introduce new major design choices, gather first-time requirements, or expand the scope of analysis. If execution reveals a new material decision or conflict, return to planning rather than stretching finalize into another analysis phase.

Tasks in this phase:

1. Freeze the chosen service set and runtime grouping.
   - prefer putting the whole project in a single pod when practical
   - if the project is a simple single-container deployment, a standalone `.container` is acceptable
   - if shared networking, shared published ports, or tighter lifecycle coupling make pod semantics useful, prefer one or more `.pod` units
   - containers in the same `.pod` can communicate over `127.0.0.1` / `localhost` because they share a network namespace
   - containers in different pods must not be treated as reachable via `127.0.0.1` / `localhost`; if you split the topology across multiple pods, use container networking and service addressing, or publish ports across the host boundary when that better matches the deployment
   - if one pod is not practical because of port conflicts or clearly incompatible groupings, split the result into a small number of pods rather than forcing an awkward topology

2. Freeze the storage strategy.
   - named volume, bind mount, or anonymous volume per storage use case
   - bind mounts must end up as absolute host paths

3. Freeze the image strategy.
   - prefer upstream prebuilt registry images when they already exist and local build is not required for correctness
   - create `.build` only when local build is actually required, or when the user explicitly wants a declarative local-build workflow
   - prefer fully qualified image names in generated output
   - if the source image omits a registry and is intended for Docker Hub, expand it explicitly instead of relying on short-name resolution
   - for images of the form `name[:tag]` with no namespace, normalize to `docker.io/library/name[:tag]`
   - for images of the form `namespace/name[:tag]` with no registry, normalize to `docker.io/namespace/name[:tag]`
   - if the source clearly points to another registry such as `ghcr.io`, `quay.io`, or a private registry, preserve that registry explicitly

4. Freeze the environment strategy.
   - use `Environment=` for a small number of stable non-sensitive values
   - use `EnvironmentFile=` for bulk variables, secrets, or values already sourced from `.env` / `env_file`
   - if Compose interpolation references variables that are missing, report them explicitly and prepare a candidate env file with placeholders or suggested defaults instead of delegating the entire review back to the user
   - treat variable names containing `PASSWORD`, `TOKEN`, `SECRET`, `KEY`, `PRIVATE`, or `PASS` as sensitive by default and avoid inlining unless the user explicitly wants that

5. Summarize already-known conflicts and their chosen resolution.
   - port collisions
   - incompatible grouping decisions
   - storage mode inconsistencies
   - unresolved required variables
   - mismatch between requested deployment mode and selected file locations

At the end of finalize, write `QUADLET-FINALIZE.md` and ask the user to review it before you start writing the final artifacts.

`QUADLET-FINALIZE.md` is a review artifact, not a questionnaire. It should summarize decisions that were already discussed in planning.

If `QUADLET-FINALIZE.md` already exists, read it first and update it intentionally. Do not blindly overwrite it without checking whether it reflects a prior review round that should be preserved or revised.

When the design has materially changed, replace outdated sections so the file remains a single current review snapshot rather than an append-only log.

`QUADLET-FINALIZE.md` should include:

- source inputs you chose and why
- selected services and omitted optional services
- pod layout
- naming prefix
- image strategy
- volume strategy
- env strategy
- only the minimal placeholders that still cannot be resolved without user secrets or environment-specific values
- detected conflicts and how they were resolved
- the list of files that will be created in execution phase

Do not start execution until the user has reviewed and confirmed `QUADLET-FINALIZE.md` or provided requested edits.

Do not use `QUADLET-FINALIZE.md` as the first place where the user sees important choices. Those choices should already have been raised in planning.

### Execution phase

Goal: write the approved runnable artifacts.

Tasks in this phase:

1. Generate the Quadlet files.
2. Generate the env file or env delta only when needed for runnable output.
3. Generate deployment notes or validation guidance only when they materially help the user operate the result.
4. Generate a README only when the user explicitly wants a self-contained handoff artifact or a packaged deliverable.

Execution should follow the approved contents of `QUADLET-FINALIZE.md`. If the implementation reveals a material conflict with the finalized design, stop and return to planning rather than silently diverging.

If you generate a README or operational notes, use the same language as the user unless the user explicitly asks for another language.

## Decision priority

When rules or signals conflict, use this priority order:

1. the user's explicit request
2. the source project's documented canonical deployment path
3. runnable and safe output
4. maintainable output
5. default style rules in this skill and its references

If a lower-priority default conflicts with a higher-priority source of truth, follow the higher-priority source and say so briefly.

## Hard stops

Stop and ask the user before finalizing or generating runnable output when any of these remain unresolved:

- required secrets, external URLs, host paths, or other deployment-specific values are missing
- multiple plausible source inputs exist and the canonical deployment entry point cannot be determined confidently
- image strategy versus local build strategy is materially ambiguous
- a lossy mapping would change runtime behavior in a way that matters
- the requested deployment mode conflicts with the intended output location or operator model

Do not keep moving forward by guessing through these gaps.

## Rootless vs rootful

Decide early whether the deployment is rootless or rootful, because this changes the output path and some operational guidance.

- For rootless deployments, use `~/.config/containers/systemd/` unless the user has a reason to use another supported path.
- For rootful deployments, use `/etc/containers/systemd/` unless the user asks for a different placement.
- For rootless long-running services, remind the user about lingering if relevant. See `references/deployment-notes.md`.

When you need authoritative details about supported search paths, unit semantics, option names, or debugging, read `references/podman-systemd.unit.5.md`.

## Reference routing

Use the reference files for detailed rules instead of restating them here:

- `references/compose-mapping.md` for Compose field mapping, topology choices, and naming conventions
- `references/env-strategy.md` for `.env`, `env_file`, interpolation, and secret-handling defaults
- `references/github-repo-intake.md` for GitHub repository discovery and canonical input selection
- `references/deployment-notes.md` for deployment guidance and rootless or rootful operational notes
- `references/validation.md` for post-generation validation and troubleshooting steps

## User collaboration

This skill is not a blind converter. For runnable output, collaborate tightly with the user.

- Confirm any environment value that controls external connectivity, credentials, storage, or deployment topology.
- If the source contains a large env template, summarize the required variables into a small decision list and ask the user to confirm only those values.
- Do not ask the user to read upstream docs or manually audit a large `.env.example` unless they explicitly want to do that themselves.
- Read the docs, comments, and example values yourself, then present the user with a reduced set of concrete decisions and a candidate env result.
- Ask the user to choose optional services and pod grouping early when the source project offers many containers or feature profiles.
- Ask the user which volume mode they want before finalizing storage mappings.
- Ask these questions before writing `QUADLET-FINALIZE.md`, not inside it.
- Preserve placeholders when the user has not provided final values yet.
- Distinguish between upstream example values and user-confirmed production values.

You should still make reasonable structural decisions yourself, but do not pretend unknown deployment inputs are settled facts.

## Deployment and validation

When the user wants runnable output, provide the relevant deployment notes from `references/deployment-notes.md` and the validation steps from `references/validation.md` as needed.

At minimum, mention the need to:

- place files in a valid Quadlet directory
- run `systemctl daemon-reload` or `systemctl --user daemon-reload`
- create required bind-mount directories before first start
- verify generator output or systemd unit validity when startup fails

## Examples

- `docker run` for a single web service -> often `advice` or `generate` with one `.container`
- small Compose app with api and db -> usually `design` or `generate`, often one `.pod` plus child containers
- GitHub repo with `.env.example` and multiple profiles -> start in `Planning`, reduce the env questions, then move to `Finalize`

## Anti-examples

- do not dump a large `.env.example` back to the user as the primary review artifact
- do not introduce first-time critical decisions inside `QUADLET-FINALIZE.md`
- do not force pod topology when a standalone `.container` is the simpler correct result
- do not keep generating through unresolved deployment-critical unknowns

## Boundaries

Do not claim perfect equivalence where Podman or Quadlet semantics differ from Docker Compose.

Be especially careful with:

- Compose interpolation and default syntax
- `depends_on` readiness assumptions
- complex `deploy` blocks
- multi-file Compose merges
- secrets and credentials
- permission behavior on rootless bind mounts

If a mapping is lossy, say so directly and explain the concrete risk.

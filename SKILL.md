---
name: quadlet-migrator
description: Convert docker run commands or Docker Compose configurations into maintainable Podman Quadlet files, default to writing reviewable output in the current directory, generate helper install/reload/start/stop/restart scripts, carry along required repo-local companion files such as config templates or init assets, help users map env/env_file/.env usage into Environment or EnvironmentFile, and explain rootless or rootful deployment details and migration risks.
---

# Quadlet Migrator

Use this skill when the user wants to migrate `docker run`, `docker compose`, or Compose-like service definitions into Podman Quadlet units, especially when environment variables, `env_file`, `.env`, rootless deployment, systemd layout, review-first generation into the current directory, or helper management scripts need to be planned or generated.

Do not rely on `podlet` as an execution dependency. You may use it only as prior-art knowledge, not as part of the runtime workflow.

## What this skill does

This skill helps you:

- discover Docker deployment entry points from a GitHub repository URL
- translate `docker run` flags and Compose service fields into Quadlet units
- choose between `.container`, `.pod`, `.network`, `.volume`, and `.build`, with a pod-first bias for multi-container services
- decide whether values belong in `Environment=` or `EnvironmentFile=`
- write reviewable output to the current directory by default before the user applies it to a live Quadlet search path
- generate helper scripts with `install.sh` as the canonical apply script name, plus `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` when producing runnable artifacts
- identify required repo-local companion files such as config files, templates, seed data, or initialization assets that must be shipped alongside Quadlet output for the deployment to run correctly
- validate env completeness before claiming runnable output, including missing required keys and suspicious env-key mismatches
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
   - identify repo-local companion files referenced by bind mounts, docs, `entrypoint`, `command`, or wrapper scripts, and decide whether they belong in the deliverable set
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
   - repo-local companion files required at runtime, such as config files, templates, migrations, seed data, or initialization assets
   - repo-local entrypoint scripts, helper scripts, and bind-mounted directories that must remain part of the deliverable set

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

If Compose files, bind mounts, startup docs, entrypoints, or helper scripts reference repo-local files or whole directories, treat those assets as candidates for the final deliverable set rather than incidental source files.
Do not assume runnable output is complete when Quadlet files exist but required mounted config, init assets, or startup scripts have not been identified.

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
   - if the project is a simple single-container deployment, a standalone `.container` is the default
   - if one logical service contains multiple containers, prefer keeping them in the same `.pod` so they share one network namespace
   - even when the source Compose topology uses bridge networking, prefer pod-based grouping over preserving bridge semantics mechanically
   - containers in the same `.pod` can communicate over `127.0.0.1` / `localhost` because they share a network namespace
   - when `Pod=` is set, never generate `AddHost=` entries whose purpose is sibling-container discovery inside that pod; intra-pod communication must use `127.0.0.1` / `localhost` instead
   - `AddHost=` remains a host-to-IP override, not an intra-pod service-discovery mechanism; because upstream Quadlet supports `AddHost=` in both `[Container]` and `[Pod]`, do not claim that `Pod=` categorically forbids `AddHost=` unless the upstream reference says so for the specific case
   - when containers are attached with `Pod=<name>.pod`, treat the pod's generated systemd service as the primary lifecycle unit; derive that service name from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name. Starting that pod service brings up the pod-managed containers, so do not add redundant per-container start commands for those child units in helper scripts
   - containers in different pods must not be treated as reachable via `127.0.0.1` / `localhost`; if you split the topology across multiple pods or preserve a shared bridge network, use container names, pod names, or explicit `NetworkAlias=` values on the shared network instead
   - `ServiceName=` controls the generated systemd unit name only and must not be treated as an application-facing network address
   - `PodName=` controls the Podman pod name only and may be part of the chosen addressing strategy, but it does not determine the systemd service name
   - if one pod is not practical because of port conflicts or clearly incompatible groupings, split the result into a small number of pods rather than forcing an awkward topology
   - avoid `.network` / bridge-first designs unless pod topology cannot express the intended deployment cleanly

2. Freeze the storage strategy.
   - named volume, bind mount, or anonymous volume per storage use case
   - bind mounts must end up as absolute host paths
   - preserve bind-mount shape from the source input: a file bind mount must stay a file bind mount, and a directory bind mount must stay a directory bind mount
   - do not widen a file mount into a directory mount, or collapse a directory mount into a file mount, unless the source is genuinely ambiguous or the upstream deployment docs explicitly require a different reviewed mapping

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
   - do not treat a variable as satisfied unless it is present in an actual final source such as `Environment=`, the final `EnvironmentFile=`, or an explicitly preserved default
   - if a likely required startup variable is still absent from the final output, keep it unresolved instead of downgrading it to an informational note
   - if referenced env keys and final env keys contain likely near-match typos, call them out explicitly before execution

5. Summarize already-known conflicts and their chosen resolution.
   - port collisions
   - incompatible grouping decisions
   - storage mode inconsistencies
   - unresolved required variables
   - suspicious env-key mismatches or typo candidates
   - missing required repo-local support files or directories
   - mismatch between requested deployment mode and selected file locations

At the end of finalize, write `QUADLET-FINALIZE.md` and ask the user to review it before you start writing the final artifacts.

Before finalizing, use this checklist template:

- [ ] support-file set identified, including mounted config, entrypoint scripts, init assets, and bind-mounted directories
- [ ] support files classified as upstream-preserved, locally generated, or locally rewritten
- [ ] env keys classified into satisfied, unresolved, default-derived, and typo-suspect states
- [ ] finalized file list reflects everything the runtime needs, not just Quadlet units
- [ ] any remaining placeholders are explicit and understood as non-runnable until filled


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
- env completeness status, including unresolved required variables and suspicious typo candidates
- repo-local support files and directories that must be copied, preserved, or rendered for the deployment to run
- which support files come from upstream unchanged versus which are generated or rewritten locally
- the intended apply target directory for rootless or rootful deployment
- the intended host-side destination paths for required support files, scripts, config trees, and initialization assets
- only the minimal placeholders that still cannot be resolved without user secrets or environment-specific values
- detected conflicts and how they were resolved
- the list of files that will be created in execution phase, including generated Quadlet files, any env file or env delta, and helper scripts such as `install.sh`, `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` when applicable; do not introduce a parallel `apply.sh` name unless the user explicitly asks for it

Do not start execution until the user has reviewed and confirmed `QUADLET-FINALIZE.md` or provided requested edits.

Do not use `QUADLET-FINALIZE.md` as the first place where the user sees important choices. Those choices should already have been raised in planning.

### Execution phase

Goal: write the approved runnable artifacts.

Tasks in this phase:

1. Generate the Quadlet files in the current working directory by default so the user can review them before applying them.
2. Generate the env file or env delta only when needed for runnable output.
3. Generate helper scripts such as `install.sh`, `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` when they materially help the user apply and operate the result.
   - Use `install.sh` as the default and canonical script name for applying the reviewed artifact set.
   - Do not also generate `apply.sh` unless the user explicitly asks for that alternate name.
   - `install.sh` should copy only Quadlet unit files into the chosen Quadlet target directory. Required env files, mounted config, scripts, and other runtime support files should remain in the reviewed current-directory deliverable set and be referenced from Quadlet via absolute host paths.
   - `install.sh` should not start, stop, restart, or reload services as a side effect.
   - `uninstall.sh` should remove only the previously installed reviewed Quadlet unit files from the chosen Quadlet target directory.
   - `uninstall.sh` should stop affected services before removing their installed unit files, and should not delete support files from the current-directory deliverable set, unrelated files, shared directories, named volumes, images, or Podman objects unless the user explicitly asks for that broader cleanup.
   - `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` should manage services only and should not silently install or overwrite files.
   - when a generated topology includes `<name>.pod` plus child containers linked with `Pod=<name>.pod`, make the pod service the lifecycle entrypoint in helper scripts; derive that service name from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name. Do not emit redundant `systemctl start/stop/restart` lines for each child container that is already managed through the pod service.
4. If the deployment depends on repo-local support files or directories, generate or copy those reviewed artifacts into the current-directory deliverable set as well.
5. Do not claim runnable output until the final env sources and support-file set are complete enough for minimal startup.
6. Generate deployment notes or validation guidance only when they materially help the user operate the result.
7. Generate a README only when the user explicitly wants a self-contained handoff artifact or a packaged deliverable.

Execution should follow the approved contents of `QUADLET-FINALIZE.md`. If the implementation reveals a material conflict with the finalized design, stop and return to planning rather than silently diverging.

Before calling the result runnable, pass this gate:

- the generated artifact set includes all required support files and directories
- every referenced `EnvironmentFile=` exists in the deliverable set and contains the required keys
- startup-critical env values are either present or explicitly unresolved
- suspected env typos have been resolved or surfaced to the user
- install, uninstall, reload, and service-management scripts match the approved artifact set

Use this execution checklist template:

- [ ] support-file set copied, generated, or preserved in the deliverable tree
- [ ] every `EnvironmentFile=` path resolves to an actual reviewed file
- [ ] startup-critical env keys present, or explicitly marked as unresolved placeholders
- [ ] support files, scripts, and config trees map to the correct host-side destination paths
- [ ] runnable-output gate passed before describing the result as runnable
- [ ] helper scripts operate on the same reviewed artifact set that finalize approved


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
- required repo-local support files or directories referenced by mounts, docs, commands, or scripts have not been identified confidently
- required runtime files are known but are still missing from the planned deliverable set
- required env values for minimal service startup are still missing from the final env sources
- likely env-key typos or mismatches remain unresolved

Do not keep moving forward by guessing through these gaps.

## Rootless vs rootful

Decide early whether the deployment is rootless or rootful, because this changes the apply target path and some operational guidance.

- By default, generate reviewable artifacts in the current working directory first.
- For rootless deployments, the default apply target directory is `~/.config/containers/systemd/` unless the user has a reason to use another supported path.
- For rootful deployments, the default apply target directory is `/etc/containers/systemd/` unless the user asks for a different placement.
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

- review the generated files in the current directory
- apply the reviewed Quadlet files into the correct Quadlet directory while keeping support files in the current directory at the absolute paths referenced by the units
- run `systemctl daemon-reload` or `systemctl --user daemon-reload`
- create required bind-mount directories before first start
- verify generator output or systemd unit validity when startup fails

## Examples

- `docker run` for a single web service -> often `advice` or `generate` with one `.container`
- small Compose app with api and db -> usually `design` or `generate`, often one `.pod` plus child containers
- GitHub repo with `.env.example` and multiple profiles -> start in `Planning`, reduce the env questions, then move to `Finalize`
- review-first runnable output -> `generate` often writes Quadlet files plus `install.sh`, `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` into the current directory, with `install.sh` as the canonical apply step

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

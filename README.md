# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

A skill that helps turn `docker run` commands and Docker Compose setups into Podman Quadlet files you can review, adjust, and apply.

## What it does

- converts `docker run` commands and Docker Compose setups into Podman Quadlet files
- writes the generated files to the current directory first so you can review them before installing them
- asks about the output location only when you already requested another location or existing files would conflict
- helps choose between `.container`, `.pod`, `.network`, `.volume`, and `.build`, usually preferring a pod for related multi-container services
- keeps `.env` / `env_file` workflows when they still fit the deployment
- turns large env templates into a short list of decisions the user actually needs to make
- can generate helper scripts such as `install.sh`, `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh`
- finds files from the current project that the service still needs when it runs, such as mounted config, setup data, and helper scripts
- checks that env files are complete before calling the result runnable
- asks the user to confirm important deployment choices during planning, then uses clear review and execution checklists
- can optionally plan `AutoUpdate=registry` when the chosen image uses a complete image name that includes the registry, such as `docker.io/...` or `ghcr.io/...`
- explains rootless vs rootful install paths, deployment notes, and validation steps

## Design principles

- use the simplest mode that fits the request
- keep planning, review, and file generation as separate steps
- do not invent deployment-specific values
- call out behavior changes when a mapping is lossy
- prefer output that is easy to understand and maintain
- write files to the current directory for review before installation
- prefer pod-based grouping when it is the clearest fit for a multi-container service
- keep required extra files in the reviewed output and point to them with absolute paths on the host machine instead of copying them into the Quadlet unit directory

## Operating modes

- `advice`: explain the mapping, review source inputs, or answer targeted questions without writing final files
- `design`: do planning and a final interactive review, then stop before generating runnable files
- `generate`: do planning, the final interactive review, and execution, then generate the approved runnable files

## Workflow

The workflow has three phases: `Planning`, `Finalize`, and `Execution`.

- `advice` usually stays in `Planning` or answers a focused question directly
- `design` includes `Planning` and `Finalize`
- `generate` includes all three phases

Planning is where unresolved deployment decisions are gathered and confirmed with the user.
Finalize is a review step in the conversation after those decisions have been discussed.
Execution starts only after the user approves that review.

## References

- `SKILL.md` contains the operating modes, workflow, and high-level rules
- `references/compose-mapping.md` covers field mapping and topology decisions
- `references/env-strategy.md` covers env handling, completeness validation, and typo detection
- `references/github-repo-intake.md` covers how the skill finds the right repository entry point
- `references/deployment-notes.md` covers deployment guidance
- `references/validation.md` covers validation and troubleshooting
- `references/template/` contains simple helper-script templates for stable prefix-based Quadlet file handling and lifecycle management

## Limitations

This skill does not claim perfect equivalence between Docker Compose semantics and Podman Quadlet semantics.

## License

MIT

# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

A skill for migrating `docker run` commands and Docker Compose-style deployments into maintainable Podman Quadlet units.

## What it does

- translates `docker run` and Compose-style inputs into Quadlet-oriented designs
- helps decide between `.container`, `.pod`, `.network`, `.volume`, and `.build`
- preserves `.env` / `env_file` workflows when appropriate
- reduces large env templates into a small set of high-impact deployment questions
- explains rootless vs rootful placement, deployment notes, and validation steps

## Design principles

- prefer the lightest operating mode that matches the request
- separate planning, review, and generation into explicit phases
- do not invent deployment-specific values
- make lossy mappings explicit
- prefer maintainable output over mechanical one-to-one translation

## Operating modes

- `advice`: explain mappings or review source inputs without writing final artifacts
- `design`: perform planning and finalize review, then stop before runnable artifact generation
- `generate`: produce approved runnable artifacts after planning and finalize review

## References

- `SKILL.md` contains the operating modes, workflow, and high-level rules
- `references/compose-mapping.md` covers field mapping and topology decisions
- `references/env-strategy.md` covers env handling and secret defaults
- `references/github-repo-intake.md` covers repository discovery and canonical input selection
- `references/deployment-notes.md` covers deployment guidance
- `references/validation.md` covers validation and troubleshooting

## Limitations

This skill does not claim perfect equivalence between Docker Compose semantics and Podman Quadlet semantics.

## License

MIT

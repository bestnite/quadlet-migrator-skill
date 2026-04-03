# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

A skill for migrating `docker run` commands and Docker Compose-style deployments into maintainable Podman Quadlet units.

## What it does

- translates `docker run` and Compose-style inputs into Quadlet-oriented designs
- writes generated artifacts to the current directory by default so they can be reviewed before being applied
- helps decide between `.container`, `.pod`, `.network`, `.volume`, and `.build`, with a pod-first bias for multi-container services
- preserves `.env` / `env_file` workflows when appropriate
- reduces large env templates into a small set of high-impact deployment questions
- can generate helper scripts with `install.sh` as the canonical apply step, plus `uninstall.sh`, `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh`
- identifies required repo-local support files such as mounted config, init assets, and helper scripts that must ship with the result
- validates env completeness before claiming runnable output
- encourages explicit finalize and execution checklists for support files and env completeness
- explains rootless vs rootful apply targets, deployment notes, and validation steps

## Design principles

- prefer the lightest operating mode that matches the request
- separate planning, review, and generation into explicit phases
- do not invent deployment-specific values
- make lossy mappings explicit
- prefer maintainable output over mechanical one-to-one translation
- default to review-first output in the current directory before installation
- prefer pod-first topology over preserving bridge networking when pod grouping expresses the intent cleanly
- ensure runtime-required support files remain in the reviewed current-directory deliverable set and are referenced from Quadlet via absolute host paths, rather than being copied by `install.sh` into the Quadlet unit directory

## Operating modes

- `advice`: explain mappings or review source inputs without writing final artifacts
- `design`: perform planning and finalize review, then stop before runnable artifact generation
- `generate`: produce approved runnable artifacts after planning and finalize review

## References

- `SKILL.md` contains the operating modes, workflow, and high-level rules
- `references/compose-mapping.md` covers field mapping and topology decisions
- `references/env-strategy.md` covers env handling, completeness validation, and typo detection
- `references/github-repo-intake.md` covers repository discovery and canonical input selection
- `references/deployment-notes.md` covers deployment guidance
- `references/validation.md` covers validation and troubleshooting

## Limitations

This skill does not claim perfect equivalence between Docker Compose semantics and Podman Quadlet semantics.

## License

MIT

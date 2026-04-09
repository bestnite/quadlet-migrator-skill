# Deployment Notes

Use this file when the user wants deployment-ready instructions alongside generated Quadlet units.

## Delivery flow

For `design` mode, stop after the user reviews the finalize snapshot in conversation.
For `generate` mode, continue only after the user has reviewed and confirmed the finalize snapshot or requested edits.

1. Generate the reviewable artifacts in the current working directory by default, or in another user-requested output directory.
2. Resolve high-impact planning questions with the user before freezing the design.
3. Review the finalized design snapshot together with the generated Quadlet files, env files, helper scripts, and any required repo-local support files or directories.
4. Use `install.sh` to copy only the reviewed Quadlet unit files into the chosen Quadlet directory.
5. Run the appropriate `daemon-reload` command.
6. Use `start.sh`, `stop.sh`, and `restart.sh` to manage the deployed services.
7. Use `uninstall.sh` when the user wants to remove the installed reviewed Quadlet unit files without broad Podman cleanup.

Do not start execution until the user has reviewed and confirmed the finalize snapshot or requested edits.

## Apply target directory

### Rootless

- default apply target: `~/.config/containers/systemd/`
- user-scoped management commands use `systemctl --user`

### Rootful

- default apply target: `/etc/containers/systemd/`
- system-scoped management commands use `systemctl`

See `podman-systemd.unit.5.md` for the full search-path matrix.

## Helper scripts

- `install.sh`: canonical apply script; copy only reviewed Quadlet unit files into the selected Quadlet target directory
- do not generate a separate `apply.sh` by default; reserve that alternate name only when the user explicitly asks for it
- helper shell scripts must discover the reviewed Quadlet files through their shared generated prefix, using shared-prefix glob matching such as `<prefix>*` instead of hardcoding exact filenames or assuming a fixed file count
- `install.sh` must not start, stop, restart, or reload services as a side effect
- `uninstall.sh`: remove the installed reviewed Quadlet unit files from the selected Quadlet target directory
- `uninstall.sh` should stop affected services before removing their installed unit files, and should not delete support files from the current-directory deliverable set, unrelated files, shared directories, named volumes, images, or Podman objects unless the user explicitly asks for broader cleanup
- `reload.sh`: run only the appropriate `daemon-reload` command after installation changes
- `start.sh`, `stop.sh`, `restart.sh`: manage services only and must not silently install or overwrite reviewed files
- when a generated topology includes `<name>.pod` plus child containers linked with `Pod=<name>.pod`, helper scripts should use the pod service as the lifecycle entrypoint; derive that service name from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name. Do not add `ServiceName=` merely to simplify helper scripts, and do not duplicate per-container lifecycle commands for child units

Keep installation separate from service-management scripts so the user can review generated files before applying them.
Do not add explicit runtime naming directives such as `PodName=`, `ServiceName=`, `ContainerName=`, or `NetworkName=` by default. Use Quadlet's derived names unless the user explicitly asks for custom naming or a reviewed requirement depends on it.
Do not use `ServiceName=` as an application connection target. It controls the generated systemd unit name only. When services communicate over a shared network outside a single pod namespace, prefer container names, pod names, or explicit `NetworkAlias=` values.
Within a single pod, use `127.0.0.1` / `localhost` for container-to-container communication instead of generating `AddHost=` entries whose purpose is sibling-container discovery.
If a service inside the pod must accept connections from sibling containers, ensure its effective listen address is reachable within the shared pod namespace, typically `127.0.0.1` or `0.0.0.0`. When the upstream service exposes this through environment variables or similar runtime configuration, preserve or generate that setting explicitly.

## Review checklist before install

Review not only the Quadlet unit files but also:

- env files referenced by `EnvironmentFile=`
- repo-local mounted config files and directory trees
- initialization files such as `init.sql`, seed data, or bootstrap assets
- repo-local entrypoint and helper scripts referenced by `Entrypoint=`, `Exec=`, docs, or wrapper scripts

Do not treat the deliverable as complete if these support files are still missing from the reviewable artifact set.

Execution checklist template before install:

- [ ] all reviewed artifacts are present in the current-directory deliverable tree
- [ ] required support files and directories are included alongside the Quadlet and env artifacts
- [ ] every `EnvironmentFile=` path resolves to an actual reviewed file
- [ ] support files remain in the current-directory deliverable tree at the absolute paths referenced by mounts and scripts
- [ ] startup-critical env keys are present, or explicitly marked as unresolved placeholders
- [ ] any unresolved values are clearly marked as intentionally non-runnable placeholders
- [ ] service-management scripts operate on the same reviewed artifact set that finalize approved
- [ ] runnable-output gate passed before describing the result as runnable

## Rootless operational notes

- Bind mounts may hit UID/GID mismatches.
- Do not add `User=`, `Group=`, or `UserNS=keep-id` by default. Consider them only when the user is working through container permission or ownership behavior, or when the source explicitly requires that runtime identity mapping.
- If the service must survive logout, mention lingering:

```bash
sudo loginctl enable-linger <username>
```

## Paths and bind mounts

- Ensure bind-mount source directories exist before first start.
- Normalize relative source paths against the source Compose file directory or the directory the user specifies.
- Emit absolute host paths in generated Quadlet files when using bind mounts.
- Explain the resolved absolute path if the source used `./...`.
- If the source project bind-mounts repo-local files or directories, make sure the reviewed current-directory deliverable set preserves the required contents and that the generated Quadlet files reference their absolute paths correctly.

## Recommended service defaults

Depending on the workload, consider adding:

```ini
[Service]
Restart=always
TimeoutStartSec=900
```

Use the timeout especially when first start may need to pull large images or build locally.

## Useful optional enhancements

- `AutoUpdate=registry` for opt-in automatic image refresh workflows when the approved image strategy uses fully qualified registry images
- explicit `.volume` or `.network` units when the user wants declarative infrastructure instead of implicit Podman objects

## Output language

If you generate a README, deployment note, or operator-facing document as part of the migration, write it in the user's language unless the user explicitly asks for another language.

# Deployment Notes

Use this file when the user wants deployment-ready instructions alongside generated Quadlet units.

## Delivery flow

1. Generate the reviewable artifacts in the current working directory.
2. Review the generated Quadlet files, env files, helper scripts, and any required repo-local support files or directories.
3. Use `install.sh` to copy only the reviewed Quadlet unit files into the chosen Quadlet directory.
4. Use `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` to manage the deployed services.
5. Use `uninstall.sh` when the user wants to remove the installed reviewed Quadlet unit files without broad Podman cleanup.

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
- `uninstall.sh`: remove the installed reviewed Quadlet unit files from the selected Quadlet target directory, stopping affected services first when needed
- `reload.sh`: run the appropriate `daemon-reload` command after installation changes
- `start.sh`: start the generated units; when the topology uses a `.pod`, start the pod's systemd service derived from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name, instead of also starting each child container service individually
- `stop.sh`: stop the generated units; when the topology uses a `.pod`, stop the pod's systemd service derived from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name, instead of duplicating per-container stop commands for its child containers
- `restart.sh`: restart the generated units after reload or config changes; when the topology uses a `.pod`, restart the pod's systemd service derived from `ServiceName=` when present on the `.pod`, otherwise use Quadlet's default generated pod service name, instead of also restarting each child container service individually

Keep installation separate from service-management scripts so the user can review generated files before applying them.
`install.sh` should copy reviewed Quadlet unit files into the chosen Quadlet target directory only, and should not start, stop, restart, or reload services as a side effect.
`uninstall.sh` should remove only the installed reviewed Quadlet unit files, stop affected services before removal when needed, and leave the support files in the current-directory deliverable set, unrelated files, shared directories, named volumes, images, and Podman objects alone unless the user explicitly asks for broader cleanup.
`reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` should not silently install or overwrite reviewed files.
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
- [ ] unit files map to the intended Quadlet directory
- [ ] support files remain in the current-directory deliverable tree at the absolute paths referenced by mounts and scripts
- [ ] startup-critical env keys are present in the final env sources
- [ ] any unresolved values are clearly marked as intentionally non-runnable placeholders
- [ ] service-management scripts operate on the same reviewed artifact set that will be installed

## Rootless operational notes

- Bind mounts may hit UID/GID mismatches.
- For pod-based deployments that should preserve host ownership semantics, consider `UserNS=keep-id` on `[Pod]` when appropriate.
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

- `AutoUpdate=registry` for opt-in automatic image refresh workflows
- explicit `.volume` or `.network` units when the user wants declarative infrastructure instead of implicit Podman objects

## Output language

If you generate a README, deployment note, or operator-facing document as part of the migration, write it in the user's language unless the user explicitly asks for another language.

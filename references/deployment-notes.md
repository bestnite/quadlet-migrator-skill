# Deployment Notes

Use this file when the user wants deployment-ready instructions alongside generated Quadlet units.

## Delivery flow

1. Generate the reviewable artifacts in the current working directory.
2. Review the generated Quadlet files, env files, helper scripts, and any required repo-local support files or directories.
3. Use `install.sh` to copy only the reviewed unit files into the chosen Quadlet directory. Copy env files and any other required runtime support files into the correct host-side paths the deployment expects.
4. Use `reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` to manage the deployed services.

## Apply target directory

### Rootless

- default apply target: `~/.config/containers/systemd/`
- user-scoped management commands use `systemctl --user`

### Rootful

- default apply target: `/etc/containers/systemd/`
- system-scoped management commands use `systemctl`

See `podman-systemd.unit.5.md` for the full search-path matrix.

## Helper scripts

- `install.sh`: canonical apply script; copy only reviewed Quadlet unit files into the selected Quadlet target directory, and copy env files plus any other required runtime support files into the correct host-side paths
- do not generate a separate `apply.sh` by default; reserve that alternate name only when the user explicitly asks for it
- `reload.sh`: run the appropriate `daemon-reload` command after installation changes
- `start.sh`: start the generated units
- `stop.sh`: stop the generated units
- `restart.sh`: restart the generated units after reload or config changes

Keep installation separate from service-management scripts so the user can review generated files before applying them.
`install.sh` should copy reviewed Quadlet unit files into the chosen Quadlet target directory and place required runtime support files into their correct host-side destinations only, and should not start, stop, restart, or reload services as a side effect.
`reload.sh`, `start.sh`, `stop.sh`, and `restart.sh` should not silently install or overwrite reviewed files.

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
- [ ] support files map to the correct host-side runtime paths for mounts and scripts
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
- If the source project bind-mounts repo-local files or directories, make sure the installed artifact set preserves the required contents and places them at the correct host-side paths expected by the mounts.

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

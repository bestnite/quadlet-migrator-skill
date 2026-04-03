# Deployment Notes

Use this file when the user wants deployment-ready instructions alongside generated Quadlet units.

## Directory choice

### Rootless

- primary default: `~/.config/containers/systemd/`
- user-scoped management commands use `systemctl --user`

### Rootful

- primary default: `/etc/containers/systemd/`
- system-scoped management commands use `systemctl`

See `podman-systemd.unit.5.md` for the full search-path matrix.

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

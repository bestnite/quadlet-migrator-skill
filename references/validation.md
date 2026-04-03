# Validation

Use this file when the user asks how to verify or troubleshoot generated Quadlet units.

## Basic deployment flow

### Rootless

```bash
systemctl --user daemon-reload
systemctl --user start <unit>
systemctl --user status <unit>
```

### Rootful

```bash
systemctl daemon-reload
systemctl start <unit>
systemctl status <unit>
```

## Generator debugging

Use the Podman systemd generator dry run when units fail to appear or options look unsupported.

```bash
/usr/lib/systemd/system-generators/podman-system-generator --dryrun
```

For rootless debugging:

```bash
/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun
```

## Systemd verification

```bash
systemd-analyze verify <unit>.service
```

For user units:

```bash
systemd-analyze --user verify <unit>.service
```

## Common failure causes

- unsupported Quadlet option for the installed Podman version
- bind-mount source directory missing
- wrong rootless or rootful unit directory
- unresolved env file path
- permissions on rootless bind mounts
- readiness assumptions hidden behind `depends_on`

## Troubleshooting posture

When validation fails, report:

- what generated successfully
- what failed to generate or start
- whether the issue is syntax, unsupported feature, path resolution, or permissions

## Relationship to execution phase

Validation belongs after the files are written in the execution phase.

Before execution, the skill should already have completed planning and finalize review with the user. Do not treat validation as a substitute for design review.

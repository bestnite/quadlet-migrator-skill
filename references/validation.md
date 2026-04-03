# Validation

Use this file when the user asks how to verify or troubleshoot generated Quadlet units.

## Basic deployment flow

1. Review the generated files in the current working directory and confirm the expected Quadlet units, env files, helper scripts, and required repo-local support files exist.
2. Run `install.sh` to copy only the reviewed Quadlet unit files into the target Quadlet directory.
3. Run the appropriate reload command.
4. Start the relevant units and inspect their status.
5. If needed, run `uninstall.sh` to remove the installed reviewed artifact set before regenerating or abandoning the deployment.

If the user requested an alternate apply script name explicitly, substitute that name where needed, but treat `install.sh` as the default documentation path.

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

## Support-file and env checks

Before calling the result runnable, verify that:

- every referenced `EnvironmentFile=` exists at the path referenced by the installed unit
- required env keys are actually present in the final env sources
- bind-mounted files and directories exist at the absolute paths referenced by the generated Quadlet files
- bind-mounted file-versus-directory shape still matches the source input
- repo-local entrypoint or helper scripts referenced by the container exist and are executable when needed
- initialization assets such as `init.sql`, seeds, bootstrap files, or config templates are present where the deployment expects them

Runnable-output gate checklist template:

- [ ] the support-file set is complete
- [ ] env completeness check passed against the actual final env sources
- [ ] unit files are installed in the intended Quadlet directory
- [ ] support files remain available at the absolute paths expected by mounts and scripts
- [ ] bind-mounted file-versus-directory shape still matches the source input
- [ ] service-management scripts operate on the same artifact set that was reviewed
- [ ] no required support file, env key, or typo-suspect mismatch remains unresolved

Do not call the result runnable until every item above is checked.

## Common failure causes

- unsupported Quadlet option for the installed Podman version
- bind-mount source directory missing
- files were generated but `install.sh` has not yet copied the unit files into the target rootless or rootful unit directory
- wrong rootless or rootful apply target directory
- unresolved env file path
- required env key missing from the final env file
- likely env-key typo or mismatch between source docs and final env output
- required repo-local config, init assets, or helper scripts missing from the installed artifact set
- permissions on rootless bind mounts
- readiness assumptions hidden behind `depends_on`

## Troubleshooting posture

When validation fails, report:

- what generated successfully
- what was applied successfully
- what failed to generate, apply, or start
- whether the issue is syntax, unsupported feature, path resolution, installation path, missing support files, missing env keys, or permissions

## Relationship to execution phase

Validation belongs after the files are written in the execution phase, the Quadlet units are applied to a valid Quadlet directory, and the referenced support files remain available at the absolute host paths used by the generated units.

Before execution, the skill should already have completed planning and finalize review with the user. Do not treat validation as a substitute for design review.

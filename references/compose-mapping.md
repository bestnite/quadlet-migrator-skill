# Compose Mapping

Use this file when converting `docker-compose.yml` or `compose.yaml` into Quadlet units.

## General approach

- Model each service first, then decide how to group all resulting containers into one or more `.pod` units.
- Prefer maintainable Quadlet output over mechanical one-to-one translation.
- Keep names stable and predictable. Generated filenames should use a shared project prefix.

## Field strategy

### `name`

- Use as an application prefix when it improves unit naming clarity.
- Do not force a top-level project name into every filename if the user prefers shorter units.

### `services.<name>.image`

- Map to `Image=` in `[Container]`.
- Prefer fully qualified image names when normalizing output.
- If the source omits a registry and is using Docker Hub semantics, normalize it explicitly for Podman.
- Use these rules when filling in Docker Hub references:
  - `redis:7` -> `docker.io/library/redis:7`
  - `nginx` -> `docker.io/library/nginx`
  - `langgenius/dify-api:latest` -> `docker.io/langgenius/dify-api:latest`
- Do not guess `docker.io/library/...` for images that already include a namespace.

### `services.<name>.build`

- Prefer upstream published images over local builds when the project already documents supported registry images.
- If the user wants declarative builds, create a `.build` unit and reference it from `Image=`.
- If the build semantics are too custom, or if an equivalent upstream image is clearly available, keep this as a manual follow-up item instead of guessing.

### `container_name`

- Drop it.
- Do not generate `ContainerName=`. Let Podman and systemd derive the runtime name.

### `ports`

- For a standalone service, map to `PublishPort=` on the `.container`.
- For a pod-based topology, prefer `PublishPort=` on the `.pod` when the published ports belong to the pod boundary rather than one child container.

### `volumes`

- Bind mounts become `Volume=HOST:CONTAINER[:OPTIONS]`.
- Normalize relative host paths against the Compose file directory and emit absolute paths in the final Quadlet output.
- Named volumes can remain referenced by name, but when the user wants explicit infrastructure-as-code, create matching `.volume` units.
- Ask the user which volume mode they want when the source does not make the intended persistence model obvious.
- If a bind mount points to a repo-local file or directory, include that source in the reviewable deliverable set unless the user explicitly wants a host-managed external path instead.
- If a bind mount references a whole directory, inspect and preserve the required directory contents rather than only naming the directory root.

### `networks`

- Prefer pod-first topology over preserving Compose bridge networks mechanically.
- If the source uses a default network only, you often do not need a `.network` unit at all.
- If the source uses bridge networking for containers that can reasonably live in one pod, collapse that topology into one `.pod` so the containers share one network namespace.
- Create a `.network` unit only when services must be split across pods, or when explicit network isolation or custom network management is materially required.
- Containers in the same `.pod` can communicate over `127.0.0.1` / `localhost` because they share a network namespace.
- When `Pod=` is set, never generate `AddHost=` entries whose purpose is sibling-container discovery inside that pod; intra-pod communication should use `127.0.0.1` / `localhost` instead.
- `AddHost=` is a host-to-IP override, not an intra-pod service-discovery mechanism. Upstream Quadlet documents `AddHost=` in both `[Container]` and `[Pod]`, so do not describe `Pod=` as a blanket prohibition on `AddHost=` unless the upstream reference explicitly requires that for the case at hand.
- Containers in different pods must not be treated as reachable via `127.0.0.1` / `localhost`.
- When splitting services across multiple pods or preserving a shared bridge network, use container names, pod names, or explicit `NetworkAlias=` values on the shared network, or publish ports to the host boundary when that is the intended access pattern.
- `ServiceName=` controls the generated systemd unit name only and must not be used as an application-facing network address.
- `PodName=` controls the Podman pod name only; it can participate in the chosen addressing strategy, but it does not determine the systemd service name.

### `environment`

- Small stable values can become one or more `Environment=` lines.
- Sensitive values should be moved to `EnvironmentFile=` unless the user explicitly wants them inline.

### `env_file`

- Prefer `EnvironmentFile=`.
- If there are multiple env files, preserve order and explain precedence if the user asks.

### `.env` interpolation

- Resolve only when you have the actual source values.
- If variables are missing, surface a missing-variable list.
- Never silently replace unknown values with blanks.

### `profiles`

- Decide first which profiles are actually part of the desired deployment.
- Do not try to preserve Compose profiles as a direct Quadlet concept.
- Treat profiles as source selection inputs that decide which services become units.

### `depends_on`

- Translate to `Requires=` and `After=` when that reflects intent.
- State clearly that this controls startup ordering, not application readiness.

### `healthcheck`

- Prefer dedicated Quadlet health fields such as `HealthCmd=`, `HealthInterval=`, `HealthTimeout=`, `HealthRetries=` when representable.
- If the Compose healthcheck is only partially representable, preserve the command intent and call out missing knobs.

### `command` and `entrypoint`

- `entrypoint` typically maps to `Entrypoint=`.
- `command` typically maps to `Exec=`.
- If an entrypoint or helper script is repo-local, treat it as a support file that must be copied or preserved in the generated layout.

### `user`

- Map to `User=` and `Group=` in `[Container]` only when it is a container runtime user mapping, not a systemd service user.
- Do not use systemd `User=` to try to make a rootless Quadlet run as another login user.

### unsupported or risky fields

Handle these conservatively and usually as migration notes:

- `deploy`
- `profiles`
- `extends`
- advanced Compose merge behavior
- readiness semantics hidden behind `depends_on`

## Minimal examples

### Single service to standalone container

Source intent:

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
```

Reasonable result shape:

```ini
[Container]
Image=docker.io/library/nginx:latest
PublishPort=8080:80
```

Use this when the deployment is truly a simple single-service case. A single container should usually stay a standalone `.container` rather than being wrapped in a pod.

### Small multi-service app to one pod

Source intent:

```yaml
services:
  api:
    image: ghcr.io/example/api:1.0
    depends_on:
      - db
  db:
    image: postgres:16
```

Reasonable result shape:

- one `.pod` for the application boundary
- one container unit for `api`
- one container unit for `db`
- `api` may reach `db` over `127.0.0.1` / `localhost` because both containers share the pod network namespace
- ordering hints for startup, while explicitly noting that `depends_on` does not guarantee readiness

Use this as the default shape for a small multi-container service unless port conflicts or incompatible grouping force a split into multiple pods.

## Pod decision rule

Choose the simplest topology that preserves the source deployment intent.

Prefer a single `.pod` for multi-container applications when practical.

If one logical service contains multiple containers, default to putting them in the same `.pod` so they share networking and lifecycle.

If the project is a simple single-container deployment with no real need for pod semantics, a standalone `.container` is the preferred result.

If one pod is not practical because of port conflicts or clearly incompatible groupings, split the result into a small number of pods rather than forcing an awkward topology.

When services are split across multiple pods, do not rely on `127.0.0.1` / `localhost` for cross-pod communication. Use container names, pod names, or explicit `NetworkAlias=` values on the shared network instead.

Avoid preserving bridge networks by default when pod grouping already expresses the intended communication pattern well.

For large application stacks with optional services, ask the user to choose the desired service set before generating a minimized result.


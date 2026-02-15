# caktus.podman-containers

An Ansible role to run Podman containers on Ubuntu or RHEL-compatible servers.

## Configuration

See `defaults/main.yml` for all supported variables, defaults, and examples.

## Key Features

- **Networks or Pods**: Use networks for independent container management, or pods for tightly coupled containers
- **Quadlets** (Podman 4.4+): Declarative `.container`/`.network`/`.pod` files for systemd integration
- **Environment Files**: Write secrets to files with restricted permissions
- **Git Builds**: Clone repos and build images with Podman

## Quick Start

```yaml
podman_user: mydeployuser
podman_user_home: /home/mydeployuser

podman_use_quadlets: yes
podman_default_network: app

podman_networks:
  - name: "{{ podman_default_network }}"

podman_containers:
  - name: whoami
    image: docker.io/traefik/whoami
    tag: latest
    # No network specified - uses podman_default_network
    publish:
      - "8080:80"
```

After deploying, test with `curl http://localhost:8080`.

See [Podman Quadlet docs](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) for `quadlet_options`.

## Upgrade Guide

This section covers migrating from older versions of the role.

### Migrating from `podman_pod_name` (removed)

The `podman_pod_name` variable has been removed and no longer has any effect.

**If you were using `podman_pod_name`** to automatically assign containers to pods:

**Old approach (removed):**

```yaml
podman_pod_name: app

podman_pods:
  - name: "{{ podman_pod_name }}"
    publish:
      - 8080:80

podman_containers:
  - name: web
    image: myapp
    tag: latest
    # Implicitly joins pod via podman_pod_name
```

**New approach (explicit pod assignment):**

```yaml
podman_pods:
  - name: app
    publish:
      - 8080:80

podman_containers:
  - name: web
    image: myapp
    tag: latest
    pod: app # Explicitly specify pod
```

**Or use networks (recommended for most use cases):**

```yaml
podman_use_quadlets: yes
podman_default_network: app

podman_networks:
  - name: app

podman_containers:
  - name: web
    image: myapp
    tag: latest
    publish:
      - "8080:80"
```

### Migrating from `podman_pod_inherit_hostname` (removed)

`podman_pod_inherit_hostname` has been removed and no longer has any effect.

**If you were using `podman_pod_inherit_hostname`** to make all pods inherit the host's hostname:

**Old approach (removed):**

```yaml
podman_pod_inherit_hostname: yes # Applied to ALL pods

podman_pods:
  - name: webapp
```

**New approach (explicit per-pod opt-in):**

```yaml
podman_pods:
  - name: webapp
    hostname: "%H" # Explicit per-pod opt-in
```

For containers (not in a pod):

```yaml
podman_containers:
  - name: web
    image: myapp
    tag: latest
    hostname: "%H" # Inherits host's hostname
```

## Testing

This role uses [Molecule](https://ansible.readthedocs.io/projects/molecule/) with the Podman driver for integration testing. Dependencies are managed with [uv](https://docs.astral.sh/uv/).

### Setup

```bash
# Install dependencies
uv sync --python 3.13
```

**Requirements:** Podman must be installed on the host machine.

### Running Tests

```bash
# Run all scenarios
uv run molecule test -s build-image
uv run molecule test -s legacy-pod
uv run molecule test -s legacy-container-hostname
uv run molecule test -s quadlet
uv run molecule test -s quadlet-default-network
uv run molecule test -s quadlet-pod
uv run molecule test -s quadlet-container-hostname

# Or run individual steps for a scenario
uv run molecule create -s quadlet    # Create the test container
uv run molecule converge -s quadlet  # Apply the role
uv run molecule verify -s quadlet    # Run assertions
uv run molecule destroy -s quadlet   # Tear down
```

### Test Scenarios

| Scenario                     | Mode    | What it tests                                                                       |
| ---------------------------- | ------- | ----------------------------------------------------------------------------------- |
| `build-image`                | N/A     | Builds the systemd-enabled test container image                                     |
| `legacy-pod`                 | Legacy  | Pod with multiple containers, `podman generate systemd`                             |
| `legacy-container-hostname`  | Legacy  | Standalone containers with host hostname inheritance                                |
| `quadlet`                    | Quadlet | Standalone containers on a shared network, `.container`/`.network` files, env files |
| `quadlet-default-network`    | Quadlet | Pods and containers using `podman_default_network` with `.network` suffix           |
| `quadlet-pod`                | Quadlet | Pod + standalone containers coexisting, `.pod`/`.container`/`.network` files        |
| `quadlet-container-hostname` | Quadlet | Standalone containers with host hostname inheritance using quadlets                 |

All scenarios use a custom `molecule/Dockerfile.j2` that builds a systemd-enabled Ubuntu 25.10 image with podman 5.4 (required for `.pod` quadlet support).

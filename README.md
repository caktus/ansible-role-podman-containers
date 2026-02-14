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
uv run molecule test -s quadlet
uv run molecule test -s quadlet-pod

# Or run individual steps for a scenario
uv run molecule create -s quadlet    # Create the test container
uv run molecule converge -s quadlet  # Apply the role
uv run molecule verify -s quadlet    # Run assertions
uv run molecule destroy -s quadlet   # Tear down
```

### Test Scenarios

| Scenario      | Mode    | What it tests                                                                       |
| ------------- | ------- | ----------------------------------------------------------------------------------- |
| `build-image` | N/A     | Builds the systemd-enabled test container image                                     |
| `legacy-pod`  | Legacy  | Pod with multiple containers, `podman generate systemd`                             |
| `quadlet`     | Quadlet | Standalone containers on a shared network, `.container`/`.network` files, env files |
| `quadlet-pod` | Quadlet | Pod + standalone containers coexisting, `.pod`/`.container`/`.network` files        |

All scenarios use a custom `molecule/Dockerfile.j2` that builds a systemd-enabled Ubuntu 25.10 image with podman 5.4 (required for `.pod` quadlet support).

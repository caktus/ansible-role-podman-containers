# caktus.podman_containers

🚀 **Production-ready Podman container orchestration with Ansible**--no Kubernetes required.

- 📦 Declarative Ansible container deployment with systemd integration
- ✨ Modern Quadlet support (`.container`, `.network`, `.pod` files)
- 🔒 Security best practices built-in (restricted env files, rootless containers)
- 🛠️ Git-based image builds (optionally clone repos and build images with Podman)
- 🐧 Works seamlessly on Ubuntu and RHEL-based systems
- ⚡ Get systemd's process management without k8s complexity

## Configuration

See `defaults/main.yml` for all supported variables, defaults, and examples.

## Version Requirements

| Feature                             | Minimum Podman Version |
| ----------------------------------- | ---------------------- |
| Basic containers/pods (legacy)      | 3.0                    |
| Quadlets (`.container`, `.network`) | 4.4                    |
| Quadlet pods (`.pod` files)         | 5.0                    |

## Quick Start

**Sample playbook:**

```yaml
---
- hosts: app_servers
  tags: podman
  gather_facts: no
  become: no
  vars:
    ansible_user: "{{ podman_user }}"
  roles:
    - role: caktus.podman_containers
```

**Role variables:**

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

All scenarios test both **Ubuntu 25.10** and **CentOS Stream 9** platforms to ensure compatibility across Debian and RHEL-based distributions.

### Setup

```bash
# Install dependencies
uv sync --locked
```

### Code Quality

```bash
# Run pre-commit checks (linting, formatting, etc.)
uv run pre-commit run --all-files
```

### Running Tests

**Requirements:** Podman must be installed on the host machine.

```bash
# Run the default scenario (recommended for quick testing)
uv run molecule test

# Or run specific scenarios
uv run molecule test -s build-image
uv run molecule test -s legacy-pod
uv run molecule test -s legacy-container-hostname
uv run molecule test -s quadlet
uv run molecule test -s quadlet-default-network
uv run molecule test -s quadlet-pod
uv run molecule test -s quadlet-container-hostname
uv run molecule test -s quadlet-restart

# Or run all scenarios
uv run molecule test --all

# Or run individual steps for a scenario
export SCENARIO=quadlet
uv run molecule create -s $SCENARIO    # Create the test container
uv run molecule converge -s $SCENARIO  # Apply the role
uv run molecule verify -s $SCENARIO    # Run assertions
uv run molecule destroy -s $SCENARIO   # Tear down
```

### Test Scenarios

| Scenario                     | Mode    | What it tests                                                                |
| ---------------------------- | ------- | ---------------------------------------------------------------------------- |
| `default`                    | Quadlet | README quickstart example (single container, shared network, HTTP endpoint)  |
| `build-image`                | N/A     | Builds the systemd-enabled test container image                              |
| `legacy-pod`                 | Legacy  | Pod with multiple containers, `podman generate systemd`                      |
| `legacy-container-hostname`  | Legacy  | Standalone containers with host hostname inheritance                         |
| `quadlet`                    | Quadlet | Standalone containers on a shared network with `.container`/`.network` files |
| `quadlet-default-network`    | Quadlet | Demonstrates `podman_default_network` with shared `.network` quadlet         |
| `quadlet-pod`                | Quadlet | Pod + standalone containers coexisting with mixed `.pod`/`.container` files  |
| `quadlet-container-hostname` | Quadlet | Standalone containers with host hostname inheritance using quadlets          |
| `quadlet-restart`            | Quadlet | Container/pod restart behavior when environment variables change             |

All scenarios use systemd-enabled test container images:

- **Ubuntu:** Custom `molecule/Dockerfile.j2` builds Ubuntu 25.10 with Podman 5.4
- **RHEL/CentOS:** Custom `molecule/Dockerfile.rhel.j2` builds CentOS Stream 9 with Podman 5.6

Podman 5.0+ is required for `.pod` quadlet support tested in the `quadlet-pod` scenario.

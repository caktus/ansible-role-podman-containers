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

### Environment Files (`env_as_file`)

Use `env_as_file` on a container to write environment variables to a restricted file (mode `0600`)
instead of embedding them in systemd units. This keeps secrets out of `ps` output and unit files.

```yaml
podman_containers:
  - name: web
    image: myapp
    tag: latest
    env_as_file:
      DATABASE_URL: "postgres://db:5432/app"
      SECRET_KEY: "s3kret"
```

The role writes the dictionary to `<podman_env_file_dir>/<container_name>.env` and passes it as
`env_file` to the container module. A checksum label is added so the container is recreated when
values change.

If a container sets **both** `env_file` and `env_as_file`, the file is still written but the
role uses the explicit `env_file` value. You must include the generated file in your list yourself:

```yaml
podman_containers:
  - name: web
    image: myapp
    tag: latest
    env_as_file:
      SECRET_KEY: "s3kret"
    env_file:
      - /home/deploy/env/web.env # the generated file
      - /home/deploy/env/extra.env # additional file
```

### Container Logging

Use `log_driver` and `log_opt` to write container logs to files. Set them per-container or globally via `podman_log_driver` and `podman_log_opt`:

```yaml
# Global defaults (apply to all containers)
podman_log_driver: k8s-file
podman_log_opt:
  path: "{{ podman_user_home }}/log/{{ item.name }}.log"

podman_containers:
  - name: web
    image: myapp
    tag: latest
  - name: worker
    image: myapp
    tag: latest
```

Or per-container:

```yaml
podman_containers:
  - name: web
    image: myapp
    tag: latest
    log_driver: k8s-file
    log_opt:
      path: /home/deploy/log/web.log
```

**Podman version compatibility:**

- **Legacy mode**: `log_driver` and `log_opt` work on all Podman versions (3.0+)

- **Quadlet mode on Podman 5.0+**: `log_driver` and `log_opt` generate `LogDriver=` and `LogOpt=` directives in `.container` files (works automatically)

- **Quadlet mode on Podman < 5.0** (e.g., Ubuntu 24.04 with Podman 4.9): `LogDriver=` and `LogOpt=` directives are **not supported** and will cause systemd to reject the unit file. Use `PodmanArgs=` instead via `quadlet_options` (per-container or global `podman_quadlet_options`):

  ```yaml
  podman_containers:
    - name: web
      image: myapp
      tag: latest
      quadlet_options:
        - "PodmanArgs=--log-driver=k8s-file --log-opt path=/var/log/web.log"
  ```

## Socket Activation

The role supports systemd socket activation, which lets rootless Podman containers bind to
privileged ports (80/443) without root and keeps ports alive across container restarts.

Configure socket units via `podman_socket_units`:

```yaml
podman_socket_units:
  - name: http
    fd_name: web        # FileDescriptorName — must match the Traefik entryPoint name
    port: 80
    service: traefik    # The container service to activate
  - name: https
    fd_name: websecure
    port: 443
    service: traefik
```

Each entry creates a systemd `.socket` unit file in `~/.config/systemd/user/` and enables it.

## Traefik HTTP Proxy

The role can deploy [Traefik](https://traefik.io/) as a managed reverse proxy container with
automatic container discovery via Podman socket labels.

### Quick Start

```yaml
podman_traefik_enabled: true
podman_traefik_acme_email: "ops@example.com"   # Enable Let's Encrypt (omit to disable)
podman_uid: "1001"                             # Numeric UID of podman_user
podman_use_quadlets: yes
podman_default_network: app

podman_networks:
  - name: app

podman_containers:
  # Socket proxy — required sidecar for Traefik container discovery
  - name: socket-proxy
    image: ghcr.io/tecnativa/docker-socket-proxy
    tag: latest
    volumes:
      - "/run/user/{{ podman_uid }}/podman/podman.sock:/var/run/docker.sock:ro,Z"
    env:
      CONTAINERS: "1"
      EVENTS: "1"
      POST: "0"

  # Web container with Traefik labels
  - name: web
    image: myapp
    tag: latest
    label:
      traefik.enable: "true"
      traefik.http.routers.web.rule: "Host(`example.com`)"
      traefik.http.routers.web.entrypoints: "websecure"
      traefik.http.routers.web.tls.certresolver: "letsencrypt"
      traefik.http.services.web.loadbalancer.server.port: "8000"
```

When `podman_traefik_enabled` is true, the role automatically:
1. Creates Traefik config files (`traefik.yaml`, `middlewares.yaml`)
2. Registers HTTP/HTTPS socket units for socket activation
3. Deploys the Traefik container with proper volume mounts

### Traefik Variables

| Variable                               | Default                                     | Description                                     |
| -------------------------------------- | ------------------------------------------- | ----------------------------------------------- |
| `podman_traefik_enabled`               | `false`                                     | Enable Traefik proxy                            |
| `podman_traefik_image`                 | `docker.io/traefik`                         | Traefik container image                         |
| `podman_traefik_tag`                   | `v3`                                        | Traefik image tag                               |
| `podman_traefik_config_dir`            | `{{ podman_user_home }}/traefik`            | Directory for Traefik config files              |
| `podman_traefik_log_level`             | `INFO`                                      | Traefik log level                               |
| `podman_traefik_acme_email`            | `""`                                        | ACME email (set to enable Let's Encrypt)        |
| `podman_traefik_acme_storage`          | `/etc/traefik/acme/acme.json`               | ACME storage path inside the container          |
| `podman_traefik_extra_volumes`         | `[]`                                        | Additional volumes for the Traefik container    |
| `podman_traefik_quadlet_options`       | `[]`                                        | Extra quadlet directives for Traefik            |
| `podman_traefik_extra_static_config`   | `{}`                                        | Deep-merged into `traefik.yaml`                 |
| `podman_traefik_container`             | `{}`                                        | Full override of the Traefik container def      |
| `podman_traefik_middlewares`           | `{}`                                        | Dict merged into `http.middlewares`             |
| `podman_uid`                           | `""`                                        | Numeric UID of `podman_user` (for socket proxy) |

### Default Middlewares

The role deploys shared middlewares in `middlewares.yaml` that can be referenced in container
labels with the `@file` suffix:

- **`compress@file`** — Gzip compression for text-based content types
- **`security-headers@file`** — HSTS, frame denial, content-type nosniff, XSS filter
- **`rate-limit@file`** — Rate limiting (100 avg / 200 burst per second)

Custom middlewares can be added via `podman_traefik_middlewares`.

## Zero-Downtime Deployments

For web-facing containers that must stay live during deploys, use versioned container naming
and the following deployment sequence:

1. Start new container alongside the running one
2. Traefik discovers it and health-gates traffic (holds it DOWN until health check passes)
3. Confirm the new container is UP via Traefik API
4. Stop the old container

**Example deployment playbook** (`deploy.yml`):

```yaml
- name: Deploy new image tag
  hosts: app
  tasks:
    - name: Start new container
      ansible.builtin.include_role:
        name: podman_containers
      vars:
        podman_containers:
          - name: "web-{{ new_tag }}"
            image: myapp
            tag: "{{ new_tag }}"
            label:
              traefik.enable: "true"
              traefik.http.routers.web.rule: "Host(`example.com`)"
              traefik.http.routers.web.entrypoints: "websecure"
              traefik.http.routers.web.middlewares: "retry,compress@file,security-headers@file"
              traefik.http.services.web.loadbalancer.server.port: "8000"
              traefik.http.services.web.loadbalancer.healthcheck.path: "/health"
              traefik.http.services.web.loadbalancer.healthcheck.interval: "5s"
              traefik.http.services.web.loadbalancer.healthcheck.timeout: "3s"
              traefik.http.middlewares.retry.retry.attempts: "2"
              app.role: web

    - name: Get new container IP
      ansible.builtin.command:
        cmd: >-
          podman inspect web-{{ new_tag }}
          --format '{% raw %}{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}{% endraw %}'
      register: _new_ip
      changed_when: false

    - name: Wait for Traefik to mark new container UP
      ansible.builtin.uri:
        url: "http://127.0.0.1:8080/api/http/services"
      register: _traefik_api
      until: >-
        (_traefik_api.json
         | selectattr('name', 'equalto', 'web@docker')
         | map(attribute='serverStatus')
         | first
        ).get('http://' + (_new_ip.stdout | trim) + ':8000', 'DOWN') == 'UP'
      retries: 30
      delay: 5

    - name: Find old containers
      ansible.builtin.command:
        cmd: >-
          podman ps -q --filter label=app.role=web --filter name!=web-{{ new_tag }}
      register: _old_containers
      changed_when: false

    - name: Stop old containers
      ansible.builtin.command:
        cmd: "podman stop {{ _old_containers.stdout_lines | join(' ') }}"
      when: _old_containers.stdout_lines | length > 0
```

Run with: `ansible-playbook deploy.yml -e "new_tag=a1b2c3d"`

Rollback: re-run with the previous tag. Containers that don't need zero-downtime (workers,
beat schedulers, caches) carry no Traefik labels and are managed normally by the role.

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
uv run molecule test -s podman49-logging
uv run molecule test -s traefik-labels

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

| Scenario                     | Mode    | What it tests                                                                  |
| ---------------------------- | ------- | ------------------------------------------------------------------------------ |
| `default`                    | Quadlet | README quickstart example (single container, shared network, HTTP endpoint)    |
| `build-image`                | N/A     | Builds the systemd-enabled test container image                                |
| `legacy-pod`                 | Legacy  | Pod with multiple containers, `podman generate systemd`                        |
| `legacy-container-hostname`  | Legacy  | Standalone containers with host hostname inheritance                           |
| `quadlet`                    | Quadlet | Standalone containers on a shared network with `.container`/`.network` files   |
| `quadlet-default-network`    | Quadlet | Demonstrates `podman_default_network` with shared `.network` quadlet           |
| `quadlet-pod`                | Quadlet | Pod + standalone containers coexisting with mixed `.pod`/`.container` files    |
| `quadlet-container-hostname` | Quadlet | Standalone containers with host hostname inheritance using quadlets            |
| `quadlet-restart`            | Quadlet | Container/pod restart behavior when environment variables change               |
| `podman49-logging`           | Quadlet | Container logging using `PodmanArgs=` workaround for Podman 4.9 (Ubuntu 24.04) |
| `traefik-labels`             | Quadlet | Socket activation, Traefik config, container labels, and discovery             |

All scenarios use systemd-enabled test container images:

- **Ubuntu 25.10:** Custom `molecule/Dockerfile.j2` with Podman 5.4 (most scenarios)
- **Ubuntu 24.04:** Used in `podman49-logging` scenario with Podman 4.9
- **RHEL/CentOS:** Custom `molecule/Dockerfile.rhel.j2` builds CentOS Stream 9 with Podman 5.6

Podman 5.0+ is required for `.pod` quadlet support tested in the `quadlet-pod` scenario.

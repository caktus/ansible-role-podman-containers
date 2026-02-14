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

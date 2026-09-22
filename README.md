# Ansible Docker

Ansible role that installs Docker Engine, Buildx and Compose v2 from Docker's
own apt repository on Ubuntu.

Tested on Ubuntu 24.04 (noble) and 26.04 (resolute) with ansible-core 2.15+.

## What it does

* Installs the Docker apt repository in deb822 format with a dedicated
  keyring at `/etc/apt/keyrings/docker.asc`, so the Docker key can only
  sign Docker packages.
* Picks the apt suite from the running release, falling back to
  `docker_apt_suite_fallback` when Docker has not published for it yet.
* Installs `docker-ce`, `docker-ce-cli`, `containerd.io`,
  `docker-buildx-plugin` and `docker-compose-plugin`.
* Writes `/etc/docker/daemon.json` with log rotation, which is what keeps
  a long-running node from filling its disk with container logs.
* Adds `docker_group_member` to the `docker` **supplementary** group.
* Optionally makes published container ports obey ufw.

## Docker and ufw

By default Docker programs the `nat` PREROUTING and `filter` FORWARD chains
directly. Those are evaluated before ufw's chains, so:

```bash
ufw default deny incoming
docker run -p 8080:80 nginx     # reachable from the whole internet
ufw status                      # says nothing about 8080
```

With `docker_ufw_integration: true` (the default) the role appends a
`DOCKER-USER` block to `/etc/ufw/after.rules` that routes container traffic
through ufw's route rules and drops the rest.

**This changes how you expose a container.** Publishing a port is no longer
enough; you also have to open it — and the rule has to be written carefully.

### Match the container port, not the published port

`nat` PREROUTING rewrites the destination before the `filter` FORWARD chain
runs, so by the time ufw sees the packet the published port is gone. For
`-p 8080:80` the packet in FORWARD is addressed to `172.17.0.2:80`.

```bash
# WRONG -- 8080 no longer exists in the packet at this point, never matches
ufw route allow proto tcp from any to any port 8080

# Right -- matches the container's own address and port
ufw route allow proto tcp from any to 172.17.0.2 port 80

# Coarser, but survives the container getting a new IP
ufw route allow proto tcp from any to any port 80
```

Host-level rules (`ufw allow 8080`) do not apply either: the traffic is
forwarded, not delivered locally, so it never reaches `ufw-input`.

Because container IPs are reassigned on recreate, the tidiest pattern is to
avoid per-container rules altogether: publish on the loopback interface
(`-p 127.0.0.1:8080:80`) and terminate public traffic at a reverse proxy on
80/443, which the node role already opens.

Traffic from `docker_ufw_internal_networks` is exempt, so container to
container and host to container keep working untouched.

Set `docker_ufw_integration: false` to keep Docker's default behaviour where
publishing a port exposes it.

### Verifying

Dropped container traffic is logged, so this shows what is being blocked:

```bash
journalctl -k -f | grep "UFW DOCKER BLOCK"
```

## Variables

See `defaults/main.yml`. The ones you are most likely to change:

| Variable | Default | Purpose |
| --- | --- | --- |
| `docker_group_member` | `{{ ansible_user }}` | Account added to the `docker` group |
| `docker_daemon_options` | log rotation, `live-restore` | Merged into `daemon.json` |
| `docker_ufw_integration` | `true` | Make published ports obey ufw |
| `docker_apt_suite_fallback` | `noble` | Suite used when Docker lags a release |
| `docker_extra_packages` | `[]` | Extra packages to install alongside Docker |

Note that `docker` group membership is equivalent to root on the host.

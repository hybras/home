# AGENTS.md

This file explain the repository structure and common tasks.

## Overview

This repository describes a set of services meant for consumption for a small group of friends and family.
Each directory is a different service, containing a docker-compose.yml file, and config files for the associated service.

## Networking

## Caddy

All services are reverse proxied by caddy (caddy directory).
Containers for different services join the caddy network (as opposed to caddy joining network for different services).
This allows containers for individual services to be stopped / started independently of caddy.

Each service has a route listed for it in the Caddyfile.
Routes are intentionally left as simple as possible.
All services must have https routes.

## Publicly vs Privately Exposed services

Some services are only available behind the vpn, headscale.
These services have a caddyfile directory of `bind tailscale/xxx`, where `xxx` is the name of the tailscale node.

Other services are publicly available, like headscale itself.

## Networking Architecture and Infrastructure

* My ISP assigns a static IP address to my router.
* The router forwards ports 80, 443 to my box's ports 80/443.
  * The port forwarding is how public services are exposed
* The box is running docker, with containers for all services. The container for caddy binds ports 80,443
* Caddy is itself also logged into the vpn as a node, allowing it to privately expose services without consuming ports 80/443 on the host.

## Development Commands

### Git Commits

Give good descriptions for all git commit messages. Use normal git commands to save changes.

### Updating Configuration files

Use docker compose commands to update and inspect running containers. Use the `--project-directory` flag rather than `cd`'ing into each relevant directory for docker compose commands.

Edit config files (including `docker-compose.yaml`), and use docker compose commands to update running containers like (`exec`, `up -d`, `restart`, `stop`, etc.).

**WARNING**::
This also applies to the caddy container, **DO NOT** use `caddy reload` to update caddy after making config changes.

### Validating the Caddyfile

The `caddy` directory contains a caddy binary with the same build configuration and capabilities as the caddy binary inside the caddy docker image. Use the following commands to ensure its validity and formatting after making changes

```bash
caddy validate --config Caddyfile --envfile caddy.env --envfile duckdns.env
caddy fmt --overwrite Caddyfile
```

### Debugging Container Internal Networking

The containers are all minimal / distroless, without tools or shells. Create ephemeral docker containers with the necessary tools.

```bash
docker run
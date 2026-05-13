# AGENTS.md

This file explains the repository structure, services, networking architecture, and working guidelines for maintaining and developing this home server infrastructure.

## Overview

This repository contains a complete home server setup running multiple services for personal use and consumption by a small group of friends and family. Each service directory contains:
- `docker-compose.yml` - Service container definitions
- Configuration files specific to that service
- Optional `justfile` for common service-specific tasks
- `README.md` - Service-specific documentation

**Watchtower Integration**: Automatic container updates are enabled via Watchtower, which means configuration changes requiring manual intervention are not permitted.

---

## Networking Architecture

### Core Design Principles

1. **Reverse Proxy Pattern**: All services are reverse-proxied through Caddy
2. **Network Isolation**: Each service container joins the Caddy network independently
   - Services can be stopped/started without affecting the reverse proxy
3. **Security via VPN**: VPN (Headscale/Tailscale) provides private network access
4. **Port Efficiency**: Single port forwarding (80, 443) handles all public traffic

### Infrastructure Stack

- **ISP Setup**: Static IP assigned by ISP
- **Router**: Forwards ports 80 & 443 to the home server (127.0.0.1:80 and 127.0.0.1:443)
- **Host**: Docker daemon runs all services
- **Caddy Container**: Binds to host ports 80 & 443
  - Routes to public services directly
  - Routes to VPN services via Tailscale network
- **Headscale**: Personal VPN controller (publicly exposed for client registration)

### Service Exposure Patterns

#### Publicly Available Services

- Accessible directly from internet via domain name
- Examples: Headscale, Nextcloud, Gitea
- Configuration: Standard routes in `Caddyfile`

#### VPN-Only Services

- Only accessible from devices connected to Headscale VPN
- Use `bind tailscale/NODE_NAME` in Caddyfile
- Examples: Admin interfaces, media servers, internal tools
- Reduces attack surface for non-essential services

---

## Service Organization

### Key Services

| Service | Purpose | Exposure | Status |
|---------|---------|----------|--------|
| **Headscale** | Personal VPN controller | Public | Core infrastructure |
| **Authelia** | Authentication & authorization | VPN | Central identity provider |
| **Caddy** | Reverse proxy & SSL/TLS | All | Core infrastructure |
| **Gitea** | Git repository hosting | Public | Self-hosted alternative to GitHub |
| **Nextcloud** | File sync & calendar/contacts | Public | Personal cloud storage |
| **Invidious** | YouTube alternative frontend | Public | Privacy-respecting YouTube |
| **Nitter** | Twitter alternative frontend | Public | Privacy-respecting Twitter |
| **Libreddit** | Reddit alternative frontend | Public | Privacy-respecting Reddit |
| **Rimgo** | Imgur alternative frontend | Public | Privacy-respecting image sharing |
| **Pihole** | DNS-level ad blocking | VPN | Network-wide filtering |
| **Vaultwarden** | Bitwarden-compatible password manager | Public | Password vault |
| **Syncthing** | Decentralized file sync | VPN | Device synchronization |
| **Watchtower** | Automatic container updates | Background | Auto-deployment system |

---

## Caddy Configuration

### Overview
Caddy serves as the single entry point for all services, handling SSL/TLS termination, reverse proxying, and load balancing.

### Key Principles
1. **Simple, Readable Routes**: One route per service, minimal directives
2. **HTTPS Enforced**: All services must use HTTPS
3. **Automatic SSL**: Let's Encrypt certificates via Caddy
4. **Dynamic DNS**: Integration with DuckDNS for dynamic IP handling

### Working with Caddyfile

#### Validation & Formatting
```bash
# From caddy/ directory
just validate          # Validate Caddyfile syntax
just fmt               # Auto-format Caddyfile
just reload            # Reload Caddy without downtime
```

#### Route Structure

```caddy
# Public service
example.hybras.dev {
    reverse_proxy localhost:8000
}

# VPN-only service (Tailscale)
admin.internal {
    bind tailscale/controlplane
    reverse_proxy localhost:9000
}
```

---

## Working Guidelines

### Git Workflow

#### Commit Messages

- **Mandatory**: Write descriptive commit messages for all changes
- **Format**: Use conventional commits when possible
  ```
  feat: add Headplane UI to Headscale
  fix: resolve Nitter database connection issue
  docs: update service documentation
  ```
- **Scope**: Include affected service in message
- **Push Regularly**: Keep changes in sync with remote

#### Branch Strategy

- Work directly on `main` branch or create feature branches
- Merge only when tested and ready for deployment
- Watchtower watches for changes and auto-deploys

### Configuration Management

#### File Editing

1. **Never use `cd` into service directories** - Use `--project-directory` flag instead
2. Edit configuration files in your preferred editor
3. Use Docker Compose to apply changes:
   ```bash
   # From any directory
   docker compose -C /path/to/service up -d      # Apply changes
   docker compose -C /path/to/service exec <svc> <command>  # Run commands
   docker compose -C /path/to/service logs -f <svc>         # View logs
   ```

#### Configuration File Types

- `docker-compose.yml` - Container definitions, networks, volumes
- `*.yml` / `*.yaml` - Service-specific configs
- `.env` files - Environment variables (may contain secrets)
- Secret files in `secrets/` directories - Mount as Docker secrets

### Debugging & Troubleshooting

#### Container Network Issues
Distroless containers lack debugging tools (no shells, minimal binaries). Create temporary debug containers:

```bash
# Generic network debugging
docker run --rm -it --network caddy_network \
  nicolaka/netshoot:latest \
  bash

# DNS testing
docker run --rm nicolaka/netshoot:latest \
  nslookup service-name

# Port connectivity
docker run --rm --network caddy_network \
  nicolaka/netshoot:latest \
  curl http://headscale:8000
```

#### Log Inspection

```bash
# Follow service logs
docker compose -C /path/to/service logs -f service_name

# Check caddy routing
docker compose -C caddy logs -f caddy

# Historical logs (last 100 lines)
docker compose -C /path/to/service logs --tail 100 service_name
```

#### Common Issues

- **Connection refused**: Service not responding on expected port
  - Check `docker compose ps` to confirm service is running
  - Verify port mappings in docker-compose.yml
  - Check service logs for startup errors

- **DNS resolution fails**: Caddy can't resolve service name
  - Verify services are on same Docker network
  - Check Caddy container has network access to service
  - Use service hostname (container name) in reverse_proxy directive

- **SSL/TLS certificate issues**: Caddy can't obtain Let's Encrypt cert
  - Check domain DNS points to server IP
  - Verify port 80 & 443 are accessible from internet
  - Review Caddy logs for ACME errors

### Authentication & Authorization

#### Authelia Integration

- Central authentication point using LDAP (via LLDAP)
- Supports OAuth2/OIDC for service integration
- Key configuration locations:
  - LDAP: `authelia/data/lldap/` (user directory)
  - Authelia config: `authelia/data/authelia/config/configuration.yml`
  - OIDC secrets: `authelia/data/authelia/secrets/`

#### Adding New OIDC Consumers

Services integrated with Authelia via OIDC:
- **Headscale** (via configuration in headscale.yaml)
- New services can add OIDC auth through Authelia config

---

## Common Tasks

### Starting the Stack
```bash
# Start all services
docker compose -C caddy up -d

# Or start individual services
docker compose -C authelia up -d
docker compose -C headscale up -d
docker compose -C caddy up -d
```

### Checking Service Status
```bash
# All containers in repository
for dir in */; do 
  echo "=== $dir ==="
  docker compose -C "$dir" ps
done

# Or for specific service
docker compose -C caddy ps
```

### Updating Service Configuration
1. Edit config file in your editor
2. Validate syntax (check service README for validation command)
3. Apply changes: `docker compose -C /path/to/service up -d service_name`
4. Verify via logs: `docker compose -C /path/to/service logs service_name`
5. Commit changes: `git add . && git commit -m "config: update service_name configuration"`

### Viewing Secrets
Secrets are stored in `secrets/` subdirectories and mounted via Docker secrets:
```bash
# View available secrets
docker secret ls

# Secrets files should never be committed to git (add to .gitignore)
```

---

## Known Configurations & Lessons

### Headplane UI for Headscale
- **Status**: Configured and running at `https://headscale.hybras.dev/admin`
- **Setup**: Headplane container alongside Headscale with shared socket access
- **Authentication**: Requires Headscale API key generated during setup
- **Reference**: See [headscale/README.md](headscale/README.md)

### LDAP & LLDAP Configuration

#### Overview
LLDAP is a lightweight LDAP server that provides centralized user management for Authelia. It stores all user credentials, groups, and permissions.

#### Architecture
- **LLDAP Container**: Runs SQLite database with LDAP protocol server
- **Port 3890**: LDAP port (internal network only)
- **Port 17170**: Web admin interface (accessible via VPN or public if exposed)
- **Storage**: `/var/lib/lldap/` - SQLite database with user data

#### Key Concepts
- **Base DN**: `dc=hybras,dc=dev` - Root of LDAP directory tree
- **ou=people**: Organizational unit containing all users
- **ou=groups**: Organizational unit containing group definitions
- **uid**: Unique identifier for each user (username)
- **mail**: Email address attribute
- **displayName**: Human-readable name

#### User Management
1. **Add User**: LLDAP web interface → Users → "Create new user"
2. **Set Password**: Users must change password on first login
3. **Manage Groups**: Assign users to groups for role-based access control
4. **Delete User**: Remove from LLDAP (also invalidates sessions)

#### Common LDAP Operations
```bash
# Test LDAP connectivity from container
docker run --rm --network authelia nicolaka/netshoot:latest \
  ldapsearch -x -H ldap://lldap:3890 \
    -D "uid=admin,ou=people,dc=hybras,dc=dev" \
    -w <password> \
    -b "dc=hybras,dc=dev"

# List all users
ldapsearch -x -D "uid=admin,ou=people,dc=hybras,dc=dev" \
  -w <password> -b "ou=people,dc=hybras,dc=dev"

# Change admin password (requires access to LLDAP container)
docker compose -C authelia exec lldap \
  lldap_password_reset admin
```

#### LDAP to Authelia Integration
- Authelia config specifies LDAP backend: `server: ldap://lldap:3890`
- Connection uses service account credentials from secrets
- User login attempts validated against LLDAP password hashes
- Group membership determines access control and service permissions

#### Troubleshooting LDAP Issues
1. **"Invalid credentials" error**: Check password hash in LLDAP database
2. **Connection refused**: Verify LLDAP container is running on port 3890
3. **User exists but can't login**: Check base DN and user uid format
4. **New user can't authenticate**: Ensure password was set correctly in LLDAP

**Reference**: See [authelia/README.md](authelia/README.md)

### Nitter Debugging Considerations

#### Session Management
- Nitter maintains Twitter session cookies in `sessions.jsonl`
- Each line is a JSON object representing an authenticated session
- Sessions can expire or become invalid if Twitter changes authentication

#### Common Issues & Solutions

**Issue**: "Failed to fetch tweets" or connection errors
- **Root Cause**: Twitter session expired or rate limiting
- **Solution**: Clear sessions and let new ones be created:
  ```bash
  rm nitter/sessions.jsonl
  docker compose -C nitter restart nitter
  ```
- **Prevention**: Monitor logs for rate limiting patterns

**Issue**: Nitter crashes on startup
- **Root Cause**: Corrupted sessions.jsonl file
- **Solution**: 
  1. Stop container: `docker compose -C nitter down`
  2. Delete sessions: `rm nitter/sessions.jsonl`
  3. Start fresh: `docker compose -C nitter up -d`

**Issue**: Very slow loading or timeouts
- **Root Cause**: Twitter rate limiting this Nitter instance
- **Solution**: 
  1. Check logs for rate limit messages: `docker compose -C nitter logs | grep -i rate`
  2. Wait before retrying (rate limits often reset after 15 min)
  3. Consider reducing request frequency

#### Debugging Steps
```bash
# View recent logs
docker compose -C nitter logs --tail 50 nitter

# Monitor logs in real-time
docker compose -C nitter logs -f nitter

# Check sessions file size and date
ls -lh nitter/sessions.jsonl

# Test basic connectivity
docker run --rm --network nitter \
  nicolaka/netshoot:latest \
  curl -v http://nitter:8080/

# Check if Twitter is accessible from container
docker run --rm nicolaka/netshoot:latest \
  curl -I https://twitter.com
```

#### Container Lifecycle
- Container should auto-restart on crash (restart policy: `unless-stopped`)
- Watchtower will pull image updates automatically
- Changes to nitter.conf require container restart
- Session data persists between restarts (important for maintaining Twitter sessions)

#### Prevention & Best Practices
1. Check logs regularly for error patterns
2. Set up alerts for repeated "rate limited" messages

**Reference**: See [nitter/README.md](nitter/README.md)

---

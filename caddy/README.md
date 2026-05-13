# Caddy - Reverse Proxy & SSL/TLS Termination

Caddy is the central reverse proxy for all home server services. It handles:
- **Reverse proxy routing** to all services
- **Automatic SSL/TLS** certificates via Let's Encrypt
- **Dynamic DNS integration** via DuckDNS
- **Traffic routing** to both public and VPN-only services

## 📋 Overview

| Aspect | Details |
|--------|---------|
| **Service Name** | caddy |
| **Image** | `ghcr.io/hybras/caddy:latest` |
| **Port** | 80, 443 (host ports) |
| **Volume** | `caddy` (for certificate cache) |
| **Networks** | caddy (shared with all services) |

## 🏗️ Architecture

### Network Design
- Caddy creates the `caddy` network that all other services join
- Each service independently connects to the `caddy` network
- This pattern allows services to be stopped/started without affecting Caddy
- Caddy is also connected to Headscale (Tailscale) for VPN routing

### Routing Patterns

#### Public Services (HTTP/HTTPS to Internet)
```caddy
example.hybras.dev {
    reverse_proxy localhost:8000
}
```

#### VPN-Only Services (Tailscale Binding)
```caddy
admin.internal {
    bind tailscale/NODE_NAME
    reverse_proxy localhost:9000
}
```

## 🔧 Configuration

### Key Files
- **Caddyfile** - Main routing configuration (simple one-route-per-service format)
- **caddy.env** - Environment variables (domain, email for Let's Encrypt)
- **duckdns.env** - DuckDNS token and domain for dynamic DNS

### Caddyfile Structure

```caddy
# Auto-HTTPS and Let's Encrypt configuration
{
    acme_ca https://acme-v02.api.letsencrypt.org/directory
    email {$EMAIL}
}

# Example public service route
example.hybras.dev {
    reverse_proxy localhost:8000
}

# Example VPN-only service route
internal-admin.local {
    bind tailscale/controlplane
    reverse_proxy localhost:9000
}
```

### Environment Variables
- `EMAIL` - Email for Let's Encrypt certificate notifications
- `DUCKDNS_TOKEN` - Token for DuckDNS API (dynamic DNS)
- `DUCKDNS_DOMAIN` - Domain name for DuckDNS

## 📦 Data Persistence

### Volumes
- **caddy** - Stores Let's Encrypt certificates and cache
  - Never manually delete this volume
  - Contains rate-limit tracking for ACME
  - Persists across container restarts

## 🚀 Common Tasks

### Validate Caddyfile Syntax
```bash
cd caddy
just validate
```

### Format Caddyfile
```bash
cd caddy
just fmt
```

### Reload Caddy (No Downtime)
```bash
cd caddy
just reload
```

### View Caddy Logs
```bash
docker compose -C caddy logs -f caddy
```

### Check All Routes
```bash
# All configured routes can be viewed in Caddyfile
cat caddy/Caddyfile
```

## 🔍 Debugging

### Service Not Reachable
1. Verify service is on `caddy` network:
   ```bash
   docker network inspect caddy
   ```

2. Test connectivity inside Caddy container:
   ```bash
   docker run --rm --network caddy nicolaka/netshoot \
     curl http://service-name:port
   ```

3. Check Caddyfile syntax:
   ```bash
   just validate
   ```

### Certificate Issues
1. Check Let's Encrypt logs:
   ```bash
   docker compose -C caddy logs caddy | grep -i acme
   ```

2. Verify domain DNS resolution:
   ```bash
   nslookup example.hybras.dev
   ```

3. Ensure ports 80 and 443 are accessible:
   ```bash
   curl -I https://example.hybras.dev
   ```

### Adding a New Service Route

1. Add route to Caddyfile:
   ```caddy
   newservice.hybras.dev {
       reverse_proxy localhost:8080
   }
   ```

2. Validate syntax:
   ```bash
   just validate
   ```

3. Format and commit:
   ```bash
   just fmt
   git add Caddyfile
   git commit -m "feat(caddy): add newservice route"
   ```

4. Reload Caddy:
   ```bash
   just reload
   ```

## 🔐 Security Considerations

- **Port 80/443**: Must be accessible from internet for Let's Encrypt challenges
- **ACME Rate Limiting**: Certificates are cached in volume to avoid hitting Let's Encrypt rate limits
- **Network Isolation**: Use `bind tailscale/` for sensitive services to hide from public internet
- **HTTPS Enforced**: All routes use HTTPS with automatic redirects from HTTP

## 🐛 Known Issues & Solutions

### Rate Limiting Errors
**Symptom**: "Too many requests" ACME errors  
**Solution**: The `caddy` volume caches certificates. Ensure volume isn't deleted unnecessarily.

### DNS Resolution Failures
**Symptom**: Service connects to wrong container or hangs  
**Solution**: Verify service hostname matches container name in Caddyfile `reverse_proxy` directive.

### VPN Routes Not Working
**Symptom**: Can't access Tailscale-bound services from VPN clients  
**Solution**: Ensure Caddy container is added as Headscale node and has correct Tailscale node name in `bind` directive.

## 📚 Related Documentation

- [Root README - Architecture](../README.md#-architecture-overview)
- [AGENTS.md - Caddy Configuration](../AGENTS.md#caddy-configuration)
- [AGENTS.md - Debugging](../AGENTS.md#debugging--troubleshooting)
- [Caddy Docs](https://caddyserver.com/docs/)

## 📝 Notes

- This service should always be running (it's the entry point for all traffic)
- Changes to Caddyfile don't require container restart (just `just reload`)
- Custom Caddy image builds SSL modules for advanced features
- DuckDNS integration enables dynamic DNS without manual updates

---

**Caddy Version**: Latest | **Status**: Core Infrastructure

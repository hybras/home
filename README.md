# Home Server Infrastructure

A comprehensive personal home server setup featuring multiple privacy-focused services, centralized authentication, and automatic container management.

## 🎯 Purpose

This repository contains a complete infrastructure-as-code setup for a home server running containerized services for personal use and sharing with a small group of friends and family. All services are:
- **Reverse-proxied** through Caddy with automatic HTTPS via Let's Encrypt
- **Accessible** via public domains or private VPN (Headscale)
- **Updated automatically** by Watchtower without manual intervention required
- **Authenticated** through centralized Authelia + LDAP

## 🏗️ Architecture Overview

```
Internet
    ↓
Router (Ports 80, 443)
    ↓
Home Server (Docker)
├── Caddy (Reverse Proxy)
│   ├── Public Services (Direct routing)
│   └── VPN Services (Tailscale routing)
├── Headscale (VPN Controller)
├── Authelia (Authentication)
└── 10+ Additional Services
```

See [AGENTS.md](AGENTS.md)

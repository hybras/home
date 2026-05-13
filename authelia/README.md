# Authelia - Centralized Authentication & LDAP

Authelia provides centralized authentication and authorization for all home server services. It uses LLDAP (a lightweight LDAP server) as the user directory and supports OIDC for service integration.

## 📋 Overview

| Aspect | Details |
|--------|---------|
| **Service Name** | authelia |
| **Image** | authelia/authelia:4.39.16 |
| **Companion** | LLDAP (user directory) |
| **Port** | 9091 (internal) |
| **Networks** | authelia, caddy |
| **Auth Methods** | LDAP, OIDC, OAuth2 |

## 🏗️ Architecture

### Components

#### Authelia
- Central authentication provider
- Session management
- Multi-factor authentication (configurable)
- OAuth2 / OIDC provider for service integration

#### LLDAP
- Lightweight LDAP directory
- User and group management
- Stored in SQLite (local database)
- Administrator interface on port 17170

### Authentication Flow

```
Client → Caddy (auth endpoint) → Authelia → LLDAP (user lookup)
                                    ↓
                          Session established
                                    ↓
                        Redirected to protected service
```

## 🔧 Configuration

### Key Files
- **configuration.yml** - Authelia main configuration
- **lldap_config.toml** - LLDAP server configuration
- **secrets/** - Docker secrets for sensitive data
  - `JWT_SECRET` - Session signing key
  - `SESSION_SECRET` - Session encryption key
  - `STORAGE_PASSWORD` - Database password
  - `STORAGE_ENCRYPTION_KEY` - Database encryption key
  - `AUTHENTICATION_BACKEND_LDAP_PASSWORD_FILE` - LDAP user password

## 👥 User Management

### Creating Users in LLDAP

1. **Access LLDAP Admin Interface**:
   ```
   https://yourdomain.com/lldap/
   ```
   - Username: admin
   - Password: Set in `LLDAP_ADMIN_PASSWORD` secret

2. **Add New User**:
   - Click "Users" → "Add User"
   - Set username, display name, email
   - Set password (users can change on first login)
   - Optional: Assign to groups

3. **Manage Groups**:
   - Navigate to "Groups" tab
   - Create group for role-based access (e.g., "admins", "users")
   - Assign users to groups

### User Attributes

Users in LDAP have these key attributes:
- `uid` - Username (used for login)
- `displayName` - Display name
- `mail` - Email address
- `userPassword` - Hashed password
- Group membership for access control

## 🔐 OIDC Integration for Services

### Headscale OIDC Configuration

Services can authenticate against Authelia via OIDC:

```yaml
# In headscale.yaml or service config
oidc:
  issuer: https://yourdomain.com
  client_id: headscale
  client_secret: <secret_from_authelia>
  scopes: ["openid", "profile", "email"]
```

### Adding New OIDC Consumer

1. Generate client secret:
   ```bash
   openssl rand -base64 32
   ```

2. Add to Authelia configuration:
   ```yaml
   identity_providers:
     oidc:
       clients:
         - client_id: newservice
           client_secret: <generated_secret>
           redirect_uris:
             - https://newservice.hybras.dev/callback
           response_types: [code]
           scopes: [openid, profile, email]
   ```

3. Configure service with OIDC credentials
4. Restart Authelia: `docker compose -C authelia up -d`

## 📦 Data Persistence

### Volumes
- **authelia-db** - Authelia database (sessions, policies)
- **authelia-sessions-db** - Session store
- **lldap-data** - LLDAP user directory database

### Backup Considerations
- LLDAP database contains all user accounts and passwords
- Authelia database contains sessions and configuration
- Secrets in `/secrets/` directory should be backed up securely

## 🚀 Common Tasks

### Change Admin Password

1. Access LLDAP interface:
   ```
   https://yourdomain.com/lldap/
   ```

2. Log in as admin (current password)

3. Click user menu → "Change Password"

### Reset User Password

From LLDAP admin interface:
1. Navigate to Users
2. Select user
3. Edit user → Change password
4. User must log in again

### Check Authentication Logs

```bash
docker compose -C authelia logs -f authelia | grep -i auth
```

### Verify LDAP Connectivity

```bash
# Test LDAP connection from temporary container
docker run --rm --network authelia \
  nicolaka/netshoot:latest \
  ldapsearch -x -H ldap://lldap:3890 \
    -D "uid=admin,ou=people,dc=hybras,dc=dev" \
    -W
```

## 🔍 Debugging

### Users Can't Login
1. Check LDAP server is running:
   ```bash
   docker compose -C authelia ps
   ```

2. Verify user exists:
   ```bash
   # Via LLDAP interface
   ```

3. Check Authelia logs:
   ```bash
   docker compose -C authelia logs authelia | tail -50
   ```

### OIDC Integration Issues
1. Verify client secret matches in Authelia config:
   ```bash
   cat authelia/data/authelia/config/configuration.yml | grep -A5 "client_id: headscale"
   ```

2. Check redirect URI matches service configuration

3. Review OIDC logs:
   ```bash
   docker compose -C authelia logs authelia | grep -i oidc
   ```

### Session Issues
1. Clear browser cookies for the domain
2. Restart Authelia:
   ```bash
   docker compose -C authelia restart authelia
   ```

## 🔒 Security Best Practices

- **Change default admin password** on first setup
- **Use strong passwords** for user accounts
- **Enable MFA** where possible (TOTP configured in authelia.yaml)
- **Limit OIDC scopes** to only what services need
- **Rotate secrets** periodically (JWT_SECRET, SESSION_SECRET)
- **Never commit** secret files to git (already in .gitignore)

## 📚 Related Documentation

- [Authelia Docs](https://www.authelia.com/)
- [LLDAP Docs](https://github.com/lldap/lldap)

# Enterprise Single Sign-On (SSO) Stack

A production-grade, containerized identity and access management stack featuring **Keycloak**, **Traefik v3**, **OAuth2 Proxy**, and **PostgreSQL 18**.

This architecture demonstrates a Zero-Trust forward authentication gateway: all incoming traffic passes through Traefik (terminating TLS via local wildcard certificates) and routes to services protected by OAuth2 Proxy with Keycloak as the central OpenID Connect (OIDC) identity provider.

---

## Architecture Overview

```
                      [ Client / Browser ]
                               │
            ┌──────────────────┴──────────────────┐
            │  https://*.chetan.com (Ports 80/443)│
            ▼                                     ▼
   ┌─────────────────────────────────────────────────────────┐
   │                       Traefik v3                        │
   │  (TLS Termination, HTTP->HTTPS Redirection, API Router)  │
   └────────┬─────────────────────┬─────────────────┬────────┘
            │                     │                 │
 https://proxy.chetan.com         │        https://app.chetan.com
   (Traefik Dashboard)            │                 │
            ▼                     │                 ▼
      [api@internal]              │        ┌──────────────────┐
                                  │        │   OAuth2 Proxy   │
                                  │        └────────┬─────────┘
                                  │                 │
                        https://auth.chetan.com     │ (Pass authenticated
                          (Keycloak Public OIDC)    │  request with headers)
                                  │                 ▼
                                  │        ┌──────────────────┐
                                  │        │   Target App     │
                                  │        │ (httpd / whoami) │
                                  │        └──────────────────┘
                                  ▼
                       ┌──────────────────────┐
                       │  Keycloak v26 (JVM)  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │   PostgreSQL 18      │
                       └──────────────────────┘
```

---

## Directory Structure

```text
KeyCloak-Docker-Compose/
├── keycloak/
│   ├── conf/
│   │   └── keycloak.conf            # Native Keycloak configuration (DB, Hostname, Logging)
│   └── logs/
│       └── keycloak.log             # Rotated physical log files (10MB max, 5 backups)
├── oauth2_proxy/
│   └── custom-templates/
│       └── sign_in.html             # Custom-branded login template for OAuth2 Proxy
├── traefik/
│   ├── certs/
│   │   ├── domain_public_certificate.crt  # Wildcard / domain TLS certificate
│   │   └── domain_secret_private.key      # TLS private key
│   ├── dynamic/
│   │   ├── routes.yaml              # Dynamic HTTP routers & service definitions
│   │   └── tls.yaml                 # Dynamic TLS certificate configuration
│   └── traefik.yaml                 # Traefik static configuration (Entrypoints, API, Logs)
├── .env                             # Environment variable overrides
├── .gitignore                       # Git ignore rules for logs, secrets, and envs
├── docker-compose.yaml              # Core Compose file defining the 5 services
└── README.md                        # Documentation
```

---

## Domain & Routing Map

Ensure the following domains resolve to `127.0.0.1` in your local `hosts` file (`C:\Windows\System32\drivers\etc\hosts` on Windows, or `/etc/hosts` on Linux/macOS):

```text
127.0.0.1  proxy.chetan.com
127.0.0.1  auth.chetan.com
127.0.0.1  app.chetan.com
```

| URL | Component | Purpose |
| :--- | :--- | :--- |
| **`https://proxy.chetan.com/dashboard/`** | Traefik | Traefik monitoring dashboard & routing overview |
| **`https://auth.chetan.com/admin`** | Keycloak | Keycloak Administrative Console |
| **`https://auth.chetan.com/realms/corp/account`** | Keycloak | User self-service profile and credential portal |
| **`https://app.chetan.com`** | OAuth2 Proxy $\rightarrow$ App | Protected application (enforces OIDC login) |

---

## Quick Start

### 1. Prerequisites
- Docker & Docker Compose v2+
- Wildcard certificates placed inside `traefik/certs/` (or update `traefik/dynamic/tls.yaml`)
- Host file entries added for the domains above

### 2. Start the Stack
```powershell
docker compose up -d
```

### 3. Check Service Status
```powershell
docker compose ps
```
All containers should report `Up` or `Up (healthy)`:
- `traefik`
- `keycloak_postgres`
- `keycloak`
- `oauth2_proxy`
- `httpd`

---

## Pre-Configured Credentials

### Keycloak Master Administrator
- **URL**: `https://auth.chetan.com/admin`
- **Username**: `admin`
- **Password**: `admin` *(configurable via `.env`)*

### Enterprise Realm & Demo User
- **Realm**: `corp` (*Enterprise Portal*)
- **OIDC Client ID**: `portal-web-gateway`
- **Client Secret**: `portal-gateway-secret-2026`
- **Test User**:
  - **Username**: `john.doe`
  - **Password**: `SecurePass2026!`
  - **Email**: `john.doe@corp.internal`

---

## Keycloak Configuration Highlights

Keycloak configuration resides in [`keycloak/conf/keycloak.conf`](keycloak/conf/keycloak.conf) rather than inline Compose environment variables:

```properties
# Database Configuration
db=postgres
db-url=jdbc:postgresql://postgres:5432/keycloak
db-username=keycloak
db-password=password

# Observability
health-enabled=true
metrics-enabled=true

# Logging with Automatic Size Rotation
log=console,file
log-file=/opt/keycloak/data/log/keycloak.log
log-file-rotation-enabled=true
log-file-rotation-max-file-size=10M
log-file-rotation-max-backup-index=5

# Reverse Proxy & Hostname Resolution
http-enabled=true
proxy-headers=xforwarded
hostname=https://auth.chetan.com
hostname-backchannel-dynamic=true
```

> [!IMPORTANT]
> **Why `hostname-backchannel-dynamic=true`?**
> Public clients access Keycloak over `https://auth.chetan.com`, while backend microservices (like `oauth2-proxy`) communicate with Keycloak across the internal Docker network (`http://keycloak:8080`). Setting dynamic backchannel resolution ensures public tokens retain `https://` schemes while allowing internal network token exchanges without hair-pinning through the public proxy.

---

## Logging & Rotation Strategy

Two levels of log management are enforced:
1. **Docker Daemon Level (`json-file`)**:
   Defined at the top of `docker-compose.yaml` via YAML anchor `*default-logging`, restricting stdout/stderr logs across all services to:
   - `max-size`: 10MB
   - `max-file`: 5
2. **Application Level (Keycloak)**:
   Physical rotating log files mounted to `./keycloak/logs/keycloak.log` on the host, capped at 10MB each with 5 backups.

---

## Troubleshooting & Verification

### View Live Traefik Logs
```powershell
docker compose logs -f traefik
```

### View Live OAuth2 Proxy Logs
```powershell
docker compose logs -f oauth2-proxy
```

### View Live Keycloak Logs
```powershell
docker compose logs -f keycloak
# Or read the host file directly:
Get-Content ./keycloak/logs/keycloak.log -Wait -Tail 50
```

### Common Issues
1. **Certificate Warnings in Browser**:
   Because certificates in `traefik/certs/` are self-signed, click **Advanced** $\rightarrow$ **Proceed to site** in Chrome/Edge.
2. **Audience Mismatch (`aud` claim)**:
   Keycloak tokens must contain the client audience (`portal-web-gateway`). This is configured in Keycloak under:
   `Clients` $\rightarrow$ `portal-web-gateway` $\rightarrow$ `Client scopes` $\rightarrow$ `dedicated` $\rightarrow$ `Mappers` $\rightarrow$ `Audience`.


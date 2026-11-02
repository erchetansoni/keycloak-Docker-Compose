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
   │  (TLS Termination, HTTP->HTTPS Redirection, API Router) │
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
│   ├── import/
│   │   └── corp-realm.json          # Automated Day-0 Realm configuration (Client, Mappers, Users)
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

## Day-0 Workflow: Automated vs Manual

### Option A: Fully Automated Day-0 Startup (Zero Manual Config)

This repository includes a pre-exported realm definition at [`keycloak/import/corp-realm.json`](keycloak/import/corp-realm.json). 

Keycloak runs with `command: start --import-realm` and mounts `./keycloak/import:/opt/keycloak/data/import:ro`. 

On the very first launch (`docker compose up -d`), Keycloak **automatically provisions**:
1. The **`corp`** realm.
2. The **`portal-web-gateway`** client (with client secret `portal-gateway-secret-2026`).
3. The **Audience protocol mapper** (`aud: portal-web-gateway`).
4. The test user **`john.doe`** (with pre-set password `SecurePass2026!`).

> [!NOTE]
> If the database already exists, Keycloak skips the import to preserve existing state and modifications.

---

### Option B: Manual Step-by-Step Configuration (If Starting Fresh)

If you wipe your database volume or choose to build the realm by hand from scratch, follow this exact workflow:

#### Step 1: Log in to the Admin Console
1. Open **`https://auth.chetan.com/admin`**.
2. Log in with the master bootstrap credentials (`admin` / `admin`).

#### Step 2: Create the Enterprise Realm
1. Click the top-left dropdown (currently `master`).
2. Click **Create Realm**.
3. Set **Realm name**: `corp`
4. Set **Display name**: `Enterprise Portal`
5. Click **Create**.

#### Step 3: Create the OIDC Client (`portal-web-gateway`)
1. In the `corp` realm, click **Clients** $\rightarrow$ **Create client**.
2. **General Settings**:
   - **Client type**: `OpenID Connect`
   - **Client ID**: `portal-web-gateway`
   - Click **Next**.
3. **Capability Config**:
   - **Client authentication**: `ON` *(Confidential client)*
   - **Standard flow**: `ON` *(Authorization Code Flow)*
   - **Direct access grants**: `OFF`
   - Click **Next**.
4. **Login Settings**:
   - **Root URL**: `https://app.chetan.com`
   - **Valid redirect URIs**: `https://app.chetan.com/*`
   - **Web origins**: `+`
   - Click **Save**.
5. **Credentials Tab**:
   - Under **Client Secret**, copy the generated secret (or set it to `portal-gateway-secret-2026`).
   - If changed, update `OAUTH2_PROXY_CLIENT_SECRET` in `docker-compose.yaml` (or `.env`).

#### Step 4: Add the Audience Protocol Mapper (CRITICAL)
Without this, OAuth2 Proxy will fail with `audience does not match`:
1. In the `portal-web-gateway` client, click the **Client scopes** tab.
2. Click the link for **`portal-web-gateway-dedicated`**.
3. Click **Configure a new mapper** (or **Add mapper** $\rightarrow$ **By configuration**).
4. Select **Audience**.
5. Configure:
   - **Name**: `portal-gateway-audience`
   - **Included Client Audience**: `portal-web-gateway`
   - **Add to ID token**: `ON`
   - **Add to access token**: `ON`
6. Click **Save**.

#### Step 5: Create a Test User
1. In the `corp` realm, click **Users** $\rightarrow$ **Add user**.
2. Enter:
   - **Username**: `john.doe`
   - **Email**: `john.doe@corp.internal`
   - **Email verified**: `ON`
   - **First name**: `John`
   - **Last name**: `Doe`
3. Click **Create**.
4. Switch to the **Credentials** tab:
   - Click **Set password**.
   - **Password**: `SecurePass2026!`
   - **Temporary**: `OFF`
   - Click **Save** $\rightarrow$ **Save password**.

---

## Quick Start Commands

### 1. Start the Stack
```powershell
docker compose up -d
```

### 2. Check Service Status
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
   Keycloak tokens must contain the client audience (`portal-web-gateway`). Ensure the audience mapper is attached to the client scopes.

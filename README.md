# Homelab Docker

Personal homelab IaC — a collection of Docker Compose stacks for self-hosted services. Primarily a backup/reference for myself, but feel free to use any of it.

---

## Overview

Every service runs on a shared internal Docker network (`internal_network`) with no ports exposed directly to the host. All external traffic goes through Nginx Proxy Manager, and authentication is handled centrally by Authentik (SSO via OIDC/OAuth2).

Each service that needs a database gets its own isolated instance — no shared databases. This keeps stacks independent, avoids version conflicts, and makes it easy to tear one down without touching anything else.

---

## Deployment Order

> **Deploy `NPM-Authentik` first.** It creates the `internal_network` that every other stack attaches to, and Authentik provides the SSO that several services depend on.

After that, deploy whichever stacks you want in any order.

---

## Stacks

### NPM-Authentik — _Deploy this first_
**Path:** `NPM-Authentik/`

The backbone of the whole setup.

| Service | Description |
|---|---|
| Nginx Proxy Manager | Reverse proxy + Let's Encrypt SSL. Admin UI bound to `127.0.0.1:81` only. |
| Authentik | Identity provider / SSO (OIDC, OAuth2, LDAP). |
| PostgreSQL 16 | Dedicated DB for Authentik. |

Creates the `internal_network` bridge (`172.18.0.0/16`) used by all other stacks.

**Setup:**
```bash
cp .env.example .env
# Fill in POSTGRES_PASSWORD, POSTGRES_USER, POSTGRES_DB, AUTHENTIK_SECRET_KEY
docker compose up -d
```

---

### Arrstack — Media / Torrent
**Path:** `Arrstack/`

Torrent client and browser, both routed through a Mullvad WireGuard VPN via Gluetun.

| Service | Description |
|---|---|
| Gluetun | VPN gateway container (Mullvad + WireGuard). All traffic from dependent services exits through this. |
| qBittorrent | Torrent client. Uses Gluetun's network. |
| Firefox | Sandboxed browser. Also uses Gluetun's network. |

**Setup:**
```bash
cp .env.example .env
# Fill in WIREGUARD_PRIVATE_KEY, WIREGUARD_ADDRESSES, SERVER_CITIES, download paths
docker compose up -d
```

---

### BentoPDF — PDF Tools
**Path:** `BentoPDF/`

Lightweight PDF utility suite (running in simple mode). No configuration needed.

```bash
docker compose up -d
```

---

### Blinko — Notes
**Path:** `Blinko/`

A self-hosted note-taking / quick-capture app.

| Service | Description |
|---|---|
| Blinko | Web app. |
| PostgreSQL 14 | Dedicated DB. |

**Setup:**
```bash
cp .env.example .env
# Fill in PGBLPASS and NXTSCT (NextAuth secret)
# Update NEXTAUTH_URL and NEXT_PUBLIC_BASE_URL in docker-compose.yml
docker compose up -d
```

---

### Dashy — Monitoring Dashboard
**Path:** `Dashy/`

Server monitoring dashboard, configured with Glances widgets for CPU, RAM, and disk. The config lives in `dashy/conf/dashy_conf.yml` — update the Glances API hostnames there to point at your Glances instances.

```bash
docker compose up -d
```

---

### Docmost — Wiki / Docs
**Path:** `Docmost/`

Collaborative wiki and documentation platform. Requires WebSockets to be enabled in your reverse proxy config.

| Service | Description |
|---|---|
| Docmost | Web app. |
| PostgreSQL 16 | Dedicated DB. |
| Redis 7.2 | Cache layer. |

**Setup:**
```bash
cp .env.example .env
# Fill in APP_SECRET and POSTGRES_PASSWORD
# Update APP_URL in docker-compose.yml
docker compose up -d
```

---

### Excalidraw — Whiteboard
**Path:** `Excalidraw/`

Self-hosted Excalidraw virtual whiteboard. No configuration needed.

```bash
docker compose up -d
```

---

### Navidrome — Music Streaming
**Path:** `Navidrome/`

Self-hosted music streaming server (Subsonic/Airsonic-compatible). Drop your music into `navidrome/Music/`.

```bash
docker compose up -d
```

---

### Nextcloud — Cloud Storage
**Path:** `Nextcloud/`

Self-hosted file sync and cloud storage.

| Service | Description |
|---|---|
| Nextcloud | Web app. |
| MariaDB 10.6 | Dedicated DB. |

A `config.php.example` is included as a reference for the Nextcloud config file (lives inside the `nextcloud` Docker volume at `/var/www/html/config/config.php`).

**Setup:**
```bash
cp .env.example .env
# Fill in MYSQL_ROOT_PASSWORD and MYSQL_PASSWORD
# Update NEXTCLOUD_TRUSTED_DOMAINS in docker-compose.yml
docker compose up -d
```

---

### Onlyoffice — Document Server
**Path:** `Onlyoffice/`

Document editing server, intended to integrate with Nextcloud.

| Service | Description |
|---|---|
| Onlyoffice Document Server | Document editor. JWT-secured. |
| RabbitMQ 4 | Message broker. |
| PostgreSQL 12 | Dedicated DB. |

**Setup:**
```bash
cp .env.example .env
# Fill in JWT_SECRET
docker compose up -d
```

---

### OpenWebUI — AI Chat Interface
**Path:** `OpenWebUI/`

Web interface for local LLMs via Ollama. Configured for Authentik SSO (login form disabled — OAuth only), Brave Search for RAG web search, and points at a remote Ollama instance.

**Setup:**
```bash
cp .env.example .env
# Fill in WEBUI_SECRET_KEY, BRAVE_SEARCH_API_KEY, OLLAMA_BASE_URL,
# OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET, OPENID_PROVIDER_URL, OPENID_REDIRECT_URI
docker compose up -d
```

---

### Portainer — Docker Management
**Path:** `Portainer/`

Docker container management UI (Community Edition).

```bash
docker compose up -d
```

---

### Privmeta — Metadata Manager
**Path:** `Privmeta/`

Self-hosted metadata management app. This stack builds from source — you need to clone the upstream repo first.

```bash
git clone https://github.com/DScaife/privmeta Privmeta/
docker compose up -d
```

---

### Samba — Network File Share
**Path:** `Samba/`

SMB/CIFS network share. Exposed on port `445` (the only stack with a direct host port).

**Setup:**
```bash
cp .env.example .env
# Fill in S_PATH (path to share), U_NAME, P_WORD
docker compose up -d
```

---

### Starbase 81 — Homepage / Dashboard
**Path:** `Starbase81/`

Custom server homepage with Authentik widget support. Builds from source — clone the upstream repo first, then mount your config and assets.

```bash
git clone https://github.com/suckharder/starbase-81 Starbase81/starbase-81-src
# Add config.json, favicon, logo, background, and icons to Starbase81/
docker compose up -d
```

---

## Network Architecture

```
Internet
   │
   ▼
Nginx Proxy Manager (ports 80, 443)
   │
   ▼  (internal_network: 172.18.0.0/16)
┌──────────────────────────────────────────────┐
│  Authentik   Nextcloud   Navidrome   ...etc  │
│  Portainer   Docmost     OpenWebUI           │
└──────────────────────────────────────────────┘

Arrstack traffic:
   └── Gluetun (Mullvad VPN) ──► qBittorrent / Firefox

Samba:
   └── Host port 445 (direct, no proxy)
```

All services communicate over `internal_network`. NPM handles SSL termination and routing. No service ports are bound to `0.0.0.0` on the host (except Samba and NPM's 80/443).

---

## Secrets & `.env` Files

Each stack that needs secrets has a `.env.example`. Copy it to `.env` and fill in the values before deploying. Secrets should be generated with:

```bash
openssl rand -base64 36   # for passwords / secret keys
openssl rand -hex 32      # for hex tokens / JWT secrets
```

Never commit your `.env` files. They are excluded from this repo.

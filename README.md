# Traefik

Shared Traefik reverse proxy for all BluesBraces projects.

Runs as a standalone Docker Compose stack on the VPS at `/opt/traefik`. Auto-discovers containers via Docker labels on the `proxy` network and provisions Let's Encrypt TLS certificates.

## Structure

```
├── docker-compose.yml   # Traefik service definition
├── .env.example         # Template for ACME email
└── .gitignore           # Excludes .env and acme.json
```

## Deployment

### CI/CD (GitHub Actions)

Push to `main` triggers `.github/workflows/deploy.yml` which SSHs into the VPS, clones/pulls the repo via `GHCR_TOKEN`, and runs `docker compose up -d`.

Required secrets: `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `GHCR_TOKEN`, `ACME_EMAIL`.

### Manual deployment

SSH into the VPS and run:

```bash
# First time only
docker network create proxy

# Clone / update repo
cd /opt
git clone https://github.com/JanKrajewskiIT/traefik.git traefik 2>/dev/null || (cd traefik && git pull)
cd /opt/traefik

# Configure (first time only)
cp .env.example .env    # Edit ACME_EMAIL

# Ensure acme.json exists with correct permissions
touch acme.json && chmod 600 acme.json

# Deploy
docker compose pull
docker compose up -d --remove-orphans
```

## How projects connect

Each project adds Traefik labels to its services in a `docker-compose.traefik.yml` overlay and joins the `proxy` network. Traefik auto-discovers them — no central config needed.

### Current subdomains

| Subdomain | Project | Service |
|-----------|---------|---------|
| `app.bluebraces.online` | Recron | SolidJS frontend |
| `api.bluebraces.online` | Recron | YARP API gateway |
| `auth.bluebraces.online` | Keycloak | Identity provider |
| `docs.bluebraces.online` | Recron | MkDocs |
| `velvet.bluebraces.online` | VelvetUi | Demo app |

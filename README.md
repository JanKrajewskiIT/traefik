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

```bash
# On VPS — first time
docker network create proxy
cd /opt/traefik
cp .env.example .env    # Edit ACME_EMAIL
touch acme.json && chmod 600 acme.json
docker compose up -d
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

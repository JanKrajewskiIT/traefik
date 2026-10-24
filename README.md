# Traefik

Shared reverse proxy and deployment configuration for hosted projects.

Shared service documentation: [Recron project docs](https://docs.bluebraces.online/projects/traefik/).

## QuantCore beside Recron

The existing Docker provider already discovers containers on the external
`proxy` network with `traefik.enable=true`. Its `websecure` entrypoint and
`letsencrypt` resolver match QuantCore's deployment labels. No static Traefik
configuration change or proxy restart is required to add QuantCore.

Point these DNS names to the same VPS before deploying the application:

| Host | Route owner |
|---|---|
| `quantcore.bluebraces.online` | QuantCore frontend WAF |
| `quantcore-api.bluebraces.online` | QuantCore API and Scalar WAFs; Scalar UI at `/` |
| `quantcore-docs.bluebraces.online` | QuantCore documentation container |

The routes are declared in QuantCore's `deploy/docker-compose.frontend.yml` and
`deploy/backend/docker-compose.yml`. Only the WAF containers and documentation
join `proxy`; application upstreams remain on their private Compose networks.
Scalar serves `/`, `/scalar.js` and `/favicon.svg` through its higher-priority
router on the API hostname; all other API paths retain the API WAF route.
Router and service names use the `quantcore` prefix, leaving Recron's routes
separate. DNS can use an existing wildcard record if it already targets this VPS.

Keycloak keeps its existing `auth.bluebraces.online` route; QuantCore adds a realm
on that server rather than a second identity-provider hostname. Validate public
HTTPS after deploying QuantCore; the existing proxy configuration alone does not
prove that DNS, application health or certificate issuance have succeeded.

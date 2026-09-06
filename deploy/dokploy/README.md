# OpenMAIC on Dokploy (Tailscale-only)

Topology: `ghcr.io/sesal78/openmaic:tailnet` + Postgres 16 on the attachable `dokploy-network` overlay (no Traefik domain), published on the Dokploy
host's Tailscale IP only (`http://100.121.168.67:3300`). No Traefik domain, no public
exposure.

## Pipeline

1. Push to `feat/dokploy-tailnet-deploy` (or run the workflow manually) builds the image
   in GitHub Actions and pushes `:tailnet` + `:sha-<short>` tags to GHCR.
2. In Dokploy, project **OpenMAIC** > compose **openmaic** > Environment: set
   `OPENMAIC_IMAGE_TAG=sha-<short sha>` (printed by the workflow), then *Redeploy*.
   Always pin the immutable `sha-` tag: `docker compose up` does not re-pull a moving
   tag such as `:tailnet`, so redeploying on it silently keeps the old image.
   Rollback = set the previous `sha-` tag and redeploy.

## Required env (Dokploy > compose > Environment)

| Key | Notes |
| --- | --- |
| `POSTGRES_PASSWORD` | Postgres role password; only applied on first init of the volume |
| `OPENMAIC_IMAGE_TAG` | immutable `sha-<short>` tag from the workflow run |
| `OPENMAIC_AGENT_RUNTIME_ENABLED` | must be `true`: the home page lists courses via `/api/stages`, which 404s when the runtime is off |
| `PERSISTENCE_DEV_TOKEN` | Must equal the build's `NEXT_PUBLIC_PERSISTENCE_TOKEN` (default `openmaic-tailnet-dev`) |
| at least one LLM key | e.g. `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY` |

Optional: `ACCESS_CODE` (site password), `MODEL_ROUTES` routing `maic-agent-driver` to
an `openai-completions` model so agent runs can execute, `OPENMAIC_PORT` / `TAILSCALE_BIND_IP`.

## Not included

- `render-service` (MP4 export) needs 8 GB RAM; the host has ~2.5 GB free. Video export
  falls back to ZIP download.
- HTTPS. Traffic is WireGuard-encrypted on the tailnet, but the browser sees plain
  `http://`, so mic-based features (ASR) that need a secure context won't work until
  `tailscale serve` is added on the host (needs SSH access to the dokploy node).

## Host CPU requirement

sharp 0.35 prebuilt Linux x64 binaries require the x86-64-v2 microarchitecture. The
Dokploy VM originally ran a generic CPU model, and the agent runner failed at startup
with `Unsupported CPU`. Fixed on 2026-09-06 by setting the Proxmox VM CPU type to
`host` and restarting. The image also ships `@img/sharp-wasm32` as a slower automatic
fallback for hosts without v2 (see `pnpm-workspace.yaml` and the Dockerfile mirror step).

## Known-good state (2026-09-06)

- `/api/health`, `/api/stages`, `/api/agent/sessions` answer 200 over the tailnet.
- Agent runner logs `[AgentRunner] runner ... started` after boot.
- Public IP `115.69.2.80:3300` refuses connections.

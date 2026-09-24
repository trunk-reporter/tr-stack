# AGENTS.md

This file provides guidance to Codex and other coding agents when working in `tr-stack`.

## What This Project Is

Single Docker Compose stack for Trunk Reporter: trunk-recorder, tr-engine, tr-dashboard, imbe-asr, PostgreSQL, Mosquitto, and Caddy.

## Commands

```bash
cp sample.env .env
docker compose config
docker compose up -d
docker compose logs -f tr-engine
```

Use `docker compose -f docker-compose.yml -f docker-compose.pi.yml up -d` for Raspberry Pi / CPU-oriented deployments.

## Change Guidance

- Keep the default path friendly to first-time hobbyist installs.
- Do not require NVIDIA GPU support in the default compose file; GPU support should be an override or clearly optional.
- Auth docs must use the current tr-engine model: `open`, `token`, and `full` modes derived from `AUTH_TOKEN` and `ADMIN_PASSWORD`.
- Avoid exposing anonymous MQTT broadly by default; document LAN-only assumptions when ports are published.

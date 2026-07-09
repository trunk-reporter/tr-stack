# tr-stack

Full P25 transcription stack in a single `docker compose up`. Includes trunk-recorder, tr-engine, tr-dashboard, imbe-asr, PostgreSQL, and Mosquitto.

## What's Included

| Service | Image | Purpose |
|---|---|---|
| `trunk-recorder` | `ghcr.io/trunk-reporter/trunk-recorder` | P25 scanner with common plugins (incl. MQTT + DVCF) |
| `tr-engine` | `ghcr.io/trunk-reporter/tr-engine` | Backend API + call ingestion + transcription routing |
| `tr-dashboard` | `ghcr.io/trunk-reporter/tr-dashboard` | Live web UI (static files on port 3000) |
| `imbe-asr` | `ghcr.io/trunk-reporter/imbe-asr-server` | IMBE vocoder → text (no audio reconstruction) |
| `caddy` | `caddy:2-alpine` | Reverse proxy: `/api/*` → engine, UI → dashboard |
| `postgres` | `postgres:17-alpine` | Database |
| `mosquitto` | `eclipse-mosquitto:2` | MQTT broker |

## Quick Start

```bash
git clone https://github.com/trunk-reporter/tr-stack.git
cd tr-stack

# 1. Configure
cp sample.env .env
# Edit .env — set POSTGRES_PASSWORD and SITE_ADDRESS at minimum
# SITE_ADDRESS should be http://YOUR_SERVER_IP (what your browser uses to reach it)

# 2. Edit config.json — set your SDR source, frequency, and system details
# See: https://trunkrecorder.com/docs/intro for trunk-recorder config reference

# 3. Add your talkgroup CSV
mkdir -p talkgroups
cp /path/to/your/talkgroups.csv talkgroups/talkgroups.csv

# 4. Start
docker compose up -d

# 5. Watch logs
docker compose logs -f tr-engine
```

**Dashboard:** http://YOUR_SERVER_IP (Caddy on port 80)  
**API:** http://YOUR_SERVER_IP/api/v1  

Auth starts in **open mode** when `AUTH_TOKEN` and `ADMIN_PASSWORD` are unset (fine for trusted LAN). See [Authentication](#authentication) before exposing the stack externally.

> **GPU:** The default compose file reserves an NVIDIA GPU for `imbe-asr`. On hosts without NVIDIA / nvidia-container-toolkit, use [CPU-only](#gpu) or the [Pi override](#running-on-raspberry-pi).

## Configuration

### .env

Copy `sample.env` to `.env` and set your values. Minimum required:

```env
POSTGRES_PASSWORD=your-secure-password
SITE_ADDRESS=http://192.168.1.100    # your server's IP or hostname
```

See `sample.env` for the full annotated list (auth, MQTT topics, IMBE model, device).

### Authentication

tr-engine uses a **three-mode** auth model (same as standalone tr-engine). Mode is derived from environment variables — **`AUTH_ENABLED` is obsolete** and is ignored.

| Mode | Config | Behavior |
|------|--------|----------|
| **open** | Neither `AUTH_TOKEN` nor `ADMIN_PASSWORD` set | No auth; API and UI are public |
| **token** | `AUTH_TOKEN` only | Shared bearer token required |
| **full** | `ADMIN_PASSWORD` set | JWT login (user `admin`). Optional `AUTH_TOKEN` = public read token from `/api/v1/auth-init` |

**Public-facing example (recommended):**

```env
ADMIN_PASSWORD=       # openssl rand -base64 32 — login as admin
AUTH_TOKEN=           # optional: openssl rand -base64 32 — guest read access
```

- Dashboard users log in with username **`admin`** and your `ADMIN_PASSWORD`.
- Caddy can inject `AUTH_TOKEN` for browser requests that do not already send `Authorization` (see `Caddyfile`). Prefer letting the dashboard use `auth-init` / JWT where possible.
- For upload plugins and scripts, create `tre_...` API keys in full mode rather than using the deprecated `WRITE_TOKEN`.

See the [tr-engine auth migration guide](https://github.com/trunk-reporter/tr-engine/blob/master/docs/migrating-auth.md) for details.

### Transcription (IMBE)

Default stack config:

```env
STT_PROVIDER=imbe
IMBE_ASR_URL=http://imbe-asr:8000
```

This requires DVCF capture from trunk-recorder (`tr-plugin-dvcf` is included in the stack image plugins).

**Limitation:** With `STT_PROVIDER=imbe`, only P25 digital calls that produce `.dvcf` data are transcribed. **Analog / conventional calls get no transcription.** Dual-provider routing (IMBE + audio STT fallback) is tracked as a tr-engine 1.0 blocker.

### config.json

Edit `config.json` to match your SDR hardware and radio system. Key fields:

- `sources` — your SDR device, center frequency, sample rate, gain
- `systems` — P25 control channel frequency, system type, talkgroup CSV path
- `plugins` — pre-configured for MQTT + DVCF; leave the `broker` pointing at the internal `mosquitto` service unless you use an external broker

### Talkgroup CSV

Place your talkgroup CSV at `talkgroups/talkgroups.csv`. RadioReference format works directly.

## IMBE-ASR Models

Models download from Hugging Face on first run (~560MB for the P25 fine-tuned model). Check progress with `docker compose logs imbe-asr`.

To pre-download:

```bash
pip install huggingface_hub
python3 -c "
from huggingface_hub import snapshot_download
snapshot_download('trunk-reporter/imbe-asr-base-512d-p25', local_dir='data/models')
"
```

Large model (better accuracy on clean speech, 290M params):

```bash
python3 -c "
from huggingface_hub import snapshot_download
snapshot_download('trunk-reporter/imbe-asr-large-1024d', local_dir='data/models')
"
```

Then set `IMBE_ASR_MODEL=trunk-reporter/imbe-asr-large-1024d` (and optional LM tuning vars if your image supports them) in `.env`.

## Running on Raspberry Pi

The stack runs on Raspberry Pi 5 (arm64) using CPU-only inference. A `docker-compose.pi.yml` override handles Pi-specific configuration.

### Prerequisites

- Raspberry Pi 5 (4GB+ RAM recommended, 8GB ideal)
- 64-bit Raspberry Pi OS (Bookworm or later)
- Docker Engine + Docker Compose v2
- RTL-SDR or compatible SDR dongle

### Usage

```bash
# Same setup steps as Quick Start, then:
docker compose -f docker-compose.yml -f docker-compose.pi.yml up -d
```

The Pi override:

- Switches imbe-asr to the `:cpu` image tag (multi-arch, no GPU drivers)
- Uses the smaller `imbe-asr-base-512d` model by default
- Sets `IMBE_ASR_DEVICE=cpu`
- Removes GPU reservation and privileged mode where appropriate

### Performance

CPU inference on Pi is significantly slower than GPU. Prefer the base model. Monitor RAM; reduce `IMBE_ASR_BEAM_WIDTH` if the Pi swaps or OOMs.

## GPU

GPU is strongly recommended for imbe-asr. The default `docker-compose.yml` reserves one NVIDIA device. Requirements: NVIDIA driver + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

**CPU-only (non-Pi):**

```env
IMBE_ASR_DEVICE=cpu
```

And either use `docker-compose.pi.yml` as a template for a CPU override, or edit the `imbe-asr` service to remove `deploy.resources.reservations.devices` so Compose does not require an NVIDIA GPU.

## Upgrading

```bash
docker compose pull && docker compose up -d
```

Database and audio data under `./data` persist across upgrades.

## Ports

| Service | Port | Notes |
|---|---|---|
| Caddy (dashboard + API) | `HTTP_PORT` (default **80**) | Browser entrypoint |
| Mosquitto MQTT | **1883** | For trunk-recorder on other machines |
| tr-engine | **8080** (internal Docker network) | Not published by default; use Caddy `/api/*` |
| tr-dashboard | **3000** (internal) | Static UI behind Caddy |

All published ports bind to `0.0.0.0` by default. Set `BIND_IP` in `.env` to restrict to a specific interface.

> **Security note:** Default Mosquitto allows anonymous MQTT. For untrusted networks, lock down the broker and/or bind MQTT to a private interface only.

## Troubleshooting

| Symptom | Check |
|---------|--------|
| Dashboard blank / API 502 | `docker compose logs caddy tr-engine` |
| Login fails | `ADMIN_PASSWORD` set? Restart after changing `.env` |
| No transcriptions | `STT_PROVIDER=imbe` needs DVCF plugin + healthy `imbe-asr`; analog calls will not transcribe |
| imbe-asr won't start | GPU reservation on non-NVIDIA host → use CPU override |
| MQTT not connecting | `MQTT_TOPICS` must match `config.json` plugin topics |

## Setup Guide

Full step-by-step setup: [docs.luxprimatech.com/#/imbe-asr-setup](https://docs.luxprimatech.com/#/imbe-asr-setup)

## Roadmap

See the [Trunk Reporter Roadmap](https://github.com/orgs/trunk-reporter/projects/1) for the cross-repo project tracker.

## Related

- [tr-docker](https://github.com/trunk-reporter/tr-docker) — trunk-recorder image source + CI
- [tr-engine](https://github.com/trunk-reporter/tr-engine) — backend source
- [tr-dashboard](https://github.com/trunk-reporter/tr-dashboard) — UI source
- [imbe-asr](https://github.com/trunk-reporter/imbe-asr) — ASR model source
- [tr-plugin-dvcf](https://github.com/trunk-reporter/tr-plugin-dvcf) — DVCF plugin
- [symbolstream](https://github.com/trunk-reporter/symbolstream) — live streaming plugin

# tr-stack

Full P25 transcription stack in a single `docker compose up`. Includes trunk-recorder, tr-engine, tr-dashboard, imbe-asr, PostgreSQL, and Mosquitto.

## What's Included

| Service | Image | Purpose |
|---|---|---|
| `trunk-recorder` | `ghcr.io/trunk-reporter/trunk-recorder` | P25 scanner with all common plugins |
| `tr-engine` | `ghcr.io/trunk-reporter/tr-engine` | Backend API + call ingestion + transcription routing |
| `tr-dashboard` | `ghcr.io/trunk-reporter/tr-dashboard` | Live web UI |
| `imbe-asr` | `ghcr.io/trunk-reporter/imbe-asr-server` | IMBE vocoder → text (no audio reconstruction) |
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

**Dashboard:** http://YOUR_SERVER_IP (served by Caddy on port 80)  
**API:** http://YOUR_SERVER_IP/api/v1

Auth is disabled by default — no login required for local installs. See [Securing for Public Access](#securing-for-public-access) if you want to expose this externally.

## Configuration

### .env

Copy `sample.env` to `.env` and set your values. Minimum required:

```env
POSTGRES_PASSWORD=your-secure-password
SITE_ADDRESS=http://192.168.1.100    # your server's IP or hostname
```

Auth is disabled by default (`AUTH_ENABLED=false`). The dashboard is accessible without login. See [Securing for Public Access](#securing-for-public-access) to enable auth.

### Securing for Public Access

If you're exposing the stack to the internet, enable auth:

```env
AUTH_ENABLED=true
AUTH_TOKEN=     # openssl rand -base64 32
WRITE_TOKEN=    # openssl rand -base64 32
ADMIN_PASSWORD= # dashboard login password
```

With auth enabled, Caddy injects the read token for browser requests automatically. The dashboard will show a login prompt — use username `admin` and your `ADMIN_PASSWORD`.

### config.json

Edit `config.json` to match your SDR hardware and radio system. Key fields:

- `sources` — your SDR device, center frequency, sample rate, gain
- `systems` — P25 control channel frequency, system type, talkgroup CSV path
- `plugins` — pre-configured for MQTT + DVCF, update `broker` if using external MQTT

The `broker` in the plugin config points to the internal `mosquitto` container — leave as-is unless you're using an external broker.

### Talkgroup CSV

Place your talkgroup CSV at `talkgroups/talkgroups.csv`. RadioReference format works directly.

## IMBE-ASR Models

Models download from Hugging Face on first run (~560MB for P25 fine-tuned model). The download happens inside the container on startup — check logs with `docker compose logs imbe-asr`.

To pre-download:
```bash
pip install huggingface_hub
python3 -c "
from huggingface_hub import snapshot_download
snapshot_download('trunk-reporter/imbe-asr-base-512d-p25', local_dir='data/models')
"
```

To use the large model (better accuracy on clean speech, 290M params):
```bash
python3 -c "
from huggingface_hub import snapshot_download
snapshot_download('trunk-reporter/imbe-asr-large-1024d', local_dir='data/models')
"
```
Then set `IMBE_ASR_LM_ALPHA=0.7` and `IMBE_ASR_LM_BETA=2.0` in `.env`.

## Running on Raspberry Pi

The stack runs on Raspberry Pi 5 (arm64) using CPU-only inference. A `docker-compose.pi.yml` override handles all Pi-specific configuration automatically.

### Prerequisites

- Raspberry Pi 5 (4GB+ RAM recommended, 8GB ideal)
- 64-bit Raspberry Pi OS (Bookworm or later)
- Docker Engine + Docker Compose v2 installed
- RTL-SDR or compatible SDR dongle

### Usage

```bash
# Same setup steps as Quick Start, then:
docker compose -f docker-compose.yml -f docker-compose.pi.yml up -d
```

The Pi override:
- Switches imbe-asr to the `:cpu` image tag (multi-arch, no GPU drivers needed)
- Uses the smaller `imbe-asr-base-512d` model by default (lower memory, faster on ARM)
- Sets `IMBE_ASR_DEVICE=cpu`
- Removes GPU reservation and privileged mode

### SDR on Pi

RTL-SDR is the most common SDR for Pi deployments. Your `config.json` source settings will differ from x86 setups — make sure to set the correct device index and gain for your dongle. See the [trunk-recorder docs](https://trunkrecorder.com/docs/intro) for source configuration.

### Performance

CPU inference on Pi is significantly slower than GPU. Expect higher latency on transcriptions. The base model (`imbe-asr-base-512d`) is recommended over the P25-tuned or large models to keep inference time reasonable. Monitor memory usage — if the Pi runs out of RAM, consider reducing `IMBE_ASR_BEAM_WIDTH` in `.env`.

> **Note:** The [GPU](#gpu) section below does not apply to Pi deployments.

## GPU

GPU is strongly recommended for imbe-asr. The stack assumes NVIDIA GPU with the Docker NVIDIA runtime installed. For CPU-only:

```env
IMBE_ASR_DEVICE=cpu
```

And remove the `deploy.resources` section from the `imbe-asr` service in `docker-compose.yml`.

## Upgrading

```bash
docker compose pull && docker compose up -d
```

## Ports

| Service | Port | Notes |
|---|---|---|
| tr-engine API | 8080 | Set `HTTP_PORT` in `.env` |
| Caddy (dashboard + API) | 80 | Set `HTTP_PORT` in `.env` |
| Mosquitto MQTT | 1883 | For trunk-recorder instances on other machines |

All ports bind to `0.0.0.0` by default. Set `BIND_IP` in `.env` to restrict to a specific interface.

## Setup Guide

Full step-by-step setup: [docs.luxprimatech.com/#/imbe-asr-setup](https://docs.luxprimatech.com/#/imbe-asr-setup)

## Related

- [tr-docker](https://github.com/trunk-reporter/tr-docker) — trunk-recorder image source + CI
- [tr-engine](https://github.com/trunk-reporter/tr-engine) — backend source
- [imbe-asr](https://github.com/trunk-reporter/imbe-asr) — ASR model source
- [tr-plugin-dvcf](https://github.com/trunk-reporter/tr-plugin-dvcf) — DVCF plugin
- [symbolstream](https://github.com/trunk-reporter/symbolstream) — live streaming plugin

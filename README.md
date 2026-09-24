# tr-stack

Full P25 transcription stack in a single `docker compose up`. Includes trunk-recorder, tr-engine, tr-dashboard, imbe-asr, PostgreSQL, and Mosquitto.

> **Security defaults changed.** There is no default database password, MQTT requires a login, and Caddy's bind address (`BIND_IP`) must be set explicitly. Existing installs: follow [Security defaults changed](#security-defaults-changed) before upgrading.

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

# 1. Create secrets: database password + MQTT login (there are no defaults)
cp sample.env .env && chmod 600 .env
export MQTT_USERNAME=trengine MQTT_PASSWORD=$(openssl rand -hex 16)
sed -i -e "s|^POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$(openssl rand -hex 24)|" \
       -e "s|^MQTT_USERNAME=.*|MQTT_USERNAME=$MQTT_USERNAME|" \
       -e "s|^MQTT_PASSWORD=.*|MQTT_PASSWORD=$MQTT_PASSWORD|" .env
# same login for trunk-recorder's MQTT plugins in config.json
sed -i "s|CHANGE_ME_MQTT_PASSWORD|$MQTT_PASSWORD|g" config.json
# same login for the broker (chown: mosquitto can't read a root-owned file)
docker run --rm -e MQTT_USERNAME -e MQTT_PASSWORD -v "$PWD/mosquitto:/mosquitto/config" eclipse-mosquitto:2 \
  sh -c 'mosquitto_passwd -c -b /mosquitto/config/passwd "$MQTT_USERNAME" "$MQTT_PASSWORD" && chown mosquitto:mosquitto /mosquitto/config/passwd'

# 2. Edit .env — set BIND_IP and SITE_ADDRESS
# BIND_IP: the address Caddy listens on, usually your server's LAN IP
# SITE_ADDRESS: http://YOUR_SERVER_IP (what your browser uses to reach it)

# 3. Edit config.json — set your SDR source, frequency, and system details
# See: https://trunkrecorder.com/docs/intro for trunk-recorder config reference

# 4. Add your talkgroup CSV
mkdir -p talkgroups
cp /path/to/your/talkgroups.csv talkgroups/talkgroups.csv

# 5. Start
docker compose up -d

# 6. Watch logs
docker compose logs -f tr-engine
```

Compose refuses to start until `POSTGRES_PASSWORD`, `MQTT_PASSWORD` and `BIND_IP` are set and `mosquitto/passwd` exists. That's deliberate — see [Security Defaults](#security-defaults).

**Dashboard:** http://YOUR_SERVER_IP (served by Caddy on port 80)  
**API:** http://YOUR_SERVER_IP/api/v1

Auth is disabled by default — no login required for trusted local installs. See [Securing for Public Access](#securing-for-public-access) before setting `BIND_IP=0.0.0.0` or otherwise exposing this externally.

## Configuration

### .env

Copy `sample.env` to `.env` and set your values. Minimum required (the Quick Start fills in the secrets for you):

```env
BIND_IP=192.168.1.100                # address Caddy listens on (LAN IP, 127.0.0.1, or 0.0.0.0)
SITE_ADDRESS=http://192.168.1.100    # your server's IP or hostname
POSTGRES_PASSWORD=...                # openssl rand -hex 24
MQTT_USERNAME=trengine
MQTT_PASSWORD=...                    # openssl rand -hex 16, must match mosquitto/passwd and config.json
```

Auth is in `open` mode by default when both `AUTH_TOKEN` and `ADMIN_PASSWORD` are unset. The dashboard is accessible without login. See [Securing for Public Access](#securing-for-public-access) to enable auth.

### Securing for Public Access

If you're exposing the stack to the internet, use full auth:

```env
ADMIN_PASSWORD= # dashboard login password
# Optional public read token returned by /api/v1/auth-init:
# AUTH_TOKEN=   # openssl rand -base64 32
```

When `ADMIN_PASSWORD` is set, tr-engine runs in full mode. The dashboard uses JWT login for writes and can use the optional `AUTH_TOKEN` as a public read token through `/api/v1/auth-init`. Caddy does not inject auth headers.

### config.json

Edit `config.json` to match your SDR hardware and radio system. Key fields:

- `sources` — your SDR device, center frequency, sample rate, gain
- `systems` — P25 control channel frequency, system type, talkgroup CSV path
- `plugins` — pre-configured for MQTT + DVCF, update `broker` if using external MQTT

The `broker` in the plugin config points to the internal `mosquitto` container — leave as-is unless you're using an external broker. Each MQTT plugin (`mqtt_status`, `mqtt_dvcf`, `mqtt_avcf`) needs the broker login in its `username`/`password` fields; the Quick Start's `sed` replaces the `CHANGE_ME_MQTT_PASSWORD` placeholders with your `MQTT_PASSWORD`.

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

The default stack runs imbe-asr on CPU so first-run works without NVIDIA drivers. For GPU acceleration, use the GPU override:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

## Upgrading

```bash
docker compose pull && docker compose up -d
```

If you're pulling a new version of this repo, read [Security defaults changed](#security-defaults-changed) first.

## Ports

| Service | Host port | Bind address | Notes |
|---|---|---|---|
| Caddy (dashboard + API) | `HTTP_PORT` (80) | `BIND_IP` (required) | tr-engine's API is served at `/api` through Caddy |
| Mosquitto MQTT | 1883 | `MQTT_BIND_IP` (default `127.0.0.1`) | Login required. Set `MQTT_BIND_IP` only for trunk-recorders on other hosts |
| PostgreSQL | — | never published | Use `docker compose exec postgres psql -U trengine trengine` |
| tr-engine, tr-dashboard, imbe-asr | — | never published | Reached through Caddy / the compose network |

## Security Defaults

- **No default database password.** `POSTGRES_PASSWORD` must be set in `.env`, or compose refuses to start.
- **PostgreSQL is never published** on the host.
- **MQTT always requires a login.** `mosquitto/mosquitto.conf` has `allow_anonymous false` and reads `mosquitto/passwd`; tr-engine uses `MQTT_USERNAME`/`MQTT_PASSWORD`, trunk-recorder uses the `username`/`password` fields in `config.json`.
- **No accidental all-interface binds.** Caddy binds to `BIND_IP`, which you must set. Mosquitto binds to `127.0.0.1` unless you set `MQTT_BIND_IP`. Prefer a LAN or VPN address over `0.0.0.0`.
- `.env` and `mosquitto/passwd` are gitignored. Don't commit them.

To change the MQTT password: update `MQTT_PASSWORD` in `.env` and the three `password` fields in `config.json`, rerun the `docker run ... mosquitto_passwd` command from the Quick Start, then `docker compose up -d && docker compose restart mosquitto trunk-recorder`.

### Security defaults changed

Earlier versions of this stack defaulted the database password to `trengine` (or `change-me` from `sample.env`), allowed anonymous MQTT, and bound Caddy to every interface. When you update an existing install:

1. **Rotate the database password inside PostgreSQL first.** Postgres only reads `POSTGRES_PASSWORD` when it creates `data/db`, so changing `.env` alone locks tr-engine out. Your current password is whatever `POSTGRES_PASSWORD` was in `.env` when the database was created: `change-me` if you kept the sample value, or `trengine` if the variable was unset. Rotate it while the old stack is still running:

   ```bash
   NEW_PG_PASSWORD=$(openssl rand -hex 24)
   docker compose exec postgres psql -U trengine -d trengine \
     -c "ALTER ROLE trengine PASSWORD '$NEW_PG_PASSWORD'"
   sed -i "s|^POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$NEW_PG_PASSWORD|" .env
   grep -q '^POSTGRES_PASSWORD=' .env || echo "POSTGRES_PASSWORD=$NEW_PG_PASSWORD" >> .env
   ```

   The command uses the container's local socket, so it doesn't need the old password. If you've already pulled the new `docker-compose.yml` and it refuses to parse, prefix the `docker compose exec` line with `POSTGRES_PASSWORD=placeholder MQTT_PASSWORD=placeholder BIND_IP=127.0.0.1`; `exec` only runs a command in the existing container. Keeping the old value in `.env` also works, since PostgreSQL isn't published, but rotate it as soon as you can.

2. **Create the MQTT login.** Set `MQTT_USERNAME`/`MQTT_PASSWORD` in `.env`, put the same values in the `username`/`password` fields of the MQTT plugins in `config.json`, and create `mosquitto/passwd` with the `docker run ... mosquitto_passwd` command from the Quick Start. `mosquitto/mosquitto.conf` from this repo now requires a login; if you edited your copy, set `allow_anonymous false` and `password_file /mosquitto/config/passwd`. Remote trunk-recorders need the same login in their plugin config.

3. **Choose `BIND_IP`.** It used to default to `0.0.0.0`. Set it to your server's LAN IP (or `0.0.0.0` if you really want every interface, with `ADMIN_PASSWORD` set). If remote trunk-recorders publish to this broker, you should already have `MQTT_BIND_IP` set; keep it.

4. `docker compose pull && docker compose up -d`, then check `docker compose logs tr-engine --tail 30` for `mqtt connected, subscribing` and no database errors.

## Setup Guide

Full step-by-step setup: [docs.luxprimatech.com/#/imbe-asr-setup](https://docs.luxprimatech.com/#/imbe-asr-setup)

## Roadmap

See the [Trunk Reporter Roadmap](https://github.com/orgs/trunk-reporter/projects/1) for the cross-repo project tracker with priorities and phases.

## Related

- [tr-docker](https://github.com/trunk-reporter/tr-docker) — trunk-recorder image source + CI
- [tr-engine](https://github.com/trunk-reporter/tr-engine) — backend source
- [imbe-asr](https://github.com/trunk-reporter/imbe-asr) — ASR model source
- [tr-plugin-dvcf](https://github.com/trunk-reporter/tr-plugin-dvcf) — DVCF plugin
- [symbolstream](https://github.com/trunk-reporter/symbolstream) — live streaming plugin

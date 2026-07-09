# wg-easy amd64 image

Unofficial `linux/amd64` image build and deployment template for [`wg-easy/wg-easy`](https://github.com/wg-easy/wg-easy).

This repository contains deployment templates and GitHub Actions automation only. It does not contain runtime `.env` files, WireGuard private keys, client profiles, Web UI credentials, or GitHub tokens.

## Security Model

Public internet exposure should be limited to WireGuard:

```text
51820/udp
```

The Web UI must stay off the public internet:

```text
51821/tcp
```

The provided Compose file binds the Web UI to `${LAN_BIND_IP}:51821`, so set `LAN_BIND_IP` to a private LAN address, VPN address, or SSH tunnel endpoint. Do not bind the Web UI to `0.0.0.0` on a public server.

Keep the deployment on Docker bridge networking unless the host firewall policy has been reviewed separately. `network_mode: host` increases the chance of exposing services unintentionally.

## Quick Start

Create the deployment directory:

```bash
mkdir -p /opt/wg-easy
cd /opt/wg-easy
```

Download the Compose template and environment example:

```bash
wget -O docker-compose.yml https://raw.githubusercontent.com/leolabtec/wg-easy-source-amd64/main/docker-compose.yml
wget -O .env.example https://raw.githubusercontent.com/leolabtec/wg-easy-source-amd64/main/.env.example
cp .env.example .env
```

Edit `.env`:

```bash
nano .env
```

Set `LAN_BIND_IP` to the server's private address:

```text
LAN_BIND_IP=192.168.1.10
```

Use the moving image tag:

```text
WG_EASY_IMAGE=ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
```

Or pin a specific release:

```text
WG_EASY_IMAGE=ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

Validate the rendered Compose configuration:

```bash
docker compose config
```

Expected port bindings:

```text
51820:51820/udp
192.168.1.10:51821:51821/tcp
```

Start the service:

```bash
docker compose up -d
```

## Firewall

Example UFW rules:

```bash
ufw allow 51820/udp
ufw allow from 192.168.1.0/24 to any port 51821 proto tcp
ufw deny 51821/tcp
```

For router or cloud firewall port forwarding, forward only:

```text
51820/udp -> server:51820/udp
```

Do not forward `51821/tcp` from the public internet.

## Web UI

Access the Web UI only from the allowed private network:

```text
http://<server-private-ip>:51821
```

Recommended Web UI settings:

- Use a strong admin password.
- Enable 2FA.
- Keep one-time links off public networks.
- Do not expose metrics publicly.
- Do not enable OAuth auto-registration without strict domain restrictions.

## Image Tags

Moving tag:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
```

Pinned release tag:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

Pinned tags use the upstream short commit SHA:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:source-<short-sha>-amd64
```

Current upstream source commit:

```text
78b283849895b35ad45c6f83fa01228c0f1a6774
```

## Updates

For deployments tracking `latest-amd64`:

```bash
cd /opt/wg-easy
docker compose pull
docker compose up -d
```

For pinned deployments, update `WG_EASY_IMAGE` in `.env` first, then run the same commands.

## GHCR

The image is published to GitHub Container Registry:

```text
ghcr.io/leolabtec/wg-easy-source-amd64
```

Repository visibility and GHCR package visibility are separate GitHub settings. If anonymous pulls return `unauthorized`, verify the container package visibility under GitHub Packages.

Release archives are also available for environments that cannot pull from GHCR:

```bash
wget https://github.com/leolabtec/wg-easy-source-amd64/releases/download/source-78b2838-amd64/wg-easy-source-78b2838-amd64.tar.gz
docker load < wg-easy-source-78b2838-amd64.tar.gz
docker compose up -d
```

## Automated Builds

Workflow:

```text
.github/workflows/build-upstream.yml
```

Schedule:

```text
Monday 03:23 UTC
```

The workflow:

1. Resolves the current upstream `wg-easy/wg-easy` commit.
2. Compares it with `.github/upstream-wg-easy.sha`.
3. Builds a `linux/amd64` image when the upstream commit changes.
4. Pushes `latest-amd64` and `source-<short-sha>-amd64` tags to GHCR.
5. Creates a GitHub Release with the compressed Docker image archive and sha256 file.
6. Updates `.github/upstream-wg-easy.sha` after a successful default-branch build.

The workflow uses the built-in `GITHUB_TOKEN` provided by GitHub Actions. No personal access token is committed to this repository.

## Data Location

Runtime data is stored in the Docker volume mounted at:

```text
/etc/wireguard
```

This data includes WireGuard configuration and private keys. Back it up only to trusted storage.

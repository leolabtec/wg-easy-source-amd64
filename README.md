# wg-easy source amd64 image

This repository holds a deployment template and automated amd64 build workflow for `wg-easy`.

Default image:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
```

The initial audited source commit was:

```text
78b283849895b35ad45c6f83fa01228c0f1a6774
```

## Automated builds

The GitHub Actions workflow checks `wg-easy/wg-easy` weekly. When the upstream commit changes, it builds a `linux/amd64` image, pushes these tags to GHCR, and creates a GitHub Release:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
ghcr.io/leolabtec/wg-easy-source-amd64:source-<upstream-short-sha>-amd64
```

Each release also contains a compressed Docker image tarball for manual `docker load` use.

For GHCR pushes, grant this repository write access to the `wg-easy-source-amd64` package. The workflow uses the built-in `GITHUB_TOKEN`.

GitHub package visibility is separate from repository visibility. To allow unauthenticated server pulls, open the package settings page and change the package visibility to public.

## Deploy

Copy the env example:

```bash
cp .env.example .env
```

Edit `.env` and set `LAN_BIND_IP` to your server's LAN IP, for example:

```text
LAN_BIND_IP=192.168.1.10
```

By default `.env` uses the moving `latest-amd64` tag. For a conservative pinned deployment, set `WG_EASY_IMAGE` to a release tag:

```text
WG_EASY_IMAGE=ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

Start:

```bash
docker compose up -d
```

Update to the latest automated build:

```bash
docker compose pull
docker compose up -d
```

If GHCR is still private, either run `docker login ghcr.io` on the server or download the release tarball and load it manually:

```bash
docker load < wg-easy-source-<short-sha>-amd64.tar.gz
```

## Firewall

Expose WireGuard to the public internet:

```bash
ufw allow 51820/udp
```

Allow Web UI only from your LAN:

```bash
ufw allow from 192.168.1.0/24 to any port 51821 proto tcp
ufw deny 51821/tcp
```

## Security notes

- Do not expose `51821/tcp` to the public internet.
- Do initial setup only from LAN, SSH tunnel, or a trusted network.
- Use a strong admin password and enable 2FA.
- Avoid one-time links on public deployments.
- Do not enable OAuth auto-registration unless you also enforce strict allowed domains.
- Do not expose metrics publicly.

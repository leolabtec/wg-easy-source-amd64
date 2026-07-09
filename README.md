# wg-easy source amd64 image

This repository holds a deployment template for a locally audited amd64 build of `wg-easy`.

Image:

```text
ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

Source commit:

```text
78b283849895b35ad45c6f83fa01228c0f1a6774
```

## Deploy

Copy the env example:

```bash
cp .env.example .env
```

Edit `.env` and set `LAN_BIND_IP` to your server's LAN IP, for example:

```text
LAN_BIND_IP=192.168.1.10
```

Start:

```bash
docker compose up -d
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

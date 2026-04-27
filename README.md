# devtop

My setup for a personal remote development box, accessible securely over Tailscale.

By running `docker compose up -d` this Docker Compose stack will:

- connect itself to my [Tailscale](https://tailscale.com/) network, allowing me to securely access it as if it were another machine on my local network, without having to do port forwarding.
- launch an [XFCE](https://xfce.org/) desktop environment based on [linuxserver/webtop](https://docs.linuxserver.io/images/docker-webtop/), allowing me to run GUI apps.
- launch [Selkies](https://github.com/selkies-project/selkies), a web-based remote desktop server that supports:
  - low-latency connection utilizing WebRTC
  - frame rate of up to 60fps
  - HiDPI (works with retina display)
  - clipboard sync
  - audio forwarding
- set up [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) so I can access the remote desktop by going to `https://<DEVBOX_NAME>.<your-tailnet>.ts.net`
  - Tailscale automatically sets up HTTPS certificate
  - the remote desktop is only accessible from within the tailnet
- launches a Docker daemon isolated from the host (Docker-in-Docker), letting me install, build, and run Docker containers inside the devbox without giving it access to the host machine

The stack is set up in a way that allows me to:

- easily install mise-en-place and then install OpenCode with it, so I can run coding agent sessions on the cloud
- configure memory limits via environment variables, so I can size the box according to the project’s needs

## Setup

### 1. Tailnet prerequisites

In the [Tailscale admin console](https://login.tailscale.com/admin), make sure these are enabled (both are on by default for new tailnets, but worth double-checking):

- **MagicDNS** — under DNS settings.
- **HTTPS certificates** — under DNS settings, "Enable HTTPS." This is what lets `tailscale serve` issue a real certificate for `<name>.<tailnet>.ts.net`.

### 2. Get an auth key

1. Visit [https://login.tailscale.com/admin/machines/new-linux](https://login.tailscale.com/admin/machines/new-linux).
2. Assign **Tags** (e.g. `tag:devtop`). [Assigning at least one tag](https://tailscale.com/docs/features/tags) prevents auth key expiry and also prevents the container from accessing the rest of your Tailnet unless explicitly allowed by an [access control policy](https://tailscale.com/docs/features/access-control).
3. Leave _Ephemeral_ and _Use as exit node_ unchecked.
4. Click **Generate install script**.
5. From the generated script, copy the value after `--auth-key=` — that's your `TS_AUTHKEY`.

### 3. Create `.env`

Create a `.env` file next to `docker-compose.yml`:

```bash
# Required
DEVBOX_NAME=devtop-demo
TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxx

# Optional
TZ=Asia/Bangkok
WEBTOP_MEM_LIMIT=2g
WEBTOP_MEMSWAP_LIMIT=4g
TS_MEM_LIMIT=512m
```

`DEVBOX_NAME` becomes the tailnet hostname, the browser tab title, and is what you'll use in the URL.

### 4. Bring it up

```bash
docker compose up -d
```

On first boot, the `tailscale` container authenticates with your auth key and registers itself with your tailnet, then `tailscale-serve-init` runs once to configure HTTPS termination on port 443. Both pieces of state persist in named volumes, so subsequent boots reuse the same identity and serve config without consuming the auth key again.

After a minute or so, open `https://<DEVBOX_NAME>.<your-tailnet>.ts.net` from any user device on your tailnet.

## Configuration reference

| Variable | Required | Default | Notes |
|---|---|---|---|
| `DEVBOX_NAME` | yes | — | Tailnet hostname and browser title |
| `TS_AUTHKEY` | yes | — | Only consumed on first boot; thereafter ignored |
| `TZ` | no | `UTC` | IANA timezone, e.g. `Asia/Bangkok` |
| `WEBTOP_MEM_LIMIT` | no | `2g` | RAM cap for webtop |
| `WEBTOP_MEMSWAP_LIMIT` | no | `4g` | RAM+swap cap (must be ≥ `WEBTOP_MEM_LIMIT`) |
| `TS_MEM_LIMIT` | no | `512m` | RAM cap for the Tailscale sidecar |

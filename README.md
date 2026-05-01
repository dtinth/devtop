# devtop

My setup for a personal remote development box, accessible securely over Tailscale, for running coding agents on VPS without giving it access to the whole system. Powered by [webtop](https://docs.linuxserver.io/images/docker-webtop/).

<img width="2128" height="1098" alt="image" src="https://github.com/user-attachments/assets/a2a915de-4e79-4fb2-8e79-45fe12ab6f97" />

By running `docker compose up -d` this Docker Compose stack will:

- connect itself to my [Tailscale](https://tailscale.com/) network, allowing me to securely access it as if it were another machine on my local network, so I can develop multiple projects on the same VPS without different projects competing for the same port
- launch an [XFCE](https://xfce.org/) desktop environment based on [linuxserver/webtop](https://docs.linuxserver.io/images/docker-webtop/), allowing me to run GUI apps.
- launch [Selkies](https://github.com/selkies-project/selkies), a web-based remote desktop server that supports:
  - low-latency connection utilizing WebRTC
  - frame rate of up to 60fps
  - HiDPI (works with retina display)
  - clipboard sync
  - audio forwarding
  - file transfers
- set up [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) so I can access the remote desktop by going to `https://<DEVBOX_NAME>.<your-tailnet>.ts.net`
  - Tailscale automatically sets up HTTPS certificate
  - the remote desktop is only accessible from within the tailnet
- launches a Docker daemon isolated from the host (Docker-in-Docker), letting me install, build, and run Docker containers inside the devbox without giving it access to the host machine

The stack is set up in a way that allows me to:

- easily [install mise-en-place](#install-mise-en-place) and then [install OpenCode](#install-opencode) with it, so I can run coding agent sessions on the cloud
- [configure memory limits](#configure-memory-limits) via environment variables, so I can size the box according to the project’s needs
- run web browsers (headed or headless), so I can ask the coding agent to use [agent-browser](https://agent-browser.dev/) for exploratory testing and [Playwright](https://playwright.dev/) for E2E testing

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

## Guides

### Install mise-en-place

Launch a terminal (Applications → Terminal Emulator) and run:

```bash
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
```

Reload the shell with `exec bash` then I can install tools I often use:

```bash
mise use -g gh ghq btop zellij opencode@1 node@24
```

### Install Tailscale CLI

Run once from the webtop terminal to put the `tailscale` binary in `~/.local/bin`:

```bash
docker run --rm -v "$HOME/.local/bin:/out" tailscale/tailscale:latest cp /usr/local/bin/tailscale /out/tailscale
```

The binary persists across restarts via the `config` volume. You can then use `tailscale serve`, `tailscale funnel`, `tailscale status`, etc. directly from the webtop terminal.

Read-only commands (e.g. `tailscale status`) work as-is. Commands that change configuration (e.g. `tailscale serve`) require root — use `sudo $(which tailscale)` rather than `sudo tailscale`, since `sudo` doesn't inherit the user `PATH` where the binary is installed. Moving the binary to `/usr/local/bin` would fix this, but that location isn't persistent across container recreations.

### Install OpenCode

Install with mise:

```bash
mise use -g opencode@1
```

Launch the [web server](https://opencode.ai/docs/web/) (keep this terminal open):

```bash
opencode serve
```

OpenCode listens on `localhost:4096` inside the devbox. To make it accessible to Tailscale users, run this from the webtop terminal (requires the [Tailscale CLI](#install-tailscale-cli)):

```bash
sudo $(which tailscale) serve --bg --https=4096 http://localhost:4096
```

You can now access your OpenCode instance at `https://<DEVBOX_NAME>.<your-tailnet>.ts.net:4096`.

To attach to the running instance from your local terminal, run:

```bash
opencode attach https://<DEVBOX_NAME>.<your-tailnet>.ts.net:4096 --dir /path/to/your/project
```

### Run Zellij Web

Install with mise:

```bash
mise use -g zellij
```

Create an auth token (displayed once — note it down):

```bash
zellij web --create-token
```

Launch Zellij once (you can exit or detach immediately — this is required before `zellij web` will stay running):

```bash
zellij
```

Then launch the [web server](https://zellij.dev/documentation/web-client.html) (keep this terminal open):

```bash
zellij web --port=18082
```

Zellij Web listens on `localhost:18082` inside the devbox. To make it accessible to Tailscale users, run this from the webtop terminal (requires the [Tailscale CLI](#install-tailscale-cli)):

```bash
sudo $(which tailscale) serve --bg --https=18082 http://localhost:18082
```

You can now access your Zellij Web instance at `https://<DEVBOX_NAME>.<your-tailnet>.ts.net:18082`.

You can also [attach to a session from a local terminal](https://zellij.dev/documentation/web-client.html#remote-terminal-attach):

```bash
zellij attach https://<DEVBOX_NAME>.<your-tailnet>.ts.net:18082/<session-name> --token <token>
```

### Configure memory limits

By default the webtop container is allowed up to **2 GB of RAM**. If that is not enough, Docker will swap — the combined RAM + swap usage can reach **4 GB** before the container is killed.

To change these limits, set the corresponding variables in your `.env` before (re)starting the stack:

```bash
WEBTOP_MEM_LIMIT=4g        # RAM limit
WEBTOP_MEMSWAP_LIMIT=8g    # RAM + swap limit (must be ≥ WEBTOP_MEM_LIMIT)
```

Then apply with:

```bash
docker compose up -d
```

To disable swap entirely, set both variables to the same value.

## Image setup FAQ

* **Why the `debian` variant instead of the default `alpine` image?** Alpine's musl libc breaks tools that ship glibc-linked binaries. In practice, Mise-installed runtimes and Playwright both fail on Alpine; Debian avoids those issues entirely.

* **Why `privileged: true`?** Required for Docker-in-Docker — the webtop container runs its own Docker daemon, which needs elevated capabilities to manage kernel namespaces and cgroups.

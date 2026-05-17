# Self-Hosting — Tailscale Sidecar + Dashboard

## Hosting on Laptop with Tailscale

Tailscale provides a personal private network, consisting of node machines each with its own tailscale network interface, a tailscale IP, and each with identity authentified in advance. 
Nodes could have nearly full access to each other's services, (https, SSH, etc.), via a secured Wireshark P2P session. 
Tailscale provides routing, (magic)DNS, authentication services, while keeping the actual data traffic node-to-node. 
Here's [how tailscale works](https://tailscale.com/blog/how-tailscale-works) for more.  

## Architecture

```
                         Tailnet (WireGuard mesh, MagicDNS, HTTPS certs)
                                          │
   ┌──────────────────────────────────────┼───────────────────────────────────┐
   │ Arch host (rootless Podman, one user) │                                   │
   │                                       │                                   │
   │  pod: silverbullet            pod: forgejo            pod: vaultwarden ... │
   │  ┌─────────────────┐          ┌─────────────────┐     ┌──────────────────┐│
   │  │ tailscale-      │  shared  │ tailscale-      │     │ tailscale-       ││
   │  │ silverbullet    │◀─netns──▶│ forgejo         │ ... │ vaultwarden      ││
   │  │  (tailscaled,   │  │       │  (tailscaled,   │     │  (tailscaled,    ││
   │  │   serve :443)   │  │       │   serve :443)   │     │   serve :443)    ││
   │  └─────────────────┘  │       └─────────────────┘     └──────────────────┘│
   │   silverbullet  ──────┘        forgejo                 vaultwarden         │
   │   :3000 (loopback)             :3000                   :80                 │
   │                                                                           │
   │  homepage  ── 127.0.0.1:3000 (the ONLY host port) ── ./homepage:/app/config│
   └───────────────────────────────────────────────────────────────────────────┘
```

Each app shares its sidecar's network namespace (`network_mode: "service:tailscale-<app>"`).
The sidecar terminates HTTPS on the Tailnet and reverse-proxies to the app on
`127.0.0.1:<app-port>` (same netns).

## Services

| Service | Image | App port (in netns) | Sidecar hostname | URL |
|---|---|---|---|---|
| Silverbullet | `zefhemel/silverbullet:latest` | 3000 | `silverbullet` | `https://silverbullet.<tailnet>.ts.net` |
| Forgejo | `codeberg.org/forgejo/forgejo:15` | 3000 | `forgejo` | `https://forgejo.<tailnet>.ts.net` |
| Anki Sync | `jeankhawand/anki-sync-server:25.07` | 8080 | `anki` | `https://anki.<tailnet>.ts.net` |
| Vaultwarden | `vaultwarden/server:latest` | 80 | `vaultwarden` | `https://vaultwarden.<tailnet>.ts.net` |
| Stirling PDF | `docker.io/stirlingtools/stirling-pdf:latest` | 8080 | `stirling` | `https://stirling.<tailnet>.ts.net` |
| Actual Budget | `ghcr.io/actualbudget/actual-server:latest` | 5006 | `budget` | `https://budget.<tailnet>.ts.net` |
| Syncthing | `docker.io/syncthing/syncthing:latest` | 8384 (GUI) | `syncthing` | `https://syncthing.<tailnet>.ts.net` |
| Homepage | `ghcr.io/gethomepage/homepage:latest` | 3000 → host | — | `http://127.0.0.1:3000` (local only) |

`<tailnet>` here is your Tailnet name, e.g. `ussuri-macaroni` → `ussuri-macaroni.ts.net`
(seen in `homepage/services.yaml` and at <https://login.tailscale.com/admin/dns>).

## Repository layout

```
selfhost/
├── docker-compose.yml          # the whole stack (tracked)
├── .env.example                # required env var names, no secrets (tracked)
├── .env                        # real Tailscale auth keys (GITIGNORED — never commit)
├── tailscale-serve/            # per-app serve configs (tracked)
│   ├── silverbullet.json  forgejo.json  anki.json  vaultwarden.json
│   ├── stirling.json  budget.json  syncthing.json
├── homepage/                   # Homepage dashboard config (tracked)
│   ├── services.yaml  settings.yaml  widgets.yaml  bookmarks.yaml ...
├── README.md                   # original (legacy host-serve) guide
├── README-v2.md                # this file
│
│   # ---- generated at runtime, NOT for reproducibility / DO NOT COMMIT ----
├── <app>-data/                 # app persistent data (silverbullet-data, forgejo-data,
│                               #   anki-data, vaultwarden-data, stirling-data,
│                               #   budget-data, syncthing-data)
└── tailscale-<app>-state/      # each sidecar's Tailnet node identity + keys
```

Only `docker-compose.yml`, `.env.example`, `tailscale-serve/*.json`, the `homepage/*`
config, and the READMEs are needed to reproduce the stack. Every `*-data/` and
`tailscale-*-state/` directory is regenerated on first boot and contains either
private user data or Tailscale node private keys — **keep them out of git.**

Ignore strategy: the root `.gitignore` keeps secrets out while tracking the
template:

```gitignore
.env
!.env.example
```

Every `*-data/` and `tailscale-*-state/` directory carries its own
`.gitignore` (`*` then `!.gitignore`) so the directory itself is tracked
(it must exist for the bind mount) while all its contents — user data and
Tailscale node private keys — are ignored. When you add a new service,
create the same `.gitignore` in its new data/state directories.

## Reproducing this deployment from scratch

### 0. Prerequisites

- Arch Linux (or any systemd distro) with **rootless Podman** + the
  Compose provider (`podman compose` shelling to `docker-compose`, as used here).
- A Tailscale account and a Tailnet.
- In the Tailscale admin console: **MagicDNS enabled** and **HTTPS Certificates
  enabled** (DNS page). Without HTTPS certs, the sidecars' `serve --https` fails.

### 1. Get the repo

```bash
git clone <this-repo> ~/selfhost
cd ~/selfhost
```

### 2. Create auth keys and `.env`

Generate **one auth key per sidecar** at
<https://login.tailscale.com/admin/settings/keys> (reusable or ephemeral both
work; pre-authorize / tag per your ACL policy). Then:

```bash
cp .env.example .env
$EDITOR .env        # fill every TS_AUTHKEY_*; set MAIN_HOST_TS_DN to the
                    # MagicDNS name homepage is reached as (or "*")
```

`.env` is gitignored — it holds secrets and stays on the host only.

### 3. Bring the stack up

```bash
podman compose up -d
podman compose ps          # all services Up; no restart loops
```

The `*-data/` and `tailscale-*-state/` directories are auto-created. Each sidecar
authenticates with its key, joins the Tailnet as `<hostname>`, provisions a
cert, and starts serving `https://<hostname>.<tailnet>.ts.net`.

### 4. Persist across reboots

Same mechanism as the original README:

```bash
systemctl --user enable --now podman-restart
loginctl enable-linger $USER
```

`restart: always` containers then come back on every boot without an active
login session.

### 5. Verify

```bash
podman ps                                            # all Up
podman logs tailscale-forgejo | tail                 # "Serve started" / cert ok
podman exec tailscale-forgejo wget -qO- -S http://127.0.0.1:3000 2>&1 | head
```

From any Tailnet client: open `https://forgejo.<tailnet>.ts.net` (no port, valid
cert). The Homepage launchpad is reachable **only on the host** at
`http://127.0.0.1:3000` by design — port-forward / SSH-tunnel it if you want it
from elsewhere, or give it its own sidecar.

## Key design decisions & gotchas

These are load-bearing — changing them reintroduces failures we already hit.

- **No `userns_mode: keep-id` anywhere.** Combining a uid0-spanning userns map
  with the implicit shared-namespace pod that `network_mode: service:` creates
  makes crun fail intermittently with
  `crun: mount 'devpts' to 'dev/pts': Invalid argument`. Symptom: a random
  different container fails each `up`, and "successful" boots still 502 because
  apps died. (Refs: podman discussion #24939, crun #1158.)
- **Apps run as root in-container instead.** Default rootless Podman maps
  container uid/gid `0` → the host user (`jcmaxwell:users`), so bind-mounted
  `*-data/` files are owned by you with **zero `chown`**. Forced where an image
  defaults to non-root: `user: "0:0"` (anki, budget, syncthing) and
  `USER_UID=0`/`USER_GID=0` (forgejo).
- **`TS_USERSPACE=true` on every sidecar.** Rootless Podman has no
  `/dev/net/tun`; Tailscale must run its userspace netstack.
- **Declarative serve.** Each sidecar gets `TS_SERVE_CONFIG=/config/serve.json`
  with `tailscale-serve/<app>.json` mounted read-only. `${TS_CERT_DOMAIN}` is
  substituted at runtime by Tailscale's containerboot, so the configs are
  Tailnet-agnostic and reproducible.
- **Homepage is the only container with a host port.** Everything else is
  Tailnet-only — that is the security boundary.

## Syncthing specifics

Syncthing has three independent network channels; only the GUI goes through serve:

- **GUI / API** — HTTP, `STGUIADDRESS=0.0.0.0:8384`. Proxied by serve →
  `https://syncthing.<tailnet>.ts.net`. It must be `0.0.0.0`, **not**
  `127.0.0.1`: Syncthing's anti-DNS-rebinding "Host check" only triggers on a
  loopback bind, so a localhost bind behind serve yields a **"Host check
  error"** page (the `Host:` header is the `.ts.net` name, not localhost). A
  non-loopback bind sidesteps it; serve still reaches the GUI via
  `127.0.0.1:8384` since `0.0.0.0` covers loopback. Set a GUI password —
  Syncthing has none by default and now also answers on the tailnet iface.
- **Sync protocol (BEP)** — raw TCP/QUIC `:22000`. `tailscale serve` is an HTTP
  proxy and **cannot** carry this. Peers connect directly to
  `syncthing.<tailnet>.ts.net:22000` over WireGuard; the sidecar's userspace
  netstack forwards inbound to `127.0.0.1:22000`. Requires the Tailnet **ACL to
  allow TCP/UDP 22000** between the syncing nodes. Falls back to Syncthing
  relays (encrypted, slower) if direct fails.
- **Discovery** — local broadcast (`UDP 21027`) does **not** cross a tailnet, so
  add each remote device with an explicit address `tcp://<peer-magicdns>:22000`
  rather than relying on `dynamic`.

Security: the Syncthing GUI has **no password by default** — anyone on your
Tailnet can open it. Set a GUI user/password in Syncthing settings, or scope
access with a Tailnet ACL.

## Adding a new service (recipe)

For an app `foo` listening on port `P`:

1. Add a `tailscale-foo` sidecar (copy an existing block; set `TS_HOSTNAME=foo`,
   `TS_AUTHKEY=${TS_AUTHKEY_FOO}`, state volume `./tailscale-foo-state`, serve
   mount `./tailscale-serve/foo.json`).
2. Add the `foo` app: `network_mode: "service:tailscale-foo"`,
   `depends_on: [tailscale-foo]`, `user: "0:0"` if the image is non-root by
   default, data volume under `./foo-data`.
3. Create `tailscale-serve/foo.json` proxying `http://127.0.0.1:P`.
4. Add `TS_AUTHKEY_FOO=` to `.env.example` and a real key to `.env`.
5. Add a tile to `homepage/services.yaml`.
6. `podman compose up -d`.

## Migrating existing data

The original README's Anki migration (section *"Migrating from an existing anki
sync server"*) still applies, with one simplification: because containers now
run as root → host user, you no longer need to discover the container UID/GID.
After copying data in, just make it owned by your host user:

```bash
sudo chown -R "$(id -u):$(id -g)" ~/selfhost/anki-data
```

Disable any conflicting host service (e.g. AUR `anki-sync-server`) as before,
then `podman compose up -d`.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `crun: mount 'devpts' to 'dev/pts': Invalid argument`, random container, intermittent | A userns map was reintroduced on a netns-sharing container. Remove all `userns_mode: keep-id`; run the app as root. |
| App reachable by MagicDNS ping but HTTPS returns **502** | The app isn't listening on the port its `tailscale-serve/<app>.json` proxies to. Check the app is Up (`podman ps`), and the port matches the Services table above. MagicDNS ping only proves the sidecar's `tailscaled` is alive, not the app. |
| `serve` errors / no cert | MagicDNS or HTTPS Certificates not enabled in the Tailscale admin DNS page. |
| Sidecar re-auth needed | Delete that sidecar's `tailscale-<app>-state/` and `up` again with a valid key. |
| Data owned by `100999` | Legacy from the old non-root mapping. `sudo chown -R "$(id -u):$(id -g)" ~/selfhost/<app>-data`. |

## Useful references

All entries from the original README still apply. Additions:

| Title | URL |
|---|---|
| Tailscale + Docker/Podman sidecar guide | <https://tailscale.com/blog/docker-tailscale-guide> |
| Tailscale serve config (JSON) | <https://tailscale.com/kb/1242/tailscale-serve> |
| Vaultwarden | <https://github.com/dani-garcia/vaultwarden> |
| Stirling PDF | <https://github.com/Stirling-Tools/Stirling-PDF> |
| Actual Budget | <https://actualbudget.org/docs/> |
| Syncthing | <https://docs.syncthing.net/> |
| Homepage dashboard | <https://gethomepage.dev/> |
| Podman userns/devpts (issue refs) | <https://github.com/containers/podman/discussions/24939> |

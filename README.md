# rhodes-hangar

Headless Android instances running Arknights, each driven by its own
[La Pluma](https://hub.docker.com/r/miaona/la-pluma) web console for MAA. One
command deploys an instance end to end: the Android image, the console, their
connection, and the latest game client.

An instance is a compose project (`rhodes-hangar-<name>`) with two containers:

| Container | What it is |
| --- | --- |
| `android` | redroid Android 14 with Google's AVD 34 ARM translation layer (the game ships ARM code only), limited to 8 CPUs and 4096 threads |
| `la-pluma` | The La Pluma console and MAA, connected to `android:5555` over the compose network |

Its data lives in `data/<name>/`: Android's `/data`, La Pluma's data, MAA's
settings and resources, and `instance.env` (ports, addresses, server, access
token, relay password). Android's adb stays inside the compose network unless
`--adb-bind` publishes it; `hangar` itself reaches Android through
`docker compose exec`.

## Requirements

Linux with Docker and binder in the kernel (`CONFIG_ANDROID_BINDER_IPC` or, as
on current Arch kernels, `CONFIG_ANDROID_BINDER_IPC_RUST`), `curl`, `jq`, and a
render node for the GPU (`/dev/dri/renderD129` by default). `adb` and `scrcpy`
are optional, for `hangar screen`.

## Deploy

```sh
./hangar deploy                       # instance "main", official server
./hangar deploy --server bilibili     # Bilibili server instead
```

The first deploy:

1. Builds `local/rhodes-hangar-redroid:14-avd34-c2` from `image/`, downloading the
   translation layer at a pinned commit and checksum.
2. Creates `data/main/` and `instance.env` with a random access token.
3. Writes MAA's connection profile and starts both containers.
4. Waits for Android to boot, then downloads the latest client (official from
   `ak.hypergryph.com`, Bilibili from the Bilibili game center, about 1.9 GB
   into `cache/`) and installs it, streaming the package into the container.
5. Asks La Pluma to test its connection to Android.

Then open `http://127.0.0.1:13055` with the token from `data/main/instance.env`,
open the game once from the console's live preview, and log in: the game
downloads about 12 GB of resources on its first launch.

Running `deploy` again is safe: it keeps the data and tokens, applies the
options given (`--server`, `--web-port`, `--bind`, `--public-ip`,
`--adb-bind`, `--adb-port`, `--gpu-node`), and updates the game when a newer
client is out. To move an existing console in with its token, set
`LA_PLUMA_TOKEN` in the environment for the first deploy.

## adb

adb is not published by default: only La Pluma reaches `android:5555`.
`--adb-bind 127.0.0.1` publishes it on this host at `--adb-port` (default
15556) for `hangar screen`, scrcpy or adb; `--adb-bind none` withdraws it.
Another address publishes it to the network, where anyone who reaches it gets
a root shell in a privileged container, with no authentication.

## Remote access

The web console listens on `127.0.0.1`. Another machine needs two paths to it:

| Path | What it carries | How it opens |
| --- | --- | --- |
| TCP `<web-port>` | The console, its API and the preview's signaling | `--bind 0.0.0.0` or `--bind` one IPv4 address of this host |
| UDP `<web-port>` | The live preview's video and touch input, through La Pluma's own TURN relay | `--public-ip ADDR`: the IPv4 address or domain name the other machine reaches this host at |

The relay is needed because the preview is WebRTC: without a public IP La Pluma
offers browsers only the container's address (`172.x`), which no other machine
reaches, and the console loads while its preview stays blank. With `--bind` set
to one address, `--public-ip` defaults to it, and `--bind 127.0.0.1` clears it:

```sh
./hangar deploy --bind 192.168.1.20                     # LAN: console and preview
./hangar deploy --bind 0.0.0.0 --public-ip 192.168.1.20 # every interface, LAN address for the relay
./hangar deploy --bind 127.0.0.1                        # back to this machine only
./hangar deploy --bind 127.0.0.1 --public-ip 203.0.113.7 # console behind a local TLS proxy, relay public
```

Each instance's relay uses its own web port number, so instances do not
collide, and its password is random per instance; browsers receive it only after
logging in with the token. MAA and every other console feature work the same
from any machine: La Pluma reaches Android over the compose network.

### A domain for a changing address (DDNS)

La Pluma reads the relay address once at start and takes only an IP address (a
domain there is silently ignored), so hangar resolves `--public-ip DOMAIN`
itself: at `deploy`, and again at each `./hangar refresh-ip`, which restarts
La Pluma (only that container) when the address moved. It resolves over DNS
over HTTPS (AliDNS, then Cloudflare): a proxy such as Mihomo in fake-ip mode
answers the system resolver with `198.18.x` addresses. A refresh that finds MAA
running leaves La Pluma alone and the setting unchanged, so the next one tries
again.

```sh
./hangar deploy --bind 10.100.200.1 --public-ip office.example.org
./hangar refresh-ip --timer on    # systemd user timer: refresh-ip every 5 minutes
./hangar refresh-ip --timer off
```

The timer runs while the user is logged in; `loginctl enable-linger` keeps it
running without a session. Between a change and the next check (up to 5
minutes), browsers elsewhere get the old address and the preview stays blank;
the console itself keeps working.

Over the internet:

- **Prefer a private network** such as Tailscale or WireGuard: `--bind` and
  `--public-ip` take this host's address there, and nothing is exposed publicly.
- **Public exposure** needs TCP and UDP `<web-port>` forwarded to this host,
  `--public-ip` set to the public address or a DDNS name for it (see above), and
  TLS in front of the console: it speaks plain HTTP,
  so the token would cross the internet in clear text. La Pluma's own README
  covers both ways: its built-in HTTPS (`LA_PLUMA_HTTPS_CERT_PATH` and
  `LA_PLUMA_HTTPS_KEY_PATH`), or a reverse proxy forwarding `/`, `/api`,
  `/health` and `/webrtc-signaling` with the WebSocket upgrade. Neither is
  tested here. The relay's UDP goes straight to this host. Browsers on this
  host's own LAN then reach the relay at the public address only if the router
  supports hairpin NAT.
- The console is a single token away from controlling the game account; the
  token is the only access control.

## Commands

| Command | Does |
| --- | --- |
| `./hangar deploy [-i NAME]` | Create or update an instance and start it |
| `./hangar install-game [-i NAME]` | Install or update the game only |
| `./hangar up` / `down [-i NAME]` | Start or stop an instance |
| `./hangar status [-i NAME]` | Containers, game version, La Pluma's connection check |
| `./hangar screen [-i NAME]` | Mirror the instance with scrcpy (needs `--adb-bind`) |
| `./hangar refresh-ip [-i NAME] [--timer on\|off]` | Resolve the `--public-ip` domain again and restart La Pluma if it moved |

`-i NAME` picks the instance (default `main`). A second instance needs its own
web port (and adb port, if published): `./hangar deploy -i alt --web-port 13066`.

Plain `docker compose` works in this directory too: `.env` links to
`data/main/instance.env` (made by the first `deploy` of `main`), so
`docker compose down`, `ps` or `logs` act on `rhodes-hangar-main`; its
`COMPOSE_FILE` adds `compose.adb.yaml` when adb is published. For another
instance pass its file: `docker compose --env-file data/alt/instance.env down`.

## Notes

- `hangar screen` runs scrcpy without audio. Plain `scrcpy` captures audio by
  default; this Android has no opus encoder, the scrcpy server crashes while
  it holds a video encoder, and on 2026-11-19 that left the media services and
  the display stack blocked and a crash_dump storm running until the container
  was killed. The CPU and thread limits keep such a storm inside the container.
- The image removes every boot-time write to host-wide sysctls (`kernel`,
  `vm`, `abi`) and tracing, which a privileged container would otherwise apply
  to the host (redroid's stock init clears `kernel.modprobe`, which broke
  module autoloading and the Clash TUN device).
- The containers do not restart on their own; `./hangar up` starts them.

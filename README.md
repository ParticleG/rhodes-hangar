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
| UDP 3478 | The live preview's video and touch input, through the shared TURN relay | `--public-ip ADDR`: the IPv4 address or domain name the other machine reaches this host at |

The relay is needed because the preview is WebRTC: without a public IP La Pluma
offers browsers only its own relay at the container's address (`172.x`), which
no other machine reaches, and the console loads while its preview stays blank.
With `--bind` set to one address, `--public-ip` defaults to it, and
`--bind 127.0.0.1` clears it:

```sh
./hangar deploy --bind 192.168.1.20                     # LAN: console and preview
./hangar deploy --bind 0.0.0.0 --public-ip 192.168.1.20 # every interface, LAN address for the relay
./hangar deploy --bind 127.0.0.1                        # back to this machine only
./hangar deploy --bind 127.0.0.1 --public-ip 203.0.113.7 # console behind a local TLS proxy, relay public
```

### The shared relay

Every instance with `--public-ip` tells browsers of one relay: coturn on UDP
3478 of this host (`relay.compose.yaml`, project `rhodes-hangar-relay`, host
network), so the router forwards one UDP port however many instances run.
`deploy` and `up` write `data/relay/turnserver.conf` (mode 600; coturn runs as
the invoking user) with a user `hangar-<instance>` and its random password for
each such instance, allow relaying to that instance's La Pluma container
address only (not its network's gateway, which is this host, nor Android; the
allow list is shared by all users), and start, restart or stop the relay to
match. Wrong credentials get 401; any other peer gets 403. A container that
Docker gives a new address, after a manual restart say, is picked up by the
next `./hangar up`. La Pluma's own relay still runs inside each container,
unpublished.

Why not one port per instance behind a UDP proxy (Caddy's layer4 plugin, say):
a TURN client's first packets (a STUN binding request, an unauthenticated
allocate) carry nothing that names the instance, and La Pluma advertises only
its own relay, so one relay that knows every instance is the way to share a
port. MAA and every other console feature work the same from any machine: La
Pluma reaches Android over the compose network.

### A domain for a changing address (DDNS)

La Pluma reads the relay address once at start and takes only an IP address (a
domain there is silently ignored), so hangar resolves `--public-ip DOMAIN`
itself: at `deploy`, and again at each `./hangar refresh-ip`, which restarts
La Pluma (only that container) when the address moved. It resolves over DNS
over HTTPS (AliDNS, then Cloudflare): a proxy such as Mihomo in fake-ip mode
answers the system resolver with `198.18.x` addresses. A refresh that finds MAA
running leaves La Pluma alone and the setting unchanged, so the next one tries
again.

`deploy` with a domain installs a systemd user timer,
`rhodes-hangar-refresh-ip-<instance>`, that runs `refresh-ip` every 5 minutes;
a later `deploy` with an address or `--public-ip none` removes it.

```sh
./hangar deploy --bind 10.100.200.1 --public-ip office.example.org
journalctl --user -u rhodes-hangar-refresh-ip-main.service   # what the checks did
```

The timer runs while the user is logged in; `loginctl enable-linger` keeps it
running without a session. Between a change and the next check (up to 5
minutes), browsers elsewhere get the old address and the preview stays blank;
the console itself keeps working.

Over the internet:

- **Prefer a private network** such as Tailscale or WireGuard: `--bind` and
  `--public-ip` take this host's address there, and nothing is exposed publicly.
- **Public exposure** needs the console behind TLS (it speaks plain HTTP, so the
  token would cross the internet in clear text), UDP 3478 forwarded to this
  host, and `--public-ip` set to the public address or a DDNS name for it.
  Browsers on this host's own LAN reach the relay at the public address only if
  the router also NATs that traffic back in (hairpin NAT).
- **One subdomain per instance**: La Pluma serves `/api` and `/webrtc-signaling`
  at the origin root, ignores its base path for both, and keeps its token in the
  browser's storage for the origin, so instances cannot share a host by path. A
  wildcard DNS name and a reverse proxy routing by host work; with Caddy (it
  proxies the signaling WebSocket by itself; a wildcard certificate needs a DNS
  challenge):

  ```caddyfile
  *.hangar.example.org {
  	@main host main.hangar.example.org
  	handle @main {
  		reverse_proxy 192.168.1.20:13055
  	}
  	handle {
  		respond "No such instance" 404
  	}
  }
  ```
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
| `./hangar refresh-ip [-i NAME]` | Resolve the `--public-ip` domain again and restart La Pluma if it moved (the timer runs it) |

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

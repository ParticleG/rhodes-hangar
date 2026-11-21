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
settings and resources, and `instance.env` (ports, server, access token).

## Requirements

Linux with Docker and binder in the kernel (`CONFIG_ANDROID_BINDER_IPC` or, as
on current Arch kernels, `CONFIG_ANDROID_BINDER_IPC_RUST`), `adb`, `curl`,
`jq`, and a render node for the GPU (`/dev/dri/renderD129` by default).
`scrcpy` is optional, for `hangar screen`.

## Deploy

```sh
./hangar deploy                       # instance "main", official server
./hangar deploy --server bilibili     # Bilibili server instead
```

The first deploy:

1. Builds `local/rhodes-hangar-redroid:14-avd34` from `image/`, downloading the
   translation layer at a pinned commit and checksum.
2. Creates `data/main/` and `instance.env` with a random access token.
3. Writes MAA's connection profile and starts both containers.
4. Waits for Android to boot, then downloads the latest client (official from
   `ak.hypergryph.com`, Bilibili from the Bilibili game center, about 1.9 GB
   into `cache/`) and installs it.
5. Asks La Pluma to test its connection to Android.

Then open `http://127.0.0.1:13055` with the token from `data/main/instance.env`,
open the game once from the console or `./hangar screen`, and log in: the game
downloads about 12 GB of resources on its first launch.

Running `deploy` again is safe: it keeps the data and token, applies the
options given (`--server`, `--web-port`, `--adb-port`, `--bind`,
`--gpu-node`), and updates the game when a newer client is out. To move an
existing console in with its token, set `LA_PLUMA_TOKEN` in the environment for
the first deploy.

The web console listens on `127.0.0.1`. `--bind 0.0.0.0` opens it on every
interface (or `--bind` one IPv4 address of this machine); its token is then the
only protection, so keep it on a trusted network. adb always stays on
`127.0.0.1`: it is a root shell in a privileged container.

## Commands

| Command | Does |
| --- | --- |
| `./hangar deploy [-i NAME]` | Create or update an instance and start it |
| `./hangar install-game [-i NAME]` | Install or update the game only |
| `./hangar up` / `down [-i NAME]` | Start or stop an instance |
| `./hangar status [-i NAME]` | Containers, game version, La Pluma's connection check |
| `./hangar screen [-i NAME]` | Mirror the instance with scrcpy |

`-i NAME` picks the instance (default `main`). A second instance needs its own
ports: `./hangar deploy -i alt --web-port 13066 --adb-port 15566`.

Plain `docker compose` works in this directory too: `.env` links to
`data/main/instance.env` (made by the first `deploy` of `main`), so
`docker compose down`, `ps` or `logs` act on `rhodes-hangar-main`. For another
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

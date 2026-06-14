# RDOC WebRTC Streaming Server

Low-latency SRT → WebRTC relay for the Raumdock crew. Streamers push an **SRT**
feed into a per-streamer [MediaMTX](https://github.com/bluenviron/mediamtx)
instance; viewers watch in the browser over **WebRTC (WHEP)** behind a
[Caddy](https://caddyserver.com/) reverse proxy with automatic TLS via
Cloudflare DNS.

Repo: `github.com/cccdemon/CC-webrtc-server`
Deployed on: **Proxmox LXC 103 (`streamer`)** at `/opt/webrtc`, public IP
`85.215.253.135`.

---

## Architecture

```
 Streamer (OBS)                 LXC 103  "streamer"                     Viewer
 ┌───────────┐    SRT/mpegts   ┌──────────────────────────┐   WebRTC   ┌────────┐
 │  encoder  │ ───────────────▶│  mediamtx-<name>  :889x  │◀──────────▶│ browser│
 └───────────┘   :889x udp     │   (one per streamer)     │   WHEP     │  / OBS │
                               └──────────┬───────────────┘            └────────┘
                                          │ http :8889
                                   ┌──────▼───────┐  TLS (Cloudflare DNS-01)
                                   │  Caddy :443  │  <name>.streaming.raumdock.org
                                   └──────────────┘
```

Each streamer gets a **dedicated MediaMTX container** so ports, paths and load
stay isolated. Caddy terminates TLS and reverse-proxies
`https://<name>.streaming.raumdock.org` → that streamer's MediaMTX WebRTC port
(`:8889` inside the container network).

## Services / streamers

| Streamer        | Container                  | SRT (udp) | API (tcp) | WebRTC ICE (udp) | Public host                             |
|-----------------|----------------------------|-----------|-----------|------------------|-----------------------------------------|
| jazzz           | `mediamtx-jazzz`           | 8891      | 9991      | 8181             | jazzz.streaming.raumdock.org            |
| headwig         | `mediamtx-headwig`         | 8892      | 9992      | 8182             | headwig.streaming.raumdock.org          |
| jericho         | `mediamtx-jericho`         | 8893      | 9993      | 8183             | jericho.streaming.raumdock.org          |
| polder          | `mediamtx-polder`          | 8894      | 9994      | 8184             | polder.streaming.raumdock.org \*        |
| headlesschris   | `mediamtx-headlesschris`   | 8895      | 9995      | 8185             | headlesschris.streaming.raumdock.org    |

\* `polder` exists in `docker-compose.yml` but currently has **no Caddyfile
entry** (no public TLS host). Add one to expose it.

All MediaMTX instances share WebRTC media port `:8889` *inside* their own
container; only SRT/API/ICE ports are published to the host. `RTSP` is enabled
(`:8554`), `RTMP`/`HLS` are off.

## Components

- **`docker-compose.yml`** — one `bluenviron/mediamtx:latest-ffmpeg` service per
  streamer + the Caddy service. MediaMTX is configured purely via `MTX_*` env
  vars.
- **`caddy/Dockerfile`** — builds Caddy with the
  [`caddy-dns/cloudflare`](https://github.com/caddy-dns/cloudflare) plugin via
  `xcaddy` (needed for DNS-01 TLS).
- **`caddy/Caddyfile`** — global ACME config + one reverse-proxy site block per
  streamer.

## Configuration

Caddy needs Cloudflare credentials, set in `docker-compose.yml` under the
`caddy` service:

```yaml
environment:
  - CF_API_KEY=<cloudflare API token with DNS edit>
  - CF_API_EMAIL=<account email>
```

> ⚠️ The deployed compose contains a **live Cloudflare API token**. Treat it as a
> secret — rotate it if this repo is ever public, and prefer a `.env` file
> (git-ignored) over committing it.

## Run

```bash
# on LXC 103
cd /opt/webrtc
docker compose up -d --build        # build Caddy plugin + start all streams
docker compose ps
docker compose logs -f mediamtx-jazzz
```

Add or restart a single streamer without touching the rest:

```bash
docker compose up -d mediamtx-headlesschris
```

## Streamer (publish) setup — OBS

OBS → **Settings → Stream → Service: SRT** (or custom FFmpeg output):

```
srt://85.215.253.135:889x?streamid=publish:<path>
```

- `889x` = the SRT port for your streamer row above.
- `<path>` = stream path, e.g. `jazzz_game`.

## Viewer (playback)

Open `https://<name>.streaming.raumdock.org/<path>` in a browser — MediaMTX
serves its built-in WebRTC/WHEP player.

---

## ⚠️ Known issue: no audio in OBS / WebRTC playback

**Symptom:** video plays, but viewers (incl. OBS browser source) hear nothing
when a colleague streams.

**Root cause:** MediaMTX logs on the host show:

```
INF [path jazzz_game] stream is available and online, 2 tracks (H264, MPEG-4 Audio)
WAR [WebRTC] [session …] skipping track 2 (MPEG-4 Audio)
INF [WebRTC] is reading from path 'jazzz_game', 1 track (H264)
```

The incoming SRT/MPEG-TS stream carries **AAC ("MPEG-4 Audio")**. WebRTC does
**not** support AAC, and MediaMTX does **not transcode** — so it *drops the
audio track* and only forwards H264 video. WebRTC audio must be **Opus** (or
G711/G722).

**Fixes (pick one):**

1. **Send Opus from the source (best).** Configure the encoder to put Opus into
   the MPEG-TS/SRT stream. In OBS use a *Custom Output (FFmpeg)*:
   - Container/format: `mpegts`
   - Audio encoder: `libopus`
   - (Video stays H264.)
   MediaMTX forwards Opus over WebRTC untouched → audio works.

2. **Transcode on the server.** Add a MediaMTX `runOnReady` (or a sidecar
   `ffmpeg`) that republishes the path with `-c:a libopus`, e.g.
   `ffmpeg -i srt://… -c:v copy -c:a libopus -f rtsp rtsp://localhost:8554/<path>_opus`,
   and point viewers at the `_opus` path. Costs CPU; needed only if sources
   can't emit Opus.

3. **Use a WebRTC-native publish (WHIP)** instead of SRT — browsers/OBS WHIP
   already negotiate Opus.

> Note: the MediaMTX HTTP API (`:999x`) currently returns
> `{"status":"error","error":"authentication error"}`, so codecs are confirmed
> from container logs rather than the API.

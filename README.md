# Plex Media Server

> Version: `1.43.5.11079-b3b13e506-amd64` | Image: `plexinc/pms-docker:1.43.5.11079-b3b13e506-amd64` | Upstream: https://github.com/plexinc/pms-docker

Central media streaming server for the homelab, running directly on the QNAP NAS host alongside primary media storage and an NVIDIA GPU for hardware-accelerated transcoding.

## Overview

| Property | Value |
| :--- | :--- |
| **Domain** | `https://plex.intranet.xbhl.online` |
| **Node** | `nas.srv.xbhl.intranet` (QNAP NAS bare-metal host) |
| **Ingress** | Exclusive Traefik TLS reverse proxy (`plex-router` + `plex-force-transcode` + `plex-force-auto-subtitles`); no host ports exposed |
| **Authentication** | Native Plex.tv account & token auth (Authelia omitted for client compatibility) |
| **Ecosystem** | Seerr & `*arr` library consumer, Homepage (`Media` tile & widget), WUD |

## Architecture & Key Decisions

```mermaid
flowchart LR
    Clients["Web, Mobile & Apple TV Clients"] -->|"HTTPS :443"| Traefik["Traefik Reverse Proxy"]
    Ecosystem["Seerr / Radarr / Sonarr & Homepage"] -->|"HTTPS :443"| Traefik
    Traefik -->|"modify only when present: directPlay/directStream=0, subtitles=auto"| Plex["Plex Media Server (nas.srv.xbhl.intranet + NVIDIA GPU)"]
```

- **QNAP NAS Co-Location & GPU Transcoding**: Deployed directly on `nas.srv.xbhl.intranet` rather than a Proxmox VM so Plex reads local QNAP media shares without NFS overhead, reserves an NVIDIA GPU for NVENC/NVDEC hardware acceleration, and writes ephemeral transcode segments to world-writable RAM-backed `tmpfs`.
- **Single Ingress with Forced Transcoding via Traefik**: The Plex for Apple TV app ignores its own Direct Play/Direct Stream settings and always sends `directPlay=1`, which breaks many files, while client-side "burn subtitles always" forces CPU-heavy burn-in and stuttering. Host `:32400`, Companion, DLNA, and GDM UDP discovery ports are omitted, so every client goes through Traefik, where `query-modification` middlewares (`type=modify`, rewriting only parameters the client actually sends) emulate "always transcode": `plex-force-transcode` sets `directPlay`/`directStream` to `0`, and `plex-force-auto-subtitles` rewrites `subtitles` (Apple TV sends `subtitles=none` because it expects to render subtitles itself during Direct Play) to `auto`, letting PMS copy the selected text track (SRT) into the MKV-segmented HLS stream that the Apple TV declares in `X-Plex-Client-Profile-Extra` (`protocol=hls&container=mkv&subtitleCodec=...,srt,...`), without burn-in (`segmented` and `sidecar` return `decision=unavailable` because that profile only declares in-container subtitles). The stricter `delete`+`add` chain on decision/start (!11–!13), which injected parameters into requests that never carried them, broke Apple TV playback ("Something went wrong playing your media").
- **Curated Shares vs Automated Request Libraries**: Separates curated QNAP media shares (`/multimedia/*`) from the shared `/torrent` hierarchy where Seerr, Radarr, and Sonarr provision automated `/torrent/requests/*` libraries.
- **Native Plex.tv Authentication**: Authenticates web, TV, and mobile apps directly via Plex.tv accounts and tokens, intentionally omitting Authelia forward-auth on `plex-router` so native Plex clients connect unimpeded.

### External Credentials

- `PLEX_CLAIM`: Manual Plex.tv server claim token (`https://www.plex.tv/claim/`) stored in GitLab CI/CD variables for first-run server registration when `PlexOnlineToken` is absent.

## Upgrades & Maintenance

- **Upstream & Wrapper Sources**: Release notes are published in the [Plex Media Server forum thread](https://forums.plex.tv/t/plex-media-server/30447/last) (`wud.link.template`), while container packaging and `s6` init scripts live at <https://github.com/plexinc/pms-docker>.
- **Version & Flavor Constraints**: Pin immutable `<version>-<hash>-amd64` tags (`wud.tag.include` / `wud.tag.transform` mapping 4-part builds to 3-part semver) rather than self-updating `public`/`beta` tags; `VERSION=docker` is an inert LinuxServer.io leftover ignored by `plexinc/pms-docker`.
- **Intentional Choices (Do Not Propose)**: Keep the Traefik middlewares as `type=modify` (do not switch to `delete`+`add`, which breaks Apple TV playback), do not re-expose host `:32400`/DLNA/GDM ports, omit Authelia on `plex-router`, preserve `mode=1777` RAM `tmpfs`, and rely on the image's built-in `/healthcheck.sh` without a Compose `healthcheck` block.
- **Upgrade & CI Gotchas**: `.gitlab-ci.yml` overrides `test:plex:docker-compose` as a no-op because CI DinD runners lack the reserved NVIDIA GPU and QNAP host media paths, uses singular `src/.env` Compose interpolation (`container-healthcheck-delay: 300`), and only consumes `PLEX_CLAIM` during `40-plex-first-run` if `PlexOnlineToken` is missing.

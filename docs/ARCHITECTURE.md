# Architecture

```mermaid
flowchart LR
    subgraph Phone["📱 Your iPhone"]
        IM[iMessage app<br/>thread: +1 628 267-9185]
    end

    subgraph Photon["☁️ Photon (managed, free tier)"]
        LINE[Shared iMessage line<br/>+1 628 267-9185]
        SPEC[Spectrum gRPC stream<br/>project: fpl-manager]
    end

    subgraph DO["🖥️ DigitalOcean droplet 64.23.135.50 — user: fpl"]
        subgraph GW["hermes-gateway (systemd user service, linger on)"]
            HERMES[Hermes Agent v0.19.0<br/>persona: fpl-work/AGENTS.md]
            SIDE[Photon sidecar<br/>node index.mjs · 127.0.0.1:8789]
            CRON[Cron scheduler<br/>ticks every 60s]
        end
        FPLW[fpl wrapper<br/>proxychains → WARP]
        FPLC[fpl-cli dist/index.js<br/>team · transfers · captain · chips]
        CFG[(~/.config/fpl-cli<br/>config.json · tokens.json)]
    end

    subgraph Ollama["☁️ Ollama Cloud"]
        MODEL[deepseek-v4-flash<br/>chat + cron inference]
    end

    subgraph FPLAPI["☁️ Premier League"]
        AUTH[account.premierleague.com<br/>Ping auth · login/refresh]
        API[fantasy.premierleague.com<br/>bootstrap · squad · transfers]
    end

    IM <-->|iMessage| LINE
    LINE <--> SPEC
    SPEC <-->|"outbound-only gRPC<br/>(no public ports)"| SIDE
    SIDE <-->|loopback NDJSON| HERMES
    CRON -->|4 jobs · deliver photon:+1972...| HERMES
    HERMES <-->|OpenAI-compatible /v1| MODEL
    HERMES -->|terminal tool| FPLW
    FPLW --> FPLC
    FPLC --- CFG
    FPLC -->|auth + token refresh| AUTH
    FPLC -->|reads + mutations<br/>dry-run → plan_id → confirm| API
```

## Cron jobs (CDT)

| Job | Schedule | What it does |
|---|---|---|
| `gw-deadline-check` | daily 18:00 | Countdown + flagged players + ONE recommendation if deadline < 48h |
| `price-watch` | daily 07:30 | Price movers on your squad/targets (silent if none) |
| `injury-scan` | daily 08:00 | New squad injuries + top replacement (silent if clean) |
| `gw-review` | Tue 09:00 | Post-GW points/rank/lesson summary |

## Trust & safety boundaries

- **No inbound ports** — droplet firewall allows SSH only; Photon is a persistent *outbound* gRPC stream.
- **Pairing gate** — only your number (`+19728226226`) can talk to the agent.
- **Mutation policy** — every transfer/captain/chip requires `fpl` dry-run → `plan_id` → your explicit iMessage "yes" before `--confirm`. Cron jobs are read-only.
- **WARP egress** — FPL's auth provider 429-blocks the droplet's DO IP, so all `fpl` traffic exits via Cloudflare WARP.

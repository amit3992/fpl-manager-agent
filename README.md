# fpl-manager

Self-hosted **FPL assistant manager** agent, reachable via iMessage.

## Architecture

```
you (iMessage)
   │  text
   ▼
Photon (managed iMessage relay — no Mac required)
   │  gRPC stream (persistent, no public URL/webhook)
   ▼
Hermes Agent gateway (systemd service on DO droplet 64.23.135.50)
   │  model: deepseek-v4-flash via Ollama Cloud (provider `ollama-cloud`)
   │  shell tool
   ▼
fpl-cli (installed on droplet, authenticated to fantasy.premierleague.com)
```

- **Framework:** Hermes (Nous Research). Chosen over OpenClaw because OpenClaw's
  iMessage channel requires a 24/7 Mac relay; Hermes uses Photon, which is fully
  managed and has a free tier (5,000 msgs/server/day).
- **Chat + scheduled jobs:** Hermes gateway runs native cron jobs and delivers
  results back to the iMessage line.
- **Team management:** the agent drives `fpl-cli` (~/dev/ai/fpl-cli) for reads
  and for executing transfers/captain/chips.

## Files

| File | Purpose |
|---|---|
| `SETUP.md` | Step-by-step droplet provisioning |
| `agent/SYSTEM_PROMPT.md` | Persona + fpl-cli operating rules for the agent |
| `cron/jobs.md` | Scheduled jobs to create (deadline/price/injury alerts) |
| `docs/ARCHITECTURE.md` | Mermaid diagram + trust boundaries |
| `.gitignore` | Keeps secrets out of git |

## Safety model

`fpl-cli` enforces a dry-run → `plan_id` → `--confirm` flow for every mutation.
Default policy: the agent **proposes** transfers and asks for an explicit
iMessage confirmation before running anything with `--confirm`.
See `agent/SYSTEM_PROMPT.md` to relax this to full auto-execute.

## Secrets (never committed)

- `OLLAMA_API_KEY` — set on the droplet (`~/.hermes/.env`)
- FPL email/password — set on the droplet via `fpl init` (`~/.config/fpl-cli/config.json`)
- Photon tokens — written by `hermes photon setup` (`~/.hermes/.env`)

## MoA (dual-model review)

Tiered model routing:

| You text | Engine | Use for |
|---|---|---|
| plain message | kimi-k2.6 | data lookups: "any injuries?", "my team", prices |
| `/moa <question>` | kimi-k2.6 + deepseek-v4-flash → glm-5.2 aggregates | judgment calls: wildcard timing, chips, -4 hits, captain coin-flips |

Cron jobs run glm-5.2 (their output IS the recommendation).
MoA costs ~3x a normal call, ~15-30s.

# SETUP — droplet 64.23.135.50

Run steps in order. Est. total: ~45 min.

## 1. Provision the droplet (~10 min)

```bash
ssh root@64.23.135.50

# non-root user
adduser fpl && usermod -aG sudo fpl
ufw allow OpenSSH && ufw enable   # no other ports needed — Photon is outbound-only
su - fpl

# Node 22 (Hermes Photon sidecar needs >= 18.17; fpl-cli is node)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs python3-pip pipx
```

## 2. Install fpl-cli (~5 min)

```bash
git clone https://github.com/amit3992/fpl-cli.git ~/fpl-cli
cd ~/fpl-cli && npm install && npm run build && sudo npm link

# auth (needed for live squad + executing transfers)
fpl init --team-id <YOUR_TEAM_ID> --email <FPL_EMAIL> --password <FPL_PASSWORD>
fpl login
fpl doctor   # must be green
```

## 3. Install Hermes + configure the model (~10 min)

```bash
pipx install hermes-agent    # or per https://hermes-agent.nousresearch.com/docs
hermes setup                 # provider: ollama-cloud, model: deepseek-v4-flash
hermes config set cron.model deepseek-v4-flash            # pin cron model too
```

`OLLAMA_API_KEY` (from ollama.com/settings/keys) goes in `~/.hermes/.env` — **ask user for the key at this step.**

## 4. iMessage via Photon (~10 min)

```bash
hermes photon setup --phone <YOUR_PHONE_E164>
# prints your assigned iMessage line — save this number
hermes pairing approve photon <CODE>   # after you first text the line
hermes gateway install                 # systemd user service (boot-persistent)
hermes gateway start
hermes photon status                   # sidecar healthy?
```

Free-tier caveat: the shared line can only **reply** to numbers that have
texted it first — so text the line from your phone before relying on
cron-initiated alerts. Dedicated outbound line = Photon Business tier.

## 5. Load the agent prompt + skills

Copy `agent/SYSTEM_PROMPT.md` into the agent config (or as `AGENTS.md` in the
gateway workdir `/home/fpl/fpl-work`).

## 6. Create scheduled jobs

Run the `hermes cron create` commands in `cron/jobs.md`.

## 7. Smoke test

1. iMessage the line: "how's my team?" → expect squad summary via `fpl --json team`.
2. `hermes cron run gw-deadline-check` → expect an iMessage with deadline + squad news.
3. Ask "should I transfer out Watkins?" → expect a dry-run-backed recommendation,
   NO `--confirm` without your explicit yes.

## Security notes

- Droplet holds FPL credentials (`~/.config/fpl-cli/config.json`) — keep SSH
  key-only auth, `PermitRootLogin no`, unattended-upgrades on.
- All secrets live on the droplet, never in this repo.

---

## DEPLOYED STATE (2026-08-09)

- Droplet 64.23.135.50, user `fpl`, ssh alias `fpl-droplet`
- fpl-cli @ 2f6b63a, authenticated (team 3581612); traffic routed via WARP+proxychains wrapper at /usr/local/bin/fpl (DO IP is 429-blocked by FPL auth)
- Hermes v0.19.0, model deepseek-v4-flash via ollama-cloud
- Photon plugin: enabled; project = fpl-manager (04ee6d5f-...); line +1(628)267-9185; user +19728226226 paired
- Sidecar sources pulled from GitHub main (0.19.0 wheel omitted them)
- Gateway: systemd user service hermes-gateway, enabled + linger
- Cron (CDT): gw-deadline-check 18:00, price-watch 07:30, injury-scan 08:00, gw-review Tue 09:00 — all deliver to photon:+19728226226

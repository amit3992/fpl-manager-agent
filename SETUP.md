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
hermes setup                 # provider: openrouter, model: deepseek/deepseek-v4-flash
hermes config set cron.model deepseek/deepseek-v4-flash   # pin cron spend too
```

`OPENROUTER_API_KEY` goes in `~/.hermes/.env` — **ask user for the key at this step.**

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

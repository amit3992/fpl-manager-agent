# Scheduled jobs

Create after SETUP.md steps 3–4. Delivery target: the Photon iMessage line
(origin channel). All jobs pinned to `deepseek/deepseek-v4-flash`.

> Cron jobs are read-only by policy — they recommend, never `--confirm`.

## 1. Deadline eve check — "gw-deadline-check"

```bash
hermes cron create "every 1d at 18:00" \
  "Run fpl --json team --fields name,news,position and fpl --json news. \
   Check the next GW deadline. If deadline is within 48h: iMessage me a \
   countdown, my flagged/doubtful players, and ONE recommended action \
   (transfer/captain). Otherwise stay silent." \
  --workdir /home/fpl/fpl-work --name gw-deadline-check
```

## 2. Price-change watch — "price-watch"

```bash
hermes cron create "every 1d at 07:30" \
  "Run fpl --json team and fpl --json budget. Report any of my players who \
   rose/fell in price overnight and any target whose price moved. Keep it \
   to 3 lines max. Stay silent if nothing moved." \
  --workdir /home/fpl/fpl-work --name price-watch
```

## 3. Injury/news scan — "injury-scan"

```bash
hermes cron create "every 1d at 08:00" \
  "Run fpl --json news. If any of my squad has new injury/doubt news, \
   iMessage me the player, the news, expected return, and the top \
   replacement from fpl --json transfers suggest <name>. Silent if clean." \
  --workdir /home/fpl/fpl-work --name injury-scan
```

## 4. Post-GW review — "gw-review"

```bash
hermes cron create "every tuesday at 09:00" \
  "Post-GW review: run fpl --json team --gw <last gw> and fpl --json budget. \
   Summarize my GW points, rank movement, best/worst pick, and one lesson. \
   5 lines max." \
  --workdir /home/fpl/fpl-work --name gw-review
```

## Manage

```bash
hermes cron list
hermes cron pause|resume|run|edit <name>
```

Timezone: set droplet to your TZ (`sudo timedatectl set-timezone America/…`)
so "18:00" means *your* 18:00.

# Race day — what to do, and what to do when something breaks

Sept 5, 2026. Everything below is grounded in what we actually measured during the week of
test runs, not guesswork. Likelihoods are estimates; the evidence behind each is stated.

---

## Before you leave the house

- [ ] `ssh n64 '/var/root/bin/race prep 50 strap'` — **use this, not the app's toggle.**
      It sets the goal, wipes the live page, zeroes the clock AND the per-source distance
      baselines, and starts collection plus the strap in one command.
- [ ] Confirm on the page: **0.00 of 50**, elapsed counting from 0, HR labelled **chest strap**,
      map showing you.
- [ ] Wet the H10 electrodes. (Run 1: 28 disconnects, 90% coverage. Run 2, after wetting:
      **0 disconnects, 100%**.)
- [ ] Phone on the power bank *before* you start. Measured drain 12.7 %/h → ~7.9 h on its own,
      against a ~13 h race.
- [ ] Ultra 2: Low Power Mode + **Fewer GPS and heart rate readings**, Auto-Pause **off**,
      Wrist Detection **off** if you intend to charge it mid-race.
- [ ] Flip the three switches:
      - `live.html` → `CAM_LIVE = true`, `WISHES_LIVE = true`
      - `index.html` → `var LIVE_BANNER = true`
      - then `git commit && git push`, and wait ~45 s for GitHub Pages to deploy.

---

## Tier 1 — the page goes dark

### The phone reboots  ·  likelihood ~15–20 %
**Not hypothetical: it happened on Sept 3.** Uptime showed 9 h, `racectl` was dead, and the
app's buttons did nothing until it was restarted by hand.

Nothing auto-restarts. Verified: `/Library/LaunchDaemons`, `/Library/LaunchAgents` and
`/System/Library/LaunchDaemons` are all read-only on a sealed volume, and `/var/root/Library/
LaunchAgents` does not exist. `launchctl bootstrap` from a writable path *does* work while the
phone is up, but launchd will not re-scan it at boot, so it buys nothing here.

**The app cannot fix this** — it talks to `racectl` on :8081, and `racectl` died with the reboot.

**Recovery:** the Mac at the house is the only route. You pass it every lap.
```
ssh n64 '/var/root/bin/race ctl'          # revive the control server
ssh n64 '/var/root/bin/race on'           # restart collection (keeps clock + distance)
```
Use `on`, **not** `prep` or `reset` — those would zero your distance.

**Detection:** the page's dot turns red and the status reads `feed Nm stale`. Ask whoever is
watching to shout if that happens.

### The power bank disconnects  ·  likelihood ~25 % for at least one event
13 h × 12.7 %/h ≈ 165 % of a charge, so the phone cannot finish unaided. A jostled cable over
13 h is likely; the phone actually *dying* needs it to go unnoticed for hours.

**Mitigation:** spare cable in the aid box. Glance at the battery when you pass the house.
The page's feed age is the tell.

### ThingsBoard quota exhausted  ·  likelihood ~3 %
Was the biggest risk in the project; now the smallest. TTL is 3 days (was 30) and delta-push
sends ~3 keys instead of 38, so the race costs roughly 0.5–1 M storage-days against ~6 M
remaining. At the original settings it would have been 7–10.7 M — it would have died mid-race.

**Residual risk:** the cycle appears to reset ~Sept 15, i.e. *after* the race, so there is no
top-up coming. **Mitigation not yet in place:** the phone supports pushing to a second target
(`MODE2`/`URL2` in `/var/root/race/push.conf`) and the page already reads both and renders
whichever is fresher — but `MODE2` is **not configured**, so ThingsBoard is still a single
point of failure. Configuring it is the highest-value remaining task.

---

## Tier 2 — degraded, but the race keeps showing

| What | Likelihood | What you lose | What saves it |
|---|---|---|---|
| **Strap drops out** | ~40 % of a multi-minute gap | strap HR for that window | Auto-falls back to watch HR after 25 s. Run 1 lost 5.4 min of 57 and still read fine. |
| **Watch dies or is charging** | ~70 % it needs a charge | the *workout* total; nothing live | Phone pedometer + GPS bridge carry the headline. The per-source-delta fix means it no longer freezes. |
| **HealthKit lags** | ~100 %, it is normal | nothing | Measured lags of 5.5, 18 and 27 min — and one 12-min run where **zero** samples arrived. The GPS bridge carried 0.36 of 0.80 mi. Watch `healthkit Nm behind` in the footer. |
| **GPS degrades** | ~15 % | the bridge, last-mile pace, the map | HealthKit still supplies distance, just laggy. `locwatch` restarts `locmon` if it crashes. |
| **Camera throttled or offline** | ~40 % at scale | the camera panel only | Shows "camera offline — retrying" and retries every 30 s. `pub-*.r2.dev` is Cloudflare's *dev* endpoint and is rate-limited; a custom domain would fix it. Race numbers unaffected. |
| **Too many viewers** | ~20 % | freshness, not data | Measured ceiling ~37 reads/s. At a 12 s poll that is ~300 simultaneous foreground viewers. Past that the page shows "live · busy, retrying", backs off, and keeps the last good numbers. |
| **Transient push failures** | ~100 % | a few seconds | Seen every run (`http=000`, `recovered after 1 failure`). Self-healing. Ignore it. |

---

## Tier 3 — the mistakes most likely to actually happen

### Starting with the toggle instead of `prep`  ·  likelihood ~30 %
**Happened twice this week.** The Data collection toggle starts collection but leaves the clock
and the distance baselines stale. Symptom: elapsed reads *hours*, pace reads ~2,000 min/mi, and
the page shows dashes for pace and ETA because they fail the plausibility gate.

**Fix mid-run**, without losing distance:
```
ssh n64 'python3 -c "import time; open(\"/var/root/race/start_epoch\",\"w\").write(str(int(time.time())-SECONDS_SO_FAR))"'
```

### Forgetting a flag  ·  likelihood ~40 % of missing at least one
`CAM_LIVE`, `WISHES_LIVE`, `LIVE_BANNER`. All three are in the checklist above. Harmless if
missed — the page just shows placeholders — but it is the sort of thing noticed at mile 30.

### Pre-run walking counted as race distance  ·  expected, by design
The baseline comes from *arrived* HealthKit totals, so ground covered between HealthKit's
coverage horizon and pressing start leaks in — roughly **0.2–0.9 mi** depending on the lag.
Check `healthkit Nm behind` in the footer before you start; the smaller it is, the less leaks.
Consequence: the page may read 50.00 slightly before you have run 50.

---

## The two things worth doing before Saturday

1. **Configure the second push target.** Removes the only remaining single point of failure.
   The plumbing is deployed and tested on both ends; it needs a URL that accepts a PUT.
2. **Tell whoever is watching the page what "feed 5m stale" means** — that is the signal that
   something needs a hand, and they can reach you before it becomes an hour.

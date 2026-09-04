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
`/System/Library/LaunchDaemons` are all read-only on a sealed volume, `/var/root/Library/
LaunchAgents` does not exist, and there is no `cron`, `crontab` or `atrun` on the device.
`launchctl bootstrap` from a writable path *does* work while the phone is up, but launchd
will not re-scan it at boot, so it buys nothing here.

**The app cannot fix this** — it talks to `racectl` on :8081, and `racectl` died with the reboot.

**Recovery A — from the phone, no Mac (added Sept 4).** In the Terminal app:
```
cd /var/mobile && ./go.sh
```
That restarts the pipeline as the **mobile** user and **keeps the clock and the distance**.

**Confirmed end to end on the device from the real Terminal app, Sept 4.** All six processes
came up as `mobile` and published to both targets with zero failures — `state.json` and
`last_push.json` 1–4 s old, `target_mi 50`, real GPS, and the R2 object the page reads was
8 s fresh. Then `go.sh` itself was killed with SIGHUP+SIGTERM, exactly what closing Terminal
does, and **every process survived**, reparented to ppid 1, still publishing. `nohup` covers
them, so you can put the phone away.

One cosmetic wart, so it does not alarm you at mile 30: **your prompt may not come back.**
`go.sh` prints its summary and then blocks writing to the tty as soon as you leave Terminal,
so the script lingers even though its work is finished. Harmless — the pipeline has already
detached. Switch away or close the tab and it keeps running. Running `go.sh` a second time is
also safe; the guards handle it.

Why it works: the binaries that need privilege get it from entitlements on the binary, not
from the uid. Verified at uid 501 vs uid 0 — `hkctl` returns byte-identical HealthKit output
(same sums, same provenance rows), `locmon` produces real GPS fixes, `curl` publishes to both
targets (R2 PUT → 200, read back live from the phone). `/var/root` and `/var/root/bin` are
`drwxr-xr-x`, and every state file needed to resume is `0644`, so mobile can read the session
and carry on from it. There is no `ssh` client, no `sudo` and no `su` on this phone and
`/private/var` is `nosuid`, so getting back to *root* is not possible — running as mobile is.

Two guards, both tested:
- It refuses to start if root's `state.json` is under 180 s old, so you can never end up with
  two collectors double-publishing. (`sh /var/mobile/go.sh force` overrides.)
- If root's `start_epoch` differs from the copy it holds, a new run has been prepped since,
  so it discards its stale state rather than resuming into the wrong session.

Also: `status`, `stop`, and `nostrap`. `stop` waits up to 20 s and then escalates to `-9`,
because `racehist`/`racecollect` sit inside ~13 s `hkctl` calls and a short wait reports a
stop that did not happen.

**The strap keeps working in mobile mode.** `rrd2` runs fine at uid 501 — verified Sept 4, it
opened CoreBluetooth with no permission error and wrote its logs. It takes the output
directory as `argv[1]` and `racecollect`'s `RRDIR` is now overridable, so the pair lives
entirely under `/var/mobile`. `go.sh` starts it automatically, *unless* an `rrd2` is already
running — the H10 only supports ~2 concurrent connections and Health holds one, so in that
case it reads root's `/var/root/rrlog` (world-readable) instead of competing for the device.

**Do not bother trying to setuid `go.sh` — it cannot work, tested Sept 4.** Two independent
blockers: every writable volume mobile can reach is mounted `nosuid` (the bit sets fine and
the kernel ignores it; a setuid copy of `/usr/bin/id` still reported uid 501), and Darwin
ignores setuid on `#!` scripts regardless, so it would have to be a compiled, code-signed
binary on a volume that does not exist here. Running as mobile is not a workaround for the
lack of root — it is the answer, and it costs almost nothing.

**What you actually lose in mobile mode:** only `racectl`, the app's control channel on
:8081. That binary hardcodes the root state dir, so the app's buttons stay dead — but `go.sh`
replaces the app. Distance, map, GPS bridge, last-mile pace, strap HR and both publish
targets all continue.

**`/var/mobile/race/push.conf` is the load-bearing piece.** Root's copy is `0600 root`, so
mobile cannot read it; a copy is staged at `/var/mobile/race/push.conf` (`0600`, mobile-owned,
in a `0700` directory). Without it `go.sh` collects but publishes nothing — it says so loudly
rather than looking healthy. **If you re-mint the R2 URL, re-stage this copy too.**

**Recovery B — from the Mac**, still the cleanest if you are at the house:
```
ssh n64 '/var/root/bin/race ctl'          # revive the control server (restores the app)
ssh n64 '/var/root/bin/race on'           # restart collection (keeps clock + distance)
```
Use `on`, **not** `prep` or `reset` — those would zero your distance. Prefer this one when
it is available: it also brings back `racectl`, so the app's buttons work again. `go.sh` does
not restart `racectl` (that binary hardcodes the root state dir).

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

**Mitigated as of Sept 4.** The phone now pushes to ThingsBoard **and** to R2, and the page
reads both and renders whichever carries the newer phone timestamp. Verified live: identical
`t` from both sources, matching values, zero second-target failures.

- Second target: a **presigned PUT URL** scoped to the single object `handel-cam/race/state.json`,
  valid until **Sept 11**. Deliberately not full write credentials — the URL can only write
  that one key, and only until it expires.
- Public read: `https://pub-8cd8f10809a84abbb9915ef5cdd8e378.r2.dev/race/state.json`, which the
  page has as `DEFAULT_SRC`. R2 returns CORS for phandel.com even on a 404, so a missing object
  degrades cleanly instead of erroring.
- **`URL2` must stay quoted** in `push.conf`: the file is sourced by the shell and a presigned
  URL contains `&`, which unquoted empties the variable and silently disables the second target.
  The pusher now logs a warning if `MODE2` is set with an empty `URL2`.
- Re-mint after expiry with `~/bin/r2-presign.py` (reads credentials from the environment,
  stores none).

**Residual risk:** the ThingsBoard cycle appears to reset ~Sept 15, i.e. after the race, so
there is no quota top-up coming — but with two independent publish targets, exhausting it no
longer takes the page down.

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

## Still worth doing before Saturday

1. ~~Configure the second push target.~~ **Done Sept 4** — ThingsBoard and R2, verified live.
2. ~~Test `go.sh` from the Terminal app.~~ **Done Sept 4** — ran from Terminal, all six
   processes came up as `mobile`, published to both targets with zero failures, and survived
   `go.sh` being SIGHUP'd/SIGTERM'd the way closing Terminal would. See Recovery A above.
3. **Tell whoever is watching the page what "feed 5m stale" means** — that is the signal that
   something needs a hand, and they can reach you before it becomes an hour. With no automatic
   restart after a reboot, a human noticing is the entire detection layer.
4. **Rotate the R2 access keys after the race.** They were pasted into a chat transcript on
   Sept 4. The presigned URL on the phone is unaffected by a rotation only until it expires on
   Sept 11, so re-mint it afterwards if the second target is still wanted — and re-stage
   `/var/mobile/race/push.conf` at the same time.

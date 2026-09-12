# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Lectura obligatoria al iniciar sesión

Lee [`SESSION_LOG.md`](./SESSION_LOG.md) antes de cualquier tarea. Contiene el historial de cambios realizados en sesiones anteriores: qué se implementó, qué archivos se modificaron y decisiones de diseño tomadas. Actualiza ese archivo al final de cada sesión con un resumen de lo que hiciste.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run locally
uvicorn api:app --host 0.0.0.0 --port 8000 --reload

# Docker
docker build -t miruro-api .
docker run -p 8000:8000 miruro-api
```

There is no test suite and no linting configuration.

## Deployment: 5 nodes, one is special

This app runs on 5 nodes behind a load balancer: 4 cloud nodes + **this physical home machine**
(reachable from the cloud nodes over ZeroTier at a private address — see `NOTIFY_RELAY_URL`
below). Only this home machine has Hermes (a personal agent framework) installed, which is how Telegram alerts get
sent — see `NOTIFY_RELAY_URL` below. All 5 nodes share the same Redis instance and the same
`API_KEY`.

## Architecture

The entire API lives in a single file: `api.py` (~1100 lines). It is a FastAPI app that acts as a thin, authenticated proxy over two upstream sources:

1. **AniList GraphQL** (`https://graphql.anilist.co`) — all anime metadata: search, filter, collections, info, characters, relations, recommendations.
2. **Miruro Pipe** (`{MIRURO_BASE_URL}/api/secure/pipe`) — episode lists and M3U8 streaming URLs. Miruro's pipe protocol base64-encodes (no gzip) every request and gzip+base64-compresses every response; `_encode_pipe_request()` and `_decode_pipe_response()` handle this transparently.

### Cloudflare `cf_clearance` — required to reach the pipe at all

Miruro's Cloudflare zone serves an interactive JS challenge ("Just a moment...") to this app's
traffic. A valid `cf_clearance` cookie (plus the exact browser headers it was solved with) is
required on every pipe request, or Cloudflare 403s it. Two things had to be true at once for
this to work reliably, both found the hard way (see `SESSION_LOG.md`, sessions 2026-09-07):

1. **Getting the cookie**: solving the challenge requires a real, non-headless browser. Headless
   Chromium (plain Playwright, and `patchright`'s stealth fork) gets stuck on the challenge
   forever. Beyond that, **the browser must have ZERO automation attached while the challenge
   resolves** — confirmed live: launching Chrome with Playwright/patchright controlling it from
   the first navigation (CDP active before the page even loads) either never cleared the
   challenge, or cleared it with an incomplete header capture that made the resulting cookie
   fail live pipe calls anyway. `cf_refresher.py`/`mac_agent/refresher.py` now launch the browser
   as a **raw subprocess** (`--remote-debugging-port` open but nothing connected), wait
   `NAKED_LAUNCH_WAIT_SECONDS` (default 20, under **Xvfb** on Linux via `xvfb-run -a`) completely
   untouched, and only THEN attach via `connect_over_cdp` to pull the cookie. Header capture also
   has to merge two CDP events — `Network.requestWillBeSent` (has `sec-ch-ua-*`/`user-agent`) and
   `Network.requestWillBeSentExtraInfo` (has `sec-fetch-*`/`cache-control`/`pragma`/`priority`,
   which Chromium always attaches but Playwright's simple `request.headers()` doesn't expose) —
   using only the first one was silently producing incomplete, non-working cookies. Publishes
   `{cookie, headers}` as JSON to the Redis key `miruro_api:cf_clearance:{FALLBACK_TOPIC}`
   (`_get_pipe_headers()` reads it, cached in-process for `CF_CLEARANCE_LOCAL_CACHE_SECONDS`).
   The cookie is **not** tied to the requesting IP (verified: a cookie solved on one device
   works fine replayed from this server) — it's tied to the header set (`sec-ch-ua`/`user-agent`/
   etc.) matching exactly what Cloudflare saw when it was issued. That's *why* sharing one cookie
   across every node used to work at all technically — but don't read that as "one shared cookie
   is fine": see the `FALLBACK_TOPIC` isolation note below for why every group now has its own key
   regardless, and why this is not up for debate again.
2. **Replaying the cookie**: even with a byte-for-byte matching cookie+headers, the pipe still
   403s if the request goes out over plain **HTTP/1.1** — `curl_cffi` (any `impersonate=` profile)
   and httpx's default both failed live; only HTTP/2 (`httpx.AsyncClient(http2=True)`, matching
   what a real browser and system `curl` both negotiate by default) gets a 200. `_pipe_get()` uses
   `httpx` with `http2=True` whenever a `cf_clearance` blob is present in Redis, falling back to
   the old `curl_cffi` (`impersonate=PIPE_IMPERSONATE`) `pipe_session` only when Redis has nothing
   cached (rare/degraded path, effectively dead weight now but kept as a fallback).

**Keeping the cookie fresh — two triggers, not one:**
- *Proactive*: `cf_refresher.py` is meant to run on a timer (`mi-api-cf-refresh.timer`/`.service`,
  **not yet installed** as of 2026-09-07 — see SESSION_LOG). It checks the cookie's real AGE
  (`updated_at` in the stored blob, not a Redis TTL — see below) first and skips (no browser
  launch) unless it's older than `MIN_REFRESH_AGE_SECONDS` (10 min) — cuts real Chromium/
  challenge-solve runs down from one per timer tick to only when actually useful.
- *Reactive* (the one that matters for uptime): when a live pipe request gets a 403 **or a 444**
  with a cookie set, `api.py`'s `_pipe_get()` fires `_trigger_reactive_cf_refresh()` via
  `asyncio.create_task` (fire-and-forget — never blocks the real client response) — a Redis lock
  (`miruro_api:cf_refresher:reactive_trigger_lock:{FALLBACK_TOPIC}`, 60s TTL) de-dupes concurrent
  failures within the same group into one browser launch, and `cf_refresher.py --force` (bypasses
  the TTL-skip check) runs in the background. Recovery for subsequent requests: ~15-30s. A Redis
  TTL that says "still valid" is **not proof the cookie actually works** (learned the hard way) —
  the reactive path is what actually catches real breakage, the proactive timer is just cheap
  insurance between failures.
  - **444 was NOT part of this trigger until 2026-09-10** — a real, confirmed-live gap: a cookie
    can stay 200 on `episodes` (still valid for Cloudflare) while `sources` 444s across EVERY
    provider at once (a genuine break, not the normal single-provider categorical-failure noise
    — see `CANARY_CHECK_SOURCES` below). Because the trigger only fired on 403, that scenario sat
    broken **for hours**, silently, with zero automatic recovery attempt, zero Mac/Windows
    fallback ping, and zero escalation alert — nothing ever even looked at it, since the whole
    reactive chain never started. Fixed by adding 444 alongside 403 in both `_pipe_get()` branches
    (the `httpx`/cookie path and the `curl_cffi` fallback path). Safe against false-positive
    browser launches because `_cf_clearance_actually_broken()` — the function the trigger calls
    first — already tries every available provider before concluding "broken"; a single
    provider's 444 alone (the common, harmless case) still won't launch anything. Verified with
    real, unmocked code before deploying: with the old code, a real live 444 called the trigger
    **0 times**; with the fix, the same real 444 called it and the trigger's own real Redis
    writes (lock, `break_detected_at`, `need_mac_refresh` flag) and real `subprocess.Popen`/
    `publish` calls all fired as expected.
- If the forced refresh itself fails (e.g. Cloudflare escalates to an interactive Turnstile a
  non-headless-but-still-automated browser can't solve), the service **stays down** — there's no
  further automatic fallback. A human has to solve the challenge in a real browser and hand the
  `cf_clearance` + full header set over to be pushed into Redis manually.
- **Alerting is per-type opt-in, not global** (revised 2026-09-09 — this was originally
  deliberately unfiltered/spammy, but the user found that too noisy in practice). Four distinct
  alert types exist, each gated by its own `NOTIFY_ON_*` env var: `NOTIFY_ON_BREAK_DETECTED`
  (api.py, fires on every real live 403 — default `false`), `NOTIFY_ON_ESCALATION` (api.py,
  fires only if NEITHER this node's own attempt NOR any fallback node fixed it within
  `MAC_ESCALATION_TIMEOUT_SECONDS` — default **`true`**, this is the one alert meant to survive
  the cut, since it's the only one that means "a human needs to act"), `NOTIFY_ON_SOLVE_FAILURE`
  (cf_refresher.py, a solve attempt exhausted `MAX_SOLVE_ATTEMPTS` — default `false`), and
  `NOTIFY_ON_RECOVERY` (cf_refresher.py, a break was successfully closed out — default `false`).
  Every node that can trigger any of these needs the SAME values set in its own `.env` — a node
  still running old code (no flag check at all) keeps sending every type unconditionally
  regardless of what any other node's `.env` says, since the gating logic itself has to be
  present in the code it's running.

### The `cf_clearance` cookie key had a real design flaw: it self-destructed on a fixed 25-min clock

Until 2026-09-11, `cf_refresher.py` wrote `miruro_api:cf_clearance:{FALLBACK_TOPIC}` with a hard
Redis TTL (`ex=25*60`) — meaning Redis itself would delete the key 25 minutes after it was last
written, **completely independent of whether the cookie actually still worked**. The user
correctly called this out as bad design: an arbitrary self-destruct timer has nothing to do with
real validity, which the project already has a proper way to check (`_cookie_actually_works()`/
`_cf_clearance_actually_broken()` — real HTTP calls against the pipe). A cookie that was still
perfectly good at the 25-minute mark got thrown away anyway; a cookie that died at minute 3 sat
in Redis, looking "valid" by TTL, for another 22 minutes.

**Fixed: the key now has no expiry at all.** It persists in Redis forever until something REAL
replaces it — a verified-broken check triggering a fresh solve (reactive 403/444,
`--proactive-monitor` finding it dead), or a routine/manual refresh. It never vanishes on its
own clock. Consequences of this fix, all handled:
- `cf_refresher.py`'s "none" mode (one-shot, skip-if-fresh) used to decide whether to bother
  solving by checking the Redis key's remaining TTL (`_current_ttl()`, `MIN_TTL_BEFORE_REFRESH_SECONDS`).
  With no TTL to read, this is now based on the cookie's own real AGE (`updated_at` in the stored
  JSON blob) instead — renamed `_current_cookie_age_seconds()` / `MIN_REFRESH_AGE_SECONDS` (still
  10 min default). This was always just an efficiency optimization (don't re-solve something
  solved 2 minutes ago), never a correctness mechanism — actual validity was, and still is,
  decided entirely by the real HTTP checks, never by this age cutoff.
- `mi_api_mcp.py`'s `estado_cf_clearance()` diagnostic tool dropped the now-meaningless
  `ttl_restante_seg` field (would always read -1 with no expiry) in favor of `hay_cookie` +
  `actualizado_hace_seg` (real age).
- `api.py`'s `REDIS_KEY_COOKIE_LIFETIME_SAMPLES` measurement (real elapsed time from write to a
  confirmed-dead request) was already the honest metric here — it never depended on the Redis
  TTL, only got its explanatory comment updated to stop referencing the now-removed 25-min cap.

**This does NOT replace the `--proactive-monitor`/reactive-detection work above** — a cookie that
Cloudflare has actually revoked still needs a real check to catch it; removing the Redis TTL only
stops the system from throwing away a cookie for no real reason. Both fixes are complementary:
this one stops false deaths, the earlier ones catch real ones faster.

### `NOTIFY_RELAY_URL` — Telegram alerts from the 4 cloud nodes

Only the home node has Hermes installed, so `notify_telegram()`/`_notify_telegram()` check for
a local Hermes binary first (path from the `HERMES_BIN_PATH` env var, set only on the home
node's own `.env` — never hardcoded in code); if it's unset/missing (any cloud node), they POST
`{"message": ...}` to `f"{NOTIFY_RELAY_URL}/internal/notify"` instead, authenticated with this
deployment's own `API_KEY`. `POST /internal/notify` (in `api.py`) is what actually calls Hermes
on the receiving end — it's a normal endpoint (not in the auth-bypass list), so it's protected
by the same `x-api-key` check as everything else. Leave `NOTIFY_RELAY_URL` unset on the home
node; set it to `http://<home-node-zerotier-ip>:8848` on the 4 cloud nodes.

### `cf_refresher.py`'s `--listen`/`--proactive-monitor` must NEVER exit — confirmed live 2026-09-11

A transient `redis.exceptions.TimeoutError` inside `_pubsub_loop`'s `pubsub.listen()` propagated
unhandled all the way up through `asyncio.gather` and killed the ENTIRE `--listen` process on
the real Windows fallback node for `group-ubuntu-windows`. The process just exited — no crash
loop, no alert, nothing — and the group sat with zero fallback coverage until, shortly after, its
`cf_clearance` genuinely expired, the proactive monitor correctly detected it and asked the
fallback to regenerate it, and **nobody answered** for ~21 minutes because the thing meant to
answer was already dead on the floor with no one aware of it.

Fixed on three layers, all confirmed live before deploying (see `SESSION_LOG.md`, 2026-09-11):
1. **`_pubsub_loop`**: the `subscribe`/`listen` pair now lives inside its own `while True` +
   `try/except`, never allowed to propagate. On any failure it logs, sends an unconditional
   Telegram alert (NOT gated by any `NOTIFY_ON_*` opt-in — deliberately, per explicit user
   demand: a dead fallback listener is worse than any amount of alert noise), sleeps
   `CRASH_ALERT_INTERVAL_SECONDS` (30s), and retries — repeating the alert every 30s for as long
   as it stays broken. `_poll_loop` and `_proactive_monitor_loop` got the same full-iteration
   try/except treatment (parts of both were previously unguarded: `_poll_loop`'s
   `run_refresh_once()` call, and `_proactive_monitor_loop`'s entire "cookie stopped working"
   branch after its inner `try` block).
2. **`_proactive_monitor_loop` now schedules its own escalation** (`_escalate_if_fallback_never_fixed_it`,
   `PROACTIVE_MONITOR_ESCALATION_TIMEOUT_SECONDS` default 120s, gated by `NOTIFY_ON_ESCALATION`
   — same flag name/intent as `api.py`'s, now also read here) — mirrors `api.py`'s
   `_escalate_if_still_broken` exactly. This didn't exist before: `api.py`'s escalation only
   fires from ITS OWN reactive trigger, in a totally separate process, so a break the proactive
   monitor detects (in its own process) had nothing watching whether the fallback ever actually
   fixed it. This is precisely how the ~21-minute silent gap happened.
3. **Top-level supervisor** in `if __name__ == "__main__":` for `--listen`/`--proactive-monitor`:
   wraps `asyncio.run(main())` in a `while True`/`try/except` as a last-resort net — if literally
   anything still escapes the per-loop guards above, it logs, sends the same unconditional
   30s-repeating Telegram alert, and restarts `main()` in a fresh event loop. Belt and suspenders
   with layer 1 above, deliberately — the user's exact words were "no debe romperse la ejecución
   por NADA".

Verified live before deploying: forced 2 consecutive simulated Redis connection failures inside
`_pubsub_loop` (real code, only the failure itself was injected) — the process stayed alive
through both, sent 2 real Telegram-alert-call invocations 30s apart, and reconnected for real on
the 3rd attempt. Never crashed.

### `cf_refresher.py` — ONE script, any OS, four modes

Solves the Cloudflare challenge and refreshes `miruro_api:cf_clearance:{FALLBACK_TOPIC}`. Runs **unchanged** on
this home server, a Mac, a Windows box, or any extra Linux/Ubuntu machine you add later — it
detects the OS at runtime (`_find_chrome_path`) and finds the right Chrome binary; only the
`.env` differs per machine (`NODE_ID`, `NOTIFY_RELAY_URL`, etc — same one `.env` as `api.py`,
never a separate copy per machine). There used to be separate `mac_agent/`/`windows_agent/`
directories with near-duplicate code — collapsed into this one file once it became clear the
only real per-OS difference is "how do I find/launch a bare Chrome", not the surrounding logic.

**Modes** (CLI args):
| Mode | What it does |
|---|---|
| *(none)* | One-shot: skip if the cached cookie still has plenty of TTL left, else refresh. This is what `api.py`'s reactive trigger calls. |
| `--force` | One-shot, skip the TTL check — always attempt. |
| `--listen` | Run forever as an active fallback node: Redis Pub/Sub (instant reaction) + a periodic poll (durable fallback for whenever this machine was asleep/offline when the trigger was published). Deploy this on any extra machine you want acting as a second/third/etc. `cf_clearance` source. |
| `--dry-run` | Solve + verify only — prints PASS/FAIL against the real pipe endpoints (`episodes` then `sources`, cache-busted). Does **not** write to Redis or notify anyone. Use this to test whether a machine's IP is even viable before deciding to run it with `--listen`. |
| `--proactive-monitor` | Run forever as its own separate process/systemd unit (`mi-api-proactive-monitor.service`, distinct from `mi-api-fallback-agent.service`): every `PROACTIVE_MONITOR_INTERVAL_SECONDS` (default 600 = 10 min, raised to 900 = 15 min on 2026-09-11 — see below), re-verifies whatever cookie is CURRENTLY in Redis with a pure HTTP check (`_cookie_actually_works` — no browser). Closes a real gap confirmed live 2026-09-10: a cookie that expires with **zero live traffic** hitting it during its whole lifetime never gets noticed by the purely-reactive path (nothing failed, so nothing triggered) — it just sits dead until someone finally makes a real request. If the check passes, does nothing. If it fails, asks the real fallback node (Mac/Windows) to regenerate it — via the same `need_mac_refresh` flag + pub/sub publish as the reactive trigger — but **deliberately never attempts a local browser solve itself**, since this is the same automation IP CLAUDE.md already warns gets progressively distrusted by Cloudflare from solving too many challenges; a proactive check firing every few minutes forever must not add to that. |

**Durable stats, so its real hit rate can be checked later without depending on `journalctl`**
(which rotates/purges — not a real record): `miruro_api:proactive_monitor:stats:{FALLBACK_TOPIC}`
(a Redis hash — `total_checks`, `valid_checks`, `broken_detected`) and
`miruro_api:proactive_monitor:break_log:{FALLBACK_TOPIC}` (a capped list of every real break
detection, each entry `{at, reason, interval_seconds}` — `reason` is `no_cookie_in_redis` or
`cookie_stopped_working`, and `interval_seconds` records what the cadence was AT THAT MOMENT, so
a cadence change (e.g. 10min → 15min) can be compared using real before/after data instead of
re-reading logs that may no longer exist by the time you go looking).

**Interval tuning (2026-09-11):** first 5 real break-lifetime samples (measured by the reactive
path, `REDIS_KEY_COOKIE_LIFETIME_SAMPLES`) were `220s, 250s, 1370s, 1252s, 505s` — quite spread
out. Raised from 10 to 15 min (`PROACTIVE_MONITOR_INTERVAL_SECONDS=900`) since 4 of 5 samples are
under 15 min; 20 min was considered and rejected (one sample, 1370s ≈ 22.8 min, would exceed it).
Caveat worth remembering if this gets revisited: those reactive-path samples are almost
certainly biased short — live traffic samples far more often than a 10-15 min proactive check,
so it catches brief all-provider Miruro-backend blips (not real sustained Cloudflare-level cookie
death) that a periodic monitor would almost always miss entirely — which is consistent with the
proactive monitor finding the cookie broken in only ~1 of ~20 combined cycles across both groups
in its first ~10 hours running. Decided to validate the interval change with real data (the
counters above) rather than deliberately breaking the production cookie to force more samples —
that specific destructive test was tried and blocked twice by Claude Code's safety classifier
(see `SESSION_LOG.md`, 2026-09-10) and wasn't worth pursuing further for a tuning decision this
minor.

**Why it needs to exist at all:** headless Chromium (plain Playwright, and `patchright`'s
stealth fork) gets stuck on the challenge forever. Beyond that, **the browser must have ZERO
automation attached while the challenge resolves** — confirmed live: launching Chrome with
Playwright/patchright controlling it from the first navigation (CDP active before the page even
loads) either never cleared the challenge, or cleared it with an incomplete header capture that
made the resulting cookie fail live pipe calls anyway. `_solve_challenge_and_capture` launches
the browser as a **raw subprocess** (`--remote-debugging-port` open but nothing connected), waits
`NAKED_LAUNCH_WAIT_SECONDS` (default 20) completely untouched, and only THEN attaches via
`connect_over_cdp` to pull the cookie. Header capture also has to merge two CDP events —
`Network.requestWillBeSent` (has `sec-ch-ua-*`/`user-agent`) and
`Network.requestWillBeSentExtraInfo` (has `sec-fetch-*`/`cache-control`/`pragma`/`priority`,
which Chromium always attaches but Playwright's simple `request.headers()` doesn't expose) —
using only the first one was silently producing incomplete, non-working cookies.

**Why an extra machine at all:** even this home server's own Xvfb+patchright automation
eventually gets distrusted by Cloudflare — it's the same IP auto-solving hundreds of challenges
a day, which is exactly the pattern a bot-management WAF learns to flag. A residential Mac (or
any machine on a different, less-flagged IP) running a real Chrome, with a normal mixed traffic
history, gets trusted far more.

**Second-tier trigger model** (server side lives in `api.py`'s `_trigger_reactive_cf_refresh`,
fired alongside — not after — the home node's own one-shot attempt):
- `SET miruro_api:need_mac_refresh:{FALLBACK_TOPIC} EX 600` — a durable flag, polled by every
  `--listen` node in the same topic every `FALLBACK_POLL_INTERVAL_SECONDS` (default 30 min) as
  the fallback for whenever that machine was asleep/offline at the moment of the real event.
  (Key name is historical — "mac" isn't literal, it means "whichever fallback node answers for
  this topic".)
- `PUBLISH miruro_api:mac_refresh_channel:{FALLBACK_TOPIC}` — instant reaction whenever a
  `--listen` node in the same topic's listener happens to already be connected. Redis Pub/Sub
  does **not** queue messages for offline subscribers, so this is the fast path, never the only
  path. Multiple `--listen` nodes in the same topic can be subscribed at once — a single trigger
  fans out to all of them in that topic, and whichever actually produces a working cookie first
  wins (harmless if more than one succeeds within the same topic; they just overwrite the same
  Redis key with an equally-valid cookie).
- `asyncio.create_task(_escalate_if_still_broken())` — scheduled the moment a break is detected,
  wakes once after `MAC_ESCALATION_TIMEOUT_SECONDS` (120s) and checks whether
  `miruro_api:cf_refresher:break_detected_at` is *still* set. If so — nothing anywhere fixed it
  in time — sends an escalation Telegram alert. Deliberately checks the real outcome (is the
  cookie still broken) rather than "did some node acknowledge the message" — an ack only proves
  the message arrived, not that Chrome actually solved the challenge.

**`FALLBACK_TOPIC` isolates the cookie itself, not just who gets woken up — this is not
optional, and it must never be walked back.** Every group (`{production node(s)} + {--listen
fallback node(s)}` sharing one `FALLBACK_TOPIC` value) has its own, completely independent
`cf_clearance` cookie at `miruro_api:cf_clearance:{FALLBACK_TOPIC}` — as well as its own
need/refresh flag and pub/sub channel, both shown above. **Do not go back to one shared
`cf_clearance` key across groups, even though a cookie is technically valid replayed from any
IP** (see the header-set-not-IP note above) — that was the original design here, and it was
wrong: sharing one cookie coupled every group's fate together. A break in one group's cookie
made every node reading that same key 403 at once (regardless of group), and — because each
node also holds its own few-second in-process copy (`CF_CLEARANCE_LOCAL_CACHE_SECONDS`) — two
completely unrelated fallback groups could appear to "trigger each other" seconds apart. That
was never one group waking the other (the topic-scoped trigger channels were already isolated
even before this fix); it was two independent nodes racing against the *same* shared cookie,
each hitting its own stale local copy at a slightly different moment. Confirmed live 2026-09-07
(see `SESSION_LOG.md`) and fixed by giving every group its own cookie key, full stop — a group
must live or die entirely on its own cookie, with zero coupling to any other group's.

**Every piece of shared state for a group must be topic-scoped, no exceptions** — found
2026-09-09: `CF_REFRESHER_TRIGGER_LOCK_KEY` and `REDIS_KEY_BREAK_DETECTED_AT` were left as flat,
global keys during the fix above, missed because they weren't part of the visible "groups
triggering each other" symptom. Consequence: two groups breaking at the same moment meant only
one could win the shared reactive-trigger lock — the LOSING group's
`_trigger_reactive_cf_refresh` returned immediately with no local browser attempt, no Telegram
alert, no fallback ping to its own Mac/Windows, and no lifetime sample, for that entire group,
until its next failing request 60s later. Both are now `f"...:{FALLBACK_TOPIC}"` too. When adding
any NEW piece of shared per-group state in the future, scope it by `FALLBACK_TOPIC` from the
start — don't wait for a visible symptom to notice it wasn't.

**Setup on a new machine** (Mac, Windows, or another Linux box — needs to be joined to the same
ZeroTier network as the home node, to reach both Redis and `NOTIFY_RELAY_URL`):
```bash
git clone <this repo> ~/MI-API   # or wherever
cd MI-API
cp .env_example .env
# edit .env: REDIS_HOST/PORT/PASSWORD (same as the server's), API_KEY (same shared key),
# MIRURO_BASE_URL, NOTIFY_RELAY_URL (home node's ZeroTier address), NODE_ID (e.g. "mac", "cloud-ubuntu-2")

python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
patchright install chromium       # Linux only — Mac/Windows use the real installed Chrome instead

# Test it works at all before wiring it into the fleet:
python cf_refresher.py --dry-run   # Linux: xvfb-run -a python cf_refresher.py --dry-run

# Run it as the active fallback node:
python cf_refresher.py --listen    # Linux: xvfb-run -a python cf_refresher.py --listen
```

Keep it running persistently:
- **Linux**: `mi-api-fallback-agent.service` (systemd) — edit the placeholder paths/user, then
  `sudo cp` it to `/etc/systemd/system/`, `daemon-reload`, `enable --now`.
- **Mac**: `com.mi-api.fallback-agent.plist` (launchd) — edit the placeholder paths, then
  `cp` to `~/Library/LaunchAgents/`, `launchctl load`.
- **Windows**: Task Scheduler, "run at log on", pointed at `venv\Scripts\python.exe
  cf_refresher.py --listen` with the repo as the working directory.

**`--proactive-monitor` is a SEPARATE unit from the one above** — `mi-api-proactive-monitor.service`
(same edit-placeholders-then-`sudo cp`/`daemon-reload`/`enable --now` flow). Deploy it on the same
node that runs `api.py` for a given group (this server for `group-mac-ubuntu`, `comba-server-1`
for `group-ubuntu-windows`) — it watches THAT group's cookie, not the fallback node's. No
`xvfb-run` needed (it never launches a browser, just HTTP checks), unlike the other two units.

Uses a dedicated, throwaway Chrome profile (`.chrome-profile/`, gitignored) rather than the
user's live daily-driver profile — this never conflicts with them actually using Chrome at the
same time as a refresh runs.

### `mi_api_mcp.py` — MCP server for manual diagnosis (home node only)

MCP server (stdio, `mcp.server.fastmcp.FastMCP`) exposing `estado_cf_clearance()` and
`refrescar_cf_clearance()` for manual diagnosis/triggering from Hermes chat. Registered in
`~/.hermes/config.yaml` under `mcp_servers.mi_api` (home node only — that's where Hermes runs).
**Pin `mcp[cli]==1.28.1` in requirements.txt** — `mcp` v2.x renamed `FastMCP` to `MCPServer` and
breaks this import; all the other MCP servers on this machine (`camaras_ip`, `jkanime_relator`,
etc.) are on 1.28.1 too, for the same reason.

### Security middleware (`secure_api`)

Every non-doc request must pass one of two checks (checked in order):
- Valid `x-api-key` header matching `API_KEY` env var
- `Origin` or `Referer` header that starts with one of the `ALLOWED_ORIGINS`

Doc paths (`/`, `/docs`, `/redoc`, `/openapi.json`) bypass this check entirely.

### ID encoding

Episode IDs returned by the Miruro pipe are base64-encoded. `_translate_id()` decodes a single ID; `_deep_translate()` recursively walks any JSON structure and decodes all IDs. Endpoints that return episode data must call `_deep_translate()` before returning.

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `ALLOWED_ORIGINS` | localhost variants | Comma-separated CORS + auth whitelist |
| `API_KEY` | `123456` | Auth header value (`x-api-key`) |
| `API_DEBUG` | `False` | `True` renders a styled HTML homepage; `False` renders a minimal page |
| `REDIS_HOST` | `localhost` | Redis host for the `/recent-episodes` cache |
| `REDIS_PORT` | `6379` | Redis port |
| `REDIS_PASSWORD` | — | Redis password |
| `CACHE_RECENT_EPISODES_HOURS` | `2` | TTL (hours) for the `/recent-episodes` cache |
| `BLOCKED_EPISODE_PREFIXES` | `` (empty) | Comma-separated episode ID prefixes (part before `:`, e.g. `animepahe`) to strip out of `/episodes` responses — use to hide a provider whose source is broken/hanging upstream, no code change needed |
| `MIRURO_BASE_URL` | — (required) | Base domain for the Miruro pipe, e.g. `https://www.miruro.to`. No hardcoded fallback — update this if Miruro changes domains again |
| `PIPE_USER_AGENT` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)` | User-Agent sent to the pipe |
| `PIPE_EXTRA_HEADERS` | `{}` | JSON object merged into pipe request headers (e.g. `sec-ch-ua`, `accept`, `cf_clearance`-adjacent headers) — used to adapt to Cloudflare without touching code |
| `CACHE_EPISODES_HOURS` | `1` | TTL (hours) for the `/episodes/{id}` cache |
| `NOTIFY_RELAY_URL` | `` (empty) | Base URL of the home node (its ZeroTier address, e.g. `http://10.x.x.x:8848`), used by cloud nodes to relay Telegram alerts through `POST /internal/notify` when no local Hermes install exists. Leave unset on the home node itself. |
| `HERMES_BIN_PATH` | `` (empty) | Absolute path to the local Hermes CLI binary. Only set on the home node's own `.env`; unset/missing anywhere else falls through to `NOTIFY_RELAY_URL`. |
| `NODE_ID` | OS hostname | Human-readable label for this node (e.g. `cloud-1`), appended to Telegram alerts as `[nodo: ...]` so you know which of the 5 nodes actually detected the failure. |
| `PROACTIVE_MONITOR_INTERVAL_SECONDS` | `600` (10 min) | How often `cf_refresher.py --proactive-monitor` re-verifies the currently stored cookie. Only read by that mode/service, not by `api.py`. |
| `PROACTIVE_MONITOR_ESCALATION_TIMEOUT_SECONDS` | `120` | How long `--proactive-monitor` waits after asking the fallback node to regenerate a dead cookie before alerting (via `NOTIFY_ON_ESCALATION`) that nobody fixed it. Mirrors `api.py`'s hardcoded `MAC_ESCALATION_TIMEOUT_SECONDS`, but as its own env var since this runs in a separate process. |

### Deployment targets

- **Vercel**: `vercel.json` maps all routes to `api.py` via the Python runtime (`mangum` adapter is imported for ASGI compatibility).
- **Koyeb/Docker**: `Dockerfile` uses Python 3.11 slim, installs requirements, and starts uvicorn directly.

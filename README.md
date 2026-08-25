# Play heatmap

One collector on the Synology, feeding two surfaces on jessevaughan.com.

The heatmap is a GitHub-contribution-style map of daily playtime across Steam
and Nintendo Switch. The card panel on /now is `RecentGames.astro`, which
reads Steam from a committed snapshot and gets a matching Switch file from
here.

## Three sources, three different jobs

**Steam** returns a lifetime total per game and never a daily history, so its
series is built by polling every fifteen minutes and diffing. A session
crossing midnight is misattributed by at most one poll interval. Nothing
before the first poll is recoverable, ever, and no third party can supply it
because Valve keeps no session logs.

**Nintendo Parental Controls** returns real calendar days, so those are taken
as authoritative. A day is replaced rather than appended to on every ingest,
which makes reruns idempotent and lets today correct itself once `result`
moves from CALCULATING to ACHIEVED. It keeps 31 days and no more.

**Exophase** supplies lifetime Switch totals, which Nintendo's 31 day window
can never give. It reads the console's own activity log through a friend code,
which is the one route nobody outside their site has replicated. Their rows
carry the same Nintendo application ID the daily summaries use, so the join is
on an ID rather than a title string.

Set the Switch to the same timezone as `COLLECT_TZ`. Nintendo reports in the
console's own offset and the collector does not second-guess it.

## Files

    bin/store.py            schema and shared helpers
    bin/switch_auth.py      one-time Nintendo login, and an API probe
    bin/switch_ingest.py    Nintendo's daily records
    bin/exophase_ingest.py  lifetime Switch totals
    bin/exophase_probe.py   inspects their page when the shape moves
    bin/collector.py        Steam polling, both exports, both pushes
    docker-compose.yml      one service
    .env.example            copy to .env on the NAS

`PlayHeatmap.astro` lives in the portfolio repo and has been adapted there.
Don't copy an older version over it.

## Setup

1. Make the directories.

       mkdir -p /volume2/docker/play-heatmap/{bin,data,secrets} && chmod 700 /volume2/docker/play-heatmap/secrets

2. Copy `bin/*.py` into `bin/`, and `docker-compose.yml` plus `.env.example`
   alongside them.

3. Create the env file and fill it in.

       cp /volume2/docker/play-heatmap/.env.example /volume2/docker/play-heatmap/.env && chmod 600 /volume2/docker/play-heatmap/.env

   The Nintendo token does not go in here. It lives in `secrets/`.

4. Register the console in Nintendo's own Parental Controls app on your phone,
   under your own account. Set no restrictions, no play-time limit, no
   bedtime. The app is the only way to add a console. Confirm Play Activity
   shows yesterday before going further.

5. Log in. Interactive, so `-it` matters. Run once.

       cd /volume2/docker/play-heatmap && sudo docker run --rm -it -v "$PWD/bin:/app/bin:ro" -v "$PWD/secrets:/secrets" python:3.12-slim sh -c 'pip install -q pynintendoparental && python -u /app/bin/switch_auth.py login'

   It prints a Nintendo login URL. Sign in, then right-click the red button
   and copy the link address rather than clicking it. Paste that back. The
   token is written to `secrets/switch-token.json` at mode 600.

6. For lifetime totals, create an Exophase account, set your Switch Play
   Activity visibility to All Users, enter your friend code on their account
   page, and run a profile sync. Then put your Nintendo profile URL in `.env`
   as `EXOPHASE_URL`. Skip this and the cards say "recorded" instead of
   "all time".

7. Run one poll by hand.

       cd /volume2/docker/play-heatmap && sudo docker run --rm --env-file .env -e DB_PATH=/data/play.db -e JSON_PATH=/data/heatmap.json -e SWITCH_TOKEN_PATH=/secrets/switch-token.json -e RECENT_JSON_PATH=/data/switch-recent.json -v "$PWD/bin:/app/bin:ro" -v "$PWD/data:/data" -v "$PWD/secrets:/secrets" python:3.12-slim sh -c 'pip install --no-cache-dir -q pynintendoparental && python -u /app/bin/collector.py --once'

   You want `baseline recorded for N games`, then `switch: N days ingested`
   with a date range, then an `exophase:` line, then `poll ok`. Steam
   contributing nothing at this point is correct.

8. Start it.

       cd /volume2/docker/play-heatmap && sudo docker compose up -d && sudo docker logs -f play-collector

An `.env` change needs `docker compose up -d --force-recreate`, since a plain
restart reuses the environment the container was created with. Changes under
`bin/` are bind-mounted, so a restart is enough for those.

## Inspecting

`switch_auth.py probe` reports what Nintendo returns: the date range held, the
mix of `result` values, raw minute figures for the busiest days, and the shape
of `players[].playedGames[]`.

    cd /volume2/docker/play-heatmap && sudo docker run --rm -v "$PWD/bin:/app/bin:ro" -v "$PWD/secrets:/secrets" python:3.12-slim sh -c 'pip install -q pynintendoparental && python -u /app/bin/switch_auth.py probe'

`exophase_probe.py` does the same for their page, decoding the escaped JSON
and reporting the objects carrying playtime. Reach for it if the lifetime
ingest starts finding zero rows.

    python3 bin/exophase_probe.py https://www.exophase.com/nintendo/user/<your id>/

## Decisions worth knowing

**One ramp, source on hover.** A blended square is total minutes. Which
platform they came from lives in the hover card. Splitting by hue would put a
second signal in a palette that runs on one, and the question a heatmap
answers is how much, not where.

**Coverage, not a start date.** The payload carries which days each source was
actually reporting on. A day inside the window but outside every range is
unknown rather than empty, and the component draws it the fainter shade.

This matters because a console that stops syncing leaves a stretch nobody
reported on. Drawing that as zero-play would be a claim about you rather than
about the data. Coverage accumulates rather than being replaced, since
Nintendo's window slides and today's response no longer mentions days recorded
weeks ago.

**Per-title minutes are summed rather than the device figure used.** A daily
summary carries a device total that runs higher than the games beneath it,
because it counts menus and the home screen. Steam counts time in a game, so
this does too. Expect the map to read slightly lower than the phone app.

**A lagging lifetime figure falls back to the recorded one.** Exophase syncs
on its own schedule, so the titles whose lifetime figure trails are the ones
being played right now, which are exactly the ones the card panel shows. A
lifetime total is at least the last week's play, so the larger of the two is
never an overstatement. It can understate until their next sync, and it
corrects rather than accumulating.

**Chat data is never read.** Each player entry carries `chatHistory` and
`firstChattedFriendshipIds`. The parser touches neither, and the test suite
fails if anything from them reaches an output file.

**Two push targets.** The heatmap payload and the card snapshot both go to the
`play-data` repo, which is public so the site can fetch unauthenticated. CI
pulls the card snapshot into `src/data/` before the art refresh step. Nothing
commits to the portfolio repo, so a token compromise cannot reach the site.
Both pushes short-circuit on an unchanged blob, and the card file reuses its
previous `generatedAt` when nothing else moved, so a quiet week costs no
commits.

**Why not nxapi.** It was the obvious choice and it does not work. Nintendo
moved the Parental Controls API to `app.lp1.znma.srv.nintendo.net/v2` and
nxapi still calls the v1 `moon` endpoints, so every request comes back
`update_required`. `pynintendoparental` followed them across and tracks the
app version deliberately.

## What this cannot do

Steam cannot be recovered before its first poll. Valve keeps only lifetime
totals and a last-played date, with no session logs, so the GDPR export
doesn't help and neither does any third party. Achievement timestamps mark
moments without durations, and Steam Replay is monthly. Neither belongs on a
map measured in minutes.

Steam playtime lands on the day the change was observed. A Deck session played
offline for a week lands entirely on the day the Deck next syncs, as one very
dark square. Nintendo does not have this problem.

Nintendo reports per device across every console linked to the account, and
sums every player on them. Set `SWITCH_ONLY_PLAYER` to your nickname to
filter.

Today's Switch figure is provisional while `result` reads CALCULATING. It
settles on its own and the next ingest overwrites it.

Playtime everywhere is wall-clock with the software open, so a paused game
counts the same as one being played.

The site is up to six hours behind, since both surfaces update on the
scheduled build rather than on a fetch.

## If it stops working

`update_required` from Nintendo means the app version moved again. Set
`SWITCH_APP_VERSION` and `SWITCH_APP_BUILD` in `.env` to whatever the Parental
Controls app on your phone reports. If that does not clear it, the library
needs an update.

`InvalidSessionTokenException` means the Nintendo token was revoked, which a
password change does. Rerun step 5. Nothing on the site will show this, so if
Switch data goes stale this is the first thing to check.

`exophase: no Nintendo rows parsed` means their page shape moved. Run
`exophase_probe.py` and correct the field map. Existing lifetime values are
left alone rather than wiped.

Steam failures are logged and skipped. A failed poll is a gap, not a stop, and
the next one reads the same cumulative totals.

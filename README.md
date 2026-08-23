# Play heatmap

One collector on the Synology, feeding two surfaces on jessevaughan.com.

The heatmap is a GitHub-contribution-style map of daily playtime across Steam
and Nintendo Switch. The card panel on /now is `RecentGames.astro`, which reads
Steam from a committed snapshot and gets a matching Switch file from here.

## How the two sources differ

Steam returns a lifetime total per game and never a daily history, so its
series is built by polling every fifteen minutes and diffing. A session
crossing midnight is misattributed by at most one poll interval. Nothing
before the first poll is recoverable, ever.

Nintendo returns real calendar days through the Parental Controls API, so
those are taken as authoritative. A day is replaced rather than appended to on
every ingest, which makes reruns idempotent and lets today correct itself once
`result` moves from CALCULATING to ACHIEVED.

Set the Switch to the same timezone as `COLLECT_TZ`. Nintendo reports in the
console's own offset and the collector does not second-guess it.

## Files

    bin/store.py           schema and shared helpers
    bin/switch_auth.py     one-time login, and a probe for inspecting the API
    bin/switch_ingest.py   pulls Nintendo's daily records into the database
    bin/collector.py       Steam polling, both exports, both pushes
    docker-compose.yml     one service
    .env.example           copy to .env on the NAS
    PlayHeatmap.astro      the heatmap component, new to the site

## Setup

1. Make the directories.

       mkdir -p /volume2/docker/play-heatmap/{bin,data,secrets} && chmod 700 /volume2/docker/play-heatmap/secrets

2. Copy `bin/*.py` into `bin/`, and `docker-compose.yml` plus `.env.example`
   alongside them.

3. Create the env file and fill in the Steam and GitHub values.

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

6. Run one poll by hand.

       cd /volume2/docker/play-heatmap && sudo docker run --rm --env-file .env -e DB_PATH=/data/play.db -e JSON_PATH=/data/heatmap.json -e SWITCH_TOKEN_PATH=/secrets/switch-token.json -e RECENT_JSON_PATH=/data/switch-recent.json -v "$PWD/bin:/app/bin:ro" -v "$PWD/data:/data" -v "$PWD/secrets:/secrets" python:3.12-slim sh -c 'pip install --no-cache-dir -q pynintendoparental && python -u /app/bin/collector.py --once'

   You want `baseline recorded for N games`, then `switch: N days ingested`
   with a date range, then `poll ok`. The heatmap being empty at this point is
   correct, since Steam has only laid down its baseline.

7. Start it.

       cd /volume2/docker/play-heatmap && sudo docker compose up -d && sudo docker logs -f play-collector

8. Copy `data/switch-recent.json` over the file at `src/data/switch-recent.json`
   in the portfolio repo and commit. If the shape disagrees with what
   `RecentGames.astro` expects, `RECENT_FIELDS` at the top of `collector.py`
   is the one block to change.

9. Drop `PlayHeatmap.astro` into the components directory, point `DATA_URL` at
   the raw URL for your data repo, swap the token fallbacks in the style block
   for the real semantic tokens, and place it on the page.

## Inspecting the API

`switch_auth.py probe` reports what Nintendo actually returns: the date range
held, the mix of `result` values, raw minute figures for the busiest days, and
the full shape of `players[].playedGames[]`. Reach for it whenever a field
seems to have moved.

       cd /volume2/docker/play-heatmap && sudo docker run --rm -v "$PWD/bin:/app/bin:ro" -v "$PWD/secrets:/secrets" python:3.12-slim sh -c 'pip install -q pynintendoparental && python -u /app/bin/switch_auth.py probe'

## Decisions worth knowing

**One ramp, source on hover.** A blended square is total minutes. Which
platform they came from lives in the hover title. Splitting by hue would put a
second signal in a palette that runs on one, and the question a heatmap
answers is how much, not where.

**Both sources start the day the collector first runs.** Nintendo hands back a
window of earlier days and those are stored but not drawn. Otherwise the early
weeks would show Switch-only squares reading as months of not touching Steam,
which is a claim about you rather than about the data.

Filtering at render rather than on ingest is the part worth knowing. Nintendo
keeps 31 days and no more, so a day dropped on ingest is gone permanently,
while a day stored and hidden costs a few kilobytes and leaves the choice
reversible. To draw the backfill after all, change `grid_start`.

**Per-title minutes are summed rather than the device figure used.** A daily
summary carries a device total that runs higher than the games beneath it,
because it counts menus and the home screen. Steam counts time in a game, so
this does too. Expect the map to read slightly lower than the phone app.

**The card totals say recorded, not all time.** Nintendo reports only what its
own records cover. The collector sums everything stored, including days the
heatmap hides, so the card figure is as complete as Nintendo allows.

**Chat data is never read.** Each player entry carries `chatHistory` and
`firstChattedFriendshipIds`. The parser does not touch either, and the test
suite fails if anything from them reaches an output file.

**Two push targets.** The heatmap payload goes to the data repo and is fetched
at build time. The card snapshot goes to the portfolio repo, because that is
where `RecentGames.astro` reads it. Leave `RECENT_REPO` empty and the NAS
writes it locally for you to commit by hand, worth doing for the first week
before letting it commit on its own. Both pushes short-circuit on an unchanged
blob, and the card file reuses its previous `generatedAt` when nothing else
moved, so a quiet week costs no commits.

**Why not nxapi.** It was the obvious choice and it does not work. Nintendo
moved the Parental Controls API to `app.lp1.znma.srv.nintendo.net/v2` and
nxapi is still calling the v1 `moon` endpoints, so every request comes back
`update_required`. `pynintendoparental` followed them across and tracks the
app version deliberately.

**Why not the friend-code route.** That is what Exophase uses. It gives
cumulative totals for the last twenty games rather than days, and needs a
third-party service to generate the f-parameter Nintendo's login expects.

## What this cannot do

Steam cannot be recovered before its first poll.

Steam playtime lands on the day the change was observed. A Deck session played
offline for a week lands entirely on the day the Deck next syncs, as one very
dark square. Nintendo does not have this problem.

Nintendo reports per device across every console linked to the account, and
sums every player on them. Set `SWITCH_ONLY_PLAYER` to your nickname to
filter.

Today's Switch figure is provisional while `result` reads CALCULATING. It
settles on its own and the next ingest overwrites it.

Retro titles played through Nintendo Switch Online may collapse into a single
entry. That affects names, not minutes.

Playtime on both sides is wall-clock with the software open, so a paused game
counts the same as one being played.

The heatmap is up to six hours behind, since it updates on the site's
scheduled build rather than on a fetch.

## If it stops working

A `update_required` from Nintendo means the app version moved again. Set
`SWITCH_APP_VERSION` and `SWITCH_APP_BUILD` in `.env` to whatever the
Parental Controls app on your phone reports, which overrides what the library
ships. If that does not clear it, the library needs an update.

An `InvalidSessionTokenException` means the token was revoked, which a
password change does. Rerun step 5.

Steam failures are logged and skipped. A failed poll is a gap, not a stop, and
the next one reads the same cumulative totals.

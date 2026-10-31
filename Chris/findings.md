# Album Jukebox: findings

Orientation review of the Jukebox feature, written 2026-10-31.

- **Scope:** read-only code review. No application code was changed.
- **Sources read:** `HANDOVER.md`, `AGENTS.md`, `DESIGN.md`, the Jukebox spec / plan / differential review in `docs/superpowers/`, and all the code listed in section 3.
- **Tests:** not run. The Python and Node dependencies are not installed in this environment (`pytest` and `node_modules` are missing). The test status quoted in section 7 comes from the earlier review document, not from me.

---

## 1. What the Jukebox is

The Album Jukebox is a card on the **Overview page**. It picks a whole album for the user to listen to. It exists to cut down choice paralysis and to push listeners away from shuffled single tracks.

Two rules shape it:

1. **Lucky Dip.** It picks one album at random from the user's catalogue, optionally narrowed by filters.
2. **Commitment rule.** After the first Lucky Dip, the user gets 3 re-rolls. When they run out, the controls lock. Pressing **Open in Spotify** commits the session and resets the count.

It does no in-app playback. The only exit is a link out to Spotify.

It has two skins sharing one engine:

| Skin | Component | Notes |
|---|---|---|
| **Flat** (default) | `FlatJukebox.jsx` | Follows `DESIGN.md`. Uses CSS tokens such as `--color-primary` and `--color-surface`. |
| **Arcade / Retro** | `RetroJukebox.jsx` + `RetroJukebox.module.css` | Deliberately skeuomorphic. Pastels, shadows, gradients, marquee and a spinning vinyl. **Protected:** `AGENTS.md` and `DESIGN.md` forbid restyling it during theme or accent-colour work. |

The choice of skin is stored in `localStorage` under `jukebox_theme_mode`, so it is per browser and not per account.

The feature came from a spec the maintainer's brother wrote. Recent history (newest first): `05a23b3` settings section, `16538b0` source selector, `a5878d7` dark-mode fix, `f1368ae` filters, `f7c4ecd` nginx route fix, `75f2573` protect retro styling, `20e27ae` initial feature. The frontend `package.json` is at v0.29.0.

---

## 2. How it works end to end

```
Spotify  --(OAuth scope: user-library-read)-->  worker (APScheduler)
   |  /v1/me/albums (50 per page) + /v1/artists (genres)
   v
worker/spotify_ingestion.py :: sync_saved_albums_for_user
   |  POST /spotify/internal/saved-albums   (X-Worker-Token)
   v
backend/app/api/spotify_library.py -> services/spotify_library_service.upsert_saved_albums
   v
saved_albums table (Postgres)
   ^
   |  GET /jukebox/albums, GET /jukebox/daily, POST /jukebox/record-pick   (JWT bearer)
backend/app/api/jukebox.py
   ^
   |  fetchJukeboxAlbums / recordJukeboxPick   (frontend/src/api.js)
JukeboxCard.jsx -> useJukeboxEngine.js -> FlatJukebox | RetroJukebox (+ JukeboxFilters)
   ^
OverviewPage.jsx
```

### Data flow in words

1. **Sync.** Every `WORKER_LIKED_TRACKS_INTERVAL_MINUTES` (default 30) the worker job `saved-albums-ingestion` runs. For each active user with `saved_albums_sync_enabled`, it fetches **one page** of up to 50 saved albums. It then looks up the primary artists' genres in one batch call and posts the normalised albums to the backend.
2. **Walk logic.** `saved_albums_backfill_offset` tracks progress through the library. When a page comes back short, the offset resets to `None` and `saved_albums_last_synced_at` is set. A new full walk will not start until 24 hours after that.
3. **Store.** The backend upserts into `saved_albums`, keyed on `(user_id, spotify_album_id)`.
4. **Serve.** `GET /jukebox/albums` returns the user's whole catalogue in one lightweight payload. Each album carries a `play_count`, worked out by matching album and artist names against the user's `listening_events`.
5. **Pick.** The browser does everything else. Filtering, the 25% pool maths and the random pick all run client-side in `useJukeboxEngine`.
6. **Commit.** Clicking **Open in Spotify** resets the session and calls `POST /jukebox/record-pick`. That sets `last_picked_at` on the album, fire-and-forget.

### Fallback source: `library_archive`

`GET /jukebox/albums` switches to the user's top 100 most-scrobbled albums, grouped from `listening_events`, in two cases:

- the user set `jukebox_source = library_archive` in Settings, or
- the user has no saved albums at all.

These rows have no year, genres, duration or Spotify URL, so the year, decade, genre and length filters can do little with them. The Spotify link falls back to a Spotify search URL.

---

## 3. File map

### Backend (`backend/`)

| File | Role |
|---|---|
| `app/api/jukebox.py` | The three user-facing routes (`/jukebox/albums`, `/jukebox/daily`, `/jukebox/record-pick`). Registered in `app/main.py`. |
| `app/api/spotify_library.py` | `POST /spotify/internal/saved-albums`, the worker-token-protected ingestion route. |
| `app/services/spotify_library_service.py` | `upsert_saved_albums`. |
| `app/schemas/spotify_library.py` | `WorkerSavedAlbumItem`, `WorkerSavedAlbumsSyncRequest`, `SavedAlbumsUpsertResponse`. |
| `app/models.py` | `SavedAlbum` model. New `UserScrobbleSettings` columns: `saved_albums_sync_enabled`, `saved_albums_backfill_offset`, `saved_albums_last_synced_at`. New `UserPreferences.jukebox_source` with a CHECK constraint. |
| `app/services/preferences_service.py`, `app/schemas/preferences.py` | Validate and expose `jukebox_source` (`spotify_saved` or `library_archive`). |
| `migrations/versions/0028_add_saved_albums_and_jukebox_settings.py` | Creates the table, adds the columns and the constraint. Has a downgrade. |
| `app/tests/test_jukebox_api.py`, `test_saved_albums_model.py` | Backend tests. |

### Worker (`worker/`)

| File | Role |
|---|---|
| `spotify_ingestion.py` | `SpotifyClient.saved_albums`, `normalize_saved_album_item`, `submit_saved_albums_with_retries`, `sync_saved_albums_for_user`. |
| `app.py` | `run_saved_albums_ingestion` and the scheduler job `saved-albums-ingestion`. Skips while Spotify is rate-limited. |
| `tests/test_saved_albums_sync.py` | Worker tests. |

### Frontend (`frontend/src/`)

| File | Role |
|---|---|
| `components/jukebox/JukeboxCard.jsx` | Container. Fetches albums, owns the skin toggle, calls `recordJukeboxPick`. |
| `components/jukebox/useJukeboxEngine.js` | All logic: filters, pool, roll counter, session state. |
| `components/jukebox/FlatJukebox.jsx` | Flat skin. Uses inline styles. |
| `components/jukebox/RetroJukebox.jsx` and `.module.css` | Arcade skin. **Do not restyle.** |
| `components/jukebox/JukeboxFilters.jsx` | Collapsible filter panel. Shared by both skins. |
| `components/jukebox/*.test.*` | Four test files covering the card, both skins and the engine. |
| `api.js` (around line 567) | `fetchJukeboxAlbums`, `recordJukeboxPick`. |
| `pages/OverviewPage.jsx` (line 70) | Mounts `<JukeboxCard>` directly under `<RediscoverCard>`. |
| `pages/SettingsPage.jsx` (around line 483) | "Album Jukebox" section with the catalogue-source dropdown. |
| `changelogData.js` | Changelog entry. |

### Infrastructure

- `frontend/nginx.conf` (line 22) and `frontend/vite.config.js` (line 65) both proxy `/jukebox` to the backend. A new top-level route must be added to **both**. The initial feature missed this, and commit `f7c4ecd` fixed it.

---

## 4. How it connects to the rest of the app

| Area | Connection |
|---|---|
| **Overview page** | Hosts the card. It renders only once the summary has loaded. |
| **Rediscover card** | Sits next to it. Same page, same "bring back older music" theme, but no shared code. |
| **Spotify OAuth** | The `user-library-read` scope is already requested in `config.py`. No new scope is needed. |
| **Worker pattern** | Copies the liked-tracks sync: one page per tick, a backfill offset, an internal worker-token route, retry with backoff, and a rate-limit check. This follows the rule that Spotify is never called inside a web request. |
| **Scrobble data** | `play_count` comes from `listening_events`, matched on album name and artist name. The archive fallback is built entirely from `listening_events`. |
| **Genres** | Looked up at sync time from Spotify's `/v1/artists`. Albums have no genres of their own, so each album gets its primary artist's genres. This does **not** use the existing `genre_cache` table, although the spec suggested it might. |
| **Preferences / Settings** | The `jukebox_source` column lives on `user_preferences` and is edited through the generic settings save. |
| **Auth** | The user routes need a JWT (`get_current_user`). The ingestion route needs the worker token. Every query is filtered by `user_id`. |
| **Design system** | The Flat skin uses the theme tokens. The Retro skin is an explicit exception in `AGENTS.md` and `DESIGN.md`. |
| **Changelog / guides** | There is a changelog entry. I found no mention in `UserGuideSection.jsx` or `FaqSection.jsx`, and none in `agents/page_specs.md` or `HANDOVER.md`. `AGENTS.md` expects user-facing features to be added to the guide and FAQ. |

---

## 5. Engine behaviour (`useJukeboxEngine.js`)

- **Filters.** Artist is a case-insensitive "contains" match. Decade, genre and length are multi-select, OR within a group and AND across groups.
  - Length buckets: under 35 min, 35 to 50 min, over 50 min.
  - An album with no `duration_ms` counts as 0 minutes, so it always lands in "Short".
- **Modes.** `any` uses the whole filtered set. `least`, `most` and `recent` sort and then keep the top 25% (at least one album).
- **Rolls.** The first Lucky Dip starts a session and sets `rollsLeft = 3` **without spending one**. Each later Lucky Dip or Re-roll then spends one. A session therefore allows **four picks in total** (the first pick plus three re-rolls). This matches the wording "initializes a session with `rollsLeft = 3`", though it could be read the other way.
- **No immediate repeats.** If the pool has more than one album, the next pick excludes the current one.
- **Commit.** Clicking the Spotify link resets the session and rolls.

---

## 6. Gaps and risks found

The spec (`docs/superpowers/specs/2026-10-30-album-jukebox-design.md`) is still marked "Draft / Pending User Approval". Several things it describes are not in the code, and a few things in the code look like bugs. In rough order of importance:

### Spec features not implemented

1. **Album of the Day is never shown.** The backend has `GET /jukebox/daily`, but `api.js` has no function for it and no component calls it. On load the card shows "No Album Picked Yet" under the heading "Today's Featured Pick". The spec says the initial view should be the daily album.
2. **Roll state is not persisted.** The spec says session state lives in `sessionStorage` so a refresh does not reset the roll limit. The engine uses plain React state only, so refreshing the page gives a fresh set of rolls. (`JukeboxCard.test.jsx` clears `sessionStorage`, but nothing in the code uses it.)
3. **"Added date" filter does nothing.** `useJukeboxEngine` has an `added` filter set, but there is no UI for it and no filtering logic.
4. **No empty-filter state.** The spec asks for a "Clear filters" prompt when nothing matches. The buttons just disable. A "Clear all filters" link does appear whenever any filter is active.
5. **Retro skin is simplified.** The spec describes a full port with a claw, indicator lamps and a tone arm. The component has a marquee, a vinyl and a Lucky Dip button, and no claw or lamps.
6. **Mode is not stored.** `record-pick` accepts `mode`, and the frontend sends it, but the backend ignores it.

### Likely bugs

7. **Archive-mode picks can be recorded against the wrong album.** In `library_archive` mode the album ids are made up (`idx + 1`). `record-pick` looks up `SavedAlbum.id == album_id` for that user. A user who also has saved albums could have `last_picked_at` set on an unrelated saved album. Archive picks otherwise record nothing.
8. **Daily pick is unstable and uses UTC.** The seed is the UTC date, not the local calendar day the spec asks for. The pick also changes whenever the catalogue changes size (`seed % len`), so a new sync can change "today's" album mid-day.
9. **Play-count matching is fragile.** Counts are fetched with a case-sensitive `IN (album names)` but then looked up lower-cased. Names that differ only by case, and albums sharing a name by different artists, can be missed or merged. Names that differ in punctuation or edition ("Deluxe") will show 0 plays, which also skews the Least Played and Most Played modes.
10. **Synthetic ids repeat the same problem for the daily pick.** In archive mode the ids are positions in a top-100 list, so they shift as plays accumulate.
11. **Albums removed from Spotify are never deleted.** The sync only inserts and updates. If someone un-saves an album, it stays in the Jukebox forever.
12. **Duration may be understated.** The duration is summed from the `tracks.items` in the saved-albums response. Spotify embeds only a limited first page of tracks, so very long albums can be undercounted and land in the wrong length bucket.
13. **Sync is on by default for every user** (`saved_albums_sync_enabled` defaults to true), unlike liked-tracks sync which is opt-in. It is throttled (one page per 30 minutes per user, then a 24-hour pause), but it is a new background Spotify load, and the first walk of a big library takes many ticks. This is the area the handover warns about after the 2026-09 rate-limit block.
14. **The genre lookup can fail silently.** On error it logs a warning and stores albums with empty genres. Because a completed walk then waits 24 hours, those albums keep no genres until the next walk. Albums that are already stored are overwritten with empty genres if a later walk fails the same lookup.
15. **Unused code.** `previousPick` is set but never read, and `FlatJukebox` and `RetroJukebox` accept `busy` but `setBusy` is never called.

### Convention / hygiene points

16. **Flat skin uses a lot of inline styles and hard-coded colours** (`#FFFFFF`, `#111111`, `#1DB954`, `#E11D48`). The rest of the app uses CSS classes and tokens. `DESIGN.md` says to use only approved colour tokens. The Spotify green and the dark vinyl are likely intentional but not documented.
17. **The Flat skin's spin uses a CSS transform transition.** Worth checking against the reduced-motion handling the app added in v0.11.10.
18. **Accessibility has not been audited.** The roll count is shown only in a button label. The result changes without any `aria-live` announcement. Emoji are used in button text.
19. **Documentation drift.** The spec says routes are `/api/jukebox/*` and the internal route is `/internal/users/{id}/saved-albums`. The real paths are `/jukebox/*` and `/spotify/internal/saved-albums`.
20. **Tests are thin on the backend.** `test_jukebox_api.py` uses a hand-rolled fake DB, so it only proves "returns 200 with a list". There is no test for the archive fallback, `record-pick`, `daily`, or user isolation, and none against real Postgres (the handover warns that SQLite-style tests hide Postgres failures). The worker sync has its own test file.
21. **Review claims to check.** `docs/superpowers/specs/2026-10-30-differential-review-jukebox.md` reports zero findings of any severity. Several of the points above (7, 9, 11) would normally show up in a review, so that "all clear" should not be taken as proof the feature is bug-free.

---

## 7. Test status (as reported, not re-run)

- The earlier review document says the full suite passed: 381 Python tests (13 skipped) and 319 frontend tests.
- Jukebox-specific tests exist in `backend/app/tests/`, `worker/tests/test_saved_albums_sync.py` and `frontend/src/components/jukebox/`.
- To re-run locally, use the commands in `AGENTS.md`:
  `PYTHONPATH=worker python -m pytest backend/app/tests worker/tests -q` and `cd frontend && npm run lint && npm test && npm run build`.

---

## 8. Things to bear in mind when working on this area

- **Read the protected-file rule first.** Never touch `RetroJukebox.jsx` or `RetroJukebox.module.css` during theme or accent changes.
- **Do not call Spotify from a request.** New data must come through the worker and a table.
- **Any new top-level route needs both `nginx.conf` and `vite.config.js` updated.**
- **Schema changes need a new numbered Alembic migration.** The next is after `0028_add_saved_albums`. Never edit 0028.
- **Per `AGENTS.md`,** a user-facing change also needs a `changelogData.js` entry, a `package.json` version bump, and guide and FAQ updates.
- **Process rule:** any fix goes through the planner agent and needs approval before implementation, then a differential review before commit.

---

## 9. Suggested starting points

If the aim is to make the feature match its own spec, the highest-value items are:

1. Wire up Album of the Day (item 1) and make it stable for the local day (item 8).
2. Persist roll state in `sessionStorage` (item 2).
3. Fix archive-mode `record-pick` (item 7) and the play-count matching (item 9).
4. Decide whether sync should be opt-in (item 13) and whether un-saved albums should be removed (item 11).
5. Add the missing user guide and FAQ entries, and backend tests for the archive and daily paths.

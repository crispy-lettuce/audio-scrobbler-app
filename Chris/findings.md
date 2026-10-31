# Album Jukebox: findings

Orientation review of the Jukebox feature, written 2026-10-31.

- **Scope:** read-only code review. No application code was changed.
- **Sources read:** `HANDOVER.md`, `AGENTS.md`, `DESIGN.md`, the Jukebox spec / plan / differential review in `docs/superpowers/`, and all the code listed in section 3.
- **Update (fifth pass):** section 14 investigates whether the Jukebox can be exempt from the design standards and live on its own tab. Short answer: yes. The rules are documentation plus one test on `styles.css`, and a dedicated page with CSS modules fits cleanly.
- **Update (fourth pass):** section 13 investigates building the curated list from a Spotify playlist of single tracks, by tracing each track back to its album. Verdict: feasible and the most efficient of the options. It also surfaced a possible problem with the existing genre lookup (finding 25).
- **Update (third pass):** section 12 investigates whether a user could curate their own album list instead of relying on Spotify or scrobbles. Short answer: not today, but the app has most of the building blocks.
- **Update (second pass):** the two original reference files (the brother's product spec and HTML mockup) have been added to `Chris/referencejukebox/` and compared against the implementation. See **sections 10 and 11**. That comparison also corrected a few statements in section 6, which are marked "(revised)".
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

The feature came from a product spec and a working HTML mockup the maintainer's brother wrote (now in `Chris/referencejukebox/`). The repo's own design doc (`docs/superpowers/specs/2026-10-30-album-jukebox-design.md`) adapted that original to this app's architecture, and the code was built from the adapted version. The Flat skin was added during that adaptation and is **not** in the original. Recent history (newest first): `05a23b3` settings section, `16538b0` source selector, `a5878d7` dark-mode fix, `f1368ae` filters, `f7c4ecd` nginx route fix, `75f2573` protect retro styling, `20e27ae` initial feature. The frontend `package.json` is at v0.29.0.

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
2. **Roll state is not persisted (revised).** The repo's design doc says session state lives in `sessionStorage` so a refresh does not reset the roll limit. The engine uses plain React state only, so refreshing the page gives a fresh set of rolls. (`JukeboxCard.test.jsx` clears `sessionStorage`, but nothing in the code uses it.) The **original** spec says the opposite: reloading resets the session (§7.5), and persistence is only an open question (Q1). So this is a gap against the repo's design doc, but the code matches the original spec.
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
18. **Accessibility is largely missing (revised).** A search of the jukebox components finds **no** `aria-*` or `role` attributes and no reduced-motion handling. The original spec lists accessibility as a requirement (§13); see the §13 row in section 10.2 for the detail.
19. **Documentation drift.** The spec says routes are `/api/jukebox/*` and the internal route is `/internal/users/{id}/saved-albums`. The real paths are `/jukebox/*` and `/spotify/internal/saved-albums`.
20. **Tests are thin on the backend.** `test_jukebox_api.py` uses a hand-rolled fake DB, so it only proves "returns 200 with a list". There is no test for the archive fallback, `record-pick`, `daily`, or user isolation, and none against real Postgres (the handover warns that SQLite-style tests hide Postgres failures). The worker sync has its own test file.
21. **Review claims to check.** `docs/superpowers/specs/2026-10-30-differential-review-jukebox.md` reports zero findings of any severity. Several of the points above (7, 9, 11) would normally show up in a review, so that "all clear" should not be taken as proof the feature is bug-free.

### Further bugs found during the comparison (by reading the code; not reproduced)

22. **"Open in Spotify" is live before anything is picked.** With no pick, `spotifyLink` is `"#"`, but the link still has `target="_blank"` and an `onClick` that commits the session. Clicking it opens a blank tab on the same page. The mockup never has this problem because the album of the day is always present, so the link is always valid. Affects both skins.
23. **Double-clicking Lucky Dip can bypass the roll limit.** `handleRoll` in both skins checks `busy` and `canRoll`, but not `spinning`. `busy` is never set to true anywhere, so the button stays enabled during the 450 ms (Flat) or 600 ms (Retro) delay. Two quick clicks each schedule a pick. Both closures were created while `sessionActive` was still false, so both take the "start a new session" path and reset the rolls to 3 instead of spending one. The original spec (§7.2) calls out exactly this loophole and says it must not be possible.
24. **The Jukebox ignores the user's blocks.** `AGENTS.md` says to exclude blocked items (`not_blocked_clause`), and the Library lets users block an artist, album or track (`blocked_items`). `backend/app/api/jukebox.py` never reads that table. In archive mode, an album or artist the user has blocked can still be picked, and in Spotify-saved mode a blocked album still appears. (Found during the curation investigation, section 12.)
25. **Album genres may be empty in production (needs checking).** The album sync looks up genres with Spotify's batch artist endpoint (`GET /v1/artists?ids=…`, `SpotifyClient.artists`), and the backend artwork backfill uses the batch track endpoint (`GET /v1/tracks?ids=…`). Search summaries of Spotify's February 2026 changelog say batch endpoints, including Get Several Artists and Get Several Tracks, were removed for **Development Mode** apps. I could not read Spotify's own pages from this environment, so this is unconfirmed (see 13.7). If the app is in Development Mode, the genre call fails, the sync catches the error and logs "Could not resolve artist genres during album sync", and albums are stored with **empty genres**, so the Genre filter has nothing to offer. Quick check: look for that log line in the worker logs, and look at whether any `saved_albums.genres` are non-empty.

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
6. Fix the two bugs from the comparison (items 22 and 23). They are small and independent of any redesign.

Sections 10 and 11 below cover the comparison with the original reference and the options for closing the gap.

---

## 10. Comparison with the original reference spec and mockup

**Reference files** (copied unchanged into `Chris/referencejukebox/`):

- `jukebox-spec.md`: the product / UX spec (15 sections).
- `jukebox-mockup.html`: a single-file, no-dependency working mockup with 40 dummy albums (about 1,080 lines).

### 10.1 Summary

The **logic** was ported faithfully. The **presentation, data model and polish** were not.

| Layer | Verdict |
|---|---|
| Selection logic (filters AND/OR, 25% pool, ceil with a minimum of 1, repeat guard, roll rules, FNV-1a hash seed) | **Matches** the mockup almost line for line (`useJukeboxEngine.js`). |
| Filters | **Partial.** 4 of the spec's 8 are in the UI. |
| Visuals (rack, lamps, claw, tone arm, grille, arch lights) | **Mostly missing.** The Retro skin has about a quarter of the mockup's elements. |
| Animation | **Missing.** Only a short vinyl spin remains. |
| Album of the day | **Backend only, unused by the UI.** |
| Result card (cover, last-picked meaning) | **Partial.** |
| Accessibility | **Largely missing.** |
| Data | **Different but workable.** Mood is missing, and "favourites" became "Spotify saved albums". |

### 10.2 Section-by-section against the spec

| Spec | Reference says | Implementation | Status |
|---|---|---|---|
| §2 Users | One person, no accounts. | Multi-user app. Everything is scoped by `user_id`. | **Adapted** (a necessary change). |
| §4 Layout | Cabinet with marquee, selector lamps, counter, record rack, turntable and claw, mode control and Lucky Dip. Separate result card and filters panel. Two columns with a sticky cabinet from 880 px. | Single card inside the Overview column. In Retro, the card and filters sit *inside* the cabinet. No rack, lamps or claw. No two-column layout. | **Gap** |
| §4 Rack feedback | Tiles dim, shrink and grey out as filters change. Pool-excluded tiles are half-dimmed. The current album is outlined. | No rack at all, so no live feedback. | **Missing** |
| §4 Counter | "14 records · picking from 4 least played" | Flat: "N records available (pool: N)". Retro: "N records match filters", with no pool figure. | **Partial** |
| §5 Filters | Artist (with suggestions), Album, Year, Genre, Mood, Length, Label, Added. | Artist (no suggestions), Decade, Genre, Length. **Missing: Album, Mood, Label, Added.** Genre chips are capped at the top 15. | **Gap** |
| §5 Lamps | One lamp per filter, lit when active; tapping jumps to that filter. | None. | **Missing** |
| §5 "Mode is not a filter" | Changing the mode never lights a lamp. | Mode counts towards "N active filters", and "Clear all filters" also resets the mode. | **Differs** |
| §6 Modes | Any / Least / Most / Recent, top or bottom 25%. | Same. The mode labels are shortened and carry emoji. | **Matches** |
| §7 Roll limit | 3 re-rolls. A mid-session Lucky Dip also spends a roll. Both disable at zero. Commit on Open in Spotify. Reload resets. Filters and mode changes don't reset. | Engine matches. The double-click bypass (item 23) breaks it in practice. The out-of-rolls message exists only in Flat, and its wording differs ("give it a spin on Spotify!"). | **Matches, with a bug** |
| §8 Album of the day | Shown on open, labelled "Album of the day · 30 Oct". Seeded by the **local** date. Persisted per date. Refreshes after midnight or when the tab regains focus. Ignores filters. Not a pick. | `GET /jukebox/daily` exists but nothing calls it. It uses the **UTC** date, is not persisted, and has no midnight refresh. The card says "Today's Featured Pick" but shows "No Album Picked Yet". | **Gap** |
| §9 Card | Cover art, title, artist, year, plays, last picked. Wraps long titles. | Text only. **`artwork_url` is fetched but never rendered in either skin.** Flat truncates long titles with an ellipsis. | **Gap** |
| §9 Last picked | Shows the value from **before** this pick. Written **at every reveal**, including re-rolls. | Written only when the user clicks Open in Spotify (the "commit"). Re-rolled albums are never recorded. The pre-pick value is computed (`previousPick`) and then never used. | **Differs** |
| §9 Spotify link | Use the stored URL if known, else a search. | Same. | **Matches** |
| §10 Animation | 7 steps, about 3.5 s: highlight, claw out, grip, carry, release, tone arm, spin, reveal with focus on the title. Reduced-motion skips it. Reveal must still happen if the animation fails. | A 450 ms (Flat) or 600 ms (Retro) delay, a vinyl spin, then an instant swap. No claw, no rack, no tone arm, no pending state, no focus move, no reduced-motion handling. | **Missing** |
| §11 Empty states | "No records match these filters" box with Clear filters. Buttons disabled. "Favourite some albums first" when the library is empty. | Buttons disable. No empty box. With no saved albums the app silently switches to the scrobble archive, so the "no favourites" case never appears. | **Gap** |
| §12 Data | id, artist, title, year, genre, **mood**, length, label, added date, plays, lastPicked, cover URL, Spotify URL. | All present except **mood**. Genre is a list (the artist's genres). | **Gap (mood only)** |
| §13 Accessibility | Real controls, visible focus, 44 px targets, `aria-pressed` chips, lamp labels, debounced `aria-live` counter, focus to the title after a reveal, decorative parts `aria-hidden`, reduced motion. | Real buttons, but **no ARIA at all**, chips about 24 px high, no live region, no focus management, no reduced-motion handling. | **Gap** |
| §14 Out of scope | No sound, history, sharing or in-app playback. | Respected. | **Matches** |

### 10.3 What the mockup does that the code does not (visual inventory)

Compared with `jukebox-mockup.html`, the Retro skin (`RetroJukebox.jsx` + `.module.css`, 124 lines of CSS) has only: the arched cabinet body, the marquee, a stage with turntable, platter and spinning vinyl, the Lucky Dip button and a simple card.

Missing: 13 arch bulbs, speaker grille, the 8 selector lamps, the record rack, the tone arm, the claw and its fingers, the flying record, generated cover art and initials on the label, the empty-state box, the separate result card with a pending state, and the segmented "Pick from" control as a labelled fieldset (the app uses chips inside the filters panel).

### 10.4 Where the implementation is *better* than the mockup

- Real data from Spotify, with real artwork URLs, Spotify links and per-user storage.
- Server-side `last_picked_at`, where the mockup only keeps it in memory.
- A worker-based, rate-limit-aware sync that follows the app's rules.
- The Flat skin, which respects `DESIGN.md` (the original has no equivalent).
- A skin choice that is remembered.
- Tests for the engine and components.

---

## 11. Can this app's architecture support an exact replica?

**Short answer: yes, with one hard data gap (mood) and a few places where the app's rules force a different approach.** Nothing in the stack (React 18 and plain CSS, FastAPI, the Postgres + worker pattern) prevents reproducing the mockup. The current implementation chose to build a simplified version, not because the architecture forced it.

### 11.1 Real constraints

| # | Constraint | Effect on an exact replica | How to work within it |
|---|---|---|---|
| 1 | **Mood has no data source.** Nothing in this app stores mood, and Spotify does not provide it in a way this app can rely on. (Spotify has restricted its audio-analysis endpoints for newer apps; this needs checking against current Spotify docs before relying on it either way.) The repo's own design doc already lists mood as a non-goal. | The Mood filter and lamp cannot work as in the mockup. | (a) Drop Mood, as now. (b) Add a small table such as `album_moods (user_id, spotify_album_id, mood)` plus a tagging control. This needs a new numbered migration and goes against the original spec's "no editing metadata" rule (§14), so it is a product decision. (c) Derive a coarse mood from genre with a fixed mapping. Cheap but crude. This is the spec's own open question 3. |
| 2 | **Design system bans gradients and box-shadows** (`AGENTS.md`, `DESIGN.md`). The mockup relies heavily on both: the cabinet's offset shadow, lamp glow, the vinyl grooves (`repeating-radial-gradient`), rack stripes and generated gradient covers. | The mockup's look can only exist in the **Retro skin**. It can never be the default look. | This is exactly why the dual skin exists, and the Retro skin is already a documented exception. Put all the mockup styling in `RetroJukebox.module.css`. A flat version can still get the rack, lamps and animation in a flat style (solid colours, borders, no glow). |
| 3 | **`/jukebox` is reserved for the API.** `frontend/nginx.conf` proxies any path starting `/jukebox` to the backend. The file's own comment explains that `/library` and `/reports` needed special handling because the same collision broke page refreshes. | A dedicated page at `/jukebox` would work in-app but return an API error on refresh or a direct link. | Use a different page path (for example `/album-jukebox`; anything that doesn't start with `/jukebox`), or add the same special-case block the app uses for `/library` and `/reports`. `vite.config.js` would need the matching rule. |
| 4 | **Layout.** The mockup is a full page with a sticky two-column layout at 880 px and above. The app mounts the Jukebox as one card in the Overview column. | The sticky cabinet, a 10-column rack and the claw's travel need much more room than a card gives. | See the options in 11.3. |
| 5 | **Protected files.** `AGENTS.md` forbids editing `RetroJukebox.jsx` and `.module.css` during theme work. | Replicating the mockup *means* rewriting them. | This is a deliberate feature change, not a theme change, so it is allowed, but it needs explicit sign-off and a plan (the repo's process requires one). |
| 6 | **Play counts have no album ID.** `listening_events` stores only album *names*. | Counts depend on name matching (see bugs 9 and 10), so "Least / Most played" can be wrong. | Compute counts in the worker and cache them (for example a `play_count` on `saved_albums`), as the app already does for genres. Or add an `album_spotify_id` column to `listening_events` and backfill it from `raw_metadata`. Either needs a migration. |
| 7 | **"Favourites" don't exist as albums.** The spec assumes a ready list of favourited albums "from any source". The app has liked *tracks* and imported history, but no favourite-album concept. | The Jukebox can only draw from Spotify saved albums, or from the top-100 scrobble archive. Apple Music, YouTube and file imports are reachable only through the archive. | Alternatives: derive albums from `liked_tracks`, add a "favourite album" action in the Library, or widen the archive beyond 100 (it is capped at 100 and has no year, genre or length). |
| 8 | **Real covers at rack size.** The sync stores only the largest image (`images[0]`, 640 px). | A rack of dozens or hundreds of tiny tiles would download 640 px images for 30 px squares. | Store a smaller image (Spotify supplies 300 px and 64 px) in the worker, or use the existing artwork route, then backfill. |
| 9 | **Rack size at real library sizes.** The mockup shows 40 albums. A saved library may hold hundreds. | A tile per album would be unreadable and slow. | Cap the rack (for example 40 to 60 tiles drawn from the matching set), keep the true count in the counter, and mark the rack `aria-hidden` as the spec already does. The dim / half-dim / outline states still work for the tiles shown. |
| 10 | **React 18 and jsdom.** Tests run in jsdom, which does not implement `element.animate()` (used by the mockup's claw). React 18 also does not support the `inert` prop as a boolean (the mockup uses `card.inert = busy`). | Direct ports of those two lines would break tests or be ignored. | Guard the animation behind a feature check (which also gives the reduced-motion path), and set `inert=""` through a ref. Both are small. |
| 11 | **Pick-before-animate ordering.** The mockup chooses the album first, animates to it, then reveals. The current engine picks only after the delay. | The claw has nothing to travel to. | Reorder: `luckyDip` returns the chosen album immediately, the skin animates, then calls a `reveal()` that commits state and writes `last_picked_at`. This also gives `busy` a real job and fixes bug 23. |

### 11.2 Things the architecture handles easily

- **All the selection maths:** already in `useJukeboxEngine`, client-side, matching the spec.
- **Missing filters (Album, Label, Added):** the data is already in the API payload (`label` and `added_at` are returned). It is a UI and engine change only.
- **Album of the day:** needs no new infrastructure. Either keep the client-side approach in the mockup (`localStorage` keyed by local date, which the app already uses for the skin choice) or send the local date to the endpoint (`/jukebox/daily?date=YYYY-MM-DD`) so the server stops using UTC.
- **Real covers on the card and on the platter label:** `artwork_url` is already delivered.
- **Accessibility:** purely front-end.
- **Plain CSS port:** the app doesn't use Tailwind, and the mockup is plain CSS with variables, so it moves into a CSS module with little change.

### 11.3 Alternative ways to implement it

Each option respects the existing rules (worker for Spotify calls, JWT-protected user routes, `DESIGN.md` for the default skin).

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. Keep the Overview card, fill in the gaps** | Fix bugs 22 and 23, wire up album of the day, add the missing filters, add accessibility and the empty states. Leave the Retro skin simple. | Smallest change. Lowest risk. | Not close to the mockup. No rack or claw. |
| **B. Full replica in the Retro skin, on the Overview card** | Rebuild `RetroJukebox` to the mockup (rack, lamps, claw, tone arm). Keep Flat as the default. | Faithful look. Keeps the existing placement. | A card-width cabinet is cramped: no two-column layout, so the 10-column rack and claw get small. |
| **C. Dedicated page plus a compact Overview teaser** (recommended if the goal is a faithful replica) | New page at a non-`/jukebox` path (constraint 3), with the full two-column mockup layout. Overview keeps a small card (album of the day and a link). The skin toggle applies on the page. | The only option that reproduces the sticky two-column layout. The Overview stays light. Fits how other features (Set Lists, Reports) have their own pages. | Needs a route, a nav entry, `page_specs.md` updates and an nginx / Vite check. |
| **D. Flat-style replica as the default** | Rebuild the rack, lamps and animation using only flat, token-based styling (solid fills, borders, no glow). Keep Retro as an opt-in with the full mockup look. | Gets most of the spec's behaviour into the default view without breaking `DESIGN.md`. | Two visual implementations to maintain. More work. |
| **E. Retro only** | Drop the Flat skin. | One implementation. Exact look. | Breaks the "flat by default" rule, so it needs a recorded design-system exception. |

**Suggested path:** A first (it fixes real bugs and spec gaps cheaply), then C with the Retro skin as the full replica, with D as a later stretch. The engine is already shared and correct, so the new work is almost entirely presentation, plus the data items in constraints 6 to 8.

### 11.4 Decisions needed before any build

1. Mood: drop it, tag it by hand, or derive it? (Spec open question 3.)
2. Placement: stay on Overview, or get a dedicated page? (Option B versus C.)
3. Should "last picked" be written at every reveal (spec) or only on commit (current code)? (Spec open question 2.)
4. Should roll state survive a reload? The original spec says no; the repo's design doc says yes. (Spec open question 1.)
5. Where should "favourites" come from for non-Spotify listeners?
6. Is a rack of a capped number of tiles acceptable for large libraries?

Any of these would go through the planner agent for approval first, as `AGENTS.md` requires.

---

## 12. Investigation: can a user manually curate their own album list?

**Question:** instead of relying on Spotify saved albums or the scrobble archive, could a user build the Jukebox catalogue by hand?

**Short answer:**

- **Today: no.** There is no way for a user to add, remove or edit a Jukebox album.
- **Feasible: yes.** The app already has the tables, patterns and screens to support it. It needs one migration, a few user-facing routes and some UI. No architectural rule stands in the way, though a few rules shape how the work should be done.

### 12.1 What exists today (evidence)

| Question | Finding |
|---|---|
| Can a user write to `saved_albums`? | **No.** The only code that writes to it is `upsert_saved_albums`, reached through `POST /spotify/internal/saved-albums`, which needs the worker token. `backend/app/api/jukebox.py` has only three routes: `GET /albums`, `GET /daily` and `POST /record-pick`. `record-pick` only updates `last_picked_at` on an existing row. |
| Is there a way to tell a hand-made row from a synced one? | **No.** `saved_albums` has no `source`, `is_manual` or similar column. |
| Can the user choose a "manual" source? | **No.** `jukebox_source` is limited to `spotify_saved` or `library_archive`, both by a database CHECK constraint (migration 0028) and by validation in `preferences_service.py`. The Settings dropdown has two options. |
| Does the catalogue ever fall back silently? | **Yes.** If the user has no saved albums, the API switches to the archive whatever the setting says. An empty curated list would be swapped for scrobbles unless that rule is changed. |
| Are there lists, tags or "favourites" for albums? | **No.** No table models a user-built album list. `liked_tracks` is a read-only mirror of Spotify likes, and `LikedHeart` in the Library only displays state. |
| Can the user save an album to Spotify from this app? | **No.** The requested Spotify scopes include `user-library-read` but not `user-library-modify`. |
| Can the user hide an album from the Jukebox? | **Not directly.** The Library's Block action is the nearest thing, but the Jukebox ignores blocks (finding 24). |

### 12.2 What partly works today

1. **Curate inside Spotify.** Saving an album in Spotify makes the worker sync add it to the Jukebox within a day or so. This is "manual curation" in effect, but it only works one way: the sync never deletes (finding 11), so un-saving an album in Spotify does not remove it. Albums that are not on Spotify cannot be added at all.
2. **Treat the archive as a shortlist.** Not practical. The archive is the user's top 100 by plays and cannot be edited.
3. **Insert rows directly into the database.** Technically possible, because a hand-written row with a made-up `spotify_album_id` would survive the sync (the sync only matches on real Spotify IDs). It is not a supported route: there is no UI, no validation and no way to tell the rows apart later. It should not be recommended to users.

### 12.3 Building blocks the app already has

| Need | Existing piece | Where |
|---|---|---|
| Pick albums you already listen to | Library "Albums scrobbled" list. Each entry carries album name, artist, artwork and play count, and the list already supports a selection mode with a bulk action bar. | `LibraryRankList.jsx`, `LibraryPage.jsx` |
| Per-item actions menu | `EntryMenu` already offers Delete / Block / Clear Artwork on each Library entry. An "Add to Jukebox" item would sit naturally here. | `EntryMenu.jsx` |
| Manual data entry with validation and duplicate checks | `POST /setlists/manual` (gigs typed in by hand). A good model for the route, schema and errors. | `api/setlists.py`, `setlist_attendance_service.py` |
| Look up album metadata without Spotify | Deezer and iTunes search helpers, already used by imports and artwork backfill. | `processed_import_service.py`, `setlist_artwork_service.py` |
| Accept pasted Spotify links or IDs | `normalize_spotify_uri` and `_extract_playlist_id`, plus the Rediscover feature's raw-ID handling. | `spotify_playlist_service.py` |
| Enrich records later, off the request path | The worker enrichment pattern (genre cache: pending list, worker job, cache table). The worker can also get a client-credentials token. | `worker/app.py`, `spotify_ingestion.py` |
| Read a Spotify playlist | `get_playlist_info`, `get_user_playlists`, with the playlist-read scopes already granted. | `spotify_playlist_service.py` |
| Per-user settings | The "Album Jukebox" section already exists in Settings. | `SettingsPage.jsx` |

### 12.4 Data-model options

All options need a **new** Alembic migration (0029 or later). Migration 0028 is already applied and must not be edited.

| Option | Description | Pros | Cons |
|---|---|---|---|
| **A. Add `source` to `saved_albums`** (recommended) | New column, for example `source` = `spotify` or `manual`. Manual rows use a made-up ID such as `manual-<uuid>` (the column is a required string, and the unique key is `(user_id, spotify_album_id)`). | Smallest change. The API, engine and both skins already read this table. Synced and hand-made albums can mix. | The sync's upsert must not overwrite manual edits. The sync must never delete manual rows when removal support is added. |
| **B. Separate `jukebox_albums` table** | A dedicated curated table, with a reference back to Spotify only if known. | Clean separation. No interaction with the sync. | Two sources to merge in the API. Duplicates when the same album is both synced and curated. |
| **C. Exclusion flag only** | Add `hidden` to saved and archive albums, so the user curates by removing. | Very small. Works with every source. | Cannot add albums that are not already present. Does not answer the question on its own. |
| **D. Named lists** | A `jukebox_lists` table with memberships ("Sunday morning", "Road trip"). | Most flexible. | Over-built for now. The original spec has one library and no lists. |

**Catalogue selection.** The single-choice `jukebox_source` setting would need to change. It would be better as "which sources to include" (Spotify saved / scrobbled archive / my list), so a user can have only their own list, or their list plus Spotify. The CHECK constraint and `preferences_service.py` validation must change together, and the "empty means fall back to archive" rule must be dropped for a curated-only choice, so that an empty list shows an empty state instead.

### 12.5 Ways a user could add albums

| Method | Metadata you get | Effort | Notes |
|---|---|---|---|
| **From the Library** (single via `EntryMenu`, bulk via the selection bar) | Album, artist, artwork, play count. **No** year, genre, length, label or Spotify URL. | Low | Natural, and works for every source (Spotify, Apple Music, YouTube, file imports). Albums with no year or length drop out of the decade and length filters, and a missing duration currently counts as "short" (engine, section 5). |
| **Type it in** (artist, album, optional year, genre, length) | Whatever the user types | Low | Same pattern as manual gigs. Tedious for many albums. |
| **Paste a Spotify album link or ID** | Full metadata once resolved | Medium | A background job fills in the details (see below). Gives the real Spotify link. |
| **Search and add** | Full metadata | Medium to high | See the "which search" note below. |
| **Import from a Spotify playlist** (user curates a playlist, the app takes its albums) | Full metadata | Medium | Lets curation happen in Spotify. Uses scopes already granted. **Investigated in detail in section 13; it looks like the best option.** |
| **Upload a list** (CSV or plain text of artist and album) | Names only until enriched | Medium | Fits the existing file-import style. |

**Which search for lookups?**

- **Spotify.** The backend has never called Spotify's search or album endpoints. `AGENTS.md` forbids inline Spotify calls in a request, so a Spotify lookup should be queued and done by the worker, like genres. That makes "add" asynchronous: the album appears straight away with basic details and gains year, genre, length, label and Spotify URL later. Note that parts of the app already call Spotify from request handlers (for example `/tools/scope-creep/playlists` calls `get_user_playlists`), so the rule is not applied uniformly today.
- **Deezer or iTunes.** Already used server-side for imports and carry no Spotify rate-limit risk. They give year, artwork and often genre, but they do **not** give a Spotify link, so "Open in Spotify" would keep using the search-URL fallback. Deezer's album detail is believed to include track-count, duration and label; this should be checked against the live API before relying on it.
- Spotify has been tightening API access for newer apps. Confirm the current limits for album search and album lookup on the app's developer-mode registration before committing to a Spotify-based lookup.

### 12.6 Backend and UI shape (sketch)

**Routes** (all JWT-protected, scoped by `user_id`, with Pydantic schemas):

- `POST /jukebox/albums`: add one album, or a list for bulk add. Reject duplicates.
- `PATCH /jukebox/albums/{id}`: edit manual fields.
- `DELETE /jukebox/albums/{id}`: remove. For synced rows, "remove" would need a `hidden` flag, since the next sync would add the row back.

**UI:**

- "Add to Jukebox" in the Library `EntryMenu` and the bulk bar.
- A "My list" manager (add by typing or pasting, edit, remove) inside the existing Settings "Album Jukebox" section, or behind a "Manage" button on the card.
- The catalogue source setting becomes a set of toggles.

**Rules to respect:**

| Rule | Why it matters here |
|---|---|
| **Validate every URL field.** `spotify_url` is rendered straight into an `href`, and `artwork_url` would be rendered as an image. At present only the worker sets them. Once users can type them, a `javascript:` or other hostile scheme becomes possible. Allow only `https://open.spotify.com/…` for links and `https://` for images. | Security (untrusted input) |
| Limit lengths to the column sizes (255), cap the number of albums per user (for example 2,000), and treat text as plain text only. | Input validation, abuse |
| Apply the user's blocks (finding 24) to every source, including the curated list. | Access control rule |
| Never rewrite migration 0028. Add a new migration with an upgrade and downgrade. Test against a real throwaway Postgres, because SQLite hides dialect failures (see `HANDOVER.md`). | Migration rules |
| Make the sync **never overwrite or delete** manual rows, and decide what happens when the same album is later saved in Spotify (suggested: keep it as manual, but merge in the Spotify ID and metadata). | Data safety |
| **Back up and export.** Synced albums can be rebuilt from Spotify. A hand-made list cannot. The pg_dump backups cover it, but the Settings export (`GET /import/export`) covers listening history only. Consider including the curated list. | Data loss |
| Update the changelog, bump the version, and add user guide and FAQ entries (all required by `AGENTS.md`). Also add the feature to `agents/page_specs.md`. | Process |
| Go through the planner agent for approval before any build. | `AGENTS.md` process |

### 12.7 How this relates to the reference spec

- The original spec's data requirement (§12) is "every **favourited album, from any source**". The app has no such list (section 11, constraint 7). A user-curated list is the most direct way to meet that requirement, and it is the only route for albums that are not on Spotify.
- It also gives a home for **mood** (section 11, constraint 1): a manually added album could carry a user-chosen mood, which removes the need for a separate tagging feature.
- The spec's out-of-scope list (§14) says "editing favourites or metadata from the Jukebox". A curated list is therefore a deliberate extension, not part of the original brief. It should be recorded as a product decision.

### 12.8 Suggested phasing

1. **Foundation.** Migration adding `source` (and a `hidden` flag), the catalogue source becomes multi-select, the empty-state fix, and block handling. Routes to add, edit and remove manual albums, with URL validation.
2. **Easy entry.** "Add to Jukebox" from the Library (single and bulk), plus typing an album in by hand. This already makes curation usable.
3. **Enrichment.** A worker job (or Deezer / iTunes lookup) that fills in year, genre, length, label, artwork and the Spotify link for albums added with names only.
4. **Bulk and extras.** Paste a Spotify link, import from a Spotify playlist, upload a list, optional mood, and include the list in the export.

### 12.9 Decisions needed

1. **Replace or combine?** Should a curated list replace the other sources, or be added to them?
2. **Where does curation happen?** From the Library, from a manager on the Jukebox, or both?
3. **Which lookup?** Spotify (queued, slower, gives the real link) or Deezer / iTunes (simple, no Spotify link)?
4. **Removing synced albums.** Should "remove" hide a Spotify album for good (the flag), or should the sync be made to mirror Spotify (finding 11)?
5. **Mood.** Include a user-chosen mood on curated albums? (Reference spec open question 3.)
6. **Export.** Should the curated list be included in the data export?

None of these were built or changed as part of this investigation. Any build would go through the planner agent for approval first.

---

## 13. Investigation: build the list from a Spotify playlist of single tracks

**Idea:** the user makes a Spotify playlist containing one song from each album they want (for example 100 songs for 100 albums). The app reads the playlist, traces each song back to the album it belongs to, and adds those albums to the Jukebox list.

**Verdict:** feasible, and the most efficient of the options in section 12.5. It needs no search feature, because Spotify's own search is the search box. It uses Spotify scopes the app already requests, and the track data already contains most of what the Jukebox needs. It has to run as a background job, and it has some caveats about what "origin album" means (13.4).

> **Evidence note.** The code findings below come from reading the repo. The Spotify API behaviour comes from search-result summaries of Spotify's February 2026 changelog and migration guide. I tried to read the official pages directly, but `developer.spotify.com` is not reachable from this environment, so those points are marked **(unverified)** and should be checked against the live documentation before any build.

### 13.1 How it would work

| Step | What happens | Calls to Spotify |
|---|---|---|
| 1. Choose the playlist | The user pastes a playlist link or ID, or picks one from a list of their own playlists. The app already has both: `_extract_playlist_id` parses links, and `GET /tools/scope-creep/playlists` plus a picker in `ScopeCreepTools.jsx` list the user's playlists. | 0 (or 1 to list playlists) |
| 2. Queue a job | The backend records a small import job and returns straight away. | 0 |
| 3. Read the playlist | The worker fetches the playlist items, 100 per page. Each track object already carries a simplified **album**: Spotify album ID, name, type (`album` / `single` / `compilation`), release date, artwork, artists, link and track count. | 1 per 100 tracks |
| 4. Trace tracks to albums | Take each item's album. **De-duplicate by album ID** (three songs from one album become one album). Skip local files, podcast episodes and unavailable items. Skip albums already in the list. | 0 |
| 5. Store | The albums are saved straight away with what the playlist gave (title, artist, year, artwork, Spotify link). They are playable immediately. | 0 |
| 6. Enrich | Fill in length, label and genres over the next few worker runs, a few at a time. | 1 per album (or 1 per 20 if batching is allowed; see 13.7) plus artist genre lookups |
| 7. Report | The user sees "Imported 87 albums; 9 already in your list; 4 tracks skipped (local files)", with progress while it runs. | 0 |

**Call budget for 100 songs from 100 different albums:** about 1 call to read the playlist, up to 100 album lookups and up to 100 artist lookups to enrich. That is small if spread over several worker runs. If batch endpoints are still allowed it is roughly 8 calls in total. The playlist is readable in a single call, so the Jukebox becomes usable almost immediately even while genres and lengths are still filling in.

### 13.2 What the track already tells us

| Jukebox field | From the playlist track's album object? |
|---|---|
| Spotify album ID, title, artist | **Yes** |
| Release date and year (decade filter) | **Yes** |
| Artwork | **Yes** |
| Spotify link (for "Open in Spotify") | **Yes** |
| Album type (album / single / compilation) | **Yes**, useful for the review step (13.4) |
| Length (length filter) | **No.** Needs an album lookup (sum of track durations). |
| Label | **No.** Needs an album lookup. |
| Genre | **No.** Albums carry none; it comes from the artist, via a separate lookup. |
| Mood | **No** (unchanged from section 11). |

So the year, decade, artwork and link are free; only length, label and genre need the background step. Albums are usable meanwhile, but the length filter should treat "unknown length" as unknown rather than "short" (the current engine counts a missing duration as 0 minutes).

### 13.3 How it fits the architecture

| Concern | Position |
|---|---|
| **No Spotify calls inside a web request** (`AGENTS.md`) | Use a job table and a worker job. The repo already has the pattern: `setlist_import_jobs` (status, remaining payload, progress counts, message, `resume_after`) and the `setlist-import-resume` worker job. Latency equals the worker interval, so use a short interval such as the 2-minute `venue-geocoding` job. *(Note: some existing code, such as the playlist picker, does call Spotify from a request handler. The new work should follow the rule, not that precedent.)* |
| **Token handling** | The worker already decrypts the user's stored refresh token, refreshes the access token and persists a rotated token (`sync_saved_albums_for_user`). Reuse that. |
| **Rate limits** | Reuse `spotify_rate_limit_blocked_until()` and `SpotifyClient._request` (retries, honours `Retry-After`). Cap how many albums are enriched per run. |
| **Scopes** | `playlist-read-private` and `playlist-read-collaborative` are already requested. Users who signed in before they were added are handled by the existing `needs_scope` / re-authorise flow in `ScopeCreepTools.jsx`. |
| **Playlist read code** | The app can read a playlist's name and create or append to a playlist, but **nothing reads a playlist's tracks yet**. A new `SpotifyClient.playlist_items` method is needed. The app's write path already uses the newer `/playlists/{id}/items` form, which suggests the code was updated for Spotify's rename. |
| **Data model** | Needs the `source` column from section 12.4 (Option A), plus a reference to the playlist. See 13.5. |
| **Design system** | Plain forms, buttons and a progress line. No gradients or shadows needed. |
| **Blocks** | The imported albums must respect the user's blocks (finding 24). |

### 13.4 "Trace the song to its origin album": what that really means

Spotify gives the album that **that particular track entry** belongs to. It does not say which album a song was first released on. This matters because the same song often exists on several releases.

| Situation | What the app gets | Suggested handling |
|---|---|---|
| Song added from its original studio album | The right album. | Nothing needed. |
| Song added from a **single** or EP release | The single, whose album ID is not the parent album. | Use the `album_type` field to flag it in a review step, and let the user remove it or replace it later. |
| Song added from a **compilation** or "Greatest Hits" | The compilation. | Flag it the same way. |
| Song added from a **deluxe / remastered / live** edition | That edition (a different ID from the standard album). | Accept it. It is probably the version the user meant. Show the full title so they can see. |
| Several songs from the same album | One album (de-duplicated). | Show the number of songs, for example "3 songs". |
| **Local files, podcast episodes, removed or unavailable tracks** | No usable album. | Skip and count them in the report. |
| A song that is relinked to a different regional release | Occasionally a different album ID from the one shown in Spotify. | Rare. Ignore. |

Finding the "canonical" original album for a song would need a separate music database such as MusicBrainz. That is out of scope here; the review step covers most of the risk more simply.

**Recommended: a review screen before saving.** After the worker reads the playlist, show the list of albums found (artwork, title, artist, "from 3 songs", a badge for single or compilation), with all ticked except flagged ones, and a button to add the ticked ones. This keeps the user in control and avoids silently importing singles. It needs a "pending review" job status, and means the first worker pass only reads and previews, while the save is a second step.

### 13.5 One-off import or a live link?

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **One-off import** | Run the import once; the albums become ordinary entries in the user's list. | Simple. The user can edit or remove entries freely afterwards. | Adding more albums later means another import. |
| **Linked playlist (live)** | The user nominates one playlist as the Jukebox list. The worker re-reads it periodically (same cadence as the saved-albums sync) and keeps the Jukebox in step: new songs add albums, removed songs remove them. | Curating becomes "edit a playlist in Spotify", which users already know. **Solves the "removals never sync" problem (finding 11) for this source**, because the playlist is the source of truth. | Needs a stored playlist ID and a way to avoid clobbering manual entries (rows must record which playlist added them). A deleted playlist or lost access must degrade safely instead of emptying the list. |

A sensible path is to build the one-off import first, with each imported row recording where it came from (for example `source = 'playlist'` and the playlist ID), so the live link can be added later without a redesign.

### 13.6 Limits and edge cases

- **Only the user's own playlists (unverified).** Search summaries of the February 2026 change say a playlist's items are returned only for playlists the user **owns or collaborates on**; for others the items are missing. That is fine for the described workflow (the user builds their own playlist), but followed, shared or Spotify-made playlists will not work. The app should detect an empty or absent items list and show a clear message ("Spotify only lets us read playlists you own or collaborate on. Copy it into one of your own first.").
- **Playlist size.** Spotify returns 100 items per page. Set a cap per import (for example 500 albums) and one active import per user.
- **Duplicates across imports.** Match on the Spotify album ID, which is exact, unlike the name matching used for play counts.
- **Album already saved in Spotify.** The existing saved-albums sync uses the same album ID, so decide which source wins (suggested: keep it once, mark it as manual / playlist so the sync never removes it).
- **Non-Spotify listeners.** This only helps people who use Spotify. Apple Music or file-only users still need the Library route from section 12.
- **Partial failure.** If Spotify rate-limits halfway, the job should pause and resume later (as the setlist import does), not fail and lose progress.

### 13.7 The Spotify API risk (needs checking first)

According to search summaries (unverified; the official pages could not be read here):

- In February 2026 Spotify changed **Development Mode** apps. New apps from 11 February 2026 and existing ones from 9 March 2026 are reported as affected. Apps in Extended Quota Mode are reported as unaffected.
- **Batch endpoints were removed** for Development Mode apps, including Get Several Albums, Get Several Artists and Get Several Tracks. The replacement is one call per item (`GET /albums/{id}`).
- **Playlist endpoints moved** from `/playlists/{id}/tracks` to `/playlists/{id}/items`, with items returned only for playlists the user owns or collaborates on. Response field names were reported as renamed (`tracks` to `items`, and per-entry `track` to `item`). **A new playlist reader must use the new names**, and the code should accept both while this is uncertain.
- A later (March 2026) note reportedly reversed one removed field (`external_ids`). The details of which fields stayed or went were not confirmed.

**What this means for the playlist idea:** it still works, but enrichment may cost one call per album instead of one per twenty, and field names must be confirmed. At 100 albums that is manageable if spread across worker runs; at 1,000 it would take a long time.

**What this means for existing code (finding 25):** the album sync's genre lookup uses the batch artist endpoint, and the artwork backfill uses the batch track endpoint. If this app is registered in Development Mode, both may already be failing. Because the album sync swallows the error, nobody would notice except through empty genres.

**To settle it:**

1. Ask the maintainer whether the Spotify app is in Development Mode or has Extended Quota (the developer dashboard shows this).
2. Read Spotify's February 2026 migration guide and changelog directly.
3. Search the worker logs for "Could not resolve artist genres during album sync".

### 13.8 How it compares with the other ways to add albums

| Method (from 12.5) | Metadata | Effort for the user | Effort to build | Verdict |
|---|---|---|---|---|
| Add from the Library | Names and artwork only | One click each, or bulk | Low | Good for albums already scrobbled |
| Type it in | What the user types | High per album | Low | Fallback for odd cases |
| Paste an album link | Full | One link per album | Medium | Fine for a few |
| **Spotify playlist of songs** | **Year, artwork and link immediately; the rest after enrichment** | **Very low: build the playlist in Spotify, paste one link** | **Medium** (new job table, worker method, review screen) | **Best for bulk** |
| Search inside the app | Full | Medium | High | Not needed if the playlist route exists |
| Upload a text or CSV list | Names only until enriched | Medium | Medium | Useful for non-Spotify users |

### 13.9 Suggested build outline (for the planner step, not started)

1. **Migration:** `source` and `source_ref` on `saved_albums` (or a separate table), plus a job table such as `jukebox_import_jobs` (user, playlist ID, status such as `queued` / `reading` / `review` / `saving` / `enriching` / `done` / `failed`, counts, message, found albums as JSON for the review step).
2. **Backend:** `POST /jukebox/import/playlist` (validate the playlist ID against `[A-Za-z0-9]{22}`, create the job, one active job per user), `GET /jukebox/import/{job}` (status and preview), `POST /jukebox/import/{job}/confirm` (selected album IDs), plus a worker-only internal route to save albums. All JWT-protected and scoped by `user_id`.
3. **Worker:** `SpotifyClient.playlist_items`, a job that reads and previews, a job that enriches a few albums per run, both rate-limit aware and resumable.
4. **Frontend:** "Import from a Spotify playlist" in the Jukebox settings: paste or pick, progress line, review list with ticks, a result summary. The picker already exists.
5. **Hygiene:** validate that any stored `spotify_url` is an `https://open.spotify.com/…` link; treat all text as plain text; apply blocks; add tests (worker, backend, frontend) including one against real Postgres for the migration; update the changelog, version, user guide, FAQ and `agents/page_specs.md`; get planner approval first.

### 13.10 Decisions needed

1. **Review step or straight in?** Show a tick-list first (recommended), or import everything immediately?
2. **One-off or live-linked playlist?** (13.5) Or one-off first, with the live link later?
3. **What to do with singles and compilations?** Flag and untick by default (suggested), skip, or import as they are?
4. **Is the Spotify app in Development Mode?** This decides whether enrichment is one call per album (13.7).
5. **Upper limit** on albums per import and per user.
6. **Should imported albums be removable one by one**, and should removing one stop the live link from re-adding it?

None of this was built or changed. Any build would go through the planner agent for approval first.

---

## 14. Investigation: design-standard exceptions and a dedicated Jukebox tab

**Questions:** (1) Can the design standards be relaxed for the Jukebox? (2) Can the Jukebox have its own tab with different rules?

**Short answer:** yes to both. Nothing technical blocks it, and the repo already has one exception. Making it a separate page is the cleanest way to do it, because the exception then has a clear boundary.

### 14.1 What the "rules" actually are

| Layer | What it is | What enforces it |
|---|---|---|
| `DESIGN.md` | Written rules: Flat-By-Default (no shadows or gradients), One Voice (approved colour tokens, accent used sparingly), segmented tabs, outline icons, 0.5 px borders. | Documentation only. Agents and reviewers are told to follow it. |
| `AGENTS.md` | Repeats the key rules and **names one protected exception**: `RetroJukebox.jsx` and `RetroJukebox.module.css`. | Process only. The planner and implementation agents read it. |
| `frontend/src/cardConsistency.test.js` | Reads `src/styles.css` and fails if it contains gradients or any box-shadow other than `none` (or a hairline outline). | **An automated test, but only on that one file.** |
| `frontend/src/settingsFormTypography.test.js` | Checks form styling in `styles.css`. | Automated, same scope. |
| ESLint (`eslint.config.js`) | JavaScript rules only. | No CSS or design rules. There is no stylelint. |

**What this means:** the rules are a written convention plus a test on the shared stylesheet. A CSS module such as `RetroJukebox.module.css` is not scanned, which is exactly how the current exception passes CI. A new Jukebox page with its own CSS module would pass too, with no test changes.

### 14.2 Can exceptions be made? Yes, and how

An exception is a documented decision, not a code change. To do it properly:

1. **Record it in `DESIGN.md`.** Add a "Jukebox exception" section stating the scope (the route and files), what is allowed (gradients, shadows, its own colour palette, custom animation, decorative imagery), and what is still required (see 14.4).
2. **Extend the protected-file note in `AGENTS.md`.** Today it names two files. It should name the whole Jukebox page and its CSS, so that theme, accent or dark-mode work does not restyle them (the same reason the current exception exists).
3. **Keep the shared stylesheet clean.** Put all Jukebox styling in CSS modules (class names are scoped, so nothing leaks into other pages). Do not add Jukebox rules to `styles.css`, which would break the two tests above.
4. **Add a small guard test** that the Jukebox's exempt CSS is not in `styles.css`, so the exception cannot creep.
5. **Update `agents/page_specs.md`**, which currently says the header links to six pages and has no Jukebox entry.

**Who decides:** this repo is a fork, so the fork owner can set its own rules. If changes are ever meant to go back to the original maintainer, they would need to accept the `DESIGN.md` and `AGENTS.md` wording changes. `HANDOVER.md` also says the maintainer's standing process (plan, approval, implementation) applies to any change.

### 14.3 A dedicated Jukebox tab

"Tab" can mean two things here; the first is what I assume you mean.

| Meaning | Fit |
|---|---|
| **A top-level navigation entry with its own page** (like Overview or Set Lists) | **Recommended.** A page gives a natural boundary for different rules. |
| A tab inside an existing page (Settings and Reports use tabs) | Possible but not suitable. The Jukebox would still sit inside a flat page, and a cabinet with a claw does not fit a small tab panel. |

**What a dedicated page involves:**

| Piece | Detail |
|---|---|
| **Route** | Add a route in `frontend/src/App.jsx` inside `Layout`, so it keeps the shared header and footer. **The path must not start with `/jukebox`**, because nginx and the Vite dev server send those paths to the API, which would break refreshes and direct links (see section 11, constraint 3). Use, for example, `/album-jukebox`. The API routes can stay at `/jukebox/...`. |
| **Nav link** | Add one entry to `NAV_LINKS` in `HeaderNav.jsx` (currently six: Overview, Library, Reports, Set Lists, Profile, Connect). Seven fits on desktop; on mobile the links already collapse into a menu. `HeaderNav.test.jsx` lists the expected links and needs the new one. |
| **Width** | `.app-shell` is capped at 1180 px, which fits the mockup's 1100 px two-column layout, so the shared shell needs no change. |
| **Header clash** | The page header stays flat and the page below it can be as decorative as wanted. Giving the Jukebox a self-contained "stage" panel (its own background, border and padding) keeps the join clean. |
| **Overview** | Keep a **compact, flat teaser** on Overview (Album of the Day and a link), which complies with the flat rules. The full Jukebox lives on its own page. |
| **Skins** | With a page that has its own rules, the Flat/Arcade toggle is no longer needed for design reasons. Decide whether to keep Flat as an accessible, lighter alternative or make the page arcade-only. |
| **Code** | A new page component (for example `AlbumJukeboxPage.jsx`) uses the existing `useJukeboxEngine`, which does not change. |

### 14.4 What an exception should *not* waive

Exceptions are about **appearance**. These should stay mandatory on the Jukebox page:

- **Accessibility:** real buttons, visible focus, 44 px touch targets, `aria-pressed` on chips, a polite live region for the count, focus moved to the album title after a pick, decorative parts hidden from screen readers (reference spec §13). Colour contrast must still pass.
- **Reduced motion:** the claw and spin animations must be skipped when the user asks for it (reference spec §10). The app already added this care in v0.11.10.
- **Responsive:** it must work at phone width (the reference spec was designed at 375 px).
- **No leakage:** nothing in the page's styles may affect any other page.
- **Security and privacy rules** (authentication, blocks, input validation) are not design rules and are unaffected.

### 14.5 Theme, dark mode and accent colour

- The mockup is a **light-theme** design with a fixed palette (coral, mint, amber, pastel cabinet). The app has light, dark and system themes and eight accent colours that change `--color-primary`.
- **Recommended:** the Jukebox page ignores the accent colour (as the Retro skin does today) and uses its own fixed palette inside its stage panel, so it looks the same in light and dark mode. The surrounding page stays themed.
- A dark-mode variant of the cabinet is possible but is extra design work. It would not be needed for a faithful replica.
- The "One Voice Rule" (accent used for no more than 5% of a screen) would be explicitly waived inside the stage panel only.

### 14.6 Risks and trade-offs

| Risk | Mitigation |
|---|---|
| Exception creep (other pages copying the Jukebox look) | Name the exact route and files in `DESIGN.md`, and add the guard test (14.2). |
| The page feels disconnected from the rest of the app | Keep the shared header, footer and typography, and confine the decoration to one stage panel. |
| A bigger stylesheet loads on every page | The page's CSS module is small (the mockup's CSS is about 340 lines). If it grows, load the page lazily (`React.lazy`) so other pages do not download it. |
| A second thing to maintain in two skins | If the page is arcade-only, there is only one skin on the page, and the flat teaser on Overview is tiny. |
| Mobile menu gets longer | One extra item; acceptable. |

### 14.7 Effect on the earlier sections

- It removes constraints 2, 4 and 5 from section 11 (the shadow and gradient ban, the card layout and the protected-file concern), leaving the data constraints (mood, play counts, covers, favourites) and the nginx path rule.
- It makes **option C** in section 11.3 (dedicated page plus a compact Overview teaser) the natural choice, and removes the main reason for **options D and E**.
- The curated list (section 12) and playlist import (section 13) would live naturally on this page, for example in a "My albums" panel, instead of being squeezed into Settings.

### 14.8 Decisions needed

1. **Confirm the page approach:** a new nav entry and route (and a flat teaser on Overview)?
2. **Path and label:** for example `/album-jukebox` with the label "Jukebox"?
3. **One skin or two?** Arcade-only on the page, or keep a flat alternative?
4. **Scope of the exception:** only the stage panel, or the whole page including its headings and controls?
5. **Dark mode:** fixed light cabinet on a dark page (simplest), or a dark variant?
6. **Upstream:** will this fork's changes ever go back to the original maintainer? If so, the wording changes to `DESIGN.md` and `AGENTS.md` need their agreement.

None of this was changed. Any build would go through the planner agent for approval first, and the `DESIGN.md` and `AGENTS.md` wording changes should be part of that plan.

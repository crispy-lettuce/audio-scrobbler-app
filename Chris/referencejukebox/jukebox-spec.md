# Album Jukebox – Product / UX Spec

A feature for the scrobbler app. Working mockup: [jukebox-mockup.html](jukebox-mockup.html). Open it straight from disk. It has no dependencies and uses 40 dummy albums.

## 1. Purpose
Spotify's algorithm and shuffle keep narrowing what gets played. The Jukebox does two things:
- It gets **whole favourited albums** played again.
- It removes **choice paralysis**. You set a few filters, press one button and get one album.

## 2. Users
One person: the owner of the scrobbler app, using it for personal listening. Mostly on a phone, sometimes on a desktop. No accounts, sharing or social features.

## 3. Core concepts
| Term | Meaning |
|---|---|
| Jukebox | The screen. A flat, stylised jukebox that shows the library as a rack of records. |
| Record | One favourited album (any source) from the scrobbler app. |
| Filtered set | The records that match all active filters. |
| Pick pool | The part of the filtered set that the random pick draws from. It depends on the pick mode. |
| Lucky Dip | The main button. It picks one record at random from the pick pool. |
| Re-roll | Rejects the current pick and draws again. Limited (see §7). |
| Album of the day | A random record shown on open. It stays the same for the whole calendar day. |
| Roll limit | A maximum of 3 re-rolls per Lucky Dip, which forces you to commit. |

## 4. Screens & layout (mobile-first)
A single screen, designed at **375px** wide. Light theme, flat shapes, few colours.

**Mobile (stacked, top to bottom)**
1. **Jukebox body**, with:
   - a rounded top and a "JUKEBOX" marquee
   - **selector lights**: one per filter type, lit when that filter is active. Tapping a light jumps to that filter.
   - a **live counter**, e.g. "14 records · picking from 4 least played"
   - the **record rack**, a grid of tiny covers (8 per row on mobile, 10 on wider screens)
   - the **turntable** (platter, record, tone arm) and the **claw arm**
   - the **Pick from** segmented control
   - the large **Lucky Dip** button
2. **Result card**, showing the current pick or the album of the day.
3. **Filters panel**, made of collapsible groups with a "Clear all" link.

**Desktop (≥ 880px):** two columns. The jukebox is on the left and stays in view as you scroll. The result card and filters are on the right.

**Rack feedback:** this happens live on every filter change.
- Matching records stay at full size.
- Non-matching records dim, shrink and turn grey.
- Records that match but fall outside the pick pool are half-dimmed.
- The current record has an outline.

## 5. Filters
Different filter types combine with **AND**. Selections within one type combine with **OR**. Every change updates the rack, counter and lights immediately.

| Filter | Control | Logic |
|---|---|---|
| Artist | Text search (with suggestions) | Case-insensitive "contains" |
| Album | Text search | Case-insensitive "contains" on title |
| Year | Decade chips (1950s…2020s) | Multi-select, OR |
| Genre | Chips | Multi-select, OR |
| Mood | Chips | Multi-select, OR |
| Length | Chips: Under 35 min / 35–50 min / Over 50 min | Multi-select, OR |
| Label | Chips (collapsed by default, since the list is long) | Multi-select, OR |
| Added to favourites | Chips: Last 30 days / 1–6 months ago / 6–12 months ago / Over a year ago | Multi-select, OR. The buckets don't overlap. |

"Clear all" resets every filter. Changing the pick mode is not a filter, so it never lights a selector lamp.

## 6. Pick modes ("Pick from")
| Mode | Pick pool |
|---|---|
| Any (default) | The whole filtered set |
| Least played | The lowest 25% of the filtered set by play count |
| Most played | The highest 25% by play count |
| Recently added | The newest 25% by date added to favourites |

- Pool size = `max(1, ceil(filteredCount × 0.25))`. The pick is uniformly random within the pool.
- Ties at the cut-off can be broken in any order.
- There is **no** "no repeats until everything has played" rule. The only repeat guard: any pick (Lucky Dip or Re-roll) avoids the album currently on screen when the pool has more than one album.

## 7. Roll-limit rule
1. **Lucky Dip** with no session open starts a **pick session** with **3 re-rolls**.
2. **Re-roll** uses one roll. Pressing Lucky Dip again mid-session **also uses a roll**, so the limit can't be bypassed.
3. When the rolls run out, **Re-roll and Lucky Dip are both disabled**. The message reads **"No more rolls – give it a spin!"**
4. The session ends (**commit**) when the user taps **Open in Spotify**. The next Lucky Dip starts a fresh session with 3 rolls.
5. **Reloading** the screen also resets the session.
6. Changing filters or the pick mode does **not** reset the count. Remaining rolls draw from the current filters and mode.
7. The album of the day is not part of a session. Re-roll stays disabled until the first Lucky Dip.

## 8. Album of the day
- Shown on the card every time the screen opens, labelled "Album of the day · 30 Oct".
- It's random but **stable for the local calendar day**. The RNG is seeded with today's date (`YYYY-M-D`) and picks from the favourites sorted by `id`, so library order doesn't matter.
- The pick is **persisted per local date** (`{date, albumId}`). On open, the stored album is reused if the date matches. It's re-picked only if that album is no longer a favourite. If storage is unavailable, the seeded pick is used.
- If the app is left open past midnight, or brought back to the foreground on a new day, it refreshes the album of the day and date-relative labels ("Today", "Yesterday", Added buckets).
- It picks from **all favourites and ignores filters**, so it's the same no matter how the screen was left.
- It does **not** count as a pick: it doesn't update "last picked" or use any rolls.

## 9. Result card
| Field | Notes |
|---|---|
| Cover art | Real artwork from the source where available. The mockup uses a generated gradient plus title initials. |
| Title, Artist, Year | |
| Plays | Play count from the scrobbler |
| Last picked | Date this album was last picked by the Jukebox, or "Never". Shown as "Today" or "Yesterday" when recent. |

The card shows the "last picked" value from *before* the current pick, so you see when it last came up. The stored value is set to now when the card is revealed. This happens on both Lucky Dip and Re-roll.

**Actions**
- **Open in Spotify**: opens in a new tab with `rel="noopener noreferrer"` and commits the session. The mockup links to `https://open.spotify.com/search/<artist + album>`. The real app should use the album's Spotify URI/URL where it's known.
- **Re-roll**: see §7. Under the buttons is a status line, e.g. "Re-rolls left: 2".

## 10. Selection animation
It plays on every Lucky Dip or Re-roll. There is no sound. Total time is about 3.5s.

| # | Step | Approx. duration |
|---|---|---|
| 1 | The chosen record in the rack highlights and grows. The card fades to a pending state. Buttons are disabled, including Open in Spotify (the card is inert). | 250ms |
| 2 | The claw arm moves from its home position (top right) to the record. | 600ms |
| 3 | The claw grips: its fingers close. | 150ms |
| 4 | The claw carries the record to the platter, and the record grows to full size on the way. Its slot in the rack shows as empty. | 800ms |
| 5 | The claw releases and returns home. The record sits on the platter showing its label. The tone arm swings on. | 450–500ms |
| 6 | The record spins (about 3 turns, easing out). | 1400ms |
| 7 | The card reveals the album, tagged with the pick mode used at pick time, and focus moves to its title. | 300ms |

**Reduced motion:** if `prefers-reduced-motion: reduce` is set, skip steps 1–6. Load the record onto the platter and reveal the card at once.

**Failure:** the pick must still reveal if the animation fails, and the controls must re-enable.

## 11. Empty & edge states
| State | Behaviour |
|---|---|
| Filters match 0 albums | Counter shows "0 records". All records dim. An empty-state box says "No records match these filters" and has a **Clear filters** button. **Lucky Dip and Re-roll are disabled.** |
| Pool of 1 | Lucky Dip still works. A re-roll returns the same album, since there's nothing else to pick. |
| Out of rolls | See §7. The message appears both on the card and under Lucky Dip. |
| Filters change during the animation | Rack and counter update live; the in-flight pick is unaffected. |
| Long titles | Wrap on the card. They are never truncated. |
| Never picked | "Last picked: Never" |
| No favourites at all | (Not in mockup) Show "Favourite some albums in the scrobbler first" and hide the Jukebox controls. |

## 12. Data needed from the scrobbler app
Per favourited album:

| Field | Used for |
|---|---|
| `id` | Identity |
| `artist`, `title` | Card, search filters, Spotify link |
| `year` | Card, decade filter |
| `genre` (one primary value is enough) | Genre filter |
| `mood` (one value) | Mood filter. This may need tagging if the scrobbler doesn't hold it. |
| `lengthMinutes` | Length filter |
| `label` | Label filter |
| `addedDate` (when favourited) | Added filter, Recently-added mode |
| `playCount` | Card, Least/Most played modes |
| `lastPicked` (nullable, **written** by the Jukebox) | Card |
| `coverUrl` (optional) | Artwork. Fall back to a generated cover. |
| `spotifyUrl` / URI (optional) | Direct link. Fall back to search. |

## 13. Accessibility
- All controls are real buttons, inputs or radios, with visible focus rings. Touch targets are at least 36–44px; selector lights and "Clear all" are at least 44px. Light labels fit at 375px (about 0.75rem, with ellipsis if needed).
- Filter chips use `aria-pressed`. Selector lights have labels such as "Genre filter on – jump to filter".
- The visual counter updates instantly. A separate visually hidden `aria-live="polite"` region announces the count about 500ms after changes pause, so typing doesn't flood the screen reader.
- After the reveal, focus moves to the album title, which is read once. There is no separate "Now playing" announcement.
- The rack, turntable and claw are decorative (`aria-hidden`). All their information is also available as text (counter, card).
- `prefers-reduced-motion` is respected (§10). State is never shown by colour alone: lights and chips also change border or fill.
- Data goes into the DOM as text only, never as HTML.

## 14. Out of scope
Sound or audio previews. A no-repeat or "play through everything" mode. Pick history or log screens. Sharing. Stats or charts. Playback control inside the app. Editing favourites or metadata from the Jukebox.

## 15. Assumptions & open questions
**Assumptions**
- The Least/Most played and Recently added modes narrow the filtered set to its top or bottom 25% (at least 1 album), then pick at random. This interprets "acts like a sort".
- The album of the day is seeded by the local date, picked from all favourites, and doesn't count as a pick.
- "Committing" means tapping Open in Spotify (§7).
- "Last picked" is updated on every reveal, including re-rolls, because it records what the Jukebox *picked*, not what was played.
- Each album has a single genre and a single mood.
- The "Added" buckets don't overlap, rather than "within the last X".
- Opening Spotify and coming back, then pressing Lucky Dip, starts a fresh session with new rolls. This loophole is accepted for personal use.

**Open questions**
1. Should roll state survive a reload (for example, stored per day), or is resetting on reload acceptable?
2. Should a re-rolled-away album also get "last picked" updated, or only the album you commit to?
3. Where does mood come from? Manual tags, the source's metadata, or derived?
4. For albums from non-Spotify sources with no Spotify match, should the button change to "Open in <source>"?
5. Should Pick-from mode and filters persist between visits?
6. Is 25% the right pool size for the sorted modes, or should it be a fixed N (e.g. 10)?

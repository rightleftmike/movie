# Rewatchables · Mikel — Cloud Edition

**Live app:** https://ylexjdgfkcinnxjzmakt.supabase.co/functions/v1/app
Add to phone home screen: open in Safari/Chrome → Share → "Add to Home Screen".

## Architecture

- **Supabase project** `rewatchables` (ref: `ylexjdgfkcinnxjzmakt`, org: Brilliant Bar, free tier, us-east-1)
- **Hosting:** Edge Function `app` serves the single-page app. The HTML itself lives in the `site_assets` table — the function reads it with a 60s cache, so the app can be updated with plain SQL, no redeploy.
- **Episode list:** fetched live in-browser from the public Google Sheet CSV endpoint
  `https://docs.google.com/spreadsheets/d/13KsNb1SxA7DYYQAzhaYxj_qKX0PXV1PXyKAZ7cEapRc/gviz/tq?tqx=out:csv&sheet=episode list`
  Fallbacks: last successful load cached in localStorage → bundled snapshot at `.../app?rw=1` (292 films).
- **Film facts:** TMDB v3 API (key baked in as default, overridable in Settings).
- **AI features:** Claude API (`claude-sonnet-4-5`), key entered in Settings, stored only in browser localStorage.

## Database schema (public, anon key has full read/write via RLS — low-sensitivity personal data)

### `ratings` (93 rows migrated from v10, 2026-07-17)
| column | type | notes |
|---|---|---|
| id | uuid pk | auto |
| title | text | required |
| year | int | |
| rating | text | one of: All-timer, Loved, Liked, Meh, Disliked (CHECK constraint) |
| tier | text | Top 5, Top 10, Top 25, Top 50, Top 100, Outside Top 100 (CHECK) |
| rewatch | text | Yes / No |
| times_seen | int | |
| first_seen_year | int | |
| vibe, pulled_in, fav_actor, fav_scene, didnt_click, notes | text | written notes |
| saved_at | timestamptz | default now() |

Unique index on `(lower(title), year)` — one rating per film.

### `watchlist`
id uuid pk, title text, year int, source text (e.g. "instagram crime list", "claude suggestion"), added_at timestamptz.

### `followed_people`
id uuid pk, name text unique, added_from text, created_at timestamptz.

### `site_assets` (app code — anon read-only)
path text pk ('app.html', 'rw.json'), ctype text, content text, updated_at.

## Managing data from Claude chats (the whole point)

Any Claude chat with the Supabase connector can read/write this data. Examples:

```sql
-- "I watched Fargo — Loved it, Top 50"
insert into ratings (title, year, rating, tier, rewatch, times_seen)
values ('Fargo', 1996, 'Loved', 'Top 50', 'Yes', 1);

-- add to watchlist
insert into watchlist (title, year, source) values ('Thief', 1981, 'chat');

-- follow someone
insert into followed_people (name, added_from) values ('Denis Villeneuve', 'chat');
```

The app reflects changes on next load/refresh. Ratings values must match the CHECK constraints above.

## Updating the app itself from a chat

```sql
update site_assets set content = $NEW$ ...full new HTML... $NEW$, updated_at = now()
where path = 'app.html';
```
Takes effect within ~60s (edge function cache). Local source copy: `app.html` alongside this README.

## Engineering notes (carried over from v10 lessons)

- No dynamic strings in inline onclick — all clicks go through one delegated listener using a `data-ref` index into a JS registry (apostrophe/diacritic-proof).
- No nested template literals; HTML built by concatenation with an `esc()` helper everywhere.
- All network paths have visible error states + toasts; Supabase writes are optimistic with reload-on-failure.

## Open items

- The Departed rating: was only in old browser localStorage — needs re-entry (chat or app).
- Instagram crime/thriller watchlist (~100 films): not in the v10 file; paste it into a chat to seed `watchlist`.
- v10 ratings blob only covered A–M + Top Gun: Maverick (93 films); any N–Z ratings need re-adding.

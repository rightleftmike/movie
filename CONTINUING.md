# Rewatchables App — Continue-Building Guide

For any future Claude/Cowork session. Read this whole file before touching anything.
Live copy of this guide: https://rightleftmike.github.io/movie/CONTINUING.md

## What this is

Mikel's personal film-tracking web app built around Bill Simmons's Rewatchables podcast. He rates films (All-timer/Loved/Liked/Meh/Disliked + rank tiers), keeps a watchlist, follows actors/directors/writers, and cross-references everything against the podcast episode list and TMDB. Data lives in Supabase so both the app AND Claude chats can read/write it. Long-term ambition: multi-user version as a portfolio piece for podcast companion apps (see MULTIUSER_PLAN.md in the repo).

## Where everything lives

| Thing | Location |
|---|---|
| Live app | https://rightleftmike.github.io/movie/ |
| Source of truth (code) | GitHub repo `rightleftmike/movie`, branch `main`. `index.html` = the whole app (single file). `rw.json` = bundled episode-list snapshot. |
| Database | Supabase project `rewatchables`, ref `ylexjdgfkcinnxjzmakt`, org "Brilliant Bar" (free tier, us-east-1). Use the Supabase MCP connector. |
| Edge function | `app` (verify_jwt=false). Routes: bare GET → 302 to the app; `?rw=1` → episode snapshot JSON from `site_assets`; `?feed=1` → CORS proxy for the podcast RSS (1h cache). |
| Episode list (live) | Google Sheet `13KsNb1SxA7DYYQAzhaYxj_qKX0PXV1PXyKAZ7cEapRc`, fetched in-browser: `/gviz/tq?tqx=out:csv&sheet=episode list` (no gid needed, CORS-friendly) |
| Podcast RSS | https://feeds.megaphone.fm/the-rewatchables — official, has MP3s + descriptions, but only a rolling ~39-episode window (archive is Spotify-exclusive) |
| TMDB key | `9f6a7cc8ad0b39dcb6381c44a16761cc` (v3, baked into app as default) |
| Anthropic key | User-entered in app Settings → browser localStorage only. NEVER hardcode. |

## Database schema (all public schema, RLS on, anon key = full read/write; single-user by design)

- `ratings` — id uuid pk, title, year int, rating (CHECK: All-timer/Loved/Liked/Meh/Disliked), tier (CHECK: Top 5/Top 10/Top 25/Top 50/Top 100/Outside Top 100), rewatch (Yes/No), times_seen int, first_seen_year int, vibe, pulled_in, fav_actor, fav_scene, didnt_click, notes, saved_at. Unique on (lower(title), year). ~93 rows migrated from v10.
- `watchlist` — id, title, year, source, added_at.
- `followed_people` — id, name (unique), added_from, created_at.
- `site_assets` — path pk, ctype, content, updated_at. Holds `rw.json` (used by `?rw=1`). Its `app.html` copy is STALE/deprecated — GitHub is the only code source of truth now.

Chat data ops are plain SQL via the connector, e.g. `insert into ratings (title, year, rating, tier, rewatch, times_seen) values ('Fargo', 1996, 'Loved', 'Top 50', 'Yes', 1);` — the app shows it on next refresh.

## How to modify the app

1. The Cowork sandbox CAN reach github.com (git push works) but NOT api.github.com, supabase.co, or github.io — verify with curl before relying on network; use the web_fetch tool for github.io/supabase URLs.
2. Get a fresh GitHub PAT from Mikel (public_repo scope) — tokens are short-lived/deleted after sessions.
3. Clone/pull `https://github.com/rightleftmike/movie.git`, edit `index.html`, then ALWAYS validate before pushing: extract the `<script>` body and run it through node `new Function()`.
4. Push to `main` → GitHub Pages auto-deploys in ~1 minute.

### Hard-won engineering rules (do not violate)

- NEVER put dynamic strings (titles, names) in inline onclick/HTML attributes — apostrophes and diacritics break it. The app uses one delegated click listener + `data-ref` indexes into the `REF` registry array. Follow that pattern.
- No nested template literals. The app is written entirely with string concatenation + an `esc()` helper on every interpolated value. Keep it that way.
- Every network path needs a visible error state (toast + inline message), never a silent spinner.
- Supabase writes are optimistic (update UI first, reload from cloud on failure).
- Supabase CANNOT serve HTML on its shared domain (gateway rewrites text/html → text/plain on both Edge Functions and Storage). Don't retry that path; hosting is GitHub Pages.
- Never rely on AI memory for film facts — TMDB only. Suggestion prompts must cite the user's actual ratings/notes.

## App feature map (v11, all in index.html)

Tabs: Search (TMDB, autocomplete over show list + ratings, full movie card), My Ratings (stats bar, filters, sorts), Suggest (3 modes, taste profile from ratings+notes+followed people — followed people are explicitly weighted), Show List (live sheet, search, unrated filter, sort oldest/newest/A–Z, inline +Rate), Watchlist (add/watched→rate/remove), People (follow/unfollow, TMDB photos, person drawer with On-Show/In-Ratings/On-Watchlist/Notable sections), Settings (Anthropic key, TMDB key, connection statuses, JSON export). Plus: Watch Tonight button (one pick, time-aware), rating form (cast chips, follow toggles, ✨ Refine original-vs-refined via Claude), movie cards show a 🎙 Episode section (inline RSS audio player + episode notes when in feed window, Spotify search link otherwise).

## Backlog (in rough priority order)

1. **The Departed rating** — still missing (was only in old localStorage). Ask Mikel for details, insert via SQL. Doubles as the chat-workflow verification (task never completed).
2. **Instagram crime/thriller watchlist (~100 films)** — not recovered; Mikel needs to paste it, then seed `watchlist` with source 'instagram crime list'.
3. **Missing N–Z ratings** — the v10 blob only covered A–M + Top Gun: Maverick; Mikel may re-add others over time.
4. **Multi-user version** — full plan in MULTIUSER_PLAN.md (repo + outputs): Phase 1 Supabase Auth + per-user RLS, Phase 2 AI proxy with quotas + paid tier, Phase 3 aggregate/community views. Do Phase 1 first with a few friends.
5. **Episode transcription → key takeaways** (parked): feed MP3s → speech-to-text (~$0.50–0.75/episode, needs an OpenAI or similar key) → Claude distills takeaways → cache in a `takeaways` table → show on movie cards. Only reaches the feed's recent window; run monthly to build the library forward.

## Kickoff prompt for a new session

Paste this to start:

---
Continue building my Rewatchables film-tracking app. First, fetch and read https://rightleftmike.github.io/movie/CONTINUING.md — it has the full architecture, database schema, engineering rules, and backlog. The Supabase connector in this session is connected to project `ylexjdgfkcinnxjzmakt` (my ratings/watchlist/followed_people live there — you can read/write them directly via SQL). The app code is the single `index.html` in GitHub repo `rightleftmike/movie` (GitHub Pages hosts it); if we're changing code, ask me for a fresh GitHub token (public_repo scope), and always syntax-validate the script with node before pushing. Follow the engineering rules in the guide exactly — especially: no dynamic strings in inline handlers, no nested template literals, visible error states everywhere. Today I want to: [DESCRIBE TASK].
---

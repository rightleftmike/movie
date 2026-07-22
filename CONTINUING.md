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

## Database schema (public schema, RLS on — MULTI-USER as of Phase 1, 2026-07-22)

Every user's data is isolated by `user_id` + Row-Level Security. Signed-in users can only see/change their own rows. The app talks to Supabase via the **supabase-js** client (publishable key `sb_publishable_NnuHKulCrQ1xfiy19jLjtA_goENmK-c`), which carries the user's JWT automatically. The chat/connector workflow uses the **service role**, which BYPASSES RLS — so SQL you run here sees/edits everyone's rows.

- `ratings` — id uuid pk, **user_id uuid NOT NULL default auth.uid()**, title, year int, rating (CHECK: All-timer/Loved/Liked/Meh/Disliked), tier (CHECK: Top 5/Top 10/Top 25/Top 50/Top 100/Outside Top 100), rewatch (Yes/No), times_seen int, first_seen_year int, vibe, pulled_in, fav_actor, fav_scene, didnt_click, notes, saved_at. **Unique on (user_id, lower(title), year)**. 94 rows, all owned by Mikel.
- `watchlist` — id, **user_id uuid NOT NULL default auth.uid()**, title, year, source, added_at.
- `followed_people` — id, **user_id uuid NOT NULL default auth.uid()**, name, added_from, created_at. **Unique on (user_id, lower(name))** (was a global unique on name).
- `profiles` — id uuid pk → auth.users(id), display_name, created_at, plan (default 'free'). One row per user, auto-created on signup by trigger `on_auth_user_created` → `public.handle_new_user()` (leaves display_name NULL so the app prompts for it on first login). The app's "___'s Screening Room" subtitle reads `display_name`.
- `site_assets` — path pk, ctype, content, updated_at. Holds `rw.json` (used by `?rw=1`). Still `anon`-readable. Its `app.html` copy is STALE/deprecated — GitHub is the only code source of truth now.
- `*_backup_20260722` (ratings/watchlist/followed_people) — pre-Phase-1 snapshots, RLS-enabled with no policies (admin-only). Safe to drop once you're confident. An offline JSON backup also exists in the Cowork outputs folder.

**Auth:** Email magic-link is enabled and working. Google sign-in is coded (button in the login screen) but needs a Google Cloud OAuth client + Client ID/Secret pasted into Supabase → Auth → Providers → Google before it works. Supabase Auth **Site URL** = `https://rightleftmike.github.io/movie/`, with `https://rightleftmike.github.io/movie/**` in Redirect URLs.

**RLS policies:** per-table `select/insert/update/delete`, all `to authenticated using (user_id = auth.uid())` (insert uses `with check`). `profiles` similarly scoped to `id = auth.uid()`. No anon policies on user tables (anon sees nothing).

Chat data ops are plain SQL via the connector (service role, bypasses RLS). **You MUST include `user_id` on inserts now** (the `auth.uid()` default is NULL in the admin context and the column is NOT NULL). Mikel's user_id = `447deb87-0ce3-431a-a168-f5ed1f0ba6b2`. Example: `insert into ratings (user_id, title, year, rating, tier, rewatch, times_seen) values ('447deb87-0ce3-431a-a168-f5ed1f0ba6b2', 'Fargo', 1996, 'Loved', 'Top 50', 'Yes', 1);` — the app shows it on next refresh.

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
- Data layer is the **supabase-js client** (`SUPA = supabase.createClient(...)`, loaded from jsDelivr CDN in `<head>`). Use `SUPA.from('table').select()/.insert()/.update()/.delete()`, each returning `{data, error}` — always check `res.error` and surface it (visible error state). The old raw-REST `sb()` helper is gone. Do NOT hardcode user_id in the client; the DB default `auth.uid()` fills it for authenticated inserts.
- Auth gating: nothing loads until signed in. `initAuth()` runs on boot; `enterApp()` (post-auth) is the only place that calls `loadCloud()`/`loadSheet()`. Login/onboarding are full-screen `.auth-gate` overlays; `body.preauth` hides the header + app until authed.
- Supabase CANNOT serve HTML on its shared domain (gateway rewrites text/html → text/plain on both Edge Functions and Storage). Don't retry that path; hosting is GitHub Pages.
- Never rely on AI memory for film facts — TMDB only. Suggestion prompts must cite the user's actual ratings/notes.

## App feature map (v12 — multi-user, all in index.html)

**Auth (Phase 1):** login gate (email magic-link + Google button), first-login "what should we call you?" onboarding, header user chip (avatar/name + Sign out), subtitle "___'s Screening Room" from the profile's display_name, Settings → Account (email, editable display name, sign out).

Tabs: Search (TMDB, autocomplete over show list + ratings, full movie card), My Ratings (stats bar, filters, sorts), Suggest (3 modes, taste profile from ratings+notes+followed people — followed people are explicitly weighted), Show List (live sheet, search, unrated filter, sort oldest/newest/A–Z, inline +Rate), Watchlist (add/watched→rate/remove), People (follow/unfollow, TMDB photos, person drawer with On-Show/In-Ratings/On-Watchlist/Notable sections), Settings (account, Anthropic key, TMDB key, connection statuses, JSON export). Plus: Watch Tonight button (one pick, time-aware), rating form (cast chips, follow toggles, ✨ Refine original-vs-refined via Claude), movie cards show a 🎙 Episode section (inline RSS audio player + episode notes when in feed window, Spotify search link otherwise).

## Backlog (in rough priority order)

1. **The Departed rating** — still missing (was only in old localStorage). Ask Mikel for details, insert via SQL. Doubles as the chat-workflow verification (task never completed).
2. **Instagram crime/thriller watchlist (~100 films)** — not recovered; Mikel needs to paste it, then seed `watchlist` with source 'instagram crime list'.
3. **Missing N–Z ratings** — the v10 blob only covered A–M + Top Gun: Maverick; Mikel may re-add others over time.
4. **Multi-user version** — full plan in MULTIUSER_PLAN.md. **Phase 1 (Auth + per-user data + RLS) is DONE & verified (2026-07-22).** Before inviting the 3–5 guinea-pig friends: (a) optionally finish Google sign-in (Google Cloud OAuth client → paste Client ID/Secret into Supabase Auth → Providers → Google); (b) optionally customize the magic-link email template (Auth → Emails) — default is generic Supabase branding; (c) consider a privacy/terms blurb + feedback link in Settings; (d) note built-in email is rate-limited (~a few/hour) — fine for a handful of friends, add SMTP if it grows. **Then just share the URL — sign-up is self-serve.** Phase 2 = AI proxy + quotas + paid tier (BYO Anthropic key stays meanwhile). Phase 3 = aggregate/community views.
5. **Episode transcription → key takeaways** (parked): feed MP3s → speech-to-text (~$0.50–0.75/episode, needs an OpenAI or similar key) → Claude distills takeaways → cache in a `takeaways` table → show on movie cards. Only reaches the feed's recent window; run monthly to build the library forward.

## Kickoff prompt for a new session

Paste this to start:

---
Continue building my Rewatchables film-tracking app. First, fetch and read https://rightleftmike.github.io/movie/CONTINUING.md — it has the full architecture, database schema, engineering rules, and backlog. The app is now MULTI-USER (Phase 1 done): per-user data behind Supabase Auth + RLS. The Supabase connector in this session is connected to project `ylexjdgfkcinnxjzmakt` and uses the service role (bypasses RLS, sees all users) — my rows are owned by user_id `447deb87-0ce3-431a-a168-f5ed1f0ba6b2`, and any rows you insert MUST include a user_id. The app code is the single `index.html` in GitHub repo `rightleftmike/movie` (GitHub Pages hosts it), and its data layer is the supabase-js client; if we're changing code, ask me for a fresh GitHub token (public_repo scope), and always syntax-validate the script with node before pushing. Follow the engineering rules in the guide exactly — especially: no dynamic strings in inline handlers, no nested template literals, visible error states everywhere. Today I want to: [DESCRIBE TASK].
---

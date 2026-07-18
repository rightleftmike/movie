# Rewatchables → Multi-User: Upgrade Plan

Goal: open the app to the public with accounts, per-user data you can aggregate, and AI features powered by your Anthropic key behind a paid tier. Everything builds on the existing Supabase project and single-file app — nothing gets thrown away.

## Phase 1 — Accounts + per-user data (the foundation)

**Auth.** Turn on Supabase Auth with email magic-link and Google sign-in (no passwords to manage, lowest friction on phones). The app gets a small login screen; the supabase-js client library replaces the raw REST calls so sessions are handled automatically.

**Schema change.** Add a `user_id uuid` column (default `auth.uid()`) to `ratings`, `watchlist`, and `followed_people`, plus a `profiles` table (id, display_name, created_at, plan). Your existing 93+ ratings get assigned to your account. The unique index on ratings becomes `(user_id, lower(title), year)`.

**Row-level security — the important flip.** Replace today's "anon can do anything" policies with: each signed-in user can select/insert/update/delete only rows where `user_id = auth.uid()`. This is the line that makes it safe to have strangers in the same database. Your chat-workflow access is unaffected (the connector uses admin access).

**Data collection for you.** You automatically get every user's ratings, watchlists, and follows in your tables — that *is* the repository. Add an `events` table (user_id, event, payload, at) if you also want lightweight usage analytics (searches, suggestion requests). This is the point where you publish a short privacy policy + terms page in the app.

Effort: the largest phase — roughly a solid session of work, mostly rewiring the app's data layer around auth. One-time data migration for your rows.

## Phase 2 — AI proxy + paid tier

**Proxy.** A new edge function `ai` receives {task, payload} with the user's auth token, verifies the session, checks their plan and quota, then calls Anthropic with **your** key (stored as a Supabase secret, never in the browser). The app's `askClaude()` swaps its URL from api.anthropic.com to this function — a five-line change. BYO-key stays as a fallback path in Settings for power users.

**Quotas.** An `ai_usage` table (user_id, day, calls). The proxy enforces e.g. free = 3 lifetime trial calls, paid = 30/day. This caps your worst-case spend per user at pennies.

**Billing.** Simplest v1: Stripe Payment Link → on success, you flip `profiles.plan` to 'paid' (manually at first, webhook later). Don't build billing infrastructure before there are users.

Effort: small — the proxy is ~80 lines, and the quota logic is a table and an upsert.

## Phase 3 — The aggregate layer (your pitch material)

Postgres views over the now-multi-user data, exposed read-only:
- consensus rankings ("audience Top 25 Rewatchables" by average tier/rating)
- most-followed people across all users
- most-watchlisted unrated episodes ("what the audience wants covered next")

These power a public "Community" tab — which is exactly the demo for your podcast-companion-app pitch: *listeners interacting with the catalog the show covers, generating data the show itself would want to see.*

Effort: small once Phase 1 exists; each view is a dozen lines of SQL.

## What doesn't change

Episode list still streams live from the Google Sheet. TMDB stays client-side (free, per-user rate limits are generous; move behind the proxy later only if abuse shows up). GitHub Pages keeps hosting the static app — auth works fine from a static page. The chat workflow (you managing your data through Claude) keeps working throughout.

## Sequencing advice

Do Phase 1 alone first and invite 3–5 friends as guinea pigs while AI stays BYO-key. If they actually use it, add Phase 2's proxy with a free trial quota, then the Stripe link. Phase 3 whenever you're ready to make the pitch — it's also the feature that makes the app feel alive to users.

Also on the checklist before "public": pick a neutral name/domain for the public version (the "unofficial companion" framing), and a feedback link in Settings.

# CLAUDE.md — NFP (Natural Feline Planning)

Context for Claude Code (and humans) working in this repo. Read this before
making changes.

---

## 1. What this is

**NFP — Natural Feline Planning** is a private summer/hangout planner for a
group of **five friends** ("the squad"). Each person marks upcoming days as
**FREE / BUSY / IDK**, the app surfaces the days where everyone overlaps, and
there's a group chat. It's deliberately silly in tone (cats, cars, technology,
the universe, jokes/friendship) but must **look solid and function properly**.

It is **not critical software** — the data is cat hangout availability, not
secrets. Optimize for simplicity, reliability, and a fun, polished feel over
enterprise rigor.

The five felines (ids): `eeps`, `zarch`, `moog` (Professor Moogenborks),
`nat`, `coral`.

---

## 2. Tech stack & hard constraints

- **Single static file**: the entire app lives in `index.html`. **No build
  step, no bundler, no framework CLI.** This is a firm constraint — it must
  stay drop-in deployable on any static host (currently GitHub Pages). Do not
  introduce npm/Vite/webpack/etc. unless the user explicitly asks to change the
  hosting model.
- **React 18 via CDN** (`unpkg.com/react@18`, `react-dom@18`) + **Babel
  Standalone** (`@babel/standalone`) transpiling a single
  `<script type="text/babel">` in the browser at load time.
- **Supabase JS v2** (`jsdelivr` CDN) as the backend client.
- **Icons** are hand-rolled inline SVG via a local `<Icon name=... />`
  component — there is **no icon library dependency**. Add new icons to that
  component's `switch`.
- **Styling**: inline `style={{}}` objects + one injected `<style>` block
  (`CSS`) for `@import` fonts, keyframes, and pseudo-selectors. **No Tailwind /
  no CSS framework.** Colors come from the `C` palette object.
- **Fonts**: Bebas Neue (display/headers) + Space Mono (body/mono), imported
  from Google Fonts in the `CSS` block, with system fallbacks.
- **No localStorage for shared state.** localStorage is used *only* for the
  device-local identity key `nfp-me` (which feline you are). All shared data
  lives in Supabase.

---

## 3. Run & deploy

- **Local**: open `index.html` in a browser (double-click or any static
  server). Requires `config` values filled in (see §5) and the Supabase tables
  created (§4).
- **Production**: GitHub Pages, **Deploy from a branch → `main` / root**.
  Repo: `karlnoelle/<repo>` → site at `https://karlnoelle.github.io/<repo>/`.
- **No CI.** Earlier there was a GitHub Actions deploy workflow; it was removed
  when the project moved to bundled/inline config (see §9). Pages serves the
  branch directly.
- **Sanity-check a change without a browser**: extract the babel script and
  transpile it, e.g. pull the contents of the `<script type="text/babel">` into
  a `.jsx` file and run it through `esbuild`/`babel`. A clean transpile means
  the JSX is at least syntactically valid.
- **Manual smoke test**: open the site in two browsers (or one incognito),
  log in with two different magic keys, mark a day / send a chat in one, and
  confirm it appears in the other within the poll interval.

---

## 4. Data model (Supabase)

Two tables. **Run this SQL once** in the Supabase SQL editor:

```sql
create table if not exists nfp_state (
  uid text primary key,
  days jsonb default '{}'::jsonb,
  vibe text default '',
  updated_at bigint default 0
);
create table if not exists nfp_messages (
  id text primary key,
  uid text not null,
  ts bigint not null,
  body text not null
);
create index if not exists nfp_messages_ts on nfp_messages (ts);

alter table nfp_state enable row level security;
alter table nfp_messages enable row level security;
create policy "nfp_state anon all" on nfp_state for all to anon using (true) with check (true);
create policy "nfp_messages anon all" on nfp_messages for all to anon using (true) with check (true);
```

- **`nfp_state`** — one row per feline. `days` is a JSON map of
  `"YYYY-MM-DD" -> "free" | "busy" | "idk"` (absent = untracked). `vibe` is a
  short status string. `updated_at` is `Date.now()` ms.
- **`nfp_messages`** — one row per chat message (`id`, `uid`, `ts`, `body`).

**RLS is intentionally wide open** to the `anon` role. The app's access control
is the client-side magic-key gate, not the database. This is acceptable because
the data is non-sensitive (see §9 for the security reasoning and upgrade path).

### Key data-flow patterns (preserve these)

- **Collision-free writes**: each user only ever writes **their own**
  `nfp_state` row (upsert on `uid`), and chat is **individual row inserts**.
  This avoids last-write-wins clobbering between users. Do not introduce a
  single shared document that multiple users rewrite.
- **Identity**: `localStorage["nfp-me"]` stores the current device's feline id
  so they stay "logged in." Cleared on "switch feline."
- **Polling, not realtime** (current): refresh every **6s while on the Chat
  tab, 20s elsewhere**, plus a manual refresh button. See §10 for the Realtime
  upgrade.
- **Optimistic + debounced save**: day/vibe edits update React state
  immediately and write to Supabase on a **550ms debounce**. On refresh, the
  user's own row is kept if local `updatedAt >= server` so in-flight edits
  aren't stomped.
- **Chat dedup**: messages are merged with `unionById()` (dedup by `id`, sort
  by `ts`) so optimistic sends + polled reads never duplicate or drop. Query
  loads `order(ts asc).limit(300)`; the feed renders the last 200.

---

## 5. Config (inline, by design)

Near the top of `index.html`:

```js
const SUPABASE_URL = "...";        // Supabase → Project Settings → API
const SUPABASE_ANON_KEY = "...";   // anon public key (public by design)

const USERS = [ { id, name, key, emoji, accent, blurb }, ... ];  // 5 felines
```

- **Magic keys** are the `key` field on each `USERS` entry. They are committed
  in the file (see §9 for why this is acceptable and the alternative).
- `DB_READY` is false until both Supabase values are set (and not the `PASTE_…`
  placeholders); until then the app shows the **Setup** screen with the SQL and
  instructions.
- To change the squad: edit `USERS` (names/emojis/accents/bios are public;
  keys too, in this bundled model).
- Key matching is case-insensitive and also accepts a pasted `…key=<key>` form.

---

## 6. Feature scope (what must keep working)

- **Auth gate**: magic-key login → identity persisted in localStorage →
  "switch feline" logout. Unknown key shows a playful "ACCESS DENIED" error.
- **MY DAYS**: per-user calendar over a **rolling 5-year horizon**
  (`HORIZON_YEARS = 5`, months built from the current month). Tap a day to
  cycle FREE → BUSY → IDK → clear. **Past days are locked**; **today** is ring-
  highlighted. Month nav with a "↺ back to this month" jump. Per-user **vibe**
  input. Month stat counts.
- **CONVERGENCE**:
  - **Full-house highlight** (the headline feature): when **all five** are free
    on an upcoming day (within 365 days), show the pulsing **FULL HOUSE** banner
    (soonest date + count + "LOCK IT IN" → chat), a count **badge on the
    Convergence tab**, and **lime-tinted rows** in the matrix.
  - **"When the stars align"**: best free-day overlaps in the **next 120 days**,
    `free >= 2`, top 6, sorted by free-count then soonest.
  - **Overlap matrix**: per-day grid of all five felines' statuses, with a
    free-count bar. Range toggle: **NEXT 14 / THIS MONTH / NEXT 90**.
- **CHAT** ("Transmissions"): group chat, bubbles aligned right for you / left
  for others in each feline's accent color, consecutive-message grouping,
  timestamps, auto-scroll, Enter to send.
- **THE SQUAD**: roster cards — emoji, bio, current vibe, free/busy tallies,
  online/quiet status, last-seen, and a "YOU" marker.
- **Ambient**: cosmic animated background, bottom joke ticker, toast messages,
  save-state indicator in the header.

---

## 7. Design language & copy voice

- **Aesthetic**: dark cosmic "terminal / mission-control" — deep space
  background with drifting starfield + nebula glows + faint scanlines; neon
  accents; translucent panels with subtle borders and blur.
- **Palette** (`C` object): bg `#070711`; accents cyan `#34e8e0`, magenta
  `#ff3df0`; FREE `#56e39f`, BUSY `#ff5470`, IDK `#ffc24d`. Each feline has an
  `accent`. Use these tokens; don't hardcode new hex unless adding to `C`.
- **Type**: Bebas Neue for big condensed headers/labels (wide letter-spacing),
  Space Mono everywhere else.
- **Voice**: heavy on memes, internet culture, jargon, and silliness — themed
  around cats, cars, technology, the universe, and jokes/friendship. Keep it
  **not serious at all in copy, but solid and functional in behavior**. Target
  a voice that reads ~32 years old. Examples in-app: "do not perceive me",
  "the past is immutable, like prod on a Friday", "consult the cosmos",
  "re-sync with the mothership". Match this register when adding UI copy.

---

## 8. Conventions for editing this repo

- **Keep it one file, no build step.** If a change seems to need a bundler,
  flag it and ask before changing the hosting model.
- **Stay copyright-safe**: no copyrighted song lyrics, no copyrighted/branded
  meme images or characters. Original jokes and emoji only. The app uses no
  external images on purpose.
- **Preserve the collision-free write model** (§4) — per-user rows, per-message
  inserts.
- **Don't move shared data into localStorage**; localStorage is identity only.
- **Use the `C` tokens and the existing component patterns** (Panel, Icon,
  Tabs, etc.) rather than introducing new styling systems.
- **Validate** with a transpile + the two-browser smoke test before declaring a
  change done.

---

## 9. Decision log (auth + secrets) — important for future versions

This records *why* the app is built the way it is, so a future iteration can
make informed trade-offs instead of relitigating settled ground.

### Auth: why magic keys, not accounts

- The ask was "magic link" auth so random people can't access or contribute —
  but only for five people. A true emailed magic link needs a mail server,
  which a static client can't run.
- Decision: **per-feline "magic keys"** — unguessable passphrases handed to
  each person (the browser-native equivalent of a magic link: passwordless,
  token-based, gated). Identity persists in `localStorage`.
- This is access *friction/gating*, not real security. Anyone with the keys (or
  who reads them from the client) can act as that feline. Accepted because the
  data is non-sensitive.

### Hosting: why we left Claude artifacts for Supabase

- The app first lived as a Claude.ai published artifact using Claude's built-in
  persistent storage. Problem discovered in testing: **storage-backed published
  artifacts force every viewer to sign into a Claude account** (a login wall
  appears). The squad shouldn't need Claude accounts.
- Decision: **self-host with a Supabase backend**, so friends just open a URL +
  paste a key — **no accounts of any kind**.

### Secrets: the path we explored vs. what we shipped

- Concern raised: with a public GitHub repo (so friends can open PRs), the
  magic keys and Supabase anon key would be committed and visible.
- **Path explored (not shipped):** externalize all secrets into a gitignored
  `config.js`; commit only a `config.example.js`; have a **GitHub Actions**
  workflow write `config.js` from **repo Secrets** at deploy time; add a guard
  workflow that fails CI if `config.js` is ever committed. This keeps secrets
  out of the repo, history, and PRs (Actions secrets aren't exposed to fork
  PRs).
- **What we shipped:** the owner decided the data isn't critical and chose
  **simplicity** — bundle the config and keys **inline in `index.html`**, no CI,
  Pages deploy-from-branch. This is the current state.
- **The unavoidable truth either way:** on a static site the browser downloads
  everything it runs, so these values can never be *truly* secret from a site
  visitor. The gitignored-config path only hides them from the repo/PRs, not
  from the deployed page.
- **If a future version wants real secrecy:** move auth and DB access behind a
  **server** the client can't bypass — e.g. a **Supabase Edge Function** (or
  Cloudflare Worker) that validates the magic key server-side, holds the
  Supabase service key, and returns only scoped data. Then tighten the RLS
  policies (per-identity instead of `anon` all). That's the "better version"
  upgrade and would also let keys be true secrets injected via env, not shipped
  to the client. GitHub Pages alone can't host that (static only).

---

## 10. Known limitations & roadmap ideas

- **Polling, not realtime.** Upgrade: **Supabase Realtime** subscriptions on
  `nfp_state` and `nfp_messages` for instant updates (requires adding the
  tables to the `supabase_realtime` publication and wiring `.channel()` /
  `.on('postgres_changes', …)`; can replace or supplement the poll loop).
- **Secrets visible in client** (see §9). Upgrade path: Edge Function login
  proxy + per-identity RLS.
- **No message edit/delete or reactions/@mentions.**
- **Last-write-wins on a user's own `nfp_state` row** (fine — single writer).
- **`file://` CORS** can be flaky for Supabase calls; prefer hosting at a URL.
- **No automated tests.**
- Other requested-but-unbuilt ideas: one-tap "whole weekend" fill on MY DAYS,
  iCal / Google Calendar export for locked-in days.

---

## 11. File structure

```
/
├── index.html      # the entire app (config + keys inline)
├── README.md       # setup + Pages deploy instructions + key list
├── .gitignore
└── CLAUDE.md        # this file
```

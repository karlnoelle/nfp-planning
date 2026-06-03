# NFP — Natural Feline Planning 🐈🪐

A tiny summer-planning app for five busy cats. Mark days **free / busy / idk**,
see when the stars align, and chat about it. One static HTML file backed by
[Supabase](https://supabase.com) — **no accounts needed** for the squad, just a
magic key.

Heads up: this is a static site with the config and magic keys baked into
`index.html`, so they're visible to anyone who opens the page. That's fine here
— it's cat schedules, not state secrets.

## Setup

**1. Supabase (one time).** Make a free project, then run this in the SQL Editor:

```sql
create table if not exists nfp_state (
  uid text primary key, days jsonb default '{}'::jsonb,
  vibe text default '', updated_at bigint default 0
);
create table if not exists nfp_messages (
  id text primary key, uid text not null, ts bigint not null, body text not null
);
create index if not exists nfp_messages_ts on nfp_messages (ts);
alter table nfp_state enable row level security;
alter table nfp_messages enable row level security;
create policy "nfp_state anon all" on nfp_state for all to anon using (true) with check (true);
create policy "nfp_messages anon all" on nfp_messages for all to anon using (true) with check (true);
```

**2. Fill in `index.html`.** Near the top, set `SUPABASE_URL` and
`SUPABASE_ANON_KEY` (Supabase → Project Settings → API). Magic keys live in the `USERS` array just below that — tweak names, emojis, or keys there.

**3. Publish on GitHub Pages.** Push this repo (public), then
**Settings → Pages → Build and deployment → Source: Deploy from a branch →
`main` / `root`**. Your site goes live at `https://karlnoelle.github.io/<repo>/`.

Share the URL + each person's magic key. Done.

## The squad's magic keys

| Feline                | Key                |
|-----------------------|--------------------|
| Eeps                  | eeps-zoomies-4f2k  |
| Zarch                 | zarch-vroom-9x7q   |
| Professor Moogenborks | moog-quantum-2v8z  |
| Nat                   | nat-nebula-6k3p    |
| Coral                 | coral-reef-1m9t    |

## Notes

- The app polls for updates (~6s in chat, ~20s elsewhere). Fine for five people.
- Anyone with the URL could read the data if they went digging; the magic-key
  wall just gates who can pick an identity and post.

# SocialSpinner

A small shared web app for social event planning. It has two tabs:

- **Spinner** — add ideas to a wheel and spin to pick one randomly.
- **Whose Hosting Anyway?** — track whose turn it is to host across one or more named topics (e.g. "Monthly Dinner", "Game Night"). Each topic has a list of names; hitting **Next** advances the highlighted person and wraps back to the top.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML/CSS/JavaScript (single `index.html`, no build step) |
| Database & Auth | [Supabase](https://supabase.com) (PostgreSQL + Realtime) |
| Hosting | [GitHub Pages](https://pages.github.com) (served from `main` branch) |
| Supabase client | Loaded from CDN via `https://esm.sh/@supabase/supabase-js@2` |

No npm, no bundler, no framework — open `index.html` in a browser and it works.

---

## Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Production — auto-deployed to GitHub Pages |
| `develop` | Active development — merge to `main` to deploy |

---

## Supabase Setup

### 1. Create a project

Sign up at [supabase.com](https://supabase.com), create a new project, and note your **Project URL** and **anon public key** from Settings → API.

### 2. Paste credentials into `index.html`

Find these two lines near the top of the `<script>` block:

```js
const SUPABASE_URL     = "https://your-project.supabase.co";
const SUPABASE_ANON_KEY = "your-anon-key";
```

Replace the values with your own.

### 3. Create the database tables

Run the following SQL in the Supabase **SQL Editor** (Dashboard → SQL Editor → New query):

```sql
-- ── Spinner ideas ────────────────────────────────────────────────
create table if not exists ideas (
  id         uuid primary key default gen_random_uuid(),
  text       text not null,
  created_at timestamptz default now()
);

alter table ideas enable row level security;
create policy "allow all" on ideas for all using (true) with check (true);

-- ── Hosting topics ───────────────────────────────────────────────
create table if not exists hosting_topics (
  id         uuid primary key default gen_random_uuid(),
  name       text not null,
  created_at timestamptz default now()
);

alter table hosting_topics enable row level security;
create policy "allow all" on hosting_topics for all using (true) with check (true);

-- ── Hosting names (one row per person per topic) ─────────────────
create table if not exists hosting_names (
  id         uuid primary key default gen_random_uuid(),
  topic_id   uuid references hosting_topics(id) on delete cascade,
  name       text not null,
  position   integer not null default 0,
  is_current boolean not null default false,
  created_at timestamptz default now()
);

alter table hosting_names enable row level security;
create policy "allow all" on hosting_names for all using (true) with check (true);

-- ── Enable Realtime on all three tables ──────────────────────────
alter publication supabase_realtime add table ideas;
alter publication supabase_realtime add table hosting_topics;
alter publication supabase_realtime add table hosting_names;
```

> **Note:** The RLS policies above allow anonymous read/write access, which is intentional for this shared app. Tighten them if you need access control.

---

## Hosting on GitHub Pages

1. Merge `develop` → `main`.
2. In the GitHub repo: Settings → Pages → Source → Deploy from branch → `main` / `/ (root)`.
3. The site will be live at `https://<your-org>.github.io/<repo-name>/`.

---

## Feature Notes

### Spinner tab
- Ideas are stored in the `ideas` table.
- All connected users see additions and removals in real time (Supabase Realtime).

### Whose Hosting Anyway? tab
- **Topics** live in `hosting_topics`. Create as many as you like (e.g. one per recurring event).
- **Names** live in `hosting_names`, linked to a topic via `topic_id`.
  - `position` (integer) determines display order; up/down arrows swap adjacent positions.
  - `is_current` (boolean) marks whose turn it is. Exactly one name per topic should be `true` at a time.
  - Hitting **Next** clears `is_current` on the whole topic then sets it on the next name, wrapping from the last name back to the first.
  - The first name added to a topic is automatically marked current.
  - Deleting a topic cascades and removes all its names.
- All changes are real-time; every browser watching the page sees the update immediately.

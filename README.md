# Messenger

A standalone AIM-style messenger. Buddy list, multiple chat windows, minimize,
presence (Available / Away / Sign off), away auto-responses, clickable links,
file & picture sharing, and per-user login passcodes.

It runs entirely from a single `index.html` on GitHub Pages and shares the **same
Supabase project** as the Task Board — so the buddy list, logins, messages, and
presence are identical across both apps. Sign in on one, you're the same person on
the other.

## Deploy

1. Create a new GitHub repo (e.g. `messenger`) and add this `index.html`.
2. Settings → Pages → deploy from `main` / root. Your link is
   `https://<you>.github.io/messenger/`.
3. Open `index.html` and confirm the `CONFIG` block (near the bottom) has the same
   Supabase **Project URL** and **anon public** key as your Task Board. It's
   pre-filled with your current values.
4. Commit and push. Done.

## Shared backend

This app expects the same `users` and `messages` tables the Task Board already
uses, plus an `attachments` storage bucket for shared files/pictures. If they
already exist (they do, if the Task Board works), you don't need to do anything.

For a brand-new Supabase project, run this once in **SQL Editor**:

```sql
create extension if not exists pgcrypto;

create table if not exists users (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  username text,
  email text,
  title text,
  color text,
  last_seen timestamptz,
  status text,
  away_message text,
  pin_hash text,
  created_at timestamptz default now()
);

create table if not exists messages (
  id uuid primary key default gen_random_uuid(),
  from_user uuid references users(id) on delete cascade,
  to_user uuid references users(id) on delete cascade,
  body text,
  att_path text,
  att_name text,
  att_mime text,
  read boolean default false,
  created_at timestamptz default now()
);

alter table users enable row level security;
alter table messages enable row level security;
create policy "open" on users for all using (true) with check (true);
create policy "open" on messages for all using (true) with check (true);
```

If you're reusing the Task Board's project and only need the presence + passcode
columns, this is enough:

```sql
alter table users add column if not exists status text;
alter table users add column if not exists away_message text;
alter table users add column if not exists pin_hash text;
```

For file/picture sharing, create a **public** Storage bucket named `attachments`
(Storage → New bucket → name `attachments`, Public).

## Features

- **Buddy list** grouped into Online / Offline, with status dots (green active,
  yellow idle/away, red offline) and unread badges.
- **Multiple chat windows** open at once, docked bottom-right; each minimizes to a
  title bar and restores on click. Unsent text is preserved.
- **Presence:** Available, Away (with a custom message), or Sign off (appear
  offline). Away shows a truncated note under the buddy's name — hover for the full
  text — and auto-replies your away message when someone messages you.
- **Clickable links** in messages.
- **File & picture sharing** via the 📎 button (max 10 MB).
- **Login passcodes:** picking a profile requires that person's passcode, so nobody
  can log in as someone else.

## Notes on security

Because the app talks to Supabase with the public anon key, passcode checks happen
in the browser. This stops casual impersonation but isn't hardened against a
determined attacker reading network traffic. For true enforcement, move to Supabase
Auth with row-level security.

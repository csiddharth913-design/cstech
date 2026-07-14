# CS Tech Contact Tracker — Supabase Setup Guide

Follow these steps exactly, in order. It takes about 10 minutes.

---

## Step 1 — Create a free Supabase account

1. Go to **https://supabase.com** and click **Start your project**
2. Sign in with GitHub (easiest) or create an account
3. Click **New project**
4. Fill in:
   - **Name:** `cs-contact-tracker` (or anything you like)
   - **Database password:** choose something strong and save it somewhere
   - **Region:** pick the one closest to you
5. Click **Create new project** — wait ~1 minute for it to provision

---

## Step 2 — Create the database tables

1. In your Supabase project, click **SQL Editor** in the left sidebar
2. Click **New query**
3. Paste the entire block below and click **Run**

```sql
-- Contacts table
create table contacts (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  title text,
  company text,
  linkedin text,
  rel text,
  connection text,
  trigger text,
  last_touch_date date,
  last_touch_type text,
  lifecycle text,
  priority text,
  assigned text,
  touch_type text,
  engagement text,
  notes text,
  created_at timestamptz default now()
);

-- Interaction log table
create table interaction_log (
  id uuid primary key default gen_random_uuid(),
  date date not null,
  contact_name text not null,
  type text,
  template_id uuid references templates(id) on delete set null,
  response text,
  notes text,
  created_at timestamptz default now()
);

-- Templates table
create table templates (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  body text,
  created_at timestamptz default now()
);

-- Settings table (stores config as JSON — one row, id=1)
create table settings (
  id int primary key default 1,
  config jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);

-- Insert default settings row
insert into settings (id, config) values (1, '{}'::jsonb)
on conflict (id) do nothing;
```

> **Note:** If you get a foreign key error on `interaction_log`, create `templates` first.
> Run this order if needed: first create `templates`, then `contacts`, then `interaction_log`, then `settings`.

---

## Step 3 — Disable Row Level Security (for internal team use)

By default Supabase blocks all reads/writes. Since this is an internal tool (not public), the easiest approach is to allow all operations via the anon key.

Run this in the SQL Editor:

```sql
alter table contacts enable row level security;
alter table interaction_log enable row level security;
alter table templates enable row level security;
alter table settings enable row level security;

create policy "allow all" on contacts for all using (true) with check (true);
create policy "allow all" on interaction_log for all using (true) with check (true);
create policy "allow all" on templates for all using (true) with check (true);
create policy "allow all" on settings for all using (true) with check (true);
```

---

## Step 4 — Get your API credentials

1. In your Supabase project, click **Settings** (gear icon, bottom of left sidebar)
2. Click **API**
3. You need two values:
   - **Project URL** — looks like `https://xyzabcdef.supabase.co`
   - **anon / public key** — a long string starting with `eyJ…`

---

## Step 5 — Add credentials to supabase.js

Open `supabase.js` in your repo and replace the placeholder values:

```js
const SUPABASE_URL     = 'https://xyzabcdef.supabase.co';   // ← your actual URL
const SUPABASE_ANON_KEY = 'eyJhbGci...';                    // ← your actual anon key
```

Save the file.

---

## Step 6 — Push to GitHub and enable GitHub Pages

Your repo should contain exactly these three files:
```
index.html
supabase.js
SETUP.md        ← optional, can delete after setup
```

1. Commit and push both `index.html` and `supabase.js`
2. In your GitHub repo → **Settings** → **Pages**
3. Under **Source**, select **Deploy from a branch**
4. Choose **main** (or master) branch, **/ (root)** folder
5. Click **Save**
6. Wait ~1 minute, then visit the URL shown (e.g. `https://yourname.github.io/repo-name/`)

The app will load, connect to Supabase, and show an empty tracker ready to use.

---

## Step 7 — Seed your existing data (optional)

If you want to pre-load the 10 sample contacts, run this in the Supabase SQL Editor:

```sql
insert into templates (name, body) values
('Past client – Check-in', 'Hi [Name],

Hope you''re well. I was thinking about the work we did together on [project] and wanted to reach out to see how things are going on your end.

We''ve recently completed a similar engagement for [relevant client type] — happy to share notes if ever useful.

Best,
[Sender Name]'),
('Job change – Congratulation', 'Hi [Name],

Saw you''ve moved to [New Company] — congratulations! [New Company] is doing interesting things in [space].

Would love to stay in touch as you settle in.

Best,
[Sender Name]'),
('Relevant content share', 'Hi [Name],

We just published a case study on [topic] — given your work in [their area], thought it might be worth a read.

[Link]

No agenda — just thought it might be useful.

Best,
[Sender Name]');
```

---

## How it works after setup

| Action | What happens |
|--------|-------------|
| Add / edit a contact | Saved to Supabase instantly |
| Log an interaction | Saved to Supabase, contact's last-touch date updates |
| Add / edit a template | Saved to Supabase |
| Change settings | Saved to Supabase when you click **Save settings** |
| Another team member opens the URL | They see the same data in real time |
| Someone else makes a change | Refresh the page to see their changes |

---

## Security note

The anon key in `supabase.js` is **visible in your public GitHub repo**. This is fine for an internal team tool because:
- Row Level Security (Step 3) controls what the key can access
- The anon key cannot access your database password or admin functions
- If you want stricter access, you can add Supabase Auth (login with email/password) — ask for help if needed


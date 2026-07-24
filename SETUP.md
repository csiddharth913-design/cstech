# CS Tech Contact Tracker — Full Setup Guide

---

## What this app is

A shared contact tracker for the team: contacts, interaction history, follow-up templates, a lightweight sales pipeline, priority scoring, and two AI-assisted features (an interaction summary and a web-researched company/contact brief). It's a static site (just `index.html`) backed by Supabase for data storage and two small serverless functions for the AI features.

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

## Step 2 — Create the full database schema

In your Supabase project, click **SQL Editor** in the left sidebar → **New query** → paste the entire block below → **Run**.

```sql
-- Templates (created first — interaction_log references it)
create table templates (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  body text,
  created_at timestamptz default now()
);

-- Contacts
create table contacts (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  title text,
  company text,
  email text,
  linkedin text,
  rel text,
  connection text,
  trigger text,
  last_touch_date date,
  last_touch_type text,
  lifecycle text,
  priority text,
  priority_score int default 0,
  priority_criteria jsonb default '[]'::jsonb,
  assigned text,
  assignees jsonb default '[]'::jsonb,
  touch_type text,
  engagement text,
  notes text,
  birthday date,
  important_dates jsonb default '[]'::jsonb,
  industry text,
  pipeline_stage text,
  deal_value numeric,
  next_action text,
  ai_summary text,
  ai_summary_updated_at timestamptz,
  ai_enrichment text,
  ai_enrichment_updated_at timestamptz,
  created_at timestamptz default now()
);

-- Interaction log
create table interaction_log (
  id uuid primary key default gen_random_uuid(),
  date date not null,
  contact_name text not null,
  type text,
  template_id uuid references templates(id) on delete set null,
  reply_to_id uuid references interaction_log(id) on delete set null,
  logged_by text,
  message text,
  response text,
  reply_message text,
  reply_date date,
  notes text,
  created_at timestamptz default now()
);

-- Comments on individual interaction log entries
create table log_comments (
  id uuid primary key default gen_random_uuid(),
  log_id uuid references interaction_log(id) on delete cascade,
  author text,
  comment text not null,
  created_at timestamptz default now()
);

-- Settings (stores config as JSON — one row, id=1)
create table settings (
  id int primary key default 1,
  config jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);
insert into settings (id, config) values (1, '{}'::jsonb)
on conflict (id) do nothing;
```

---

## Step 3 — Set up Row Level Security

By default Supabase blocks all reads/writes. Since this is an internal team tool (not public), the simplest approach is to allow all operations via the anon key. Run this in the SQL Editor:

```sql
alter table contacts enable row level security;
alter table interaction_log enable row level security;
alter table templates enable row level security;
alter table settings enable row level security;
alter table log_comments enable row level security;

create policy "allow all" on contacts for all using (true) with check (true);
create policy "allow all" on interaction_log for all using (true) with check (true);
create policy "allow all" on templates for all using (true) with check (true);
create policy "allow all" on settings for all using (true) with check (true);
create policy "allow all" on log_comments for all using (true) with check (true);

grant usage on schema public to anon, authenticated;
grant select, insert, update, delete on contacts, interaction_log, templates, settings, log_comments to anon, authenticated;
```

---

## Step 4 — Get your API credentials

1. In your Supabase project, click **Settings** (gear icon, bottom of left sidebar) → **API**
2. You need two values:
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

Your repo should contain:
```
index.html
supabase.js
SETUP.md              ← optional, can delete after setup
PROVIDER_SWAPS.md     ← optional, can delete after setup
```

1. Commit and push
2. In your GitHub repo → **Settings** → **Pages**
3. Under **Source**, select **Deploy from a branch**
4. Choose **main** (or master) branch, **/ (root)** folder → **Save**
5. Wait ~1 minute, then visit the URL shown (e.g. `https://yourname.github.io/repo-name/`)

The app will load, connect to Supabase, and show an empty tracker ready to use. Everything through here is enough to run the app — the AI features below are additional, but recommended.

---

## Step 7 — Set up the AI features (summaries + research)

Two things live on each contact card: an **AI summary** of the interaction history, and an **AI research** brief on the company/contact (industry, recent news, growth signals). Both run on one free Groq API key.

### 7a. Get a free Groq API key

1. Go to **https://console.groq.com** and sign up (no credit card needed)
2. Click **API Keys** → **Create API Key** → copy it

### 7b. Deploy the two functions

These instructions assume deploying through the Supabase dashboard (Edge Functions → Deploy a new function), since that's the most common path if you don't have the Supabase CLI installed. If you do use the CLI, the equivalent commands are `supabase functions deploy <name>` and `supabase secrets set GROQ_API_KEY=...`.

1. In your Supabase project, go to **Edge Functions** in the left sidebar → **Deploy a new function**
2. Name it **exactly** `summarize-contact` (this must match exactly — the app calls this URL directly, and even small differences like a British-vs-American spelling will silently 404 (I made this mistake))
3. Paste in the full contents of `supabase/functions/summarize-contact/index.ts` and deploy
4. Repeat for a second function named **exactly** `ai-enrich-contact`, using `supabase/functions/ai-enrich-contact/index.ts`
5. Turn off JWT verification for both. Depending on where you find it, this is either a toggle labeled **"Enforce JWT verification"** at creation/in the function's settings, or — if deploying via CLI — the `supabase/config.toml` file included alongside this guide. Both functions are called directly from the browser with no login step, so they need this off; otherwise every request is blocked before your code ever runs.
6. Go to **Secrets** in the left sidebar → add `GROQ_API_KEY` with the key from step 7a. One key powers both functions.

### 7c. Verify it

Open the app, open any contact with some interaction history, and click **Generate summary**. Then try **Enrich with AI** on any contact with a company name. Both should return text within a few seconds.

---

## How it works day to day

| Action | What happens |
|---|---|
| Add / edit a contact | Saved to Supabase instantly |
| Log an interaction | Saved to Supabase; shows as **"Awaiting reply"** until a reply is logged |
| Log a reply | Use the **Log reply** action on an interaction to record the response type, date, and text |
| Edit a log entry | Every interaction has an Edit action, so details can be corrected or filled in later |
| Click an interaction | Opens the full thread it belongs to (replies chained via `reply_to_id`), with per-entry team comments |
| Comment on a log entry | Saved to `log_comments`, visible to the whole team |
| Add / edit a template | Saved to Supabase |
| Change settings | Saved when you click **Save settings** (an "● Unsaved changes" badge shows if you haven't yet) |
| Priority scoring | Calculated automatically from a weighted checklist (editable under Settings) rather than picked manually |
| Assignees | Multi-select checklist rather than a single owner |
| Pipeline | Stage / deal value / next action per contact; a funnel + total value on the Summary tab |
| Key dates | Birthday + any number of custom dates, surfaced as upcoming reminders within 30 days |
| Click "Generate summary" on a contact card | Calls `summarize-contact`, which asks Groq to summarize that contact's interaction history |
| Click "Enrich with AI" on a contact card | Calls `ai-enrich-contact`, which uses Groq's web-search-enabled model to research the company + contact |
| Another team member opens the URL | They see the same data in real time |
| Someone else makes a change | Refresh the page to see it |

---

## Troubleshooting

The app shows a **red banner at the top of the screen** (not just a quick toast) whenever a save fails, with the exact error message — read that first.

**"I can't add contacts" / "settings don't save"**

1. *You edited Settings but didn't click Save settings* — editing tags/fields updates them in your browser only until you press the button. Watch for the "● Unsaved changes" badge.
2. *Row Level Security is blocking writes* — if you ever drop and recreate a table, RLS resets. Re-run the Step 3 policy SQL (the `create policy` statements will fail with "already exists" if nothing changed — that's fine, or add `drop policy if exists "allow all" on <table>;` before each one if you need to re-run it safely).
3. *Missing grants* — re-run the `grant` lines from Step 3, harmless if they already exist.
4. *Check the browser console* (F12 → Console) — the exact Supabase error is logged there. Common ones:
   - `permission denied for table contacts` → re-run Step 3
   - `new row violates row-level security policy` → same fix
   - `null value in column "name" violates not-null constraint` → the Name field was empty
   - `Failed to fetch` → check your internet connection, or that `SUPABASE_URL` in `supabase.js` is correct

**AI features: "network error" or CORS error on Generate summary / Enrich with AI**

1. Open DevTools (F12) → **Network** tab, retry the action, click the failed request, check its **Response** tab for the actual error text — that's the fastest way to know what's actually wrong.
2. **404 on the function URL** → the function name deployed doesn't exactly match what the app calls (`summarize-contact` / `ai-enrich-contact`). Check Edge Functions in the dashboard — click into the function and confirm its real URL matches exactly (watch for spelling drift, e.g. "summarize" vs "summarise").
3. **401 / "Invalid JWT" / CORS preflight failure** → JWT verification is still on for that function. Turn it off (see Step 7b.5).
4. **"Request too large" from the AI research feature** → this is a real Groq-side limit occasionally hit by the web-search model on a very open-ended query; the function is already configured to use lighter "basic search" and a single search pass to avoid this, but if it recurs, it's worth trying again — it's not something wrong with your setup.
5. **GROQ_API_KEY not set** → check **Edge Functions → Secrets** in the dashboard.

---

## Security note

- The Supabase anon key in `supabase.js` is **visible in your public GitHub repo**. This is fine for an internal team tool because Row Level Security (Step 3) controls what it can access, and it can't touch your database password or admin functions. For stricter access, add Supabase Auth (login with email/password).
- The Groq API key is **not** in the repo — it's stored as a Supabase Edge Function secret, and the two AI functions act as a proxy so the browser never sees it.
- If you later want to swap any of these providers out, see `PROVIDER_SWAPS.md`.

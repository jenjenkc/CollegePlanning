# Senior Year Command Center

Single-file web app (`index.html`) for tracking college applications, deadlines, tasks, a calendar, and now an inbox for forwarded college emails — shared live between you and Savannah.

## Publish on GitHub Pages
1. Create a repo (e.g. `college-dashboard`) and add `index.html` and this README.
2. Settings → Pages → Deploy from branch → `main` / root.
3. Your site is live at `https://<username>.github.io/<repo>/`.

## Turn on shared live sync (you + Savannah, same live data)
Without this, each of you sees your own private copy in your own browser, and forwarded emails have nowhere to land.

1. Go to supabase.com → New project (free tier is enough). Save the database password somewhere safe.
2. In the SQL Editor, run:
   ```sql
   create table dashboard_state (
     id text primary key,
     data jsonb not null,
     updated_at timestamptz default now()
   );
   alter table dashboard_state enable row level security;
   create policy "read" on dashboard_state for select using (auth.role() = 'authenticated');
   create policy "insert" on dashboard_state for insert with check (auth.role() = 'authenticated');
   create policy "update" on dashboard_state for update using (auth.role() = 'authenticated');
   alter publication supabase_realtime add table dashboard_state;

   create table raw_inbox (
     id uuid primary key default gen_random_uuid(),
     received_at timestamptz default now(),
     from_email text,
     subject text,
     college text,
     body text,
     status text default 'new',
     created_at timestamptz default now()
   );
   alter table raw_inbox enable row level security;
   create policy "read inbox" on raw_inbox for select using (auth.role() = 'authenticated');
   create policy "update inbox" on raw_inbox for update using (auth.role() = 'authenticated');
   create policy "insert inbox" on raw_inbox for insert with check (true);
   alter publication supabase_realtime add table raw_inbox;
   ```
3. Go to Authentication → Users → Add user, and create one login for yourself and one for Savannah (any email + password works).
4. Go to Project Settings → API. Copy the **Project URL** and the **anon public** key.
5. Open `index.html` and find this block near the top of the `<script>` section:
   ```js
   const SUPABASE_URL='';
   const SUPABASE_ANON_KEY='';
   ```
   Paste your Project URL and anon key between the quotes, save, and re-upload to GitHub.
6. Open the live link. You'll both see a sign-in screen. Once signed in, changes either of you makes appear for the other within a second or two ("🟢 Synced" in the header).

If you'd rather send me the Project URL and anon key, I can drop them into the file for you.

## Getting college emails into the Inbox tab
The Inbox tab reads directly from the `raw_inbox` table above, so anything written there shows up live for both of you — with buttons to turn an email into a task or a calendar event, or dismiss it.

**Forwarding setup (no code), using Zapier or Make.com:**
1. Create one dedicated address for this — a new Gmail address, or a forwarding rule from Savannah's school email — e.g. `savannah.colleges@gmail.com`.
2. In Zapier (or Make), create a Zap: **Trigger** = "New Email" on that inbox. **Action** = Supabase "Create Row", table `raw_inbox`, mapping the email's subject/from/body into the matching columns.
3. Test it by sending yourself a sample email — it should appear in the app's Inbox tab within moments.
4. You'll still tag which college it's about and decide whether it becomes a task or event — this captures every email automatically, but doesn't guess the college or deadline for you. (That extra step is possible later with an AI-parsing add-on, but adds cost and complexity — worth doing once you're using the basic version regularly.)
5. Until you set up forwarding, use the **Paste an email** button in the Inbox tab to add one by hand.

**One trade-off to know:** the `insert inbox` rule above allows anyone with the app's public key to add a row (this is what lets Zapier write to it without needing its own login). That key is visible in the page source, same as the rest of the app. The realistic risk is someone submitting junk rows, not reading your existing data — reading and editing are still locked to signed-in accounts. If that ever becomes a problem, tell me and I'll switch it to a private webhook with a shared passcode instead.

## Before selling this to other students
This version is built for exactly two people sharing one record. Turning it into a real multi-family product needs per-family accounts and data isolation (not just two shared logins), a privacy policy, and parental consent for under-13 users (COPPA), since these are minors' education records. Ask me when you're ready for that version — it's a bigger change than this one.

# Pulse — personal fitness tracker

A private, installable fitness app: workouts & live workout mode, rest timer, exercise library (~130 exercises), programs & goals, calendar, running tracker (GPS optional), stopwatch, body progress & photos, nutrition, meal ideas (Filipino‑friendly), meal planner, water, habits, weekly/monthly statistics and personal records.

Built with **React 19 + TypeScript + Vite** and **Supabase** (Auth, Postgres with Row Level Security, private Storage). Deploys to **Netlify**. Installs to your phone's home screen as a **PWA**.

---

## Setup (about 15 minutes)

### 1. Create the Supabase project
1. Go to [supabase.com](https://supabase.com) → **New project** (free tier is fine). Pick a region close to you (e.g. Singapore).
2. Open **SQL Editor → New query**, paste the whole of [`supabase/schema.sql`](supabase/schema.sql) and click **Run**. It creates every table, the Row Level Security policies, the workout save function and the private `progress-photos` storage bucket. It is safe to run again.
3. Open **Project Settings → API** and copy the **Project URL** and the **anon public** key.

### 2. Run it locally (optional)
Requires Node 18.18+.
```bash
cp .env.example .env        # then paste your URL and anon key into .env
npm install
npm run dev                 # open the printed http://localhost:5173
```
Set `VITE_ALLOW_SIGNUP=true` in `.env` the first time so the **Create account** button appears.

### 3. Create your account
1. Open the app → **Create account** → enter your email and a password (8+ characters).
2. Supabase sends a confirmation email by default — click the link, then sign in.
   (To skip email confirmation: Supabase → **Authentication → Providers → Email** → turn off *Confirm email*.)

### 4. Lock it down to just you
1. Supabase → **Authentication → Providers → Email** → turn **off “Allow new users to sign up”** (in newer dashboards this is under **Authentication → Sign In / Providers → User Signups**).
2. Set `VITE_ALLOW_SIGNUP=false` (or delete it) in `.env` / Netlify.

Even if someone found your site and the public anon key, Row Level Security means the database only ever returns rows owned by the logged‑in user.

### 5. Deploy to Netlify
1. Push this folder to a GitHub repository (or drag‑and‑drop a built `dist/` — but Git is easier for updates).
2. Netlify → **Add new site → Import an existing project** → pick the repo. Build settings come from `netlify.toml` (`npm run build`, publish `dist`).
3. **Site configuration → Environment variables** → add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` (and `VITE_ALLOW_SIGNUP=false`). Redeploy.
4. Supabase → **Authentication → URL Configuration** → set **Site URL** to your Netlify URL (needed for password‑reset links).

### 6. Install on your phone
- **Android (Chrome):** open the site → menu ⋮ → **Install app** / **Add to Home screen**.
- **iPhone (Safari):** open the site → Share → **Add to Home Screen**.

---

## How your data is kept safe and in sync
- **Supabase is the source of truth.** Every add/edit/delete is written to your database; data is there after refresh, browser restart and on any device you sign in on.
- **Sync between devices:** the app reloads your data when you return to it, when the connection comes back, and every few minutes while open.
- **No accidental duplicates:** every record gets its ID on the device and is saved with an *upsert*, so a retry updates the same row instead of creating another. Workouts are saved through one database function that replaces the workout's exercises and sets in a single transaction.
- **Offline:** an “Offline” badge and banner appear. New entries are kept on the device, marked **“N to sync”**, and uploaded automatically when you reconnect. Nothing is shown as saved to your account until it actually is. Progress photos need a connection to upload.
- **Deleting** asks for confirmation and removes the row from Supabase (child rows cascade). Settings → Data → *Export all data (JSON)* gives you a full backup.
- localStorage is only used for: the login session, a read cache for fast/offline start‑up, the pending‑sync queue, your theme, and in‑progress timers/live workouts (so a phone screen lock or refresh doesn't lose them).

## Project structure
```
supabase/schema.sql        database: tables, RLS, save_workout(), storage bucket
public/                    PWA manifest, service worker, icons, Netlify _redirects
src/
  main.tsx, App.tsx        entry, providers, auth gate, routes
  router.tsx               tiny history-API router (Link, useRouter)
  pages/                   one file per screen
  components/              Icon, RestTimer, GlobalSearch
    ui/                    Button, Field/Select/Toggle…, Modal, Confirm, Card, Badge…
    layout/AppShell.tsx    sidebar (desktop), bottom nav (mobile), sync status
    charts/Charts.tsx      Ring, LineChart, BarChart, Heatmap, HBars (SVG, no deps)
    workout/               exercise picker/editor, date dialog, draft model
  hooks/                   useAuth, useData (synced store + offline queue), useUnits, useTimer, useTheme…
  services/                supabase client, api (CRUD, paging past 1000 rows), storage (photos)
  utils/                   dates, units, validation, calculations, stats & PRs
  data/                    exercise library, programs, goals, meal ideas
  styles/index.css         the design system (light/dark tokens)
```
See [ARCHITECTURE.md](ARCHITECTURE.md) for the schema, routes and data flow.

## Scripts
- `npm run dev` – local dev server
- `npm run build` – production build to `dist/`
- `npm run preview` – serve the build locally
- `npm run typecheck` – optional TypeScript check

## Notes
- Calorie burn, running calories and suggested nutrition targets are rough estimates for general guidance, not medical advice.
- GPS tracking runs in the browser, so keep the screen on during a tracked run (the app requests a screen wake lock where supported). Distance can always be entered manually instead.

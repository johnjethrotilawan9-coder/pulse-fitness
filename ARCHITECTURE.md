# Architecture

## Overview
```
Phone / computer (React PWA)                       Supabase
┌──────────────────────────────────┐   HTTPS   ┌────────────────────────────┐
│ pages → hooks/useData (store)    │ ───────▶ │ Auth (email + password)    │
│   ├─ save/remove/saveWorkout     │           │ Postgres + Row Level Sec.  │
│   ├─ offline queue (localStorage)│ ◀─────── │ save_workout() function    │
│   └─ read cache (localStorage)   │           │ Storage: progress-photos   │
│ service worker: app shell only   │           └────────────────────────────┘
└──────────────────────────────────┘
```

## Authentication & private access
- Supabase Auth email/password; session persisted by supabase-js and refreshed automatically.
- `App.tsx` gate: no session → Login; session → `DataProvider` loads that user's data.
- Every table has `user_id` and a policy `using (user_id = auth.uid()) with check (user_id = auth.uid())`, `force row level security`, and `anon` has no grants. Child tables (`workout_exercises`, `workout_sets`, `habit_logs`) also have a restrictive policy requiring the parent to belong to the same user.
- `save_workout()` runs as the caller (security invoker), so RLS still applies; it refuses to overwrite another user's workout.
- Progress photos live in a private bucket under `<user_id>/…`; storage policies only allow that folder. Photos are shown via short-lived signed URLs.
- Sign-up is hidden unless `VITE_ALLOW_SIGNUP=true`; disable sign-ups in Supabase after creating your account.

## Synchronisation
1. After login the store loads all tables (paging in 1000-row chunks) and the profile.
2. Writes: `save()` / `saveWorkout()` → Supabase first; on success the local store updates. Client-generated UUIDs + upsert make every write idempotent.
3. Offline or network failure: the operation is applied locally, appended to a per-user queue in localStorage, and shown as “N to sync”. The queue replays in order on reconnect/focus/interval; a server rejection (e.g. validation) is reported and dropped rather than retried forever.
4. Re-sync on tab focus, `online` event and every 3 minutes → phone and computer converge.

## Database (supabase/schema.sql)
| Table | Purpose | Key relationships |
|---|---|---|
| profiles | name, height, weights, units, goals[], all targets, theme, toggles | PK user_id → auth.users |
| workouts | date, name, type, status, times, duration, muscles[], distance, calories, difficulty, energy, mood, notes, volume | user |
| workout_exercises | exercise in a workout, order, targets, rest, skipped | → workouts (cascade) |
| workout_sets | reps, weight, duration, distance, completed | → workout_exercises (cascade) |
| custom_exercises | your own library entries | user |
| favorite_exercises | starred exercise keys | unique (user, key) |
| workout_programs | custom programs; `days` jsonb of day → exercises | user |
| calendar_events | rest days, notes, other activities | user |
| running_sessions | run/jog/walk: distance, duration, pace, calories, GPS route & splits, planned/completed | user |
| body_measurements | weight, body fat, waist/chest/arms/thighs/hips/neck | user |
| progress_photos | storage path + date + caption | user, storage bucket |
| daily_metrics | steps, sleep per day | unique (user, date) |
| nutrition_entries | food log per meal (6 meal slots) with macros | user |
| foods | saved “My foods” | user |
| meal_plans / meal_templates | weekly planner and reusable meals | user |
| water_entries | each drink, per day (history kept) | user |
| habits / habit_logs | habits and daily check-offs | logs → habits (cascade), unique (habit, date) |
| personal_records | manual PRs (automatic PRs are computed) | user |

All tables: `id uuid`, `user_id`, `created_at`, `updated_at` (trigger), CHECK constraints for ranges (no negative durations, valid percentages, etc.). Values are stored metric; the UI converts to kg/lb, km/mi, cm/in.

The built-in exercise library (~130 exercises) and ready-made programs ship with the app (`src/data`) so they work offline; your custom exercises/programs are in the database.

## Routes
`/` dashboard · `/workouts` · `/workouts/new` · `/workouts/:id` · `/workouts/:id/edit` · `/live/:id` (full-screen live mode) · `/calendar` · `/run` · `/exercises` · `/programs` · `/nutrition` · `/meals` · `/water` · `/habits` · `/progress` · `/stats` · `/records` · `/timers` · `/settings` · `/more` · `/login`

Deep links used across the app: `/nutrition?add=1`, `/progress?add=1&date=…`, `/run?id=…`, `/run?plan=…`, `/calendar?date=…`, `/exercises?ex=…`, `/programs?id=…|start=…`, `/workouts/new?date=…`.

## Main components
- `AppShell` – sidebar (≥1024px) / top bar + 5-tab bottom nav (mobile), sync & offline status, global search (Ctrl/⌘+K).
- UI kit – `Button`, `Input`/`NumberInput`/`Select`/`Textarea`/`Toggle`/`Segmented`/`ChipSelect`/`Rating`, `Modal` (bottom sheet on phones), `ConfirmProvider`, `Card`, `Stat`, `Badge`, `StatusBadge` (icon + text, never colour alone), `EmptyState`, `Tabs`, `SearchBox`.
- Charts – `Ring`, `LineChart`, `BarChart`, `Heatmap`, `HBars` (SVG, measured to container width).
- Workout – `ExercisePicker`, `ExerciseEditor`, `DateDialog`; `RestTimer`; `Stopwatch`.

## Implementation phases (as built)
1. Architecture & tooling · 2. Schema + RLS (tested on Postgres 16) · 3. Auth · 4. Layout/navigation · 5. Dashboard · 6. Workout tracker · 7. Live mode + rest timer · 8. Exercise library + programs/goals · 9. Calendar · 10. Running + stopwatch · 11. Nutrition + meal ideas + planner · 12. Body progress + charts + photos · 13. Water + habits · 14. Weekly/monthly stats + PRs · 15. Settings · 16. PWA · 17. End-to-end browser tests on phone and desktop sizes.

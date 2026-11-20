# CampusConnect

A working prototype of your final-year project: **skill-matrix team formation +
DAG-based task/dependency tracking + activity telemetry + AI contribution
reports**, built with **Next.js + Supabase** instead of the MEAN stack — same
ideas, far less setup, and you can run all of it on your own laptop.

No deployment required. Everything runs locally with `npm run dev`, talking to
a free Supabase project for the database + auth.

---

## What's actually implemented

| Module from your abstract | How it's implemented here |
|---|---|
| Multi-attribute skill-matrix team formation | Greedy balanced-assignment algorithm in `lib/teamFormation.ts` |
| DAG task scheduling + timeline drift | Topological sort + critical path (longest path) in `lib/dag.ts` |
| Live activity telemetry | **Real GitHub commit sync** (`lib/github.ts`, `app/api/github-sync`) pulls actual commits by matching `github_username`, plus a manual "log session" form for non-code work |
| Real-time board updates | Supabase Realtime — task board and activity count update live across tabs the moment data changes, no page refresh needed |
| AI-driven domain extraction | `app/api/extract-skills` — reads a project description and suggests required skills (AI if you set a key, keyword-match fallback if not) |
| Candidate search/discovery | `/students` page — filter eligible students by skill domain and minimum level |
| AI report generation | `app/api/report/route.ts` — calls the Claude API if you provide a key, otherwise falls back to a rule-based summary so it still works with zero API cost |
| Academic gatekeeper (attendance/backlogs filter) | Built into `formTeams()` and the candidate search — students below 70% attendance or with backlogs are excluded automatically |
| Angular Kanban UI | Built in React/Next.js instead — a 3-column Kanban board with drift/critical-path highlighting |

**Still not built:** a VS Code IDE extension (the GitHub commit sync above is the practical substitute — see "Troubleshooting" for why), and a genetic-algorithm variant of team matching (greedy is used instead).

---

## 1. Install prerequisites (one-time)

You need **Node.js 18 or later**. Check with:
```bash
node -v
```
If you don't have it, download from https://nodejs.org (LTS version).

---

## 2. Create your free Supabase project (one-time, ~3 minutes)

1. Go to https://supabase.com and sign up (free tier is enough).
2. Click **New project**. Pick any name/region, set a database password (save it somewhere).
3. Once it's created, go to **Project Settings → API**. You'll need three values:
   - `Project URL`
   - `anon public` key
   - `service_role` key (click "reveal" — keep this one secret, never share it)
4. Go to the **SQL Editor** (left sidebar) → **New query**.
5. Open `supabase/schema.sql` from this project, copy the whole file, paste it into the SQL editor, and click **Run**.
   This creates all the tables (profiles, projects, teams, tasks, task_dependencies, activity_logs, reports).

---

## 3. Configure the project

In the project folder, copy the example env file:
```bash
cp .env.example .env.local
```
Open `.env.local` and paste in the three Supabase values from step 2:
```
NEXT_PUBLIC_SUPABASE_URL=https://your-project-ref.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-public-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
ANTHROPIC_API_KEY=
```
Leave `ANTHROPIC_API_KEY` blank if you don't have one — the AI report step will
still work, just using a simpler built-in summary instead of an LLM. If you do
want real LLM-written reports, get a key from https://console.anthropic.com
and paste it in.

---

## 4. Install and run

```bash
npm install
npm run dev
```
Open **http://localhost:3000** in your browser. That's it — no build, no deploy, no server to manage.

---

## 5. How to actually demo it

1. **Sign up** 4–8 times with different accounts (as different "students") — role = Student. On each account, go to **Profile** and set different skill levels (frontend/backend/design/etc), attendance, and backlogs, so the matching algorithm has something real to work with.
2. On any account, go to **Dashboard → New project**. Give it required skills like `frontend, backend, design` and a team size (e.g. 4).
3. Open the project → click **Run team formation**. Watch it auto-create balanced teams from the eligible student pool.
4. Inside a team, **add tasks** with durations and dependencies (e.g. "Design mockups" → "Build frontend" depends on it). The board will show which tasks sit on the **critical path** and flag any that are **drifting** (past their planned finish date and not done). Open the same project in two browser tabs and mark a task done in one — it updates live in the other tab with no refresh (Supabase Realtime).
5. Go to each student's **Profile** and set a real GitHub username (any public GitHub account works, including your own).
6. Back in the project, under **GitHub commit telemetry**, enter a public repo as `owner/repo` and click **Sync commits** — real commits by matching usernames become activity log entries automatically.
7. Try **Find candidates** in the nav bar — filter eligible students by skill domain and minimum level.
8. On **New project**, type a description and click **Suggest from description** to auto-fill required skills.
9. Click **Generate report** — it aggregates activity (manual + GitHub) and task completion into a written contribution report per student.

### If you already ran `schema.sql` before this update

Run `supabase/migration_2_github_sync.sql` once in the Supabase SQL Editor — it adds two new columns and turns on Realtime. Fresh installs don't need this, it's already in `schema.sql`.

---

## 6. Project structure

```
app/
  login/, signup/          → auth pages
  dashboard/                → project list + create
  profile/                  → skill matrix editor
  projects/[id]/             → team formation, Kanban/DAG board, activity log, report
  api/
    team-formation/          → runs the matching algorithm, persists teams
    tasks/                    → create tasks, GET returns scheduled+drift+critical path
    activity/                 → log/read activity sessions
    report/                   → aggregates data, calls AI (or fallback), saves report
lib/
  teamFormation.ts            → the skill-matrix matching algorithm
  dag.ts                       → topological sort, critical path, drift detection
  supabaseClient.ts            → browser Supabase client
  supabaseServer.ts            → server-side Supabase client (service role)
supabase/
  schema.sql                   → run this once in the Supabase SQL editor
```

---

## 7. Ideas for extending it (good for your report/viva)

- **Real telemetry instead of manual logging**: replace the "Log activity" form
  with a scheduled job that polls the GitHub REST API for each student's
  `github_username` (already a field in `profiles`) and auto-inserts
  `activity_logs` rows from real commit timestamps. No VS Code extension needed.
- **Genetic algorithm upgrade**: `lib/teamFormation.ts` currently uses a greedy
  balanced-assignment approach. You can swap in a genetic algorithm (as your
  literature survey references) without touching anything else — it's isolated
  in one file.
- **NLP domain extraction**: for the "AI-driven domain extraction" feature from
  your technical overview, you could add a step where a project description is
  sent to an LLM to auto-suggest `required_skills` instead of typing them manually.
- **Faculty view**: add a `role = faculty` dashboard that lists all teams across
  all students and can view every report without owning the project.

---

## Troubleshooting

- **"Supabase env vars are missing" warning** → you haven't filled in `.env.local` yet.
- **Team formation says "no eligible students"** → check that student profiles have attendance ≥ 70% and 0 backlogs, and have set skill levels above 0.
- **Nothing happens when you click things** → open the browser console (F12) — API errors will show there and in your terminal running `npm run dev`.

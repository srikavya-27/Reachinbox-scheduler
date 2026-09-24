# ReachInbox Scheduler — Full-Stack Email Job Scheduler

## Stack
- Backend: TypeScript, Express, BullMQ + Redis, MySQL (Prisma), Elasticsearch, Nodemailer (Ethereal), Passport-free Google OAuth (google-auth-library), Slack OAuth v2
- Frontend: React + TypeScript + Tailwind + Vite

## Quick start
```bash
docker compose up -d              # MySQL, Redis, Elasticsearch
cd backend
cp .env.example .env              # fill in Google/Slack/Ethereal creds
npm install
npx prisma migrate dev --name init
npm run dev                       # runs API + worker together (concurrently)

# in a second terminal
cd frontend
npm install
npm run dev                       # http://localhost:5173
```
Live BullMQ dashboard: http://localhost:4000/admin/queues (must be logged in)

## Getting credentials (fastest path)
- **Google OAuth**: console.cloud.google.com → APIs & Services → Credentials → OAuth Client ID (Web). Add `http://localhost:4000/api/auth/google/callback` as an authorized redirect URI.
- **Slack OAuth**: api.slack.com/apps → Create App → OAuth & Permissions → add redirect URL `http://localhost:4000/api/slack/callback`, add scopes `incoming-webhook`, `chat:write`.
- **Ethereal senders**: ethereal.email/create — generate 2+ throwaway SMTP accounts instantly, no signup needed.

## How the hard requirements are implemented

### No cron — BullMQ delayed jobs only
Every scheduled email becomes one BullMQ job added with a `delay` computed from `scheduledFor - now`. Redis persists the job; there is no polling loop or cron trigger anywhere in the codebase.

### Restart survival & idempotency
- Each `EmailJob` row in MySQL is the source of truth; its `id` is passed as the BullMQ job's `jobId` **and** its payload.
- Using the MySQL row's id as the **BullMQ `jobId`** means BullMQ itself de-dupes: adding a job with the same `jobId` twice is a no-op.
- On top of that, scheduling computes an `idempotencyKey` = sha256(userId + recipient + subject + scheduledFor) and `upsert`s on it — so re-submitting the same CSV/subject/time never creates duplicate rows.
- On restart: Redis still holds the delayed BullMQ jobs (they're persisted, not in-memory), so they fire at the original time. The worker also re-checks the DB row's `status` before sending — if it's already `SENT` it skips, so a job that was mid-flight during a crash can't double-send.

### Concurrency
`WORKER_CONCURRENCY` (env var) is passed straight into `new Worker(..., { concurrency })`. Default 5.

### Minimum delay between sends
`MIN_DELAY_MS` (env var, default 2000ms = **2 seconds between sends**) — the worker awaits this before each send. Chosen to mimic a conservative real-world SMTP provider throttle.

### Hourly rate limit (per sender)
- `EMAILS_PER_HOUR` (env var) is enforced **per sender email**, via a Redis key `ratelimit:<sender>:<hour-bucket>` incremented atomically with `INCR`, with `EXPIRE` set on first increment so it self-resets.
- `INCR` is atomic across processes, so this is safe with multiple worker instances — no in-memory counters.
- When a sender is over quota, the job is **not failed**: it's re-added to the queue with a `delay` until the top of the next hour, and its `priority` is set to the original `scheduledFor` timestamp so earlier-queued emails still tend to go out first once capacity frees up.

### Slack notification on rate-limit hit
- Real Slack OAuth (`/api/slack/connect` → Slack consent screen → `/api/slack/callback` exchanges the code for an incoming webhook URL, stored per-user in MySQL).
- The webhook URL is read fresh from the DB at the moment a limit is hit — so connecting Slack later works immediately, no redeploy.
- If no webhook is stored, `notifyRateLimitHit` returns silently — no crash, no notification.

### Behavior under load (1000+ emails at once)
- All 1000 DB rows + BullMQ jobs are created in the same request (staggered by 1s of `scheduledFor` each purely to keep ordering deterministic in the UI); the **real** pacing is enforced by `MIN_DELAY_MS` and the per-hour Redis counter in the worker, not by how they were inserted.
- Once the hourly quota for a sender is hit, the overflow reschedules into subsequent hour windows automatically (see above) — so a burst of 1000 drains itself over however many hours `EMAILS_PER_HOUR` implies, without manual intervention.

### Elasticsearch
Every email is indexed (`emails` index) on creation and on status change (sent). `/api/emails/search?q=` does a `match` on `recipient`/`subject` scoped to the logged-in user.

## Trade-offs / what to improve with more time
- Rescheduled-job ordering across hour boundaries is priority-based, not a strict FIFO queue — fine for the assignment's stated goal ("preserving order **as much as possible**").
- No per-tenant workspace concept beyond `userId` — every Google account is its own tenant.
- Frontend polls every 5s rather than using websockets — simplest thing that satisfies "view scheduled/sent emails" with reasonably live data.

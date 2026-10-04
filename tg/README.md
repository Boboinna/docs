# Courses Ecosystem — Bot + Channel Testing Phase

This is the bot/channel/storage/deletion/tracking mechanics only —
no blog, no real ad-gate, no mini app yet. The gate is stubbed
(`packages/bot-core/src/gate.ts`) so every session completes instantly.
Swapping in the real blog-driven gate later means changing that ONE file;
everything else here stays as-is.

## Where this goes on GitHub

1. Create a new **private** GitHub repo — e.g. `tg-ecosystem`. Keep it
   private since it will eventually contain real business logic and, later,
   references to real infrastructure.
2. Push this whole folder as the initial commit:
   ```bash
   cd tg-ecosystem
   git init
   git add .
   git commit -m "bot-core + channel posting + storage/deletion scaffold"
   git branch -M main
   git remote add origin https://github.com/<you>/tg-ecosystem.git
   git push -u origin main
   ```
3. **Before your first commit**, add a `.gitignore` with at least:
   ```
   node_modules
   .env
   videos/*.mp4
   ```
   Never commit `.env` — it holds your bot token and DB credentials.
   `.env.example` (committed) is the template; `.env` (not committed) is
   your real one, per-machine.
4. When your partner needs access: add them as a Collaborator on the repo
   (Settings → Collaborators) — separate from the Tailscale/Portainer
   infrastructure access discussed earlier. Repo access lets them see and
   change code; Tailscale/Portainer access lets them operate the running
   server. You can grant one without the other.

## Local setup (test on your own machine first)

You do **not** need to be online or hosted anywhere for this phase — long
polling means no public URL is required.

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   if you don't have it.
2. Register an app at https://my.telegram.org to get `TELEGRAM_API_ID` and
   `TELEGRAM_API_HASH` (required for local-bot-api).
3. Create a test bot via [@BotFather](https://t.me/BotFather) → `/newbot`
   → copy the token.
4. Create a free [Neon](https://neon.tech) project → copy the connection
   string.
5. Copy `.env.example` to `.env` in the repo root, fill in all four values.
6. Copy `apps/bot-runner/.env.example` to `apps/bot-runner/.env` too, same
   values (bot-runner reads its own env file when run outside Docker; the
   root `.env` is what `docker-compose.yml` uses).
7. Drop 2–3 small placeholder `.mp4` files into `videos/` (see
   `videos/README.md`).

## Install dependencies and set up the database

```bash
pnpm install                # or bun install, both work with this workspace layout
pnpm db:generate            # generates SQL migration files from schema.ts
pnpm db:migrate             # applies them to your Neon database
pnpm seed                   # creates one test course with 3 chapters
```

The seed script prints something like:
```
Seeded course 3fae1e2b-... — try: /start course_3fae1e2b-..._full
```
Keep that course ID handy for testing below.

## Run everything

```bash
docker compose up -d
docker compose logs -f bot-runner
```

This starts `local-bot-api` and `bot-runner` together. Leave the logs
tailing in a terminal while you test — every `/start`, delivery, and
deletion-sweep tick will print there.

## Test the bot flow

1. Open your test bot in Telegram, send:
   `/start course_<the-id-from-seed>_full`
2. Expected: the bot sends chapter 1 first (first send = real upload via
   local-bot-api, since there's no cached `file_id` yet — check the logs,
   this one takes a moment longer). Then chapter 2 immediately after
   (still first-time uploads). Chapter 3 depends on `chaptersPerUnlock` —
   the seed sets it to 2, so with `mode = "full"` all released chapters go
   out regardless of that cap (full mode ignores per-cycle limits by
   design — see `resolveChaptersForMode` in start-handler.ts).
3. To test the **chapter-by-chapter cap** specifically, send:
   `/start course_<id>_chapter:3` — this should send only chapter 3.
4. To test the **cached file_id path**: run the same `/start` command a
   second time. Check the logs — this delivery should be noticeably
   faster, and you can confirm it in the database:
   ```sql
   select number, storage_ref from chapters where course_id = '<id>';
   ```
   `storage_ref` should now be populated after the first send.
5. To test **channel-join requirements**: in Neon's SQL editor (or any
   Postgres client), insert a row into `required_channels` for your test
   course with a real channel you control, set `requiresChannelJoin` on
   the course to `true`, then try `/start` again without having joined —
   you should get the "join these channels" message instead of the
   course. Join, retry, confirm delivery.

## Test the 30-minute deletion + warning

Rather than waiting 30 real minutes every test cycle, temporarily lower
the constants in `packages/bot-core/src/deletion-job.ts` and
`packages/bot-core/src/delivery.ts` (`DELETE_AFTER_MS`,
`WARNING_LEAD_MS`) to something like 2 minutes / 30 seconds, restart
`bot-runner`, run `/start` again, and watch the logs — you should see the
warning message arrive, then the actual file disappear from the chat.
**Remember to set these back to the real 30-minute / 5-minute values
before anything resembling production use.**

## Test channel posting

1. Create a private Telegram channel, add your test bot as an admin with
   "Post Messages" permission.
2. Get the channel's numeric ID (forward any message from the channel to
   [@userinfobot](https://t.me/userinfobot), or use `getUpdates` on the
   bot API directly).
3. Write a tiny one-off script (or use `pnpm --filter admin-actions` as a
   template) calling `postCourseToChannel` with your test course ID and
   that channel ID — confirm the photo, caption, and "Download Course"
   button appear correctly, and that clicking the button opens your bot
   with the right `/start` payload.
4. Run it again with a changed course title — confirm it **edits** the
   existing post instead of creating a duplicate.

## Test funnel tracking

After a few `/start` runs (some completed, some deliberately interrupted
by e.g. not joining a required channel), query:
```sql
select outcome, count(*) from funnel_events group by outcome;
```
You should see both `completed` and `abandoned` rows reflecting what you
actually did.

## What's intentionally not here yet

- The real ad-gate (blog-driven) — `gate.ts` is stubbed on purpose.
- The mini app.
- Redis/BullMQ — the deletion loop is a plain `setInterval`, which is
  genuinely fine at this scale.
- Multi-bot config — `BOT_ID` is a placeholder env var for now; turning
  this into a real `bots` table row per bot is a small addition once
  you're adding your partners' bots.

## Moving to Oracle for always-on testing/staging

Once local testing feels solid, the exact same `docker-compose.yml` and
`.env` (with `LOCAL_BOT_API_URL` pointed at the Docker service name, which
it already is) run unchanged on the Oracle VM from your plan.md Section
12A — `git clone` the repo there, copy your `.env` over (never commit it),
`docker compose up -d`, done.

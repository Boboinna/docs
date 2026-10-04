# Tg-Bot-Download Ecosystem — Build Plan

## 0. Getting-Started Checklist

- [ ] Register domains (blog, mini app if custom domain wanted)
- [ ] Create GitHub org/repo, set up monorepo skeleton (Section 2)
- [ ] Register app at my.telegram.org → get api_id/api_hash (needed for local-bot-api)
- [ ] Decide: start on Oracle free tier (Section 12A) or go straight to Hetzner (Section 12B) — either works with the same docker-compose.yml
- [ ] Set up Tailscale tailnet, invite the partners (Section 14)
- [ ] Build packages/db schema first, then packages/storage, then packages/bot-core (Section 17)
- [ ] Deploy blog to Cloudflare Pages (NOT Vercel Hobby — see Section 11)
- [ ] Deploy mini app to Vercel free tier (no ads, so Hobby ToS is fine)
- [ ] Wire up admin-cli + mini app against the same shared admin-actions package (Section 18)

---

## 1. Goal

A reusable, multi-bot system for selling/gifting stuff via Telegram, where:
- Free/discounted access is gated behind an ad-funnel (bot → blog → bot).
- Courses can be released fully, in manually-sized batches, or chapter-by-chapter.
- Channel-join requirements, ad-cycle requirements, and chapter-unlock frequency are all per-course, admin-configurable, no redeploy needed.
- The same core logic can run under multiple bot tokens (yours, your partners') with only config differing.
- Storage starts on Telegram (free forever), can migrate to self-hosted/object storage later without a rewrite.
- Infrastructure starts free (Oracle) and can move to paid self-hosting (Hetzner) later with no code changes.
- Other people(partners) can all operate the system; you retain sole ownership of the underlying infrastructure.

---

## 2. Repo Structure (monorepo, pnpm + Turborepo)

```
courses-ecosystem/
├── apps/
│   ├── blog/                 # Next.js 15 + Velite (MDX), Adsterra ads, gating UI, course pages
│   ├── bot-runner/           # Telegram bot process(es), webhook or long-poll
│   ├── miniapp-admin/        # Telegram Mini App (self-hosted on Vercel/Cloudflare): upload chapters, manage courses/channels
│   ├── admin-cli/            # CLI for bulk/scripted admin actions
│   └── local-bot-api/        # Docker config for self-hosted telegram-bot-api
├── packages/
│   ├── db/                   # Drizzle schema + migrations + client (Postgres)
│   ├── storage/              # StorageProvider interface + adapters
│   ├── bot-core/              # Reusable command handlers, gating/session logic
│   ├── admin-actions/         # Shared functions called by BOTH miniapp-admin and admin-cli
│   └── shared-types/
├── docker-compose.yml         # postgres, redis, local-bot-api, bot-runner, portainer
└── turbo.json
```

**Why monorepo:** shared `bot-core`, `admin-actions`, and `db` packages mean adding a partner's bot is a new `.env` + a config row, not a fork. Apps still deploy independently.

---

## 3. Data Model (Postgres via Drizzle)

- `bots` — id, name, token_env_key, owner, active
- `courses` — id, title, distribution_mode (`full` | `batch` | `chapter_by_chapter`), requires_ad_cycle (bool), requires_channel_join (bool), price/free flag, image_url
- `chapters` — id, course_id, number, storage_provider, storage_ref, released (bool), released_at
- `course_settings` — course_id, chapters_per_unlock (manually editable), post_to_channel_1 (bool), post_to_channel_2 (bool)
- `course_page_buttons` — id, course_id, label, mode (`full` | `batch:N` | `chapter:N`), enabled, sort_order
- `channel_posts` — course_id, channel_id, message_id (for editing/reposting instead of duplicating)
- `required_channels` — course_id (nullable = global), channel_username_or_id, invite_link
- `sessions` — id, user_id, bot_id, course_id, mode, token (nonce), status (`pending`|`step1`|`step2`|`step3`|`completed`), created_at, expires_at
- `user_progress` — user_id, course_id, chapters_unlocked, last_ad_cycle_at
- `delivered_files` — chat_id, message_id, sent_at, delete_at (for the 30-min auto-delete job)

---

## 4. Bot ↔ Blog Flow

1. User clicks course (from a channel post or a course page) → deep link `t.me/Bot?start=course_<id>_<mode>`.
2. Bot creates a `sessions` row (status `pending`) + short-lived token, sends blog link `blog.com/course/<id>?t=<token>`.
3. Blog middleware validates token server-side against `sessions`. Valid → gated view (countdown, scroll-to-bottom, timed buttons). No/invalid token → normal blog post, no gating UI.
4. Each gated step calls a backend API route; server issues the next nonce only after verifying elapsed time + prior nonce, and advances `sessions.status`.
5. Final button (after last blog page) deep-links back to the bot with the completed session token.
6. Bot checks `sessions.status = completed` before showing "Get course" / sending any file. Incomplete cycle = no file, no matter what URL the user hits directly.
7. Bot parses the `<mode>` suffix from the original deep-link payload to know whether to deliver a specific chapter, a batch, or the full course.

## 5. Chapter Release / Unlock Logic

- On ad-cycle completion: `user_progress.chapters_unlocked += course_settings.chapters_per_unlock`, capped at count of `chapters.released = true`.
- Chapter-by-chapter courses: releasing chapter N doesn't auto-unlock it for users already at their cap — they must run the cycle again.
- Once `distribution_mode` flips to `full` or `batch`, bot skips unlock-gating and sends per that mode.
- `requires_ad_cycle = false` and/or `requires_channel_join = false` per course → bot skips those checks entirely for that course.
- Channel-join check: `getChatMember(channel_id, user_id)` for each row in `required_channels` before starting the cycle.

## 6. File Delivery & Auto-Delete

- Send file → record `delivered_files` row with `delete_at = now + 30m`.
- Caption on every delivered file: instruction to forward to Saved Messages immediately.
- Scheduled job (BullMQ + Redis, checked every minute) deletes messages past `delete_at` via `deleteMessage`.

## 7. Storage Abstraction

```ts
interface StorageProvider {
  store(file, meta): Promise<FileRef>
  getDeliverable(ref: FileRef, chatId: string): Promise<void>
  delete(ref: FileRef): Promise<void>
}
```
- `TelegramStorageProvider` (current, free forever): caches `file_id`, resends via `copyMessage`.
- `S3StorageProvider` / `R2StorageProvider` (future, if ever needed): store object key, stream to Telegram on delivery.
- `chapters.storage_provider` + `storage_ref` columns make this a per-file setting — migration is incremental, not a cutover.
- Decision: staying on Telegram storage for now (free, and client-side encryption tools like Cryptomator don't meaningfully fit this pipeline — see discussion history). Revisit only if a specific, named risk emerges.

## 8. Local Bot API Server

- Register app at my.telegram.org → `api_id` + `api_hash`.
- Run `telegram-bot-api` binary/Docker image with those + bot token(s), pointed at a local data dir.
- `bot-runner` points its API base URL at `http://local-bot-api:8081` instead of `api.telegram.org`.
- Unlocks up to 2000MB file handling and local-path file sends.
- Always-on process → lives in `docker-compose.yml`, deployed on a VPS, not serverless.

## 9. Channel Posting

- Admin action `postCourseToChannel(courseId, channelId)` in `packages/admin-actions`, called from both the mini app and the CLI.
- Uses `sendPhoto` with the course image, an HTML-formatted caption (name + details), and a single inline URL button: `https://t.me/YourBot?start=course_<id>`.
- Stores the resulting `message_id` in `channel_posts` so future edits use `editMessageCaption`/`editMessageMedia` instead of duplicate-posting.
- Two channels = two `post_to_channel_1`/`post_to_channel_2` flags per course, or just call the action twice with different channel IDs.
- For 40 courses at once, `bulkPostAllCourses(channelId)` in admin-actions loops and calls the single-course action — this is the kind of job that's much nicer from the CLI than clicking through a UI 40 times.

## 10. Individual Course Pages with Editable Buttons

- Course content (description, chapter list, images) lives in Velite/MDX — static, fast, SEO-friendly.
- The **button block is a separate, DB-driven component**, not static MDX — it fetches `course_page_buttons` rows via an API route and renders whatever's currently configured.
- Each button's `mode` (`full` / `batch:N` / `chapter:N`) becomes the deep-link suffix: `t.me/Bot?start=course_<id>_<mode>`.
- Editing labels, which buttons show, or their mode is a data edit (via mini app or CLI) against `course_page_buttons` — no redeploy needed.
- Same gating flow (Section 4) applies regardless of whether the user arrived via a channel post or a course page — the deep-link payload is what tells the bot which course + mode was requested.

## 11. Hosting Platform Rules — Read This Before Deploying

| Platform | Free tier allows ads/commercial use? | Use for |
|---|---|---|
| **Vercel Hobby (free)** | **No** — Hobby ToS restricts to non-commercial/personal use; ads or monetization technically violate it the moment they're live | Non-monetized apps only, e.g. the mini app admin panel |
| **Vercel Pro ($20/mo)** | Yes | Blog, if you later want Vercel specifically |
| **Cloudflare Pages (free)** | **Yes, explicitly allows commercial use**, unlimited bandwidth on free tier | **Put the Adsterra-monetized blog here** |

**Decision: blog → Cloudflare Pages (free). Mini app → Vercel free tier (no ads on it, so Hobby ToS is fine).**

Separately from hosting ToS: Adsterra ad formats (pop-unders, Social Bar) combined with forced-wait/forced-click funnels resemble patterns ad networks watch for as incentivized/non-genuine traffic. Read Adsterra's publisher terms directly before scaling.

---

## 12A. Self-Hosting Free: Oracle Cloud Always Free (start here, $0)

- Ampere A1 (ARM) compute: **2 OCPU / 12 GB RAM** (as of mid-2026), 200GB block storage, 10TB/month outbound transfer, permanently free (not a trial).
- Requires a card for identity verification, not charged unless you explicitly upgrade.

**Signup steps:**
1. oracle.com/cloud/free → sign up → choose home region carefully (Always Free resources are tied to it).
2. Console → Compute → Instances → Create Instance.
3. Shape: **VM.Standard.A1.Flex**, 2 OCPU / 12GB (Always Free eligible). If "out of capacity," retry later or fall back to the smaller AMD `VM.Standard.E2.1.Micro` (also Always Free).
4. Ubuntu image, upload/generate an SSH key pair.
5. Create instance, note the public IP.
6. Set a Billing → Budget alert so you're emailed if anything drifts into paid usage.
7. `ssh -i your_key.pem ubuntu@<public_ip>`

## 12B. Self-Hosting Paid: Hetzner Cloud (move here when ready to pay)

Hetzner is the standard next step once you want reliability without free-tier capacity limits — European data centers, cheap (a few euros/month for a CX22/CX32), same Docker Compose stack as Oracle, no code changes required to move.

**Signup steps:**
1. Create a Hetzner Cloud project (console.hetzner.cloud).
2. Security → SSH Keys → add your public key.
3. Servers → Add Server → location of choice → Ubuntu → shape **CX22** (2 vCPU/4GB) or **CX32** (4 vCPU/8GB) for comfortable headroom running Postgres + Redis + local-bot-api + bot-runner together.
4. Attach your SSH key, create.
5. `ssh root@<server-ip>` — install Docker exactly as in Section 13, run the same `docker-compose.yml`.
6. Point your domain's DNS A record at the Hetzner server IP (Cloudflare DNS is a good free choice here, adds DDoS protection in front of it too).
7. Resize the server later (more vCPU/RAM) without rebuilding, directly from the Hetzner console, as course volume grows.

**Migration path Oracle → Hetzner:** stop the stack on Oracle (`docker compose down`), copy the named volumes (`pgdata`, `redisdata`, `botapidata`) to the new server, `docker compose up -d` there, repoint DNS. Same compose file both places.

---

## 13. Docker — What It Is and How It Works (no prior experience assumed)

- **Image**: a frozen blueprint of an app + everything it needs (OS bits, dependencies, config) — e.g. the official `postgres` image.
- **Container**: a running instance of an image — isolated from other containers but able to talk to them over a Docker network you define.
- **Dockerfile**: instructions for building a *custom* image (used for your own `bot-runner` and `blog` code, not for off-the-shelf Postgres/Redis).
- **Volume**: a folder on the host VM that a container writes persistent data to — e.g. Postgres's actual database files — so data survives container restarts/rebuilds.
- **docker-compose.yml**: one file describing all your containers, their images, ports, volumes, and how they connect. `docker compose up -d` starts everything; `docker compose down` stops it; `docker compose logs -f <service>` tails logs.

**Install (Ubuntu, works on both Oracle and Hetzner):**
```bash
sudo apt update
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# log out and back in for the group change to apply
```

**Starter docker-compose.yml:**
```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: courses
      POSTGRES_PASSWORD: changeme
      POSTGRES_DB: courses
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    restart: unless-stopped
    volumes:
      - redisdata:/data

  local-bot-api:
    image: aiogram/telegram-bot-api:latest
    restart: unless-stopped
    environment:
      TELEGRAM_API_ID: "your_api_id"
      TELEGRAM_API_HASH: "your_api_hash"
    volumes:
      - botapidata:/var/lib/telegram-bot-api
    ports:
      - "8081:8081"

  bot-runner:
    build: ./apps/bot-runner
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
      - local-bot-api
    env_file:
      - .env

  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainerdata:/data

volumes:
  pgdata:
  redisdata:
  botapidata:
  portainerdata:
```

Run it: `docker compose up -d`. Check status: `docker compose ps`. This file runs identically on your laptop, Oracle, and Hetzner.

---

## 14. Coordinating With Partners

- **Tailscale** (free Personal plan, expanded to **6 users, unlimited devices** as of mid-2026): you create the tailnet as Owner, invite all partners by email from the admin console, tag the server as a shared resource. Everyone installs the Tailscale client; the server joins once. Result: all of you reach Postgres, Redis and Portainer as if on a local network — **no public ports exposed to the internet**, and you can revoke either partner's access anytime from the console.
- **Portainer CE** (free, already in the compose file): web UI for Docker — start/stop containers, view logs, edit the stack. Give each partner their own login with standard-user permissions (not admin) reachable at `http://<tailscale-ip>:9000` — they get real day-to-day access (restart a bot, check logs, redeploy) without your SSH key or Hetzner/Oracle account login.

**Result:** you're the sole holder of the VM, the SSH key, and the cloud account — true ownership. Both partners get functional operational access through Portainer over Tailscale.

---

## 15. Deployment Targets (Free → Paid Self-Host)

| Component | Free (Oracle) | Paid (Hetzner) |
|---|---|---|
| blog | Cloudflare Pages (free, ads allowed) | Same — Cloudflare Pages doesn't need to move |
| bot-runner | Docker on Oracle VM | Docker on Hetzner VM |
| local-bot-api | Docker on Oracle VM | Docker on Hetzner VM |
| miniapp-admin | Vercel free tier | Same, or Docker on VPS if preferred |
| admin-cli | Run locally by any of the 3 of you, hits the same backend | Same |
| db (Postgres) | Docker on Oracle VM | Docker on Hetzner VM (bigger volume as you scale) |
| redis | Docker on Oracle VM | Docker on Hetzner VM |
| container management | Portainer (free) | Same |
| team access | Tailscale (free, 6-user tier) | Same |

No code changes moving from free to paid — same docker-compose.yml, same Drizzle connection-string pattern, just bigger/paid infrastructure when ready.

---

## 16. Multi-Bot Reusability

- `bots` table + per-bot `.env` (token, owner) is the only per-instance config.
- All command handling, gating logic, and chapter logic live in `packages/bot-core`, imported by `apps/bot-runner`.
- A partner's bot = new token + new `bots` row + own courses, same codebase, same DB (or a separate DB instance for full separation if ever wanted).

---

## 17. Admin CLI + Mini App — Shared Logic

- `packages/admin-actions` holds the real functions: `postCourseToChannel`, `bulkPostAllCourses`, `releaseChapter`, `setChaptersPerUnlock`, `setDistributionMode`, `editCoursePageButtons`, etc.
- `apps/miniapp-admin`'s API routes call these functions, authenticated via Telegram `initData` HMAC validation + an admin `user_id` allow-list.
- `apps/admin-cli` (built with `commander` or similar) calls the *exact same* functions, authenticated via a service token — not raw DB access — so every action is consistently logged regardless of which interface triggered it.
- Use the CLI for anything repetitive/bulk (posting all 40 courses, batch-releasing a chapter across many courses); use the mini app for one-off edits and anything you want a partner to do without giving them CLI/server access.

---

## 18. Build Order (suggested)

1. Provision infrastructure: Oracle VM first (Section 12A), or straight to Hetzner (12B) if you'd rather skip the free-tier step.
2. Docker install + base `docker-compose.yml` (postgres, redis, portainer) running (Section 13).
3. Tailscale tailnet set up, both partners invited (Section 14).
4. `packages/db` schema + migrations (Postgres, Drizzle), confirmed connecting from your laptop over Tailscale.
5. `packages/storage` with Telegram adapter only.
6. `packages/bot-core`: `/start` handler, deep-link + mode parsing, session creation, channel-check, chapter-unlock logic.
7. `packages/admin-actions`: the shared functions from Section 17.
8. `apps/local-bot-api` added to compose + `apps/bot-runner` wired to it.
9. `apps/blog`: Velite content, Adsterra integration, no middleware, gated-view components, DB-driven course-page buttons — deployed to Cloudflare Pages.
10. `apps/admin-cli`: wraps `packages/admin-actions` for bulk/scripted use.
11. Auto-delete job (Redis/BullMQ).
12. `apps/miniapp-admin` last, once schema is stable — deployed to Vercel free tier, all partners given access.

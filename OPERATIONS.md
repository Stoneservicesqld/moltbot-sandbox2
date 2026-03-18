# OpenClaw / Moltbot-Sandbox2 — Permanent Knowledge Record

**Project:** Stoneservicesqld/moltbot-sandbox2  
**Bot name:** Max (Telegram)  
**Confirmed working as of:** 2026-03-18  
**Last updated:** 2026-03-18

---

## Architecture in One Paragraph

Max runs as an OpenClaw gateway process inside a Cloudflare Container (Durable Object). The container starts fresh on every cold start — it has no persistent disk. All state (config, conversation history, auth profiles, skills) lives in an R2 bucket (`moltbot-data`). A startup script (`start-openclaw.sh`) runs on every cold start: it restores config from R2, patches the config with secrets from Worker environment variables, then starts the OpenClaw gateway on port 18789. A Cloudflare Worker (`moltbot-sandbox2`) proxies all traffic to the container. The container has a **180-second startup timeout** — if port 18789 is not bound within 180 seconds, Cloudflare kills it.

---

## The One Bug That Caused Two Days of Outage

### Root Cause

The `openclaw/` prefix in R2 (`r2:moltbot-data/openclaw/`) became contaminated with workspace files, media files, and `node_modules` directories during the working session on 2026-03-17. This happened because the background sync-back used `rclone sync /root/.openclaw/ r2:moltbot-data/openclaw/` — and OpenClaw writes workspace subdirectories, media/inbound images, and agent session data **inside** `/root/.openclaw/`. Every 30 seconds, these were synced back into the `openclaw/` R2 prefix.

On the next cold start, the restore command `rclone copy r2:moltbot-data/openclaw/ /root/.openclaw/` faithfully copied everything back — including hundreds of files from `workspace/ms365-integration/node_modules/`, `media/inbound/*.jpg`, and `agents/main/sessions/*.jsonl`. This took over 90 seconds, pushing past the 180-second timeout and killing the container before port 18789 ever bound.

### The Fix (commits `ca09476` and `94ce4cf`)

**Restore:** Added `--exclude='workspace/**' --exclude='media/**' --exclude='agents/**'` to the `openclaw/` rclone copy command. The restore now only copies actual config files (~15 files, ~3 seconds).

**Sync-back:** Added the same excludes to the background sync loop's `rclone sync` of `$CONFIG_DIR/` back to `openclaw/`. Workspace, media, and agent data are synced to their own dedicated R2 prefixes (`workspace/`, `openclaw-agents/`) — not into `openclaw/`.

**Workspace memory:** Re-enabled workspace restore as a **background task** that runs after the gateway has already bound to port 18789. Max gets his conversation history back within ~30 seconds of cold start, without blocking startup.

---

## R2 Bucket Structure (Correct)

| R2 Prefix | Container Path | Contents |
|---|---|---|
| `openclaw/` | `/root/.openclaw/` | Config only: `openclaw.json`, `devices/`, `identity/`, `cron/`, `canvas/`, `telegram/`, `cron/` |
| `openclaw-agents/` | `/root/.openclaw/agents/` | Auth profiles, API keys, agent sessions |
| `workspace/` | `/root/clawd/` | Conversation history, learnings, tools, user files |
| `skills/` | `/root/clawd/skills/` | Custom skills |

**Critical rule:** The `openclaw/` prefix must NEVER contain `workspace/`, `media/`, or `agents/` subdirectories. If it does, cold starts will timeout.

---

## Startup Sequence (Current — Working)

```
1. Key presence check (echo all secrets)
2. rclone setup (R2 credentials from env vars)
3. Restore openclaw/ → /root/.openclaw/  [BLOCKING, ~3s, excludes workspace/media/agents]
4. Restore openclaw-agents/ → /root/.openclaw/agents/  [BLOCKING, ~2s]
5. Restore skills/ → /root/clawd/skills/  [BLOCKING, ~2s]
6. openclaw onboard --non-interactive  [BLOCKING, ~10s, only if no config]
7. node config patch (inject secrets, Telegram token, gateway token)  [BLOCKING, ~1s]
8. Background subshell starts:
   a. Restore workspace/ → /root/clawd/  [BACKGROUND, ~30s, non-blocking]
   b. Every 30s: sync changed files back to R2
9. exec openclaw gateway --port 18789  [TAKES OVER PROCESS]
```

Total blocking time before port 18789 binds: **~18 seconds** (well within 180s limit).

---

## Environment Variables / Secrets Required

All set via `wrangler secret put` in Cloudflare dashboard:

| Secret | Purpose | Status |
|---|---|---|
| `ANTHROPIC_API_KEY` | Claude API access | Required |
| `OPENAI_API_KEY` | OpenAI fallback | Set |
| `TELEGRAM_BOT_TOKEN` | Max's Telegram bot | Required |
| `OPENCLAW_GATEWAY_TOKEN` | Admin UI auth token | Required |
| `R2_ACCESS_KEY_ID` | R2 persistence | Required |
| `R2_SECRET_ACCESS_KEY` | R2 persistence | Required |
| `R2_BUCKET_NAME` | `moltbot-data` | Required |
| `CF_ACCOUNT_ID` | R2 endpoint URL | Required |

---

## How to Diagnose a Future Outage

### Step 1 — Check status
Visit: `https://moltbot-sandbox2.manager-cfa.workers.dev/api/status`

| Response | Meaning |
|---|---|
| `{"ok":true,"status":"running"}` | All good |
| `{"ok":false,"status":"not_running"}` | Container idle, will start on next request |
| `{"ok":false,"status":"not_responding","processId":"..."}` | Container starting or crashed |

### Step 2 — Get the startup log
In Cloudflare Dashboard → Workers & Pages → moltbot-sandbox2 → **Observability** tab → filter by `container` → look for `[Gateway] startup failed. Stdout:` entries.

The stdout will show exactly where the startup script stopped. Key things to look for:

- **Lots of `workspace/ms365-integration/node_modules/` files being copied** → R2 contamination has recurred. Add more excludes to the restore command.
- **`E: Could not get lock /var/lib/dpkg/lock-frontend`** → Old container image with `apt-get` in startup script. Force a new deploy.
- **`openclaw onboard` hanging** → onboard step taking too long. Check `timeout 120` is still on the onboard command.
- **Script stops after config patch with no gateway start line** → Node.js config patch script erroring. Check `openclaw.json` in R2 for corruption.

### Step 3 — If R2 contamination recurs
Run this rclone command to clean the `openclaw/` prefix (requires rclone configured with R2 credentials):

```bash
# List what's in openclaw/ that shouldn't be there
rclone ls r2:moltbot-data/openclaw/ | grep -E "workspace/|media/|agents/"

# Delete the contaminating files
rclone delete r2:moltbot-data/openclaw/ --include='workspace/**'
rclone delete r2:moltbot-data/openclaw/ --include='media/**'
rclone delete r2:moltbot-data/openclaw/ --include='agents/**'
```

---

## Deployment

**Repository:** https://github.com/Stoneservicesqld/moltbot-sandbox2  
**Deploy method:** Push to `main` branch → GitHub Actions builds and deploys automatically  
**Build time:** ~5 minutes  
**Cloudflare Worker URL:** https://moltbot-sandbox2.manager-cfa.workers.dev  
**Admin UI:** https://moltbot-sandbox2.manager-cfa.workers.dev (requires `OPENCLAW_GATEWAY_TOKEN`)

---

## Commit History — Key Milestones

| Commit | What it did |
|---|---|
| `ca09476` | **THE FIX** — excludes workspace/media/agents from openclaw/ restore |
| `94ce4cf` | Prevents re-contamination + re-enables workspace restore in background |
| `92e6385` | Adds openclaw-agents/ restore (auth profiles) |
| `73bc0ca` | Moves poppler-utils to Dockerfile (no runtime apt-get) |
| `a689479` | Last commit that was working on Mar 17 |

---

## Max's Memory — How It Works

Max remembers conversations because OpenClaw saves session data to `/root/clawd/` (the workspace directory). On every cold start:

1. The gateway starts immediately (port 18789 binds in ~18s)
2. In the background, the workspace is restored from `r2:moltbot-data/workspace/` (~30s)
3. Max has full conversation history within ~48 seconds of cold start

New conversations are saved back to R2 every 30 seconds by the background sync loop. If the container is recycled mid-conversation, at most 30 seconds of conversation history could be lost.

---

*This document should be stored in the project's GitHub repository as `OPERATIONS.md` for permanent reference.*

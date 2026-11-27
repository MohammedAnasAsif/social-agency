# Social Agency: Design

Status: draft for approval · 26 Nov 2026 · Owner: Mohammed Anas Asif

## 1. What this is and why

This is a one-person content agency that runs itself. Claude writes posts in the style of the best people in each niche, scores every post and rewrites the weak ones, then queues them. An AWS server posts one item per account per day to Instagram, YouTube, X, LinkedIn and Substack. A dashboard shows what went out, where it went, and anything that failed.

The goal is steady, high-quality posting on a few real accounts without paying for schedulers or APIs, and without the owner having to write or upload anything by hand.

## 2. Goals, non-goals, anti-goals

**Goals**
- Posts are written, rated and published with no manual step after the one-time setup.
- ₹0 running cost beyond the AWS server and the owner's existing Claude plan.
- Up to 2 accounts per platform, 1 post per account per day.
- Every attempt is logged. Nothing fails silently, and expiring logins are flagged before they cause a failure.
- Writing per platform:
  - **Instagram / YouTube:** clips (gaming, podcast or other). The owner picks the channel type for each account.
  - **LinkedIn:** reads the owner's profile, writes posts and articles, and proposes a rewritten headline/About section.
  - **X:** tweets in the style(s) the owner picks (controversy, quotes, news, teaching, psychology, ...).
  - **Substack:** posts for the owner's field.
  - All of it modelled on how the top writers in that niche write: structure, hooks and density, never copying their words or posing as them.

**Non-goals (for now)**
- More than 2 accounts per platform, or more than 1 post per account per day.
- Replying to comments or DMs, and analytics beyond "did it post".
- TikTok, Threads, Facebook.

**Anti-goals (a failure even if it "works")**
- Hiding AI use from platform disclosure systems. Where a platform requires an AI label (e.g. photorealistic AI video on YouTube/Instagram), we apply it.
- Fake or duplicate personal profiles (LinkedIn allows one per real person).
- Re-uploading blindly after an unclear result. Duplicates get accounts flagged.
- Storing anyone's passwords. Only session cookies and tokens are stored.
- Reposting other people's footage without the right to use it.

## 3. Constraints
- Posting runs on the owner's **AWS server (assumed Ubuntu)**. Browser uploads need Chrome under `xvfb-run`.
- Claude's scheduled sessions run in Anthropic's cloud and **cannot reach the AWS server directly**. They also can't download source footage from most websites.
- GitHub files are limited to 100 MB. Videos never go into the repo.
- Platform limits and facts (checked Nov 2026):
  - X has no free API tier (so we post with cookies).
  - LinkedIn's personal-profile posting API is free and self-serve, but its token lasts about 60 days.
  - YouTube API uploads from an unaudited project are locked private (so we use browser upload).
  - Substack has no official API, and its session lasts about 3 months.

## 4. Approaches considered

**How the writing gets to the server**
1. **GitHub repo as the mailbox (picked by owner).** A daily Claude scheduled task writes, rates and commits posts to `queue/`. The server pulls, posts, and pushes results to `log/`. Pros: ₹0, uses the skills already on the owner's Claude account, full history in git. Con: two moving parts.
2. Claude Code on the AWS server. Simpler for clips later, but the skills must be copied to the server and kept in sync, and it adds another login that can expire.
3. *Dropped:* this chat connecting to the server directly. Cloud sessions can't reach it.

**How posting is done (owner picked B: strict ₹0)**
- A: official X API (~₹80/month) + everything else free.
- **B: cookie-based X posting with `auto-x`**, everything else as below. It's free, but X is the platform most likely to ask "verify it's you". The dashboard surfaces this.
- *Dropped:* Postiz (no Substack, and YouTube uploads would get stuck private).

## 5. The system

```
   Claude scheduled task (daily, cloud)                 AWS server (daily cron)
 ┌──────────────────────────────────────┐            ┌──────────────────────────────┐
 │ 1. read log/ → refresh dashboard     │            │ 1. git pull                  │
 │ 2. for each account: write → rate →  │  GitHub    │ 2. health-check every login  │
 │    rewrite/drop → commit to queue/   │◀──repo────▶│ 3. post 1 item per account   │
 │                                      │            │ 4. write log/, move item     │
 └──────────────────────────────────────┘            │ 5. git push                  │
            │                                         │ (phase 4: cut clips here)    │
            ▼                                         └──────────────────────────────┘
   Dashboard (private Claude artifact)
```

### 5.1 Repo layout
```
config/accounts.yaml        one entry per account (see 5.2)
config/profiles/<id>.md     niche brief per account: LinkedIn profile text, style picks, reference writers
skills/                     the repo's own SKILL.md files (writer orchestrator, rater, poster)
queue/<account-id>/<date>-<slug>/post.json (+ images for carousels)
posted/<account-id>/...     items moved here after confirmed success
failed/<account-id>/...     items moved here after failure, with reason
log/posts.jsonl             one line per attempt (append-only)
log/sessions.json           latest health check per account
server/                     poster code, adapters, cron + setup scripts
docs/design.md              this file
```
`cookies/` lives **only on the server**, is listed in `.gitignore`, and is never committed.

### 5.2 Accounts config (example)
```yaml
- id: ig-main
  platform: instagram
  login: sau                  # adapter
  niche: gaming clips (GTA, Valorant)
  post_time: "19:00"          # IST
  enabled: true
- id: x-main
  platform: x
  login: auto-x
  styles: [psychology, teach]
  reference_writers: [handles the owner names]
```

### 5.3 post.json
```json
{
  "id": "2026-11-27-why-habits-fail",
  "account": "x-main",
  "platform": "x",
  "type": "tweet | thread | carousel | clip | linkedin_post | linkedin_article | substack_post",
  "text": "...",
  "title": "...",              // YouTube / Substack
  "tags": ["..."],
  "media": ["slide1.png"],      // relative to the item folder; clips use server paths in phase 4
  "ai_label": false,            // true when the platform's rules require disclosure
  "score": {"total": 8.2, "hook": 9, "clarity": 8, "platform_fit": 8, "value": 8, "notes": "..."},
  "status": "ready"
}
```

### 5.4 Writer (Claude scheduled task, once a day, after the posting time)
1. Read `log/` and refresh the dashboard (5.7).
2. For each enabled account, top the queue up to **3 ready items** (a buffer, so one missed writer day doesn't stop posting).
3. Write using the installed social-media skills, routed by platform:
   - **LinkedIn:** linkedin-post-writer / linkedin-growth / profile-optimization
   - **X:** text-post-and-microblog / thread-writer / x-growth / contrarian-and-opinion
   - **Instagram carousel:** carousel-writer + images rendered as PNG
   - **YouTube / Instagram clip captions:** youtube-shorts / instagram-reels-publishing
   - **Substack:** email-and-newsletter
   - Plus hook-writer, voice-builder and writing-style-and-tone for every post.
4. Rate with the rater skill (5.5). A score of **≥ 7.5 queues the post.** Below that, it's rewritten once, and if it's still below, dropped and the reason logged.
5. Commit to `queue/`.

### 5.5 Rater skill
- Scores 1–10 on: hook (first line / first 2 seconds), clarity, value to the reader, fit to the platform's format and length, and originality vs the account's last 20 posts (no near-duplicates).
- Weights are set per platform. It returns a total score, sub-scores and one line on what to fix.
- It's a quality bar, not a virality guarantee.

### 5.6 Poster (AWS, cron)
- Runs at each account's `post_time`.
- **Health check first:** each adapter has a `check` command. Failures mark the account `needs_relogin` in `log/sessions.json`, and that account's post is skipped. We never attempt an upload on a dead session.
- Picks the oldest `ready` item for the account and posts it through the adapter:

| Platform | Adapter | Login (one-time, by the owner) |
|---|---|---|
| YouTube | `social-auto-upload` (sau), with the fixes in the installed yt-ig-browser-upload skill | `sau youtube login --account <id>` in a browser window |
| Instagram | sau | `sau instagram login --account <id>` |
| X | `xtea/auto-x` | export cookies.json with Cookie-Editor → `auto-tt import-cookies` |
| LinkedIn | official Posts API, `w_member_social` | OAuth approve once; re-approve ~every 60 days (warned 7 days ahead) |
| Substack | `python-substack` | cookies.json exported from the browser |

- **Success** only with proof: exit code 0 **and** a post URL (or `published` for Instagram). The item moves to `posted/`, and the URL is logged.
- **Unknown result** (timeout after upload started): the item is marked `needs_check` and is **never re-posted automatically**. The dashboard shows it.
- **Failure before upload starts** (network, browser crash): one retry, then the item goes to `failed/` with the reason.
- Push the log at the end. On a git push conflict, pull/rebase and retry.

### 5.7 Dashboard
- A private Claude artifact the writer task republishes daily from `log/`.
- Shows: per account, today's status (posted / skipped / failed / needs check), the link, and the last 30 days of history; login health with days until expiry where known; queue depth; errors with plain-language reasons.
- *(Owner's call pending. Alternative: a password-protected page served from the AWS server, live instead of daily.)*

### 5.8 Onboarding (one time, in chat)
Per account, Claude asks for and saves to `config/profiles/<id>.md`:
- **Instagram/YouTube:** channel type and footage source.
- **X:** tweet styles.
- **LinkedIn:** the profile text (pasted, or the profile PDF export, no scraping), plus Claude proposes a new headline/About.
- **Substack:** field and audience.
- **All:** 3–5 reference writers the owner admires.

### 5.9 Access and setup (who touches what)
- **Claude cannot SSH into the server.** Server setup is one script (`server/setup.sh`) that the owner runs once on the server. Updates come through `git pull`, so new code reaches the server the same way posts do.
- **The server talks to GitHub** with a deploy key that has write access to this repo only.
- **The Claude scheduled task talks to GitHub** through the owner's GitHub connection on Claude.
- **Browser logins** (sau, and exporting cookies for X/Substack) are done on a PC with a screen, and the cookie files are copied to the server with `scp`. A small helper script for the PC does the login and the copy. This flow has already been tested for sau, from a laptop in India to an AWS server.

## 6. Build phases
1. **Posting + logging + dashboard.** Server setup script, the 5 adapters, health check, cron, log format, dashboard. Tested with private/test posts where the platform allows it.
2. **Rater skill.**
3. **Writer task + onboarding** for X, LinkedIn, Substack and Instagram carousels (all text/images, which the cloud can make).
4. **Clip pipeline on the server:** pull source footage the owner has rights to, cut highlights, caption, make 9:16. The writer adds titles/captions. (The installed twitch-chat-spike-clipper skill is a starting point for gaming.)

## 7. Left to the builder
- Exact CLI flags for sau and auto-x. Confirm with `--help` on install, don't guess.
- Python version/venv layout on the server, and cron vs systemd timers.
- How carousel PNGs are rendered (HTML → PNG with headless Chrome is the default).
- Log rotation once `posts.jsonl` grows.

## 8. Open items (owner)
1. **Repo is public** (owner's choice, 27 Nov, because private access wouldn't connect). So nothing secret ever goes in it: cookies, tokens and keys stay on the server. Queued posts, the posting log and profile briefs (including pasted LinkedIn text) are readable by anyone. Switch back to private once Claude's GitHub app is given private-repo access.
2. Which PC stores files (footage, backups). Owner will confirm.
3. Second LinkedIn presence: another real person's profile, or a company page (which would be "Claude writes, owner clicks post").
4. Posting time per account (default 19:00 IST).
5. Dashboard: Claude artifact (default) or a live page on the server.

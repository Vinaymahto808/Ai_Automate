# AI Automated Ads

A self-hosted app that **generates, previews, approves, deploys, schedules, monitors and optimises** real ad campaigns on **Meta Ads and Google Ads**, and **publishes free organic posts to a Facebook Page**. It uses Claude for copy and strategy.

Nothing here is simulated. Campaigns and posts are created through the official platform APIs in your own accounts. Analytics come from the platforms' reporting APIs, and if a platform call fails, the app shows the platform's real error.

---

## Contents

1. [Quick start](#1-quick-start)
2. [Configure `.env`](#2-configure-env)
3. [Running the server](#3-running-the-server)
4. [Connect Meta (Facebook / Instagram ads + Page posts)](#4-connect-meta)
5. [Connect Google Ads](#5-connect-google-ads)
6. [Using the app](#6-using-the-app)
7. [Posting to your Facebook Page](#7-posting-to-your-facebook-page)
8. [File structure](#8-file-structure)
9. [API reference](#9-api-reference)
10. [Security](#10-security)
11. [Troubleshooting](#11-troubleshooting)
12. [Operational notes](#12-operational-notes)

---

## 1. Quick start

Requires **Node.js 22.13 or newer**. The app uses the built-in `node:sqlite`, so there are no native builds and no separate database to install.

Every command below must be run **inside the `automate_social_media_post` folder**, which is where `package.json` lives:

```powershell
cd C:\Users\Galaxy\Desktop\ai_automate\automate_social_media_post
npm install
copy .env.example .env      # macOS/Linux: cp .env.example .env
# fill in .env (see section 2)
npm run dev                 # or: npm start
```

Open the URL set in `PUBLIC_URL` (for example http://localhost:3001) and log in with `ADMIN_EMAIL` / `ADMIN_PASSWORD`.

| Command | What it does |
|---|---|
| `npm run dev` | Starts the server and restarts it automatically when a file in `src/` changes. It does **not** reload `.env`; restart it by hand after editing `.env`. |
| `npm start` | Starts the server once, without watching for changes. |
| `run_server.bat` | Windows watchdog: runs `node src/server.js` in a loop, restarts it 5 seconds after a crash, and appends output to `data\server.log`. |

> Run **only one** of these at a time. They all use the same port, and a second copy can't start while the first one holds it (see [Troubleshooting](#11-troubleshooting)).

---

## 2. Configure `.env`

Copy `.env.example` to `.env` and fill it in. `.env` contains secrets: never commit it or share it. It's already listed in `.gitignore`.

**Minimum to start:** `SESSION_SECRET`, `ENCRYPTION_KEY`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, and an AI key: `GROQ_API_KEY` or `ANTHROPIC_API_KEY`.

Generate `SESSION_SECRET` and `ENCRYPTION_KEY` (64 hex characters each) with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

| Variable | Required | Description |
|---|---|---|
| `PORT` | yes | Port the server listens on (e.g. `3001`). |
| `PUBLIC_URL` | yes | Base URL of the app, e.g. `http://localhost:3001`. OAuth redirect URIs are built from it, so it must match the port. |
| `NODE_ENV` | no | `development` or `production`. Production enables secure cookies and HSTS (needs HTTPS). |
| `DATA_DIR` | no | Where the database, creatives and uploads are stored. Default `./data`. |
| `SESSION_SECRET` | yes | 64 hex chars. Signs login cookies. |
| `ENCRYPTION_KEY` | yes | 64 hex chars. Encrypts platform tokens in the database. **Losing it means reconnecting every platform.** |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | first run | Creates the first admin when no users exist. Password must be 12+ characters. Use a strong password. |
| `ANTHROPIC_API_KEY` | one AI key | Claude API key, used for AI generation and optimisation. |
| `ANTHROPIC_MODEL` | no | Default `claude-opus-5-5`. |
| `GROQ_API_KEY` | alternative to Claude | Groq API key from [console.groq.com/keys](https://console.groq.com/keys). Used automatically when `ANTHROPIC_API_KEY` is empty. |
| `GROQ_MODEL` | no | Default `openai/gpt-oss-120b`. |
| `AI_PROVIDER` | no | Force the AI provider: `groq` or `anthropic`. Empty = Groq if only a Groq key is set, otherwise Claude. |
| `META_APP_ID`, `META_APP_SECRET` | for Meta | From your Meta developer app (App settings → Basic). |
| `META_API_VERSION` | no | Default `v24.0`. |
| `META_CONFIG_ID` | only for Login for Business apps | Configuration ID from Facebook Login for Business. When set, it replaces the requested scope list. See [4.4](#44-facebook-login-for-business-apps-meta_config_id). |
| `GOOGLE_ADS_CLIENT_ID`, `GOOGLE_ADS_CLIENT_SECRET` | for Google | OAuth client (Web application) from Google Cloud Console. |
| `GOOGLE_ADS_DEVELOPER_TOKEN` | for Google | From your Google Ads manager account (API Center). |
| `GOOGLE_ADS_LOGIN_CUSTOMER_ID` | no | Manager (MCC) ID, digits only, if it isn't detected automatically. |
| `GOOGLE_ADS_API_VERSION` | no | Default `v21`. |
| `MAX_DAILY_BUDGET_PER_CAMPAIGN`, `MAX_TOTAL_BUDGET_PER_CAMPAIGN` | no | Hard budget caps per campaign, in account currency. Admin settings can tighten them but never raise them. |
| `SCHEDULER_TICK_SECONDS` | no | How often the scheduler runs. Default `60`. |
| `METRICS_SYNC_MINUTES` | no | How often live metrics are synced. Default `30`. |
| `AI_OPTIMIZE_EVERY_HOURS` | no | How often Claude reviews active campaigns. Default `24`. |
| `SMTP_URL`, `NOTIFY_EMAIL_FROM`, `NOTIFY_EMAIL_TO` | no | Email notifications, e.g. `smtps://user:pass@smtp.example.com:465`. |
| `SLACK_WEBHOOK_URL` | no | Slack notifications. |

---

## 3. Running the server

1. Open a terminal in `automate_social_media_post`.
2. Run `npm run dev`.
3. Open `PUBLIC_URL` in the browser.

The server prints nothing to the terminal, even on startup or when a request fails. To check that it's running, open the URL, or run:

```powershell
netstat -ano | findstr :3001
```

To stop the server, press `Ctrl+C` in its terminal. If you started `run_server.bat`, close its window; closing only the `node` process makes the watchdog start a new one.

---

## 4. Connect Meta

Meta needs a developer app. Its keys go in `.env`; then you connect your Facebook account from inside the app.

### 4.1 Create the Meta app

1. Go to [developers.facebook.com](https://developers.facebook.com) → **My Apps** → **Create app**.
2. Copy **App ID** and **App secret** (App settings → Basic) into `META_APP_ID` and `META_APP_SECRET`.
3. Under Facebook Login settings, add this **Valid OAuth Redirect URI**:
   ```
   {PUBLIC_URL}/api/oauth/meta/callback
   e.g. http://localhost:3001/api/oauth/meta/callback
   ```

### 4.2 Add the use cases and permissions

In the app dashboard, open **Use cases**:

1. **Ads**: add **"Create & manage ads with Marketing API"**. This covers `ads_management`, `ads_read` and `business_management`.
2. **Page posts**: add **"Manage everything on your Page"**. Click **Customize** next to it, then click **Add** next to each of these permissions:
   - `pages_manage_posts` (**required to post to a Page**)
   - `pages_show_list`
   - `pages_read_engagement`

   If the use case isn't listed, click **Add use cases** first.

The app requests these permissions when you connect:

```
ads_management, ads_read, business_management,
pages_show_list, pages_read_engagement, pages_manage_posts
```

While the app is in **Development mode**, only people with a role on the app (admin, developer or tester) can grant these permissions. You don't need App Review for that. To manage ad accounts or Pages that belong to other people, you need **Advanced Access** through Meta's App Review.

### 4.3 Connect inside the app

1. Restart the server if you changed `.env`.
2. Go to **Connections → Meta → Connect** (or **Reconnect**).
3. On the Facebook screen, click **Edit settings / Edit access**, tick your **Page** and ad account, and make sure **"Manage posts on your Page"** is turned on.
4. Back in the app, **select the ad account** and the **Page**. Also select a **Pixel** if you'll run Leads or Sales campaigns.

The token lasts about 60 days. The app warns you 7 days before it expires; then just click **Reconnect**.

### 4.4 Facebook Login for Business apps (`META_CONFIG_ID`)

Some Meta apps use **Facebook Login for Business**. Those apps ignore the permission list the app sends and grant only what's in a saved configuration. If reconnecting never adds `pages_manage_posts`, set up a configuration:

1. In the app dashboard, open the **Facebook Login for Business** use case → **Customize** → **Configurations** → **Create configuration**.
2. Choose **User access token**, select **Pages** (and ad accounts) as assets, and tick the six permissions listed in 4.2.
3. Copy the **Configuration ID** into `.env`:
   ```
   META_CONFIG_ID=1234567890
   ```
4. Restart the server and reconnect Meta.

If adding `pages_manage_posts` in step 4.2 is enough, leave `META_CONFIG_ID` empty.

---

## 5. Connect Google Ads

1. In [Google Cloud Console](https://console.cloud.google.com), create a project, enable the **Google Ads API**, and create an **OAuth client ID** of type *Web application*.
2. Add this authorised redirect URI:
   ```
   {PUBLIC_URL}/api/oauth/google/callback
   ```
3. Put the client ID and secret in `GOOGLE_ADS_CLIENT_ID` / `GOOGLE_ADS_CLIENT_SECRET`.
4. In your Google Ads **manager account**, open **Tools → API Center** and copy the **developer token** into `GOOGLE_ADS_DEVELOPER_TOKEN`.
5. Restart the server, then go to **Connections → Google Ads → Connect** and pick an account.

A test-level developer token only works on test accounts. Apply for **Basic access** to use real accounts. Manager (MCC) accounts are expanded to their client accounts automatically.

---

## 6. Using the app

### Workflow

```
Brief ──► AI generates ──► Preview & edit ──► Approve ──► Deploy (paused) ──► Auto-start ──► Monitor ──► Auto-end
          strategy +        (local mocks +     (manager)    all-or-nothing     at start       sync, rules,   at end time
          N variations       live Meta preview)              with rollback      time           AI suggestions or budget cap
```

1. **Brief** (*New campaign*): product or service, offer details, landing URL, audience, ages, objective, platforms, locations (looked up through each platform's geo search), total budget, start and end time, tone, language and an optional product photo.
2. **AI generation**: Claude returns a targeting strategy (Meta interests, Google keywords, negative keywords) and 2–5 variations. Each variation has primary text, 5–10 headlines (≤30 chars), 2–4 descriptions (≤90 chars), a CTA and a creative design. Platform length limits are checked, and a repair pass runs automatically if one is broken.
3. **Creatives**: the AI design is rendered into real 1080×1080 PNGs for Meta, optionally on top of your product photo. You can also upload your own finished PNG.
4. **Preview**: in-app feed and search previews for every platform, plus a **live Meta preview** rendered by Meta itself (nothing is published).
5. **Approve**: a manager approves with an explicit budget confirmation. Any later edit cancels the approval. Optional four-eyes rule: the creator can't approve their own campaign.
6. **Deploy**: everything is created **paused** on every platform. If any platform fails, everything created in that attempt is rolled back and the campaign stays approved, so you can fix the problem and retry.
7. **Schedule**: the scheduler activates the campaign at its start time and ends it at its end time. Meta ad sets also carry the end date themselves, as a backstop if this server is down.
8. **Monitor**: metrics sync every 30 minutes (configurable). Rules can pause or end the campaign, pause a weak variation, or just notify. A built-in guard ends the campaign once spend reaches the approved total.
9. **Optimise**: Claude analyses the real metrics and suggests actions (pause a variation, change the daily budget). A **manager must click Apply**; the AI never changes live campaigns by itself. Budget changes are clamped so they can't exceed the approved total.
10. **Notifications**: in-app, plus optional email and Slack.
11. **Audit log**: an append-only record of every user and system action. Database triggers block edits and deletes.

### What gets created on each platform

| | Meta | Google Ads |
|---|---|---|
| Container | Campaign (ODAX objective) | Campaign budget + Search campaign |
| Targeting | Ad set: locations, ages, AI interests (resolved to real IDs) | Location criteria + phrase-match keywords |
| Ads | Image link ad per variation (Page identity) | One responsive search ad per variation (max 3) |
| Pause / End | PAUSED / ARCHIVED | PAUSED / PAUSED (history kept) |

Objectives (awareness, traffic, engagement, leads, sales) map to each platform's equivalent. **Leads and Sales on Meta require a Pixel**, selected on the Connections page.

### Roles

| Role | Can do |
|---|---|
| **editor** | Create campaigns, generate and edit copy, upload images, post to the Page, view everything |
| **manager** | Everything an editor can, plus approve, deploy, pause/end, change budgets, manage rules, apply AI suggestions, connect platforms, view the audit log |
| **admin** | Everything a manager can, plus manage users and settings, and disconnect platforms |

---

## 7. Posting to your Facebook Page

Organic Page posts are **free**: they're not ads and don't use any budget.

**Before you start:** Meta must be connected, a Page must be selected, and the connection must include `pages_manage_posts` (section 4). If the permission is missing, the page shows a **"Posting permission missing"** banner with a **Reconnect Meta** button, and the post button stays disabled.

1. Open **Page post** in the menu (`{PUBLIC_URL}/#/page-post`, e.g. http://localhost:3001/#/page-post).
2. Write a message (up to 5,000 characters), attach an image, or both.
   - Images must be **PNG or JPEG**, up to **8 MB**.
   - Optional **Link (optional)** field: must start with `http://` or `https://`.
3. Click **Post to page**. The post goes live on Facebook immediately.
4. A confirmation appears for 10 seconds with the post's link: `Posted — https://www.facebook.com/...`.
5. The **Recent page posts** list below the form shows the Page's latest posts, each with a **view on Facebook** link.

---

## 8. File structure

```
automate_social_media_post/
├── .env                      Your secrets and settings (git-ignored, never share)
├── .env.example              Template for .env with every supported variable
├── .gitignore                Ignores node_modules/, .env, data/
├── package.json              Dependencies and npm scripts (start, dev)
├── package-lock.json         Exact dependency versions
├── README.md                 This file
├── run_server.bat            Windows watchdog: keeps the server running, writes data\server.log
│
├── public/                   Browser UI (single-page app, no build step)
│   ├── index.html            Page shell; loads app.js and styles.css
│   ├── app.js                All screens: dashboard, campaigns, Page post, connections, settings
│   └── styles.css            Styling
│
├── src/                      Server code (Node.js, ES modules)
│   ├── server.js             Entry point: Express app, security headers, API routes, static files, starts the scheduler
│   ├── config.js             Reads and validates .env
│   ├── db.js                 SQLite schema and query helpers (node:sqlite)
│   ├── auth.js               Sessions, login cookies, roles, CSRF guard, first-admin bootstrap
│   ├── crypto.js             AES-256-GCM encryption, scrypt password hashing, HMAC, random tokens
│   ├── audit.js              Append-only audit log
│   ├── notify.js             In-app, email (SMTP) and Slack notifications
│   │
│   ├── ai/                   AI integration (Claude or Groq)
│   │   ├── claude.js         generateJSON(): Claude client with structured outputs; hands off to Groq when selected
│   │   ├── groq.js           Groq client (OpenAI-compatible API, JSON-schema outputs, retries)
│   │   ├── generate.js       Generates targeting strategy and ad variations, enforces length limits
│   │   ├── optimize.js       Analyses live metrics and proposes optimisation actions
│   │   └── creative.js       Renders AI creative designs to PNG (SVG → PNG with resvg)
│   │
│   ├── platforms/            Ad platform integrations
│   │   ├── index.js          Maps platform names to their adapters
│   │   ├── connections.js    Stores encrypted tokens, account/Page selection, connection status
│   │   ├── http.js           HTTP requests with retries and platform error handling
│   │   ├── money.js          Currency unit conversion (major/minor units, micros)
│   │   ├── meta.js           Meta: OAuth, ad accounts, pixels, campaigns/ad sets/ads, metrics, Page posts
│   │   └── google.js         Google Ads: OAuth, accounts, budgets, Search campaigns, ads, metrics
│   │
│   ├── services/             Business logic
│   │   ├── campaigns.js      Campaign and variation data access
│   │   ├── lifecycle.js      Deploy, rollback, and status changes (activate/pause/end)
│   │   ├── metrics.js        Syncs real metrics from the platforms
│   │   ├── rules.js          Evaluates pause/end/notify rules and the total-spend guard
│   │   ├── optimizer.js      Stores and applies AI suggestions
│   │   └── scheduler.js      Background loop: start/end, metric sync, rules, AI optimisation, token expiry warnings
│   │
│   └── routes/               REST API (mounted under /api)
│       ├── auth.js           Login, logout, current user, users, password change
│       ├── campaigns.js      Campaigns, AI generation, variations, approval, deploy, budgets, rules, suggestions
│       ├── connections.js    Platform connections, OAuth start and callback, account/pixel selection, locations
│       ├── misc.js           Dashboard, notifications, audit log, settings, Page posts, file serving
│       └── util.js           Shared helpers: async handler wrapper, HTTP errors
│
├── data/                     Created at runtime (git-ignored); back it up
│   ├── ads.db                SQLite database (plus ads.db-shm / ads.db-wal working files)
│   ├── creatives/            Generated ad images (PNG)
│   ├── uploads/              Uploaded product photos
│   └── server.log            Output from run_server.bat
│
└── node_modules/             Installed dependencies (created by npm install)
```

### Dependencies

| Package | Used for |
|---|---|
| `express` | Web server and API |
| `@anthropic-ai/sdk` | Claude API |
| `@resvg/resvg-js` | Rendering SVG creatives to PNG |
| `multer` | Image uploads |
| `nodemailer` | Email notifications |
| `dotenv` | Loading `.env` |

---

## 9. API reference

All endpoints are under `/api`, require a logged-in session, and (for anything other than GET) the CSRF header the UI sends. The role shown is the minimum required.

| Method | Path | Role | Purpose |
|---|---|---|---|
| POST | `/auth/login` | none | Log in |
| POST | `/auth/logout` | any | Log out |
| GET | `/auth/me` | any | Current user |
| POST | `/auth/password` | editor | Change own password |
| GET / POST | `/users` | admin | List / create users |
| DELETE | `/users/:id` | admin | Delete a user |
| GET | `/dashboard` | editor | Dashboard summary |
| GET | `/notifications` | editor | Notifications |
| POST | `/notifications/read-all` | editor | Mark all notifications as read |
| GET | `/audit` | manager | Audit log |
| GET / PUT | `/settings` | editor / admin | Read / update settings |
| POST | `/uploads/product` | editor | Upload a product photo |
| GET / POST | `/campaigns` | editor | List / create campaigns |
| GET / PUT | `/campaigns/:id` | editor | Read / update a campaign |
| DELETE | `/campaigns/:id` | manager | Delete a campaign |
| POST | `/campaigns/:id/generate` | editor | AI-generate strategy and variations |
| PUT | `/campaigns/:id/strategy` | editor | Edit targeting strategy |
| PUT | `/campaigns/:id/variations/:vid` | editor | Edit a variation |
| POST | `/campaigns/:id/variations/:vid/regenerate` | editor | Regenerate one variation |
| POST | `/campaigns/:id/variations/:vid/creative` | editor | Upload or re-render the creative |
| POST | `/campaigns/:id/variations/:vid/select` | editor | Include/exclude a variation |
| GET | `/campaigns/:id/variations/:vid/meta-preview` | editor | Live Meta ad preview |
| POST | `/campaigns/:id/approve` / `unapprove` | manager | Approve / revoke approval |
| POST | `/campaigns/:id/deploy` | manager | Deploy to platforms (paused) |
| POST | `/campaigns/:id/status` | manager | Activate / pause / end |
| PUT | `/campaigns/:id/budget` | manager | Change budget |
| POST | `/campaigns/:id/variations/:vid/status` | manager | Pause/resume one variation |
| POST | `/campaigns/:id/sync` | editor | Sync metrics now |
| POST | `/campaigns/:id/rules` | manager | Add a rule |
| PATCH / DELETE | `/rules/:rid` | manager | Update / delete a rule |
| POST | `/campaigns/:id/optimize` | editor | Ask Claude for suggestions |
| POST | `/suggestions/:sid/apply` | manager | Apply a suggestion |
| POST | `/suggestions/:sid/dismiss` | editor | Dismiss a suggestion |
| GET | `/connections` | editor | Connection status for all platforms |
| POST | `/connections/:platform/oauth/start` | manager | Start the OAuth login |
| GET | `/oauth/:platform/callback` | (redirect) | OAuth return URL |
| GET | `/connections/:platform/accounts` | manager | Ad accounts available to the token |
| GET | `/connections/meta/pixels` | manager | Meta pixels |
| POST | `/connections/:platform/account` | manager | Select account / Page / pixel |
| DELETE | `/connections/:platform` | admin | Disconnect |
| GET | `/connections/:platform/locations` | editor | Location search |
| GET | `/meta/page-status` | editor | Whether the token can post to the Page |
| GET | `/meta/page-posts` | editor | Recent Page posts |
| POST | `/meta/page-post` | editor | Publish a Page post (multipart: `message`, `link`, `file`) |
| GET | `/files/creatives/:file`, `/files/uploads/:file` | editor | Serve generated creatives / uploads |

---

## 10. Security

- Platform tokens are encrypted at rest with **AES-256-GCM** (`ENCRYPTION_KEY`) and never sent to the browser. Meta calls include `appsecret_proof`.
- Sessions use signed httpOnly cookies. Mutating requests require a custom header (CSRF guard). Login attempts are throttled.
- OAuth `state` is one-time, expires after 15 minutes and is bound to the user who started the flow.
- Account selection is re-checked on the server against what the token can actually access.
- Hard budget caps live in `.env`; admin settings can only tighten them.
- A strict Content Security Policy is applied. Uploads are validated by their file signature (magic bytes), and file serving is authenticated with path-traversal checks.
- In production, run behind HTTPS with `NODE_ENV=production` (secure cookies and HSTS).
- Use a strong `ADMIN_PASSWORD` and change it after first login.

---

## 11. Troubleshooting

The server doesn't print logs or errors to the terminal. Errors are shown in the browser (red banners and toasts), and platform errors include Meta's or Google's own message.

| Problem | Cause | Fix |
|---|---|---|
| `npm error enoent Could not read package.json` | Command run from the wrong folder (e.g. `ai_automate`) | `cd automate_social_media_post` first, then run the command. |
| Changes don't appear, or the new server seems not to start | Another copy of the server (often `run_server.bat`) still holds the port | Find it with `netstat -ano \| findstr :3001`, close the watchdog window or stop that process, then start one server only. |
| `.env` changes have no effect | `npm run dev` only watches code files | Stop and restart the server. |
| "Posting permission missing" / "Meta must be reconnected with posting permission" | The Meta token doesn't include `pages_manage_posts` | Add the permission (section 4.2), then **Connections → Meta → Reconnect** and turn on "Manage posts on your Page". |
| Reconnect never adds `pages_manage_posts` | App uses Facebook Login for Business | Create a configuration and set `META_CONFIG_ID` (section 4.4). |
| Facebook says "Invalid Scopes" | The permission isn't added to the Meta app | Add it under **Use cases** (section 4.2). |
| Facebook says "URL blocked" / redirect URI error | Redirect URI not registered, or `PUBLIC_URL`/port mismatch | Register `{PUBLIC_URL}/api/oauth/meta/callback` exactly; make sure `PUBLIC_URL` uses the same port as `PORT`. |
| "OAuth state invalid or expired" | Login took more than 15 minutes, or you were logged out of the app | Log in to the app and click Connect/Reconnect again. |
| "No Facebook Page selected" | No Page chosen after connecting | **Connections → Meta → Select account** and pick the Page. |
| "No AI key is set" / "ANTHROPIC_API_KEY is not set on the server" | No AI key | Set `GROQ_API_KEY` (or `ANTHROPIC_API_KEY`) in `.env` and restart. |
| "Groq API key was rejected" | Wrong or revoked Groq key | Create a new key at console.groq.com/keys. |
| "Groq API rate limit reached" | Free-tier limits | Wait a minute and retry, or generate fewer variations. |
| "ENCRYPTION_KEY must be 64 hex characters" | Key missing or wrong length | Generate one with the command in section 2. |
| Can't log in on first run | No admin was created | Set `ADMIN_EMAIL` and `ADMIN_PASSWORD` (12+ chars) **before** the first start. They're only used when no users exist. |
| Google: "developer token is only approved for test accounts" | Test-level developer token | Apply for Basic access in the Google Ads API Center. |
| Meta Leads/Sales campaign fails | No Pixel selected | Select a Pixel on the Connections page. |

---

## 12. Operational notes

- Run a **single instance**: the scheduler and per-campaign locks live in the server process.
- Back up `data/`, and keep `ENCRYPTION_KEY` safe. Losing the key means reconnecting every platform.
- Google Search campaigns have no end date on Google's side. The scheduler pauses them at the end time, and the total-spend guard also applies.
- Before the first real campaign, test each connection with a small budget and a near-term schedule.
- API versions are configurable (`META_API_VERSION`, `GOOGLE_ADS_API_VERSION`), so you can follow each platform's deprecation schedule without code changes.
#   A i _ A u t o m a t e  
 
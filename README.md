# REDTUBE — Setup & Deploy Guide

This app has two parts:
- **Frontend** (`public/`) — the Telegram Mini App UI (index.html, app.js, style.css) and the admin panel (admin.html, admin.js)
- **Backend** (`api/`) — serverless functions that talk to MongoDB and Telegram

⚠️ **Important:** GitHub only stores your code — it does not run it. If you just push this
to a GitHub repo and open the raw file, the loading screen will show but nothing after it
will work, because `/api/...` calls have nowhere to go. You need to deploy this on **Vercel**
(free), connected to your GitHub repo, so the `api/` folder actually runs as live functions.

---

## Step 1 — Create your Telegram bot
1. Message **@BotFather** on Telegram → `/newbot` → follow the prompts.
2. Save the token it gives you (looks like `123456:ABC-DEF...`). You will paste this into
   Vercel as an environment variable — **never commit it into any file in your repo.**

## Step 2 — Create a free MongoDB database
1. Go to [MongoDB Atlas](https://www.mongodb.com/atlas), create a free cluster.
2. Create a database user + password.
3. Get your connection string (Atlas → Connect → Drivers), it looks like:
   `mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/?appName=Cluster0`
4. Same rule as above: this goes into Vercel's environment variables, not into your code.

## Step 3 — Push this project to GitHub
Push this whole folder (`api/`, `public/`, `package.json`, this README) to a GitHub repo.

## Step 4 — Deploy on Vercel
1. Go to [vercel.com](https://vercel.com) → **Add New Project** → import your GitHub repo.
2. Before deploying, add these **Environment Variables** in the Vercel project settings:

| Variable | Value |
|---|---|
| `BOT_TOKEN` | your Telegram bot token from Step 1 |
| `MONGODB_URI` | your MongoDB connection string from Step 2 |
| `ADMIN_PASSWORD` | any password you choose for `/admin.html` |
| `WEBAPP_URL` | your Vercel URL once known, e.g. `https://your-project.vercel.app` |
| `BOT_USERNAME` | your bot's username without `@`, e.g. `RedTube_bot` |

3. Deploy. Vercel gives you a URL like `https://your-project.vercel.app`.
4. Go back into the environment variables and set `WEBAPP_URL` to that real URL, then redeploy
   (Vercel → Deployments → ⋯ → Redeploy) so `bot.js` sends the right link.

## Step 5 — Connect the bot to your deployed app
Run this once (replace `<BOT_TOKEN>` and `<YOUR_URL>`), from any terminal/browser:
```
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=<YOUR_URL>/api/bot
```
This tells Telegram to forward messages to your `api/bot.js` function.

## Step 6 — Try it
Open your bot in Telegram, tap **/start**, then tap **🚀 Open REDTUBE**.

---

## What each file does

**Backend (`api/`)**
- `bot.js` — receives Telegram messages, replies to `/start` with the "Open REDTUBE" button, creates the user record and stores who referred them.
- `_db.js` — shared MongoDB connection helper used by every other file.
- `_telegram.js` — helper to call the Telegram API and to check the admin password on admin routes.
- `user.js` — creates/fetches a user, and verifies channel-join status (both channels must be joined before the join-gate unlocks).
- `earn.js` — handles "watch ad" rewards per network (Adsgram Daily/Special, Monetag, GigaPub) with daily limits.
- `task.js` — lists active tasks for users and accepts their proof submissions (text + screenshot, no passwords).
- `promo.js` — lets a user redeem a promo code you created in the admin panel.
- `referral.js` — returns a user's referral link/stats and the top-20 referrer leaderboard.
- `withdraw.js` — creates a withdraw request (Binance/Tonkeeper/bKash) and lists a user's own withdraw history.
- `admin/users.js` — admin: look up a user by UID/username, manually add or subtract balance.
- `admin/withdraws.js` — admin: list and approve/reject withdraw requests.
- `admin/tasks.js` — admin: create/delete tasks, and approve/reject task submissions.
- `admin/promo.js` — admin: create promo codes with a reward + claim limit.

**Frontend (`public/`)**
- `index.html` / `app.js` / `style.css` — the Mini App itself: loading screen → join-gate → Home/Video/Earning/Task/Refer tabs, withdraw modal, history modal, profile modal, promo modal.
- `admin.html` / `admin.js` — your private admin dashboard (password-protected). Open it at `https://your-project.vercel.app/admin.html`.

---

## Notes on what's already wired up
- Screenshot proofs are compressed to a small JPEG in the browser and stored directly in
  MongoDB — no third-party file host needed. Admin panel shows thumbnails you can click to
  view full-size.
- Withdraw minimums: Binance 2000 WTC, Tonkeeper 1600 WTC, bKash 5000 WTC (bKash asks for a
  phone number, the other two ask for a wallet address/UID).
- 20 WTC = $0.001 (i.e. 1 WTC = $0.00005) is used everywhere balances are converted to USD.
- Top 20 referrers are simply whoever has the most referrals — no separate reward tier.
- The join-gate only shows once: once a user is marked `joined: true` in the database it
  won't ask again, even after restarting the bot — unless they never actually joined, in
  which case it will keep showing until they do.

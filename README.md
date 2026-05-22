# zk-doctor-bot — GitHub App setup guide

This is the GitHub App that runs [zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) on every PR opened in a registered ZK project repo, and posts a review comment with the health score + findings.

Tier 2.1 in our auto-revenue plan: $39–99/mo per team. Niche completely open (CodeRabbit doesn't do ZK).

## Architecture

```
GitHub PR opened/sync
   ↓
GitHub webhook → POST https://smee.io/<zk-doctor-bot-channel>
   ↓
mini github_app_listener.py (launchd KeepAlive=true)
   ↓ (validates signature with webhook secret)
For each pull_request.opened/synchronize event:
   ↓
Clone the PR's head branch via gh CLI
   ↓
Run zk-doctor . --format json
   ↓
Frontier LLM enriches raw scores into narrative
   ↓
gh pr comment <pr_number> --body "<markdown report>" --repo <owner>/<repo>
   ↓
DB log to github_app_runs table
   ↓
Telegram bilingual alert: "🩺 zk-doctor reviewed <repo>#<pr> — score X/10"
```

Free tier: 3 PRs/month per repo, public report comment.
Paid tier: unlimited PRs, private reports if configured, priority detector requests.

## User one-time setup (5 minutes)

### Step 1: Create the GitHub App

1. Go to <https://github.com/settings/apps/new>
2. Fill in:
   - **GitHub App name**: `zk-doctor-bot`
   - **Homepage URL**: `https://github.com/Battam1111/zk-pipeline-doctor`
   - **Webhook URL**: (use the smee.io channel — see step 3 to create)
   - **Webhook secret**: generate a 32-byte random string (or let me give you one)
3. **Permissions** (Repository):
   - **Pull requests**: Read & write (to post comments)
   - **Contents**: Read (to clone)
   - **Metadata**: Read (default)
   - **Checks**: Read & write (optional, for status checks)
4. **Subscribe to events**:
   - Pull request
   - Push (optional, for force-push handling)
5. **Where can this GitHub App be installed?**: Any account (you want public installability eventually)
6. Click **Create GitHub App**

### Step 2: Generate a private key

1. On the App's settings page, scroll to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file downloads — save it as `secrets/zk-doctor-bot.pem` on the mini (chmod 600)
4. Note the **App ID** (top of the page) — save it

### Step 3: Smee.io channel for webhook delivery

1. Go to <https://smee.io/new>
2. Copy the URL (something like `https://smee.io/aBcDeFgHiJkLmN`)
3. Update the GitHub App's webhook URL to this smee URL
4. The mini's `github_app_listener.py` will connect to the same URL via SSE

### Step 4: Install on a test repo

1. On the App settings page, **Install App** (left sidebar)
2. Pick one of your repos as a test target
3. Open a PR on that repo — the App will fire its first review

### Step 5: Provide credentials to the system

Drop a file `secrets/github-app.json` on the mini:

```json
{
  "app_id": 123456,
  "private_key_path": "secrets/zk-doctor-bot.pem",
  "webhook_secret": "the-32-byte-secret-you-set",
  "smee_url": "https://smee.io/your-channel",
  "installation_ids": {
    "Battam1111/test-repo": 7890123
  }
}
```

(installation_id is shown in the URL after installing the app.)

Then restart the listener:

```bash
launchctl unload ~/Library/LaunchAgents/com.chenyanyun.bounty.github_app.plist
launchctl load ~/Library/LaunchAgents/com.chenyanyun.bounty.github_app.plist
```

## Verifying it works

Open a new PR on your test repo. Within 30 seconds, the App should:
1. Fire a webhook → smee → mini
2. Clone your PR branch to /tmp
3. Run zk-doctor + Frontier
4. Post a review comment on your PR

Telegram should also ping you (bilingual): `🩺 zk-doctor reviewed test-repo#1 — overall 6.5/10`.

## Going to paid tier

Once you have 3-5 users on the free tier and at least one paid customer, list on GitHub Marketplace:

1. <https://github.com/marketplace/new>
2. Use the App's existing config
3. Create paid plans (e.g., $39/mo Hobby, $99/mo Team)
4. Stripe Connect onboarding (similar to Polar)

For now, paid tier intake is handled via Polar (`Bounty Radar Team $497/mo` includes priority detector requests for zk-doctor) — the GitHub App stays free as the "discovery wedge."

## Sibling projects

- [zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) — the OSS CLI this bot runs
- [bounty-radar-data](https://battam1111.github.io/bounty-radar-data/) — live ZK + AI bounty feed (paid SaaS sibling)
- [midnight-zk-cookbook](https://battam1111.github.io/midnight-zk-cookbook/) — tutorials

---

<!-- related-projects:start -->

## Related projects

- [**zk-pipeline-doctor**](https://github.com/Battam1111/zk-pipeline-doctor) — OSS CLI the bot wraps
- [**zk-doctor-action**](https://github.com/Battam1111/zk-doctor-action) — GitHub Action — free CI tier (no bot install needed)
- [**midnight-zk-cookbook**](https://github.com/Battam1111/midnight-zk-cookbook) — ZK tutorials + paid audits + bundles

<!-- related-projects:end -->

# claude-haiku-monitor

> A tiny GitHub Actions monitor that pings you on Telegram the moment the Claude Haiku 5.5 page goes live.

## How it works

A scheduled workflow ([`.github/workflows/monitor.yml`](.github/workflows/monitor.yml)) runs every 5 minutes:

1. `curl`s `https://www.anthropic.com/claude-haiku-5-5` (no redirect following) and reads the HTTP status code.
2. If the status is **200**, it sends a message to your Telegram chat through the Bot API.
3. Right after alerting, the workflow **disables itself**, so you get exactly one notification instead of one every 5 minutes.

Any other status (404 = not launched yet, but also 403 from bot protection, 5xx, or a network error) is **not** treated as a launch. Unexpected codes other than 404 show up as a warning annotation in the run log so you can spot, for example, the runner being blocked.

Redirects are intentionally not followed: a redirect to the home page would otherwise end in a 200 and trigger a false alert.

## Setup

### 1. Create the Telegram bot

1. Open [@BotFather](https://t.me/BotFather) in Telegram and send `/newbot`. Follow the prompts and copy the **bot token**.
2. Send any message (e.g. `/start`) to your new bot. Bots can't message you until you write first.
3. Get your **chat ID** by opening this URL in a browser (replace `<TOKEN>`):

   ```
   https://api.telegram.org/bot<TOKEN>/getUpdates
   ```

   Look for `"chat":{"id": 123456789, ...}` in the response. For a group, add the bot to the group, send a message there, and use the (negative) group ID.

### 2. Add the repository secrets

In [Settings → Secrets and variables → Actions](https://github.com/Guzz7/claude-haiku-monitor/settings/secrets/actions), create:

| Secret               | Value                      |
| -------------------- | -------------------------- |
| `TELEGRAM_BOT_TOKEN` | The token from BotFather   |
| `TELEGRAM_CHAT_ID`   | Your chat (or group) ID    |

Or with the GitHub CLI:

```bash
gh secret set TELEGRAM_BOT_TOKEN --repo Guzz7/claude-haiku-monitor
gh secret set TELEGRAM_CHAT_ID   --repo Guzz7/claude-haiku-monitor
```

### 3. Enable the workflow

Push this repository to <https://github.com/Guzz7/claude-haiku-monitor>, then check that the workflow is enabled in the [Actions tab](https://github.com/Guzz7/claude-haiku-monitor/actions). Scheduled workflows only run from the default branch.

## Testing

You can't test against the real page while it still returns 404, so use the manual trigger with a URL that returns 200:

```bash
gh workflow run monitor.yml --repo Guzz7/claude-haiku-monitor -f url=https://www.anthropic.com
```

You should receive the Telegram message. Manual runs that override the URL **do not** disable the workflow, so the real monitor keeps running afterwards.

## Customizing

- **Different page:** change `TARGET_URL` in the workflow's `env` block.
- **Different interval:** change the `cron` expression. Note GitHub's minimum is every 5 minutes.
- **Re-arm after it fired:** re-enable the workflow with `gh workflow enable monitor.yml --repo Guzz7/claude-haiku-monitor` (or from the Actions tab).

## Limitations

- **Scheduling is best-effort.** GitHub may delay scheduled runs by several minutes (sometimes more) during peak load, so this is near-real-time, not real-time.
- **Inactivity pause.** In public repositories, GitHub automatically disables scheduled workflows after 60 days without repository activity.
- **Minutes quota.** Running every 5 minutes is ~8,600 runs per month. That is free for public repositories, but it exceeds the free minutes of a private repository (2,000/month on the Free plan). Keep the repo public or use a longer interval.
- **Bot protection.** If the site starts answering GitHub's runner IPs with 403, the monitor will never see a 200. Watch for the `Unexpected status` warnings in the run log.
- The page URL is a best guess; if Anthropic publishes the announcement under a different path, update `TARGET_URL`.

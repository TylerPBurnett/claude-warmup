# claude-warmup

Manipulate Claude Code's 5-hour usage window into resetting when you actually need it.

EDIT: You can apply the same concept and achieve the exact same goal in a "native" way by using a Claude Code Web scheduled task: https://claude.ai/code/scheduled. It works flawlessly! The "Why" section below is still very useful for visualizing why would you do this at all.

## Why

Claude Code gives you a token budget that resets every 5 hours. The window starts when you send your first message, floored to the clock hour.

So if you start at 8:30 AM and burn through your budget by 11, you're locked out until 1 PM. Two dead hours in the middle of your morning.

The fix is dumb and it works: fire a throwaway message before you start working. A GitHub Actions cron sends "hi" to Haiku at 6:15 AM. The window floors to 6 AM, runs until 11. By the time you've hit the limit, it resets right away. Your next message anchors a fresh window through 4 PM.

Example schedule:
```markdown
            6am    7     8     9    10    11    12    1pm    2     3     4     5    6pm
             |     |     |     |     |     |     |     |     |     |     |     |     |

Before:                  [========== window 1 =========]
                          work ~8:30am-11am  ░░ dead ░░
                                                       [========== window 2 =========]
                                                                work ~1pm-6pm
          cron trigger
               │
               ▼
After:       [========== window 1 =========]
              ░░ idle ░░  work ~8:30am-11am
                                           [========== window 2 =========]
                                                   work ~11am-4pm
                                                                         [== win 3 ==]
                                                                         work ~4pm-6pm
```

> As you can see, we are able to squeeze another fresh window starting at 4pm in this case.

## How it works

A scheduled GitHub Actions workflow installs the Claude Code CLI, authenticates with your subscription using a long-lived OAuth token, and sends one Haiku message. That's enough to anchor the window. Cost-wise, one Haiku "hi" is nothing.

## Setup

### 1. Fork this repo

```bash
gh repo fork vdsmon/claude-warmup --clone
```

### 2. Enable Actions on your fork

GitHub disables scheduled workflows on forks by default. After forking:

1. Open your fork on GitHub.
2. Click the **Actions** tab.
3. Click **"I understand my workflows, go ahead and enable them"** if prompted.

Scheduled runs will not fire until you do this.

### 3. Generate an OAuth token

On a machine where you're logged into Claude Code:

```bash
claude setup-token
```

Opens a browser for OAuth. Gives you a token starting with `sk-ant-oat01-...`, good for about a year.

### 4. Store it as a secret

```bash
gh secret set CLAUDE_OAUTH_TOKEN
```

Paste when prompted.

### 5. Set your schedule

Default is weekdays at 8:00 UTC (3:00 AM CDT / 2:00 AM CST).

GitHub Actions requires `on.schedule.cron` to be a literal value in the workflow file, so changing the schedule means editing `.github/workflows/warmup.yml` and updating this line:

```yml
- cron: '0 8 * * 1-5'
```

That's a standard cron expression in UTC. Common conversions:

Need help generating one? Try [crontab.guru](https://crontab.guru/#0_8_*_*_1-5).

| Timezone | 3:00 AM local in UTC | Cron |
| --- | --- | --- |
| US Pacific (UTC-7/PDT) | 10:00 AM | `0 10 * * 1-5` |
| US Pacific (UTC-8/PST) | 11:00 AM | `0 11 * * 1-5` |
| US Central (UTC-5/CDT) | 8:00 AM | `0 8 * * 1-5` |
| US Central (UTC-6/CST) | 9:00 AM | `0 9 * * 1-5` |
| US Eastern (UTC-4/EDT) | 7:00 AM | `0 7 * * 1-5` |
| US Eastern (UTC-5/EST) | 8:00 AM | `0 8 * * 1-5` |
| Central Europe (UTC+2/CEST) | 1:00 AM | `0 1 * * 1-5` |
| Central Europe (UTC+1/CET) | 2:00 AM | `0 2 * * 1-5` |
| Brazil BRT (UTC-3) | 6:00 AM | `0 6 * * 1-5` |
| India IST (UTC+5:30) | 9:30 PM prev day | `30 21 * * 0-4` |
| Japan JST (UTC+9) | 6:00 PM prev day | `0 18 * * 0-4` |
| Australia AEST (UTC+10) | 5:00 PM prev day | `0 17 * * 0-4` |

Pick something 2-4 hours before you usually start working (really depends on your usage pattern).

### 6. Test it

Optional but useful: set your fork as the default GitHub CLI target in this clone.

```bash
gh repo set-default <your-user>/claude-warmup
```

```bash
gh workflow run warmup.yml --repo <your-user>/claude-warmup
```

Check workflow status:

```bash
gh workflow list --repo <your-user>/claude-warmup
gh run list --workflow warmup.yml --repo <your-user>/claude-warmup
gh run view --log --repo <your-user>/claude-warmup
```

Check the logs. You should see a Haiku response or a rate-limit message. Both mean it worked.

### 7. Verify

Next morning, run `/usage` in Claude Code. The session reset time should match your anchored window.

## About the quota window

Some things about how Claude Code's 5-hour window actually works that aren't well documented anywhere (as of March 2026):

- The window is a fixed block. Once set, boundaries don't move no matter how much you use.
- Boundaries floor to clock hours. Message at 6:15 AM means window starts at 6:00 AM.
- Usage is shared between claude.ai, Claude Code, and Claude Desktop. One pool.
- Budget is in tokens, not messages. Extended Thinking and tool use eat through it faster than regular chat.
- There's a separate 7-day weekly cap on top of the 5-hour window. They don't interact.

## Questions

**Does this waste budget?** One Haiku "hi" with no tools, no context. You won't notice it.

**What if I'm already rate-limited?** Still works. The request reaches Anthropic's servers either way, and it still anchors the window.

**Can I run this locally instead?**
Yeah. `claude -p "hi" --model haiku --no-session-persistence` in a cron or macOS launchd does the same thing. GitHub Actions is just easier because your machine doesn't need to be awake at 3 AM.

**Token expiry?** About a year. Set a reminder.

## Troubleshooting

**Scheduled runs never fire**
GitHub disables scheduled workflows on forks by default. Open the **Actions** tab on your fork and enable workflows if prompted. You must do this before the first scheduled run will execute.

**`workflow not found on the default branch`**
The workflow file is missing from your fork's default branch, or GitHub Actions has not been enabled for the fork yet. Push `.github/workflows/warmup.yml` to your fork's default branch, then open the `Actions` tab once and enable workflows if prompted.

**`Must have admin rights to Repository`**
You're probably dispatching against the upstream repo instead of your fork. Run `gh workflow run warmup.yml --repo <your-user>/claude-warmup` or set the default with `gh repo set-default <your-user>/claude-warmup`.

**`CLAUDE_OAUTH_TOKEN secret is not set`**
Add the secret in your fork under `Settings > Secrets and variables > Actions`, then rerun the workflow.

**`Claude token appears invalid or expired`**
Run `claude setup-token` on a machine where you're logged into Claude Code, then update the `CLAUDE_OAUTH_TOKEN` secret in your fork and rerun the workflow. The warmup step checks for this directly.

**Unexpected Claude CLI failure**
Check the workflow logs. The job now prints the full Claude CLI output and only treats explicit rate-limit responses as expected.

## License

MIT

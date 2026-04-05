# mails-gtm-agent-skills

AI SDR (Sales Development Representative) skills for Claude Code and Openclaw. Turn your AI assistant into a cold email outreach agent.

**No infrastructure needed.** No Cloudflare Workers, no database, no OpenRouter API key. Claude Code IS the LLM. The mails CLI handles email. That's it.

## What it does

Install the skill, give it a product URL and a CSV of contacts, and your AI assistant becomes an autonomous SDR:

1. Reads your product page, builds a knowledge base
2. Generates personalized cold emails for each contact
3. Sends via your mails-agent mailbox
4. Monitors inbox for replies
5. Classifies reply intent (interested, not now, unsubscribe...)
6. Generates contextual follow-up replies
7. Tracks the full pipeline: sent → clicked → replied → converted

## vs mails-gtm-agent (the Worker)

| | mails-gtm-agent | mails-gtm-agent-skills |
|--|----------------|----------------------|
| Runtime | Cloudflare Workers | Claude Code / Openclaw |
| LLM | OpenRouter (paid) | Claude Code itself (included) |
| Database | D1 (SQLite) | Local JSON files |
| Automation | Fully autonomous (cron) | Human-in-the-loop |
| Setup | Deploy Worker + secrets | Install skill |
| Best for | Hands-off operation | Oversight + control |

## Prerequisites

- [mails-agent](https://github.com/Digidai/mails) CLI installed and configured
- A claimed mailbox (run `mails claim myagent` first)

## Install

### Claude Code

```bash
claude skill install /path/to/mails-gtm-agent-skills/skills/claude-code/gtm-agent.md
```

Or copy `skills/claude-code/gtm-agent.md` to `~/.claude/skills/`.

### Openclaw

```bash
openclaw skill install /path/to/mails-gtm-agent-skills/skills/openclaw/SKILL.md
```

## Usage

After installing, use natural language:

```
"Create a campaign for mails0.com targeting AI developers"
"Import contacts from leads.csv"
"Generate and preview emails for all pending contacts"
"Send approved emails"
"Check inbox for replies and classify them"
"Draft replies to interested contacts"
```

Or use slash commands:

```
/gtm campaign create --product-url https://mails0.com
/gtm contacts import leads.csv
/gtm preview
/gtm send
/gtm replies
/gtm status
```

## License

MIT

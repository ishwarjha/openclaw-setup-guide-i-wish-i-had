# OpenClaw: Complete Setup & Optimization Guide

> **A step-by-step guidebook to go from first install to a fully configured, production-ready OpenClaw agent setup.**

---

## Table of Contents

- [Why This Guide Exists](#why-this-guide-exists)
- [Step 0: Audit Your Current Setup](#step-0-audit-your-current-setup)
- [Step 1: Install & Initialize OpenClaw](#step-1-install--initialize-openclaw)
- [Step 2: Configure Your Models & Fallbacks](#step-2-configure-your-models--fallbacks)
- [Step 3: Personalize Your Agent](#step-3-personalize-your-agent)
- [Step 4: Set Up Persistent Memory](#step-4-set-up-persistent-memory)
- [Step 5: Activate the Heartbeat](#step-5-activate-the-heartbeat)
- [Step 6: Schedule Tasks with Cron Jobs](#step-6-schedule-tasks-with-cron-jobs)
- [Step 7: Connect Your Communication Channels](#step-7-connect-your-communication-channels)
- [Step 8: Lock Down Security](#step-8-lock-down-security)
- [Step 9: Enable Web Search & External Tools](#step-9-enable-web-search--external-tools)
- [Step 10: Build Your Use Cases](#step-10-build-your-use-cases)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Why This Guide Exists

OpenClaw exploded on GitHub, reaching over 329,000 stars in roughly 60 days and becoming one of the fastest-growing open-source projects ever.

- **Jensen Huang** (NVIDIA founder) called it "the new computer" at GTC 2026, stating at Morgan Stanley's TMT Conference that OpenClaw achieved in 3 weeks what Linux took 30 years to reach in adoption.
- It gives solo founders absurd leverage, letting one person build what previously required 50-person teams through autonomous 24/7 agents.
- Power users demonstrate overnight autonomous builds, persistent memory, dedicated agent accounts on GitHub/email/X, and self-fixing workflows.
- Massive real-world adoption went viral, including huge public install events in Shenzhen where crowds lined up outside Tencent's building for free OpenClaw setups.
- Weixin/WeChat added official OpenClaw support via a ClawBot plugin, making it instantly accessible to hundreds of millions of users.
- It marks the shift from passive AI chatting to deploying persistent, tool-equipped agents that run independently on consumer hardware (even old RTX 3060 GPUs).

**Despite all of that**, setup is hard. Here is what the struggle looks like:

- Setup takes days (15 days for the author), even for someone with technical knowledge.
- Gateway fails to start, crashes, or shows port conflicts, auth token mismatches, or "another instance is listening" errors.
- Debugging token limits, agent loops, corrupted configs, and permission issues burns time and money.
- Installation complexity and poor beginner guidance make the first run feel overwhelming.
- Security risks and fear of exposing vulnerabilities stop people from even attempting the install.
- Random glitches, unexpected behaviour like the agent "glazing", or outdated versions cause quick frustration.

This guide is the outcome of that labour. Follow all 11 steps to set up OpenClaw from scratch, or audit and improve your existing setup.

---

## Step 0: Audit Your Current Setup

Before changing anything, run a series of diagnostic commands to understand what's already configured and what needs attention.

**1.** Check that OpenClaw is installed and see which version you're running:

```bash
openclaw --version
```

**2.** Run the built-in doctor utility to scan for common issues:

```bash
openclaw doctor
```

**3.** Print your full configuration to the screen:

```bash
cat ~/.openclaw/openclaw.json
```

> **Note:** `openclaw config get` requires a specific path argument (e.g., `openclaw config get agents.defaults.model`). To view the entire config at once, read the file directly with `cat`. To query it through the gateway RPC instead, use:

```bash
openclaw gateway call config.get --params '{}'
```

**4.** Check which AI models are set up and whether their authentication is valid:

```bash
openclaw models list
openclaw models status
```

**5.** Verify that the gateway is healthy:

```bash
openclaw health
```

**6.** See what plugins and channels are currently active:

```bash
openclaw plugins list
openclaw channels status --probe
```

**7.** List any scheduled cron jobs:

```bash
openclaw cron list
```

**8.** Check recent logs for errors:

```bash
# macOS:
cat ~/Library/Application\ Support/openclaw/logs/gateway.log | tail -50

# If that path doesn't exist, try:
# cat $OPENCLAW_STATE_DIR/logs/gateway.log | tail -50

# Linux:
journalctl --user -u openclaw-gateway.service -n 50 --no-pager
```

Take note of anything that looks missing or broken. The steps below address each area one by one.

---

## Step 1: Install & Initialize OpenClaw

If you're starting fresh or if Step 0 revealed that OpenClaw isn't installed, this is where you begin.

### Install the CLI

**1.** Run the installer for your platform.

**macOS or Linux:**

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
```

**Windows (PowerShell):**

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

### Initialize Your Environment

**2.** Run the setup command. This creates your `~/.openclaw/openclaw.json` config file and your agent workspace:

```bash
openclaw setup
```

To specify a custom workspace path:

```bash
openclaw setup --workspace <YOUR_WORKSPACE_PATH>
# Example: openclaw setup --workspace ~/.openclaw/workspace
```

Or use a guided walkthrough:

```bash
openclaw setup --wizard
```

### Run the Onboarding Wizard

**3.** The onboarding wizard walks you through authentication, model selection, and daemon configuration.

**Interactive (recommended for most users):**

```bash
openclaw onboard
```

**Non-interactive (for scripted/server setups):**

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

### Start the Gateway

**4.** The gateway is the background process that keeps your agent running. First, check if it's already active:

```bash
openclaw health
```

If you see a healthy response, skip ahead to step 5. If the gateway is not running, start it:

```bash
openclaw gateway
```

> **Common error:** If you see `Gateway failed to start: gateway already running` and `Port 18789 is already in use`, the gateway is already running as a systemd service. Don't try to start a second instance. Use `openclaw health` to confirm, or `openclaw gateway restart` if you need to pick up config changes.

**5.** Confirm everything is working:

```bash
openclaw doctor
openclaw health
```

If both commands return clean output, you're good to move on.

### Set Up a Troubleshooting Project

**6.** Things will break during setup. Save yourself hours by creating a dedicated Claude or ChatGPT project pre-loaded with OpenClaw's docs.

**On Claude (recommended):**

1. Go to [claude.ai](https://claude.ai) > **Projects** > **Create Project**. Name it `OpenClaw Troubleshooting`.
2. Add this as the project description:

   ```text
   You are an OpenClaw expert. When I paste errors or config snippets,
   diagnose the issue using the uploaded documentation and suggest exact
   copy-paste fixes. Always reference the official docs.
   ```

3. Upload the OpenClaw docs to your project's knowledge base. The fastest way is through [context7.com](https://context7.com). Search for `OpenClaw`, download the packaged docs, and drag the file into your project. Alternatively, save key pages from [docs.openclaw.ai](https://docs.openclaw.ai) as PDFs (Installation, Gateway Configuration, CLI Reference, Heartbeat & Cron, FAQ).

**On ChatGPT:**

1. Go to [chatgpt.com](https://chatgpt.com) > **My GPTs** > **Create a GPT**. Add the same system prompt.
2. Under **Knowledge**, upload the same docs from Context7.

**When something breaks:**

1. Copy the **full** error output (5-10 lines of context, not just the last line).
2. Paste it into your project along with what you were doing and the relevant config section:

   ```text
   I ran openclaw gateway restart after adding a Telegram channel and got this:

   <PASTE_FULL_ERROR_OUTPUT>

   My channels config looks like:

   <PASTE_RELEVANT_CONFIG_SECTION>
   ```

3. The AI has the full docs in context and will pinpoint the issue.

---

## Step 2: Configure Your Models & Fallbacks

Your agent needs an AI model to think with. Fallbacks keep it reliable when the primary API goes down.

### Set Your Primary Model

**1.** Choose your main model:

```bash
openclaw models set <MODEL_PROVIDER>/<MODEL_NAME>
# Example: openclaw models set anthropic/claude-sonnet-4-20250514
```

### Add a Fallback

**2.** Add a fallback model. If the primary is unreachable, your agent switches to this automatically:

```bash
openclaw models fallbacks add <FALLBACK_PROVIDER>/<FALLBACK_MODEL>
# Example: openclaw models fallbacks add openai/gpt-4o-mini
```

**3.** Verify it was added (onboarding doesn't set fallbacks automatically):

```bash
openclaw models fallbacks list
```

You should see your fallback listed. If the output says `Fallbacks (0): - none`, the add command may have failed silently. Check that you have a valid API key for the fallback provider.

Want multiple layers of protection? Fallbacks are tried in the order you add them:

```bash
openclaw models fallbacks add anthropic/claude-haiku-4-5-20251001
```

### Or Configure Both in JSON

**4.** If you prefer editing the config file directly, open `~/.openclaw/openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "<MODEL_PROVIDER>/<MODEL_NAME>",
        "fallbacks": ["<FALLBACK_PROVIDER>/<FALLBACK_MODEL>"]
      }
    }
  }
}
```

### Enable Prompt Caching (Optional, Saves Cost)

**5.** If you're using Anthropic models, enable prompt caching to reduce repeated token costs:

```json
{
  "agents": {
    "defaults": {
      "models": {
        "<ANTHROPIC_MODEL_STRING>": {
          "params": { "cacheRetention": "long" }
        }
      }
    }
  }
}
```

### Verify Your Setup

**6.** Confirm models and fallbacks are registered correctly:

```bash
openclaw models list
openclaw models status
openclaw models fallbacks list
```

If any models show authentication errors, refresh your credentials:

```bash
openclaw models auth setup-token --provider <PROVIDER_NAME>
# Example: openclaw models auth setup-token --provider anthropic
```

---

## Step 3: Personalize Your Agent

Right now your agent sounds generic. This step makes it sound like *you*. This is the biggest unlock and what separates a useful agent from a toy.

### Give Your Agent an Identity

**1.** Open your `openclaw.json` and add an identity block:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "identity": {
          "name": "<YOUR_AGENT_NAME>",
          "theme": "<PERSONALITY_THEME>",
          "emoji": "<AGENT_EMOJI>",
          "avatar": "avatars/<AVATAR_FILENAME>"
        }
      }
    ]
  }
}
```

> **Watch the nesting.** The `name`, `theme`, `emoji`, and `avatar` keys must live inside the `identity` object, not directly on the agent. Placing them at the agent level will throw an `Unrecognized keys` error on gateway restart. If you hit this, run `openclaw doctor --fix` to strip the invalid keys, then re-add them inside `identity`.

### Understand the Workspace File System

**2.** OpenClaw organizes your agent's personality across two key directories:

- **Workspace** (`~/.openclaw/workspace/`) — shared context that all agents can read. Bootstrap files, writing samples, templates, and memory live here.
- **Agent directory** (`~/.openclaw/agents/<AGENT_ID>/agent/`) — agent-specific skills and custom prompt overrides. Each agent gets its own.

When your agent boots up, it reads these **bootstrap files** automatically:

| File | Purpose |
| :--- | :--- |
| `SOUL.md` | Core personality: tone, values, communication style, hard rules. |
| `USER.md` | Information about *you*: name, role, preferences, working hours, tools. |
| `IDENTITY.md` | How the agent presents itself: name, persona, introductions. |
| `AGENTS.md` | High-level instructions about what the agent does and how it behaves. |
| `TOOLS.md` | Which tools the agent has access to and how to use them. |
| `HEARTBEAT.md` | The checklist followed on each heartbeat cycle (covered in Step 5). |
| `memory/` | Directory where the agent stores accumulated knowledge over time. |

All of these are plain Markdown files. The agent reads them on startup and injects their contents into its system prompt automatically.

**3.** Navigate to your workspace and confirm these files exist:

```bash
cd ~/.openclaw/workspace
ls -la
```

If you ran `openclaw setup` earlier, most files were created automatically. If they're missing, create them manually as `.md` files.

### Create Your System Prompt: `SOUL.md`

**4.** This is the most important file. Think of `SOUL.md` as the DNA of your agent's personality. Open the file:

```bash
nano ~/.openclaw/workspace/SOUL.md
```

**5.** Replace the default content with your own. Copy this template and fill in your details:

```markdown
# Soul

## Who I Am

I am a personal AI assistant for <YOUR_NAME>. I work as their
<YOUR_ROLE>. I operate as a trusted teammate, not a generic chatbot.

## Communication Style

- I write in a <TONE_STYLE> tone.
- I keep responses <RESPONSE_LENGTH_PREFERENCE>.
- I match the energy of the person I'm talking to.
- I never use emojis unless explicitly asked.
- I never start messages with "Sure!" or "Of course!" or "Great question!"
- I avoid corporate jargon like "synergy", "leverage", "circle back".
- I write in <PERSON_PERSPECTIVE>.

## Hard Rules

- Never fabricate data, statistics, or quotes. If I don't know, I say so.
- Always ask for clarification before making assumptions on high-stakes tasks.
- When writing content, always match the voice and style of the samples in my library.
- Never share personal information about <YOUR_NAME> with anyone.
- Keep all drafts under <MAX_WORD_COUNT> words unless told otherwise.

## Domain Knowledge

- I am deeply familiar with <YOUR_INDUSTRY>.
- I understand <YOUR_TOOLS>.
- I know our key competitors are <COMPETITOR_LIST>.
- Our target audience is <TARGET_AUDIENCE_DESCRIPTION>.

## Formatting Preferences

- Emails: short paragraphs, no more than 3 sentences each.
- Social posts: punchy, hook-first, under 280 characters for X/Twitter.
- Blog posts: use subheadings every 2-3 paragraphs, include a TL;DR at the top.
- Reports: executive summary first, then details.
```

**6.** Save the file. The key is specificity. Vague instructions like "be helpful" do nothing. Specific instructions like "never start a sentence with 'I think'" or "always use the Oxford comma" produce real results.

### Define Yourself: `USER.md`

**7.** This file tells the agent about you. The more context, the better it anticipates your needs.

```bash
nano ~/.openclaw/workspace/USER.md
```

**8.** Fill it in using this structure:

```markdown
# User Profile

## Basic Info

- Name: <YOUR_NAME>
- Location: <YOUR_CITY>, <YOUR_COUNTRY> (<YOUR_TIMEZONE> timezone)
- Role: <YOUR_ROLE> at <YOUR_COMPANY>

## Work Context

- Industry: <YOUR_INDUSTRY>
- Company size: <TEAM_SIZE>
- Primary tools: <TOOL_LIST>
- Working hours: <START_TIME> - <END_TIME> <TIMEZONE>, <WORK_DAYS>

## Preferences

- <COMMUNICATION_PREFERENCE_1>
- <COMMUNICATION_PREFERENCE_2>
- <COMMUNICATION_PREFERENCE_3>

## Current Projects

- <PROJECT_1>
- <PROJECT_2>
- <PROJECT_3>

## People I Work With

- <COLLEAGUE_1_NAME> (<ROLE>) - <ROUTING_RULE>
- <COLLEAGUE_2_NAME> (<ROLE>) - <ROUTING_RULE>
```

### Set Your Agent's Persona: `IDENTITY.md`

**9.** This file shapes how the agent presents itself.

```bash
nano ~/.openclaw/workspace/IDENTITY.md
```

```markdown
# Identity

## Name

<YOUR_AGENT_NAME>

## Persona

I'm <YOUR_AGENT_NAME> - <YOUR_NAME>'s AI <AGENT_ROLE>.
I handle <TASK_LIST> so <YOUR_NAME> can focus on high-leverage work.

## How I Introduce Myself

When someone new messages me: "Hey, I'm <YOUR_AGENT_NAME> -
<YOUR_NAME>'s AI assistant. How can I help?"

## Signature

For emails I draft on <YOUR_NAME>'s behalf, I don't add a signature.
For internal Slack messages, I sign off as "- <YOUR_AGENT_NAME> <AGENT_EMOJI>"
```

### Add Writing Samples (The Secret Weapon)

**10.** This is what separates "sounds like AI" from "sounds like me." Create a library of your actual writing:

```bash
mkdir -p ~/.openclaw/workspace/samples
```

**11.** Populate it with real examples. The more variety, the better.

**Emails you've sent:**

```bash
nano ~/.openclaw/workspace/samples/email-<DESCRIPTIVE_NAME>.md
```

```markdown
# Email: <EMAIL_SUBJECT> (<DATE>)

Subject: <SUBJECT_LINE>

<PASTE_YOUR_ACTUAL_EMAIL_BODY>
```

**Social media posts:**

```bash
nano ~/.openclaw/workspace/samples/social-<PLATFORM>-posts.md
```

```markdown
# <PLATFORM> Posts - <YOUR_NAME>'s Style

## Post 1

<PASTE_YOUR_ACTUAL_POST>

## Post 2

<PASTE_YOUR_ACTUAL_POST>
```

**Blog posts or long-form content:**

```bash
nano ~/.openclaw/workspace/samples/blog-<DESCRIPTIVE_NAME>.md
```

Include the full text of 2-3 blog posts. The agent picks up on paragraph length, vocabulary, analogies, section structure, and overall voice.

**Slack messages or casual communication:**

```bash
nano ~/.openclaw/workspace/samples/slack-tone-examples.md
```

```markdown
# Slack Tone - How <YOUR_NAME> Writes Internally

## Giving feedback

"<PASTE_REAL_FEEDBACK_EXAMPLE>"

## Delegating

"<PASTE_REAL_DELEGATION_EXAMPLE>"

## Responding to a problem

"<PASTE_REAL_PROBLEM_RESPONSE_EXAMPLE>"
```

**12.** Aim for at least 5-10 sample files. Here's a good target mix:

| Type | How Many | Why |
| :--- | :--- | :--- |
| Emails (formal) | 2-3 | Shows your professional tone |
| Emails (casual/internal) | 2-3 | Shows your informal tone |
| Social media posts | 5-10 posts in one file | Teaches brevity and hooks |
| Blog posts / long-form | 2-3 full posts | Teaches structure and depth |
| Slack / chat messages | 10-15 examples in one file | Teaches your everyday voice |
| Client-facing copy | 1-2 | Shows how you talk to customers |

### Create Reusable Templates

**13.** Templates are pre-built formats for tasks you do repeatedly. Create a `templates/` directory:

```bash
mkdir -p ~/.openclaw/workspace/templates
```

**14.** Add your templates as Markdown files. Here are three to start with:

**Weekly team update:**

```bash
nano ~/.openclaw/workspace/templates/weekly-update.md
```

```markdown
# Template: Weekly Team Update

## Format

Post this in #general on Slack every Monday at <TIME>.

## Structure

**Subject line**: Week of <DATE> - <ONE_LINE_SUMMARY>

### What shipped last week

- <FEATURE_OR_TASK> - <IMPACT_SENTENCE>

### What's in progress

- <FEATURE_OR_TASK> - <OWNER> - <EXPECTED_COMPLETION>

### Blockers

- <BLOCKER> - <WHAT_IS_NEEDED>

### Priorities this week

1. <TOP_PRIORITY>
2. <SECOND_PRIORITY>
3. <THIRD_PRIORITY>

## Tone

Keep it crisp. No fluff. Use bullet points. Max 200 words total.
```

**Follow-up email:**

```bash
nano ~/.openclaw/workspace/templates/follow-up-email.md
```

```markdown
# Template: Follow-Up Email After Meeting

## Format

Email, sent within 24 hours of a meeting.

## Structure

Subject: Great chatting - <ONE_LINE_RECAP>

Hi <RECIPIENT_NAME>,

<PERSONAL_REFERENCE_FROM_MEETING>

As discussed, here's what we agreed on:

- <ACTION_ITEM_1> - <OWNER> - <DEADLINE>
- <ACTION_ITEM_2> - <OWNER> - <DEADLINE>

<NEXT_STEPS_SENTENCE>

Best,
<YOUR_NAME>

## Rules

- Never longer than 8 sentences.
- Reference something personal from the meeting.
- Always include clear action items with owners and deadlines.
```

**Content brief:**

```bash
nano ~/.openclaw/workspace/templates/content-brief.md
```

```markdown
# Template: Content Brief

## Metadata

- **Topic**: <WORKING_TITLE>
- **Format**: <CONTENT_FORMAT>
- **Target audience**: <AUDIENCE_DESCRIPTION>
- **Goal**: <DESIRED_OUTCOME>
- **Length**: <WORD_COUNT_OR_DURATION>
- **Deadline**: <DATE>

## Key Points to Cover

1. <POINT_1>
2. <POINT_2>
3. <POINT_3>

## Angle / Hook

<UNIQUE_TAKE_DESCRIPTION>

## Reference Material

- <LINK_OR_FILENAME>

## Anti-patterns (what to avoid)

- <ANTI_PATTERN_1>
- <ANTI_PATTERN_2>

## Tone

Match the tone of samples/<REFERENCE_SAMPLE_FILENAME>.md
```

### Verify Your Directory Structure

**15.** Make sure everything is in place:

```bash
cd ~/.openclaw/workspace
find . -type f -name "*.md" | head -30
```

Expected output:

```
./SOUL.md
./USER.md
./IDENTITY.md
./AGENTS.md
./TOOLS.md
./HEARTBEAT.md
./samples/email-investor-update.md
./samples/social-linkedin-posts.md
./samples/blog-onboarding-teardown.md
./samples/slack-tone-examples.md
./templates/weekly-update.md
./templates/follow-up-email.md
./templates/content-brief.md
```

### Commit Your Personalization to Git

**16.** Lock in your personality layer with a Git commit:

```bash
cd ~/.openclaw/workspace
git add SOUL.md USER.md IDENTITY.md AGENTS.md TOOLS.md HEARTBEAT.md samples/ templates/ memory/
git commit -m "Add personalization: soul, identity, writing samples, and templates"
```

### Why This Matters

Your agent generates text by following the patterns in its context. If you give it nothing personal, it defaults to generic AI voice. If you feed it your past work, your brand guidelines, and explicit style rules, it starts to sound like you.

**The bottom line: the more context you give, the less "AI slop" you get.**

---

## Step 4: Set Up Persistent Memory

Without memory, every conversation starts from zero. With memory, your agent accumulates knowledge over time.

### Enable Session Memory Search

**1.** Add this experimental feature to your config to index past conversations:

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "experimental": { "sessionMemory": true },
        "sources": ["memory", "sessions"]
      }
    }
  }
}
```

### Version-Control Your Memory with Git

**2.** Commit periodically to snapshot your agent's evolving knowledge:

```bash
cd ~/.openclaw/workspace
git add .
git commit -m "Memory snapshot - $(date +%Y-%m-%d)"
```

If you don't have a repo yet, initialize one first:

```bash
cd ~/.openclaw/workspace
git init
git add .
git commit -m "Initial agent memory snapshot"
```

### Create Backups

**3.** Use the built-in backup command regularly:

```bash
openclaw backup create
openclaw backup create --verify
```

Or create a manual archive:

```bash
tar -czvf openclaw-backup.tar.gz ~/.openclaw ~/.openclaw/workspace
```

### Manage Long Conversations

**4.** Over time, conversations get long and the context window fills up:

```bash
# Summarize and condense the current conversation
/compact Focus on decisions and open questions

# Start a fresh session (keeps memory, clears chat)
/new

# Full reset
/reset
```

Think of `/compact` as "save my progress" — it keeps the important parts and trims the noise.

---

## Step 5: Activate the Heartbeat

The heartbeat is what makes OpenClaw feel *alive*. At a regular interval, your agent wakes up, checks a task list, and takes action if something needs attention. If nothing's urgent, it goes back to sleep.

### Enable the Heartbeat

**1.** Add the heartbeat configuration to your `openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "<INTERVAL>",
        "target": "last",
        "lightContext": true,
        "isolatedSession": true,
        "activeHours": { "start": "<START_HOUR>", "end": "<END_HOUR>" }
      }
    }
  }
}
```

Settings breakdown:

| Setting | What It Does |
| :--- | :--- |
| `every` | How often the agent wakes up (e.g., `"30m"`, `"1h"`). Set to `"0m"` to disable. |
| `target: "last"` | Sends alerts to the last person who messaged it. |
| `lightContext: true` | Only loads the HEARTBEAT.md checklist, not the full conversation history. |
| `isolatedSession: true` | Each heartbeat run starts with a clean slate. |
| `activeHours` | Restricts heartbeats to specified hours. No 3 AM pings. |

After saving, restart the gateway:

```bash
openclaw gateway restart
```

### Create Your Heartbeat Checklist

**2.** Create `HEARTBEAT.md` in your workspace. This is the to-do list your agent follows every time it wakes up:

```markdown
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it's daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask <COLLEAGUE_NAME> next time.
```

Keep items small, stable, and safe. If the file is empty or effectively blank, the agent skips the heartbeat and responds with `HEARTBEAT_OK`.

### Per-Agent Overrides (Optional)

**3.** Running multiple agents? Give each one its own heartbeat schedule:

```json
{
  "agents": {
    "list": [
      { "id": "main", "default": true },
      {
        "id": "ops",
        "heartbeat": {
          "every": "<INTERVAL>",
          "target": "<CHANNEL_NAME>",
          "to": "<RECIPIENT_ID>",
          "prompt": "Read HEARTBEAT.md if it exists. Follow it strictly. If nothing needs attention, reply HEARTBEAT_OK."
        }
      }
    ]
  }
}
```

---

## Step 6: Schedule Tasks with Cron Jobs

The heartbeat handles recurring background awareness. Cron jobs are for *specific* scheduled tasks.

### Create a Daily Morning Briefing

**1.** This job runs every day at your preferred morning hour:

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 <HOUR> * * *" \
  --tz "<YOUR_TIMEZONE>" \
  --session isolated \
  --message "Generate today's briefing: weather, calendar, top emails, news summary." \
  --announce
```

> **Note:** Replace `<YOUR_TIMEZONE>` with your IANA timezone string (e.g., `Asia/Kolkata`, `America/New_York`). Omitting `--tz` defaults to the gateway's system timezone, which may not match yours.

### Create a Weekly Project Review

**2.** Runs every Monday at 9 AM using a more powerful model:

```bash
openclaw cron add \
  --name "Weekly review" \
  --cron "0 9 * * 1" \
  --tz "<YOUR_TIMEZONE>" \
  --session isolated \
  --message "Review project progress, summarize blockers, suggest priorities for the week." \
  --model opus \
  --announce
```

### Set a One-Shot Reminder

**3.** A one-time nudge that fires in 2 hours and then deletes itself:

```bash
openclaw cron add \
  --name "<REMINDER_NAME>" \
  --at "<DELAY>" \
  --session main \
  --system-event "<REMINDER_MESSAGE>" \
  --wake now \
  --delete-after-run
```

### Send a Nightly Summary to Telegram

**4.** Runs at 10 PM every night and sends to a Telegram chat:

```bash
openclaw cron add \
  --name "Nightly summary" \
  --cron "0 22 * * *" \
  --tz "<YOUR_TIMEZONE>" \
  --session isolated \
  --message "Summarize everything accomplished today and flag anything unfinished." \
  --announce \
  --channel telegram \
  --to "<TELEGRAM_CHAT_ID>"
```

### Tune for High Volume (Optional)

**5.** If you run many cron jobs, these settings keep things tidy:

```json
{
  "cron": {
    "sessionRetention": "12h",
    "runLog": {
      "maxBytes": "3mb",
      "keepLines": 1500
    }
  }
}
```

### Manage Your Jobs

**6.** View and remove cron jobs anytime:

```bash
openclaw cron list
openclaw cron remove --name "<JOB_NAME>"
```

---

## Step 7: Connect Your Communication Channels

OpenClaw meets you where you already are: WhatsApp, Telegram, Slack, Discord. You're not locked into a single app.

### Configure Multiple Channels

**1.** Add the channels you want to your `openclaw.json`. Include only the ones you actually use:

```json
{
  "channels": {
    "whatsapp": {
      "allowFrom": ["<YOUR_PHONE_NUMBER>"]
    },
    "telegram": {
      "enabled": true,
      "botToken": "<YOUR_TELEGRAM_BOT_TOKEN>",
      "allowFrom": ["<YOUR_TELEGRAM_USER_ID>"]
    },
    "discord": {
      "enabled": true,
      "token": "<YOUR_DISCORD_BOT_TOKEN>",
      "dm": { "enabled": true, "allowFrom": ["<YOUR_DISCORD_USER_ID>"] }
    },
    "slack": {
      "enabled": true,
      "mode": "socket",
      "appToken": "<YOUR_SLACK_APP_TOKEN>",
      "botToken": "<YOUR_SLACK_BOT_TOKEN>"
    }
  }
}
```

After saving, restart the gateway:

```bash
openclaw gateway restart
```

### Add a Channel via CLI

**2.** You can also add channels from the command line:

```bash
openclaw channels add --channel <CHANNEL_NAME> --token <BOT_TOKEN>
```

### Pair with WhatsApp

**3.** WhatsApp uses a pairing code system:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <PAIRING_CODE>
```

> Pairing codes expire after one hour, so approve promptly.

### Verify Your Channels

**4.** Confirm all channels are connected and working:

```bash
openclaw channels status --probe
```

If a channel shows as disconnected, check that your tokens are correct and the gateway is running.

---

## Step 8: Lock Down Security

You're about to give this agent access to your email, calendar, and business tools. Security isn't optional.

### Restrict Who Can Message Your Agent

**1.** Use allowlists and set DM scoping to `per-channel-peer` so conversations stay isolated:

```json
{
  "session": { "dmScope": "per-channel-peer" },
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["<YOUR_PHONE_NUMBER>"]
    },
    "discord": {
      "enabled": true,
      "token": "<YOUR_DISCORD_BOT_TOKEN>",
      "dm": { "enabled": true, "allowFrom": ["<YOUR_DISCORD_USER_ID>"] }
    }
  }
}
```

### Sandbox Agent Execution

**2.** Sandboxing runs agent tasks inside an isolated Docker container:

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "backend": "docker",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "readOnlyRoot": true,
          "network": "none",
          "memory": "1g",
          "cpus": 1,
          "capDrop": ["ALL"]
        }
      }
    }
  },
  "tools": {
    "sandbox": {
      "tools": {
        "allow": ["exec", "process", "read", "write", "edit", "apply_patch"],
        "deny": ["browser", "canvas", "nodes", "cron", "discord", "gateway"]
      }
    }
  }
}
```

### Use Restrictive Tool Profiles

**3.** Create agents with specific permissions. Three common profiles:

**Read-only** (can look but not touch):

```json
{ "tools": { "allow": ["read"], "deny": ["exec", "write", "edit", "apply_patch", "process"] } }
```

**Safe execution** (can run commands but can't write files):

```json
{ "tools": { "allow": ["read", "exec", "process"], "deny": ["write", "edit", "apply_patch", "browser", "gateway"] } }
```

**Communication-only** (can manage sessions but nothing else):

```json
{ "tools": { "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"], "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"] } }
```

### Never Hardcode Secrets

**4.** Always use environment variable references for API keys and tokens:

```json
{
  "gateway": { "auth": { "token": "${OPENCLAW_GATEWAY_TOKEN}" } },
  "env": {
    "ANTHROPIC_API_KEY": "${ANTHROPIC_API_KEY}",
    "BRAVE_API_KEY": "${BRAVE_API_KEY}"
  }
}
```

### Secure Your Webhooks

**5.** If you're using webhooks, lock them down with token-based auth:

```json
{
  "hooks": {
    "enabled": true,
    "token": "${OPENCLAW_HOOKS_TOKEN}",
    "defaultSessionKey": "hook:ingress",
    "allowRequestSessionKey": false,
    "allowedSessionKeyPrefixes": ["hook:"]
  }
}
```

### Verify Your Security Posture

**6.** After making security changes, restart the gateway:

```bash
openclaw gateway restart
```

**7.** Run these checks to confirm nothing is unexpectedly exposed:

```bash
# Linux - confirm no public ports are listening:
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# macOS - equivalent check:
lsof -i -P | grep LISTEN

# Run the full doctor check:
openclaw doctor
```

---

## Step 9: Enable Web Search & External Tools

Your agent is smart, but it doesn't know what happened today. Web search and plugins give it the ability to research and fetch live data.

### Set Up Web Search

**1.** The most common setup uses Brave Search:

```json
{
  "tools": {
    "web": {
      "search": {
        "enabled": true,
        "provider": "brave",
        "apiKey": "${BRAVE_API_KEY}",
        "maxResults": 5
      },
      "fetch": {
        "enabled": true
      }
    }
  }
}
```

**2.** Prefer Perplexity? Route it through OpenRouter:

```json
{
  "tools": {
    "web": {
      "search": {
        "enabled": true,
        "provider": "perplexity",
        "perplexity": {
          "apiKey": "${OPENROUTER_API_KEY}",
          "baseUrl": "https://openrouter.ai/api/v1",
          "model": "perplexity/sonar-pro"
        }
      }
    }
  }
}
```

After adding either search config, restart the gateway:

```bash
openclaw gateway restart
```

### Install Skills from ClawHub

**3.** Skills are pre-built capability packs. Search the registry and install:

```bash
clawhub search "<SKILL_KEYWORD>"
clawhub install <SKILL_NAME>
clawhub update --all
```

**4.** Want skills to auto-refresh when you edit them? Enable the watcher:

```json
{
  "skills": {
    "load": {
      "watch": true,
      "watchDebounceMs": 250
    }
  }
}
```

### Install Plugins

**5.** Plugins extend your agent with new capabilities:

```bash
openclaw plugins install <PLUGIN_PACKAGE_NAME>
# Example: openclaw plugins install @openclaw/voice-call

openclaw plugins list
```

---

## Step 10: Build Your Use Cases

Your infrastructure is solid. Now put it to work. Here are three real-world patterns you can adapt.

### Use Case A: Automated Content Creation Pipeline

A content pipeline that flows: **Ideas > Planning > Script Writing > Filming > Posting > Analytics > (back to Ideas)**.

**1.** Create an `all-ideas.md` file in your workspace. Throughout the week, you (or your agent) dump content ideas here.

**2.** Set up a weekly cron job that reads the ideas file and generates a content plan:

```bash
openclaw cron add \
  --name "Weekly content plan" \
  --cron "0 9 * * 1" \
  --session isolated \
  --message "Read all-ideas.md. Create a weekly content plan based on logged ideas and recent analytics learnings." \
  --announce
```

**3.** Store your past scripts, writing samples, and templates in the agent directory. The agent references these to match your voice and style.

**4.** Add an analytics cron job that fetches performance data and feeds learnings back into the ideas file, creating a self-improving loop.

### Use Case B: Personal CRM & Follow-Up System

A CRM you can *talk to*: "Hey, who do I need to follow up with today?"

**1.** Set up Gmail integration:

```bash
openclaw webhooks gmail setup --account <YOUR_EMAIL>
```

**2.** Configure the webhook to route Gmail events to your agent:

```json
{
  "hooks": {
    "enabled": true,
    "mappings": [{
      "id": "gmail-hook",
      "action": "agent",
      "transform": { "module": "gmail.js", "export": "transformGmail" }
    }],
    "gmail": {
      "account": "<YOUR_EMAIL>",
      "subscription": "gog-gmail-watch-push"
    }
  }
}
```

> The `gmail.js` transform module is created by the `openclaw webhooks gmail setup` wizard. If configuring manually, install the Gmail hook pack first: `openclaw hooks install @openclaw/gmail-hook-pack`.

**3.** Store follow-up templates in your workspace. The agent can draft emails using your templates and send them (or save as drafts for your review).

### Use Case C: Multi-Agent Setup

For complex workflows, run separate agents for different areas of your life:

```json
{
  "agents": {
    "list": [
      {
        "id": "home",
        "default": true,
        "identity": { "name": "Home" },
        "workspace": "~/.openclaw/workspace-home",
        "agentDir": "~/.openclaw/agents/home/agent"
      },
      {
        "id": "work",
        "identity": { "name": "Work" },
        "workspace": "~/.openclaw/workspace-work",
        "agentDir": "~/.openclaw/agents/work/agent"
      }
    ]
  },
  "bindings": [
    { "agentId": "home", "match": { "channel": "whatsapp", "accountId": "<PERSONAL_ACCOUNT_ID>" } },
    { "agentId": "work", "match": { "channel": "whatsapp", "accountId": "<WORK_ACCOUNT_ID>" } }
  ]
}
```

Each agent gets its own workspace, memory, and personality. Messages are automatically routed to the right one based on which account they arrive on.

---

## Quick Reference

Commands you'll use most often, all in one place:

| What You Want To Do | Command |
| :--- | :--- |
| Run a health check | `openclaw doctor` |
| Fix invalid config keys | `openclaw doctor --fix` |
| View your config | `cat ~/.openclaw/openclaw.json` |
| List your models | `openclaw models list` |
| Check model auth | `openclaw models status` |
| Start the gateway | `openclaw gateway` |
| Restart the gateway | `openclaw gateway restart` |
| Stop the gateway | `openclaw gateway stop` |
| Check channel status | `openclaw channels status --probe` |
| List cron jobs | `openclaw cron list` |
| List plugins | `openclaw plugins list` |
| Compact chat history | `/compact` |
| Start a new session | `/new` |
| Create a backup | `openclaw backup create --verify` |
| Update OpenClaw | `openclaw update` |
| Follow live logs | `openclaw logs --follow` |
| Set queue mode | `/queue steer` |

---

## What's Next?

You've built the foundation. From here, the key is iteration. Start with one use case, maybe the morning briefing or the heartbeat, and live with it for a week. See what feels useful and what needs tweaking.

OpenClaw is still early. There will be rough edges. But the people who start experimenting now will be the ones who know how to manage AI agents when this becomes the norm. The magical moments are worth the effort.

Happy building.

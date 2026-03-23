# OpenClaw: Complete Setup & Optimization Guide I Wish I had

> **A step-by-step guidebook to go from first install to a fully configured, production-ready OpenClaw agent setup.**

### Table of Contents

- [Before You Begin](#before-you-begin)
- [Step 0: Audit Your Current Setup](#step-0-audit-your-current-setup)
- [Step 1: Install & Initialize OpenClaw](#step-1-install--initialize-openclaw)
- [Step 2: Configure Your Models & Fallbacks](#step-2-configure-your-models--fallbacks)
- [Step 3: Personalize Your Agent](#step-3-personalize-your-agent)
- [Step 4: Set Up Persistent Memory](#step-4-set-up-persistent-memory)
- [Step 5: Activate the Heartbeat](#step-5-activate-the-heartbeat)
- [Step 6: Schedule Tasks with Cron Jobs](#step-6-schedule-tasks-with-cron-jobs)
- [Step 7: Connect Your Communication Channels](#step-7-connect-your-communication-channels)
- [Step 8: Lock Down Security](#step-8-lock-down-security)
  - [Essential Security (10 minutes)](#essential-security-10-minutes)
  - [Advanced Security (Optional)](#advanced-security-optional)
- [Step 9: Enable Web Search & External Tools](#step-9-enable-web-search--external-tools)
- [Step 10: Build Your Use Cases](#step-10-build-your-use-cases)
- [Quick Reference](#quick-reference)
- [Full Config Reference](#full-config-reference)
- [Common Problems & Fixes](#common-problems--fixes)
- [What's Next?](#whats-next)

---

## Before You Begin

OpenClaw exploded on GitHub, reaching over 329,000 stars in roughly 60 days and becoming one of the fastest-growing open-source projects ever.

Jensen Huang (NVIDIA founder) called it "the new computer" at GTC 2026, stating at Morgan Stanley's TMT Conference that OpenClaw achieved in 3 weeks what Linux took 30 years to reach in adoption. It gives solo founders absurd leverage, letting one person build what previously required 50-person teams through autonomous 24/7 agents. Power users demonstrate overnight autonomous builds, persistent memory, dedicated agent accounts on GitHub/email/X, and self-fixing workflows.

Massive real-world adoption went viral, including huge public install events in Shenzhen where crowds lined up outside Tencent's building for free OpenClaw setups. Weixin/WeChat added official OpenClaw support via a ClawBot plugin, making it instantly accessible to hundreds of millions of users. It marks the shift from passive AI chatting to deploying persistent, tool-equipped agents that run independently on consumer hardware (even old RTX 3060 GPUs).

Despite all of that, setup is hard. Here is what the struggle looks like:

- Setup takes days (15 days for me), even with some technical knowledge.
- Gateway fails to start, crashes, or shows port conflicts, auth token mismatches, or "another instance is listening" errors.
- Debugging token limits, agent loops, corrupted configs, and permission issues burns time and money.
- Installation complexity and poor beginner guidance make the first run feel overwhelming.
- Security risks and fear of exposing vulnerabilities stop people from even attempting the install.
- Random glitches, unexpected behaviour like the agent "glazing", or outdated versions cause quick frustration.

This guide is the outcome of that labour. Follow all 11 steps to set up OpenClaw from scratch, or audit and improve your existing setup.

### What You'll Need

Before diving in, make sure you have these basics ready:

- **A computer** running macOS, Linux, or Windows (with WSL2).
- **An internet connection** — OpenClaw downloads packages and talks to AI providers online.
- **An AI provider account** — You'll need at least one API key from a provider like Anthropic (for Claude) or OpenAI (for GPT). We'll cover how to get these in Step 2.
- **A terminal** — This is the text-based app where you'll type commands. If you've never used one before, here's how to open it:

**On macOS:** Press `Cmd + Space`, type `Terminal`, and hit Enter.

**On Linux:** Press `Ctrl + Alt + T`, or search for "Terminal" in your app launcher.

**On Windows:** Install WSL2 first (search "Turn Windows features on" → enable "Windows Subsystem for Linux"), then open "Ubuntu" from the Start menu. All commands in this guide run inside WSL2, not PowerShell.

> Throughout this guide, whenever you see a gray box with code like `openclaw doctor`, that's a command you type into your terminal and press Enter. You don't need to type the `$` symbol if you see one — that's just the terminal's prompt.

Let's get started.

---

## Step 0: Audit Your Current Setup

Before changing anything, let's see what you're working with. You'll run a series of diagnostic commands to understand what's already configured and what needs attention.

> **First time here?** If you haven't installed OpenClaw yet, skip straight to Step 1. Come back to this step later when you want to audit an existing setup.

**1.** Open your terminal. Check that OpenClaw is installed and see which version you're running:

```bash
openclaw --version
```

You should see something like:

```text
OpenClaw 2026.3.13 (61d171a)
```

If you see "command not found", OpenClaw isn't installed yet — skip to Step 1.

**2.** Run the built-in doctor utility. This scans for common issues with your installation and gateway:

```bash
openclaw doctor
```

**3.** Print your full configuration to the screen. Skim through it — you'll be editing parts of this throughout the guide:

```bash
cat ~/.openclaw/openclaw.json
```

> `openclaw config get` requires a specific path argument (e.g., `openclaw config get agents.defaults.model`). To view the full config at once, read the file directly with `cat`. To query through the gateway RPC, use: `openclaw gateway call config.get --params '{}'`

**4.** Check which AI models are set up and whether their authentication is valid:

```bash
openclaw models list
openclaw models status
```

**5.** Verify that the gateway (the background process that keeps OpenClaw alive) is healthy:

```bash
openclaw health
```

A healthy response looks like:

```text
✓ Gateway reachable on port 18789
✓ Agent "main" ready
```

If you see errors or "connection refused", the gateway isn't running — you'll fix that in Step 1.

**6.** See what plugins and channels are currently active:

```bash
openclaw plugins list
openclaw channels status --probe
```

**7.** List any scheduled cron jobs:

```bash
openclaw cron list
```

**8.** Finally, check recent logs for any errors you might have missed:

```bash
# macOS:
cat ~/Library/Application\ Support/openclaw/logs/gateway.log | tail -50
# If that path doesn't exist, try:
# cat $OPENCLAW_STATE_DIR/logs/gateway.log | tail -50

# Linux:
journalctl --user -u openclaw-gateway.service -n 50 --no-pager
```

Nice work! Take note of anything that looks missing or broken. The steps below will address each area one by one.

---

## Step 1: Install & Initialize OpenClaw

If you're starting fresh — or if Step 0 revealed that OpenClaw isn't installed yet — this is where you begin. You'll install the CLI, initialize your workspace, and start the gateway.

### Check Prerequisites

**1.** OpenClaw requires **Node.js 20 or newer**. Check if you have it:

```bash
node --version
```

If you see `v20.x.x` or higher, you're good. If you see "command not found" or a version below 20, install Node.js first:

**On macOS (using Homebrew):**

```bash
brew install node
```

> Don't have Homebrew? Install it first: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

**On Linux (Ubuntu/Debian):**

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

**2.** Verify Node.js is installed:

```bash
node --version
npm --version
```

Both should return version numbers. If they do, you're ready to install OpenClaw.

### Install the CLI

**3.** The installer downloads OpenClaw and sets it up on your system. Run the one that matches your platform:

On **macOS or Linux**:

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
```

On **Windows (PowerShell)**:

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

### Initialize Your Environment

**4.** Run the setup command. This creates your `~/.openclaw/openclaw.json` config file — the main settings file you'll edit throughout this guide. It also creates your agent workspace, the folder where your agent's personality and memory live:

```bash
openclaw setup
```

> **What is `~/.openclaw/openclaw.json`?** This is your agent's configuration file. It's a text file written in JSON format that controls everything — which AI model to use, which channels to connect, security settings, and more. The `~/` means it's in your home directory, inside a hidden `.openclaw` folder. You'll be editing this file often throughout the guide.

Want to specify a custom workspace path? You can do that too:

```bash
openclaw setup --workspace ~/.openclaw/workspace
```

Or if you prefer a guided walkthrough:

```bash
openclaw setup --wizard
```

### Run the Onboarding Wizard

**5.** The onboarding wizard asks you a series of questions — which AI provider to use, your API key, whether to install the gateway as a background service — and configures everything based on your answers:

```bash
openclaw onboard
```

If you're scripting this (say, for a server), use the non-interactive version instead:

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

**6.** The gateway is the background process that keeps your agent running. Think of it as a little server on your computer that's always listening for messages and running tasks. First, check if it's already running:

```bash
openclaw health
```

If you see a healthy response, the gateway is already active — most likely it was installed as a background service during onboarding (`--install-daemon`). You can skip ahead to step 7.

If the gateway is *not* running, start it:

```bash
openclaw gateway
```

> **Common error:** If you see `Gateway failed to start: gateway already running` and `Port 18789 is already in use`, that means the gateway is already running as a systemd service. This is normal. Don't try to start a second instance — just use `openclaw health` to confirm it's working, or `openclaw gateway restart` if you need to pick up config changes.

**7.** Confirm everything is working:

```bash
openclaw doctor
openclaw health
```

A clean `doctor` output looks like this:

```text
✓ Gateway running on port 18789
✓ Model authenticated: anthropic/claude-sonnet-4-20250514
✓ Workspace: ~/.openclaw/workspace
All checks passed.
```

> If `doctor` finds problems, it will list them with suggestions. For config errors specifically, try `openclaw doctor --fix` — it can auto-repair common issues like invalid keys or malformed JSON. Get in the habit of running this whenever something feels off.

If both commands return clean output, you're good to move on.

### Set Up an AI Project for Troubleshooting

**8.** Things will break during setup — config typos, auth errors, gateway crashes. You'll save yourself hours by creating a dedicated Claude or ChatGPT project pre-loaded with OpenClaw's docs, so you can paste any error and get an instant fix.

**On Claude (recommended):**

**a.** Go to **https://claude.ai** → **Projects** → **Create Project**. Name it `OpenClaw Troubleshooting`.

**b.** Add this as the project description:

```
You are an OpenClaw expert. When I paste errors or config snippets, diagnose 
the issue using the uploaded documentation and suggest exact copy-paste fixes. 
Always reference the official docs.
```

**c.** Upload the OpenClaw docs to your project's knowledge base. The fastest way is through **https://context7.com** — search for `OpenClaw`, download the packaged docs, and drag the file into your project.

> Alternatively, save key pages from **https://docs.openclaw.ai** as PDFs — focus on Installation, Gateway Configuration, CLI Reference, Heartbeat & Cron, and the FAQ.

**On ChatGPT:**

**a.** Go to **https://chatgpt.com** → **My GPTs** → **Create a GPT**. Add the same system prompt as above.

**b.** Under **Knowledge**, upload the same docs from Context7.

**c.** Save. Use it exactly the same way.

**When something breaks:**

**a.** Copy the **full** error output — not just the last line. Include 5–10 lines of context.

**b.** Paste it into your project along with what you were doing and the relevant section of `~/.openclaw/openclaw.json`:

```
I ran openclaw gateway restart after adding a Telegram channel and got this:

[paste full error here]

My channels config looks like:

[paste relevant config section]
```

**c.** The AI has the full docs in context — it'll pinpoint the issue and give you the exact command to fix it.

---

## Step 2: Configure Your Models & Fallbacks

Your agent needs an AI model to think with — this is the "brain" behind the scenes. Models like Claude (by Anthropic) or GPT (by OpenAI) are the intelligence that powers your agent's responses, writing, and decision-making.

But what happens if that model's API goes down? That's where fallbacks come in. You'll set a primary model and one or more backups so your agent stays reliable.

### What Are API Keys?

Before configuring models, you need an **API key** — think of it as a password that lets your agent connect to an AI provider's servers. Each provider issues its own key.

**To get an Anthropic API key (for Claude):**

**a.** Go to **https://console.anthropic.com** and create an account (or sign in).

**b.** Navigate to **API Keys** in the dashboard and click **Create Key**.

**c.** Copy the key — it starts with `sk-ant-`. Save it somewhere safe (you'll need it in a moment). You won't be able to see it again after you close the page.

**To get an OpenAI API key (for GPT):**

**a.** Go to **https://platform.openai.com/api-keys** and sign in.

**b.** Click **Create new secret key**, name it something like "OpenClaw", and copy it. It starts with `sk-`.

> Both providers charge based on usage (tokens processed). Most casual use costs a few dollars per month. You can set spending limits in each provider's dashboard.

### Set Your Primary Model

**1.** Choose your main model. This is the one your agent will use for every task by default:

```bash
openclaw models set anthropic/claude-sonnet-4-20250514
```

> **What does `anthropic/claude-sonnet-4-20250514` mean?** The format is `provider/model-name`. Anthropic is the company, Claude Sonnet 4 is the model, and the date is the version. You don't need to memorize these — just copy-paste from this guide.

### Add a Fallback

**2.** Now add a fallback. If the primary model is unreachable, your agent will automatically switch to this one:

```bash
openclaw models fallbacks add openai/gpt-4o-mini
```

**3.** Verify it was added. This is worth checking — onboarding doesn't set fallbacks automatically, so most fresh installs have none:

```bash
openclaw models fallbacks list
```

You should see `openai/gpt-4o-mini` listed.

> If the output still says `Fallbacks (0): - none`, the add command might not have worked. Check that you have a valid API key for the fallback provider in your `.env` file or environment.

Want multiple layers of protection? Fallbacks are tried in the order you add them:

```bash
openclaw models fallbacks add anthropic/claude-haiku-4-5-20251001
```

Now if your primary goes down, it tries `openai/gpt-4o-mini` first, then `anthropic/claude-haiku-4-5-20251001`.

### Or Configure Both in JSON

**4.** If you prefer editing the config file directly, open it in a text editor:

```bash
nano ~/.openclaw/openclaw.json
```

> **How to use nano (the text editor):** Use arrow keys to move around. Type to add text. Press `Ctrl + O` then `Enter` to save. Press `Ctrl + X` to exit. If you prefer a graphical editor, you can use `code ~/.openclaw/openclaw.json` (VS Code) or open the file from your file manager.

**5.** Find the `"agents"` section (or add it if it doesn't exist) and add the model config:

```jsonc
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-20250514",
        "fallbacks": ["openai/gpt-4o-mini"]
      }
    }
  }
}
```

### Enable Prompt Caching (Optional, Saves Cost)

**6.** If you're using Anthropic models, you can enable prompt caching to reduce repeated token costs:

```jsonc
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-opus-4-6": {
          "params": { "cacheRetention": "long" }
        }
      }
    }
  }
}
```

### Verify Your Setup

**7.** Confirm that your models and fallbacks are registered correctly:

```bash
openclaw models list
openclaw models status
openclaw models fallbacks list
```

Here's what success looks like:

```text
Models:
  Primary: anthropic/claude-sonnet-4-20250514 ✓

Fallbacks (1):
  1. openai/gpt-4o-mini
```

> If `fallbacks list` still shows `none`, go back to step 2 and add one — onboarding doesn't configure fallbacks for you. If any models show authentication errors, run `openclaw models auth setup-token --provider anthropic` to refresh your credentials.

---

## Step 3: Personalize Your Agent

Here's where things get interesting. Right now your agent sounds like... a generic AI. The goal of this step is to make it sound like *you*. This is arguably the biggest unlock — and it's what separates a useful agent from a toy.

### Give Your Agent an Identity

**1.** Open your `openclaw.json` and add an identity block. This sets a name, emoji, and optional personality theme:

```jsonc
{
  "agents": {
    "list": [
      {
        "id": "main",
        "identity": {
          "name": "MyAgent",
          "theme": "space lobster",
          "emoji": "🦞",
          "avatar": "avatars/openclaw.png"
        }
      }
    ]
  }
}
```

> **Watch the nesting.** The `name`, `theme`, `emoji`, and `avatar` keys must live inside the `identity` object — not directly on the agent. Placing them at the agent level (e.g., `"id": "main", "theme": "space lobster"`) will throw an `Unrecognized keys` error on gateway restart. If you hit this, run `openclaw doctor --fix` to strip the invalid keys, then re-add them inside `identity`.

### Understand the Workspace File System

**2.** Before you start creating files, let's understand how OpenClaw organizes your agent's personality. There are two key directories:

- **Workspace** (`~/.openclaw/workspace/`) — shared context that all agents can read. This is where bootstrap files, writing samples, templates, and memory live.
- **Agent directory** (`~/.openclaw/agents/<name>/agent/`) — agent-specific skills and custom prompt overrides. Each agent gets its own. If you have a multi-agent setup, each agent has a separate `agentDir`.

In this guide, we'll put everything in the **workspace** since most setups use a single agent. When your agent boots up, it reads these **bootstrap files** automatically:

| File | Purpose |
|------|---------|
| `SOUL.md` | Your agent's core personality — tone, values, communication style, hard rules. |
| `USER.md` | Information about *you* — your name, role, preferences, working hours, tools you use. |
| `IDENTITY.md` | How the agent presents itself — its name, persona, how it introduces itself. |
| `AGENTS.md` | High-level instructions about what the agent does and how it behaves. |
| `TOOLS.md` | Describes which tools the agent has access to and how to use them. |
| `HEARTBEAT.md` | The checklist the agent follows on each heartbeat cycle (covered in Step 5). |
| `memory/` | A directory where the agent stores accumulated knowledge over time. |

All of these are plain Markdown files. You can edit them with any text editor. The agent reads them on startup and injects their contents into its system prompt automatically.

**3.** Navigate to your workspace and confirm these files exist:

```bash
cd ~/.openclaw/workspace
ls -la
```

If you ran `openclaw setup` earlier, most of these files were created automatically with default content. If they're missing, you can create them manually — they're just `.md` files.

### Create Your System Prompt: `SOUL.md`

**4.** This is the most important file. Think of `SOUL.md` as the DNA of your agent's personality. Everything it writes, every decision it makes, will be filtered through the rules you define here.

Open the file:

```bash
nano ~/.openclaw/workspace/SOUL.md
```

**5.** Replace the default content with your own. Here's the structure to follow — copy this template and fill in your own details:

```markdown
# Soul

## Who I Am
I am a personal AI assistant for [Your Name]. I work as their [role — e.g., "digital marketing manager", "executive assistant", "content strategist"]. I operate as a trusted teammate, not a generic chatbot.

## Communication Style
- I write in a [casual / professional / witty / direct] tone.
- I keep responses [concise / detailed / somewhere in between].
- I match the energy of the person I'm talking to — if they're brief, I'm brief.
- I never use emojis unless explicitly asked.
- I never start messages with "Sure!" or "Of course!" or "Great question!"
- I avoid corporate jargon like "synergy", "leverage", "circle back".
- I write in [first person / third person].

## Hard Rules
- Never fabricate data, statistics, or quotes. If I don't know something, I say so.
- Always ask for clarification before making assumptions on high-stakes tasks.
- When writing content, always match the voice and style of the samples in my library.
- Never share personal information about [Your Name] with anyone.
- Keep all drafts under [X] words unless told otherwise.

## Domain Knowledge
- I am deeply familiar with [your industry — e.g., "B2B SaaS marketing", "real estate", "video production"].
- I understand [specific tools you use — e.g., "Google Analytics, Notion, Figma, HubSpot"].
- I know our key competitors are [list them].
- Our target audience is [describe them].

## Formatting Preferences
- Emails: short paragraphs, no more than 3 sentences each.
- Social posts: punchy, hook-first, under 280 characters for X/Twitter.
- Blog posts: use subheadings every 2–3 paragraphs, include a TL;DR at the top.
- Reports: executive summary first, then details.
```

**6.** Save the file. The key here is specificity. Vague instructions like "be helpful" do nothing. Specific instructions like "never start a sentence with 'I think'" or "always use the Oxford comma" produce real results.

### Define Yourself: `USER.md`

**7.** This file tells the agent about you — the human it's working for. The more context you provide, the better it can anticipate your needs.

```bash
nano ~/.openclaw/workspace/USER.md
```

**8.** Fill it in using this structure:

```markdown
# User Profile

## Basic Info
- Name: Raj Patel
- Location: Mumbai, India (IST timezone)
- Role: Founder & CEO at [Company Name]

## Work Context
- Industry: [your industry]
- Company size: [team size]
- Primary tools: Notion, Slack, Google Workspace, Figma
- Working hours: 9 AM – 7 PM IST, Mon–Sat

## Preferences
- I prefer bullet points over long paragraphs for internal communication.
- I like data-driven arguments — always include numbers when possible.
- I check WhatsApp more than email, so send urgent things there.
- For content, I want a conversational tone — like I'm talking to a friend who happens to be smart.

## Current Projects
- Launching [product name] by Q3 2026
- Hiring a Head of Growth
- Preparing investor deck for Series A

## People I Work With
- Priya (COO) — cc her on anything operational
- Arjun (CTO) — technical decisions go through him
- Meera (Content Lead) — she reviews all published content
```

### Set Your Agent's Persona: `IDENTITY.md`

**9.** This file shapes how the agent presents itself — its name, how it greets people, how it signs off.

```bash
nano ~/.openclaw/workspace/IDENTITY.md
```

```markdown
# Identity

## Name
Atlas

## Persona
I'm Atlas — Raj's AI Chief of Staff. I handle research, drafting, scheduling, and follow-ups so Raj can focus on high-leverage work.

## How I Introduce Myself
When someone new messages me: "Hey, I'm Atlas — Raj's AI assistant. How can I help?"

## Signature
For emails I draft on Raj's behalf, I don't add a signature — Raj signs them himself.
For internal Slack messages, I sign off as "— Atlas 🦞"
```

### Add Writing Samples (The Secret Weapon)

**10.** This is what separates "sounds like AI" from "sounds like me." You're going to create a library of your actual writing so the agent can learn your voice.

Create a `samples/` directory inside your workspace:

```bash
mkdir -p ~/.openclaw/workspace/samples
```

**11.** Now populate it with real examples of your writing. The more variety, the better. Here's what to include and the file format for each:

**Emails you've sent** — save as `.md` files:

```bash
nano ~/.openclaw/workspace/samples/email-investor-update.md
```

```markdown
# Email: Monthly Investor Update (June 2026)

Subject: June Update — 40% MRR Growth, New Enterprise Client

Hi everyone,

Quick update from the trenches.

**Revenue**: MRR hit $82K this month, up 40% from April. The product-led growth motion is finally compounding — most of this is organic.

**Team**: We hired Arjun as CTO. He was previously at Razorpay and brings exactly the infra mindset we need for the next phase.

**Challenge**: Churn in the SMB segment is still stubbornly high at 8%. We're shipping an onboarding overhaul in July that should cut this in half.

**Ask**: If you know anyone strong in growth marketing (ideally B2B SaaS), we're actively hiring. Warm intros welcome.

Talk soon,
Raj
```

**Social media posts** — save as `.md` files:

```bash
nano ~/.openclaw/workspace/samples/social-linkedin-posts.md
```

```markdown
# LinkedIn Posts — Raj's Style

## Post 1
We just hit $80K MRR with zero paid ads.

No growth hacks. No viral loops. Just a product people actually want to tell their friends about.

The unsexy truth about "product-led growth": it takes 18 months of building before the compounding kicks in. But when it does, it's a different game.

## Post 2
Hot take: most startup pitch decks are 30 slides too long.

Investors don't need your TAM slide. They don't need your "why now" slide. They need to understand 3 things in 3 minutes:

1. What's the problem
2. Why you specifically can solve it
3. Is there early evidence it's working

Everything else is noise.

## Post 3
Hired our CTO today. Took 6 months.

Here's what I learned about hiring senior people at an early-stage startup:
- They don't care about your funding. They care about the problem.
- The sell isn't "join us." The sell is "here's why this will be the most interesting work you do this decade."
- Reference checks matter more than interviews. Always call the people they didn't list.
```

**Blog posts or long-form content** — paste the full text of 2–3 posts you've written:

```bash
nano ~/.openclaw/workspace/samples/blog-onboarding-teardown.md
```

> The agent picks up on your paragraph length, vocabulary, analogies, and section structure. Full posts work better than excerpts.

**Slack messages or casual communication:**

```bash
nano ~/.openclaw/workspace/samples/slack-tone-examples.md
```

```markdown
# Slack Tone — How Raj Writes Internally

## Giving feedback
"Love the direction on the landing page. Two quick things: the hero copy feels too safe — let's make it punchier. And can we swap that stock photo for an actual product screenshot? Real > polished."

## Delegating
"@Meera can you take a first pass at the case study for [Client]? I've dropped the raw notes in the shared folder. Aim for ~800 words, conversational tone. No rush — end of week is fine."

## Responding to a problem
"Okay, not ideal but not a disaster. Let's do a quick 15 min sync to figure out the fix. I'd rather move fast on this than let it sit over the weekend."
```

**12.** Aim for at least 5–10 sample files covering different types of writing. Here's a good target mix:

| Type | How Many | Why |
|------|----------|-----|
| Emails (formal) | 2–3 | Shows your professional tone |
| Emails (casual/internal) | 2–3 | Shows your informal tone |
| Social media posts | 5–10 posts in one file | Teaches brevity and hooks |
| Blog posts / long-form | 2–3 full posts | Teaches structure and depth |
| Slack / chat messages | 10–15 examples in one file | Teaches your everyday voice |
| Client-facing copy | 1–2 | Shows how you talk to customers |

### Create Reusable Templates

**13.** Templates are pre-built formats for tasks you do repeatedly. When you ask the agent to "write a weekly update," it'll use your template instead of guessing. Create a `templates/` directory:

```bash
mkdir -p ~/.openclaw/workspace/templates
```

**14.** Add your templates as Markdown files. Here are three common ones to start with:

**Weekly team update template:**

```bash
nano ~/.openclaw/workspace/templates/weekly-update.md
```

```markdown
# Template: Weekly Team Update

## Format
Post this in #general on Slack every Monday at 10 AM.

## Structure
**Subject line**: Week of [DATE] — [one-line summary]

### What shipped last week
- [Feature/task] — [one sentence on impact]
- [Feature/task] — [one sentence on impact]

### What's in progress
- [Feature/task] — [owner] — [expected completion]
- [Feature/task] — [owner] — [expected completion]

### Blockers
- [Blocker] — [what's needed to unblock]

### Priorities this week
1. [Top priority]
2. [Second priority]
3. [Third priority]

## Tone
Keep it crisp. No fluff. Use bullet points. Max 200 words total.
```

**Follow-up email template:**

```bash
nano ~/.openclaw/workspace/templates/follow-up-email.md
```

```markdown
# Template: Follow-Up Email After Meeting

## Format
Email, sent within 24 hours of a meeting.

## Structure
Subject: Great chatting — [one-line recap of what was discussed]

Hi [Name],

[One sentence referencing something specific from the conversation — shows I was paying attention.]

As discussed, here's what we agreed on:
- [Action item 1] — [owner] — [deadline]
- [Action item 2] — [owner] — [deadline]

[One sentence about next steps or when to reconnect.]

Best,
Raj

## Rules
- Never longer than 8 sentences.
- Reference something personal from the meeting (a joke, a shared interest, a problem they mentioned).
- Always include clear action items with owners and deadlines.
```

**Content brief template:**

```bash
nano ~/.openclaw/workspace/templates/content-brief.md
```

```markdown
# Template: Content Brief

## Metadata
- **Topic**: [working title]
- **Format**: [blog post / LinkedIn post / email sequence / video script]
- **Target audience**: [who is this for]
- **Goal**: [what should the reader do/feel/know after consuming this]
- **Length**: [word count or duration]
- **Deadline**: [date]

## Key Points to Cover
1. [Point 1]
2. [Point 2]
3. [Point 3]

## Angle / Hook
[What's the unique take? Why would someone stop scrolling for this?]

## Reference Material
- [Link or file name]
- [Link or file name]

## Anti-patterns (what to avoid)
- [Don't do this]
- [Don't do this]

## Tone
[Reference a specific writing sample, e.g., "Match the tone of samples/blog-onboarding-teardown.md"]
```

### Verify Your Directory Structure

**15.** Let's make sure everything is in place. Run this command to see your full workspace layout:

```bash
cd ~/.openclaw/workspace
find . -type f -name "*.md" | sort | head -30
```

Your output should look something like this:

```text
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

**16.** Now that you've built out your personality layer, lock it in with a Git commit so you never lose this work:

```bash
cd ~/.openclaw/workspace
git add SOUL.md USER.md IDENTITY.md AGENTS.md TOOLS.md HEARTBEAT.md samples/ templates/ memory/
git commit -m "Add personalization: soul, identity, writing samples, and templates"
```

Your personalization is locked in. The more samples and context you add over time, the less your agent sounds like generic AI — and the more it sounds like you.

---

## Step 4: Set Up Persistent Memory

Without memory, every conversation starts from zero. With memory, your agent accumulates knowledge over time — it remembers your preferences, past decisions, and ongoing projects. Think of it like a colleague who keeps notes from every meeting you've had together.

### Enable Session Memory Search

**1.** This experimental feature indexes your past conversations so the agent can search and recall them later. Open your config file:

```bash
nano ~/.openclaw/openclaw.json
```

**2.** Add the `memorySearch` block inside `agents.defaults`. If the `agents` section already exists, just add the `memorySearch` part:

```jsonc
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

**3.** Save and restart the gateway so the change takes effect:

```bash
openclaw gateway restart
```

### Version-Control Your Memory with Git

**4.** **Git** is a tool that tracks changes to files over time — like an "undo history" for your entire folder. If your agent's memory gets corrupted or you want to roll back to an earlier state, Git lets you do that.

First, check if Git is installed:

```bash
git --version
```

If you see "command not found", install it:

**On macOS:**

```bash
brew install git
```

**On Linux:**

```bash
sudo apt-get install -y git
```

**5.** If you followed Step 3, your workspace already has a Git repo. Going forward, commit periodically to snapshot your agent's evolving knowledge:

```bash
cd ~/.openclaw/workspace
git add .
git commit -m "Memory snapshot — $(date +%Y-%m-%d)"
```

> **What does this do?** `git add .` stages all changed files. `git commit -m "..."` saves a snapshot with a description. You can run this daily or weekly — it's like pressing "save" on your agent's brain.

If you skipped Step 3 and don't have a repo yet, initialize one first:

```bash
cd ~/.openclaw/workspace
git init
git add .
git commit -m "Initial agent memory snapshot"
```

### Create Backups

**6.** Backups protect your entire OpenClaw setup — config, workspace, memory, everything. If something goes wrong, you can restore from a backup instead of starting over.

Run the built-in backup command:

```bash
openclaw backup create
```

**7.** Verify the backup was created successfully:

```bash
openclaw backup create --verify
```

> **Where do backups go?** By default, they're saved as `.tar.gz` files in your current directory. You can specify a different location with `openclaw backup create --output ~/Backups`. It's a good idea to run this before making any big config changes.

**8.** You can also create a manual archive (useful if you want to upload it to Google Drive, Dropbox, etc.):

```bash
tar -czvf openclaw-backup.tar.gz ~/.openclaw
```

> **What is a `.tar.gz` file?** It's a compressed archive — like a `.zip` file on Mac/Windows. It bundles your entire `.openclaw` folder into one file you can store or move anywhere.

### Manage Long Conversations

**9.** Every conversation with your agent builds up a "context window" — the agent's short-term memory for that chat. Over time, this fills up and the agent starts forgetting older parts of the conversation. Here's how to manage it:

**To summarize and condense** — keeps the important parts, trims the noise:

```text
/compact Focus on decisions and open questions
```

> You type this *inside a chat with your agent*, not in your terminal. The text after `/compact` tells the agent what to focus on when summarizing.

**To start a fresh session** — clears the current chat but keeps all persistent memory:

```text
/new
```

**To fully reset** — clears everything in the current session:

```text
/reset
```

> **Which one should I use?** Use `/compact` when a conversation gets long but you want to keep the context. Use `/new` when you're switching to a totally different topic. Use `/reset` only if something has gone wrong and you want a clean slate.

---

## Step 5: Activate the Heartbeat

The heartbeat is what makes OpenClaw feel *alive*. At a regular interval (say, every 30 minutes), your agent wakes up, checks a task list, and takes action if something needs attention. If nothing's urgent, it goes back to sleep. Think of it like a personal assistant who pokes their head in every half hour to ask "anything you need?"

### Enable the Heartbeat

**1.** Open your config file:

```bash
nano ~/.openclaw/openclaw.json
```

**2.** Add the heartbeat block inside `agents.defaults`. If `agents.defaults` already has content (like the model config from Step 2), add the `heartbeat` section alongside it — don't replace what's already there:

```jsonc
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "last",
        "lightContext": true,
        "isolatedSession": true,
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    }
  }
}
```

> **Adding to existing config?** If `agents.defaults` already has a `model` section, add a comma after it and paste the `heartbeat` block next to it. If you're unsure, paste your full config into the troubleshooting project from Step 1 and ask it to merge.

Let's break down what each setting does:

| Setting | What It Does |
|---------|-------------|
| `every: "30m"` | The agent wakes up every 30 minutes. Set to `"0m"` to disable. |
| `target: "last"` | Sends alerts to the last person who messaged it. |
| `lightContext: true` | Only loads the HEARTBEAT.md checklist — not the full conversation history. |
| `isolatedSession: true` | Each heartbeat run starts with a clean slate. |
| `activeHours` | Restricts heartbeats to daytime hours. No 3 AM pings. |

After saving, restart the gateway so it picks up the new heartbeat config:

```bash
openclaw gateway restart
```

### Create Your Heartbeat Checklist

**3.** The heartbeat file is a simple Markdown checklist that your agent reads every time it wakes up. Create it:

```bash
nano ~/.openclaw/workspace/HEARTBEAT.md
```

**4.** Type (or paste) your checklist. Here's a starter template:

```markdown
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it's daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down _what is missing_ and ask Peter next time.
```

**5.** Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

> Customize this to your workflow. For example, if you run a store, your checklist might say "check for new orders" and "flag any customer complaints older than 24 hours." Keep items small and specific — the agent takes them literally.

If the file is empty or effectively blank, the agent will skip the heartbeat and respond with `HEARTBEAT_OK`.

### Per-Agent Overrides (Optional)

**6.** Running multiple agents? You can give each one its own heartbeat schedule. For example, keep the "main" agent heartbeat-free while the "ops" agent checks in every hour:

```jsonc
{
  "agents": {
    "list": [
      { "id": "main", "default": true },
      {
        "id": "ops",
        "heartbeat": {
          "every": "1h",
          "target": "whatsapp",
          "to": "+15551234567",
          "prompt": "Read HEARTBEAT.md if it exists. Follow it strictly. If nothing needs attention, reply HEARTBEAT_OK."
        }
      }
    ]
  }
}
```

---

## Step 6: Schedule Tasks with Cron Jobs

The heartbeat handles recurring background awareness. Cron jobs are for *specific* scheduled tasks — a morning briefing, a weekly review, a one-time reminder. Think of them as calendar reminders for your agent: "At 7 AM every day, do this."

### Understanding Cron Schedules

Before creating jobs, here's a quick primer on the scheduling format. A cron schedule like `"0 7 * * *"` has five parts:

```
┌───── minute (0-59)
│ ┌──── hour (0-23, in 24-hour time)
│ │ ┌─── day of month (1-31)
│ │ │ ┌── month (1-12)
│ │ │ │ ┌─ day of week (0-6, where 0 = Sunday)
│ │ │ │ │
0 7 * * *   ← "At minute 0, hour 7, every day" = 7:00 AM daily
0 9 * * 1   ← "At 9:00 AM, only on day 1 of the week" = Monday 9 AM
0 22 * * *  ← "At 10:00 PM, every day"
```

The `*` means "every" — so `* * *` in the last three positions means "every day, every month, every day of the week." You don't need to memorize this — just copy the examples below and change the hour.

### Create a Daily Morning Briefing

**1.** This job runs every day at 7 AM in your timezone, generates a summary of your day ahead, and announces the result:

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Generate today's briefing: weather, calendar, top emails, news summary." \
  --announce
```

Here's what each flag does (applies to all cron examples below):

| Flag | What It Does |
|------|-------------|
| `--name` | A label for this job — shows up when you list your cron jobs later. |
| `--cron` | The schedule in cron format (see the primer above). |
| `--tz` | Your timezone. Find yours at **https://en.wikipedia.org/wiki/List_of_tz_database_time_zones**. |
| `--session isolated` | Runs in a fresh session — no leftover context from your last chat. |
| `--message` | The instruction your agent will follow when the job fires. |
| `--announce` | Sends the result to your connected channel (WhatsApp, Telegram, etc.). |

> Replace `Asia/Kolkata` with your timezone in all cron examples. Omitting `--tz` defaults to the gateway's system timezone, which may not match yours.

### Create a Weekly Project Review

**2.** This one runs every Monday at 9 AM using a more powerful model:

```bash
openclaw cron add \
  --name "Weekly review" \
  --cron "0 9 * * 1" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Review project progress, summarize blockers, suggest priorities for the week." \
  --model opus \
  --announce
```

### Set a One-Shot Reminder

**3.** Need a one-time nudge? Unlike cron jobs that repeat, this reminder fires once and then deletes itself. The `--at "2h"` flag means "2 hours from now":

```bash
openclaw cron add \
  --name "Call back" \
  --at "2h" \
  --session main \
  --system-event "Reminder: call back the client about the proposal." \
  --wake now \
  --delete-after-run
```

> You can use `--at "30m"` for 30 minutes, `--at "1h"` for 1 hour, or an exact time like `--at "2026-07-01T18:00:00Z"`. The `--delete-after-run` flag ensures the job cleans itself up after firing.

### Send a Nightly Summary to Telegram

**4.** This job runs at 10 PM every night, summarizes your day, and sends it to a specific Telegram chat:

```bash
openclaw cron add \
  --name "Nightly summary" \
  --cron "0 22 * * *" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Summarize everything accomplished today and flag anything unfinished." \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

> **Finding your chat ID:** Send any message to your bot, then visit `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in your browser. Look for `"chat":{"id": 123456789}`. Group IDs start with `-100`.

### Tune for High Volume (Optional)

**5.** If you end up running lots of cron jobs (10+), old session data can pile up. These settings tell OpenClaw to clean up after itself. Add this to your `openclaw.json`:

```jsonc
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

> `sessionRetention: "12h"` means cron session data is deleted after 12 hours. `runLog` caps the log file size so it doesn't grow forever. You only need this if you're running many frequent jobs — skip it for a handful of daily/weekly jobs.

### Manage Your Jobs

**6.** See all your scheduled cron jobs:

```bash
openclaw cron list
```

**7.** Remove a job you no longer need (use the exact name you gave it):

```bash
openclaw cron remove --name "Morning brief"
```

**8.** If a job isn't firing, check that the gateway is running (`openclaw health`) and that your timezone is correct. You can also check the gateway logs for cron-related errors:

```bash
openclaw logs --follow
```

Press `Ctrl + C` to stop following the logs.

---

## Step 7: Connect Your Communication Channels

One of OpenClaw's standout features is that it meets you where you already are — WhatsApp, Telegram, Slack, Discord. You're not locked into a single app. Let's wire up the channels you use.

### Get Your Bot Tokens

Before configuring channels, you'll need tokens (passwords) for each service. Here's how to get them:

**Telegram:**

**a.** Open Telegram and search for `@BotFather` — this is Telegram's official tool for creating bots.

**b.** Send `/newbot` and follow the prompts. Give your bot a name (e.g., "My OpenClaw Agent") and a username (e.g., `myopenclaw_bot`).

**c.** BotFather will reply with a token that looks like `7123456789:AAH...`. Copy it — this is your `botToken`.

**d.** To find your Telegram user ID (needed for `allowFrom`), search for `@userinfobot` on Telegram and send it any message. It'll reply with your numeric user ID.

**Discord:**

**a.** Go to **https://discord.com/developers/applications** and click **New Application**. Name it anything.

**b.** In the left sidebar, click **Bot** → **Add Bot** → **Yes, do it!**

**c.** Click **Reset Token** and copy the token — this is your Discord bot `token`.

**d.** Under **Privileged Gateway Intents**, enable **Message Content Intent** (required for the bot to read messages).

**e.** To invite the bot to your server, go to **OAuth2** → **URL Generator**, check `bot` under scopes, check `Send Messages` and `Read Message History` under permissions, then open the generated URL in your browser.

**f.** Your Discord user ID (for `allowFrom`): Enable Developer Mode in Discord settings (Settings → Advanced → Developer Mode), then right-click your username and click **Copy User ID**.

**WhatsApp:** No token needed — WhatsApp uses a pairing code system (covered in step 3 below).

**Slack:** You'll need to create a Slack app at **https://api.slack.com/apps**. Click **Create New App** → **From scratch**, enable **Socket Mode**, and copy the App Token (`xapp-...`) and Bot Token (`xoxb-...`).

### Configure Multiple Channels

**1.** Open your config and add the channels you want. Replace the placeholder values with the real tokens you just collected:

```jsonc
{
  "channels": {
    "whatsapp": {
      "allowFrom": ["+91XXXXXXXXXX"]
    },
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TELEGRAM_BOT_TOKEN",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    },
    "discord": {
      "enabled": true,
      "token": "YOUR_DISCORD_BOT_TOKEN",
      "dm": { "enabled": true, "allowFrom": ["YOUR_DISCORD_USER_ID"] }
    },
    "slack": {
      "enabled": true,
      "mode": "socket",
      "appToken": "xapp-...",
      "botToken": "xoxb-..."
    }
  }
}
```

You don't need all four — just include the ones you actually use.

**2.** After saving, restart the gateway so it picks up the new channels:

```bash
openclaw gateway restart
```

### Add a Channel via CLI

**3.** You can also add channels from the command line:

```bash
openclaw channels add --channel telegram --token <bot-token>
```

### Pair with WhatsApp

WhatsApp doesn't use a bot token like Telegram. Instead, it links your real WhatsApp account to OpenClaw through a pairing code — similar to linking WhatsApp Web.

**4.** Start the WhatsApp login process:

```bash
openclaw channels login --channel whatsapp
```

This will display a QR code in your terminal, or give you a pairing code.

**5.** On your phone, open **WhatsApp** → tap the three dots (top right) → **Linked Devices** → **Link a Device**. Either scan the QR code from your terminal, or enter the pairing code manually.

> If you have multiple WhatsApp accounts (personal + business), specify which one: `openclaw channels login --channel whatsapp --account work`

**6.** Once linked, the gateway needs to be running to receive messages. Confirm it's active:

```bash
openclaw gateway restart
```

**7.** Check that the pairing was successful:

```bash
openclaw pairing list whatsapp
```

You should see your device listed. If a pairing request is pending, approve it:

```bash
openclaw pairing approve whatsapp <CODE>
```

> Pairing codes expire after one hour. If yours expired, re-run `openclaw channels login --channel whatsapp` to get a new one.

**8.** Send a test message to yourself on WhatsApp. If your agent replies, the connection is working.

### Verify Your Channels

**9.** Confirm all your channels are connected and working:

```bash
openclaw channels status --probe
```

Here's what a healthy output looks like:

```text
whatsapp: connected (personal)
telegram: connected
discord: connected
```

If a channel shows as `disconnected`, check that your tokens are correct and the gateway is running. For WhatsApp specifically, try re-linking with `openclaw channels login --channel whatsapp`.

---

## Step 8: Lock Down Security

You're about to give this agent access to your email, calendar, and business tools. That's powerful — and it means security isn't optional. This step is split into two parts: **essentials** (required, takes 10 minutes) and **advanced** (optional, for shared environments or extra hardening).

### Essential Security (10 minutes)

These steps ensure only you can talk to your agent and your API keys aren't sitting in plain text. Everyone should do these.

#### Restrict Who Can Message Your Agent

**1.** By default, anyone with your bot's token could talk to your agent. An **allowlist** restricts it to specific people only. Open your config:

```bash
nano ~/.openclaw/openclaw.json
```

**2.** Add the security settings. Replace the placeholders with your real phone number and user IDs (you collected these in Step 7):

```jsonc
{
  "session": { "dmScope": "per-channel-peer" },
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+91XXXXXXXXXX"]
    },
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TELEGRAM_BOT_TOKEN",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    },
    "discord": {
      "enabled": true,
      "token": "YOUR_DISCORD_BOT_TOKEN",
      "dm": { "enabled": true, "allowFrom": ["YOUR_DISCORD_USER_ID"] }
    }
  }
}
```

> **Want your team to use it too?** Add their IDs to the `allowFrom` arrays: `["+91XXXXXXXXXX", "+91YYYYYYYYYY"]`. Each person gets their own isolated conversation.

#### Store Your Secrets Safely

**3.** Your config file will reference API keys and tokens. Never paste the actual keys directly into `openclaw.json` — they'll end up in Git history and backups. Instead, store them in a `.env` file:

```bash
nano ~/.openclaw/.env
```

**4.** Add your secrets, one per line. No quotes, no spaces around `=`:

```text
ANTHROPIC_API_KEY=sk-ant-your-key-here
OPENAI_API_KEY=sk-your-openai-key-here
BRAVE_API_KEY=BSA-your-brave-key-here
OPENCLAW_GATEWAY_TOKEN=your-gateway-token-here
```

> **Generate a secure gateway token** if you don't have one yet: `openssl rand -hex 32`

**5.** Lock down the file so only your user can read it:

```bash
chmod 600 ~/.openclaw/.env
```

**6.** Prevent it from ever being committed to Git:

```bash
echo ".env" >> ~/.openclaw/.gitignore
```

**7.** Now reference these variables in your `openclaw.json` using the `${VAR_NAME}` syntax:

```jsonc
{
  "gateway": { "auth": { "token": "${OPENCLAW_GATEWAY_TOKEN}" } },
  "env": {
    "ANTHROPIC_API_KEY": "${ANTHROPIC_API_KEY}",
    "BRAVE_API_KEY": "${BRAVE_API_KEY}"
  }
}
```

> If a variable is missing or empty, OpenClaw will throw an error on startup — it won't silently fail.

**8.** Run the secrets audit to make sure no plaintext keys are sitting in your config:

```bash
openclaw secrets audit --check
```

If it finds any, fix them interactively:

```bash
openclaw secrets configure
openclaw secrets audit --check
```

#### Restart and Verify

**9.** Restart the gateway so the security settings take effect:

```bash
openclaw gateway restart
```

**10.** Run a quick check:

```bash
openclaw doctor
```

You should see all checks passing. Your essential security is now in place — only you can message your agent, and your secrets are stored safely off-disk.

### Advanced Security (Optional)

Everything below is optional. Your agent is already secure if you completed the essentials above. These steps add extra hardening for shared environments, internet-facing setups, or sensitive business tools.

#### Sandbox Agent Execution

Sandboxing runs agent tasks inside an isolated Docker container — a locked-down mini-computer where your agent can work without touching your real files or network.

**11.** Check if Docker is installed:

```bash
docker --version
```

If you see "command not found", install it:

**On Linux (Ubuntu/Debian):**

```bash
sudo apt-get update
sudo apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sh
```

**On macOS:**

```bash
brew install --cask docker
```

Then open Docker Desktop from your Applications folder. You'll see the whale icon in your menu bar when it's ready.

**12.** Verify Docker is running:

```bash
docker --version
docker info
```

> If `docker info` shows `Cannot connect to the Docker daemon`, make sure Docker Desktop is running (macOS) or the service is started (Linux: `sudo systemctl start docker`).

**13.** Build the sandbox images. Navigate to your OpenClaw source directory:

```bash
scripts/sandbox-setup.sh
```

For extra build tools (Node, Go, Rust), also build the common image:

```bash
scripts/sandbox-common-setup.sh
```

**14.** Verify the images were created:

```bash
docker images | grep openclaw-sandbox
```

> If you installed OpenClaw via npm and don't have the `scripts/` directory, you can override the sandbox image to use any Debian-based Docker image in the config below.

**15.** Add the sandbox config to your `openclaw.json`:

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "backend": "docker",
        "scope": "agent",
        "workspaceAccess": "none",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "readOnlyRoot": true,
          "network": "none",
          "user": "1000:1000",
          "capDrop": ["ALL"],
          "memory": "1g",
          "cpus": 1,
          "pidsLimit": 256,
          "tmpfs": ["/tmp", "/var/tmp", "/run"],
          "env": { "LANG": "C.UTF-8" }
        },
        "prune": {
          "idleHours": 24,
          "maxAgeDays": 7
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

| Setting | What It Does |
|---------|-------------|
| `mode: "non-main"` | Sandboxes everything except the main session. Use `"all"` to sandbox everything. |
| `network: "none"` | Blocks all network access from inside the container. |
| `readOnlyRoot: true` | Container filesystem is read-only — agent can only write to `/tmp` and `/workspace`. |
| `capDrop: ["ALL"]` | Drops all Linux capabilities — minimal privileges. |
| `memory: "1g"` | Limits the container to 1 GB of RAM. |
| `prune.idleHours: 24` | Automatically removes idle containers after 24 hours. |

**16.** To mount a host directory into the sandbox (e.g., a project folder), add `binds`:

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "binds": ["/home/user/projects:/projects:ro"]
        }
      }
    }
  }
}
```

> `:ro` = read-only inside the container. Use `:rw` if the agent needs to write to it.

**17.** For multi-agent setups, you can sandbox agents differently:

```jsonc
{
  "agents": {
    "list": [
      {
        "id": "personal",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "family",
        "sandbox": {
          "mode": "all",
          "scope": "agent",
          "docker": {
            "setupCommand": "apt-get update && apt-get install -y git curl"
          }
        },
        "tools": {
          "allow": ["read"],
          "deny": ["exec", "write", "edit", "apply_patch"]
        }
      }
    ]
  }
}
```

**18.** Restart and verify the sandbox is working:

```bash
openclaw gateway restart
openclaw doctor
```

#### Use Restrictive Tool Profiles

**19.** You can create agents with very specific permissions. Three common profiles:

**Read-only** (can look but not touch):
```jsonc
{ "tools": { "allow": ["read"], "deny": ["exec", "write", "edit", "apply_patch", "process"] } }
```

**Safe execution** (can run commands but can't write files):
```jsonc
{ "tools": { "allow": ["read", "exec", "process"], "deny": ["write", "edit", "apply_patch", "browser", "gateway"] } }
```

**Communication-only** (can manage sessions but nothing else):
```jsonc
{ "tools": { "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"], "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"] } }
```

#### Use a Password Manager for Secrets

**20.** If you use 1Password, you can have OpenClaw pull secrets directly from your vault at runtime — no keys ever touch disk:

```jsonc
{
  "secrets": {
    "providers": {
      "onepassword_anthropic": {
        "source": "exec",
        "command": "/opt/homebrew/bin/op",
        "allowSymlinkCommand": true,
        "trustedDirs": ["/opt/homebrew"],
        "args": ["read", "op://Personal/Anthropic API Key/password"],
        "passEnv": ["HOME"],
        "jsonOnly": false
      }
    }
  }
}
```

> Requires the 1Password CLI (`op`) to be installed and authenticated. This replaces the `.env` approach from the essentials section.

#### Secure Your Webhooks

**21.** Webhooks let external services (Gmail, Slack) send data into your agent. Lock them down with a token:

```bash
echo "OPENCLAW_HOOKS_TOKEN=$(openssl rand -hex 32)" >> ~/.openclaw/.env
```

**22.** Add the webhook security config:

```jsonc
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

> The `token` rejects any request without it. `allowRequestSessionKey: false` prevents outsiders from targeting your personal sessions.

#### Verify Your Security Posture

**23.** Restart the gateway:

```bash
openclaw gateway restart
```

**24.** Check that nothing is unexpectedly exposed:

```bash
# Linux:
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# macOS:
lsof -i -P | grep LISTEN

# Full doctor check:
openclaw doctor
```

> The port check should show only `127.0.0.1:18789` (localhost). If you see `0.0.0.0:18789`, your gateway is publicly accessible — set `gateway.bind` to `"loopback"` in your config.

Your agent is now locked down. Only you (and anyone you explicitly allow) can talk to it, secrets are off-disk, and tools are sandboxed.

---

## Step 9: Enable Web Search & External Tools

Your agent is smart, but it doesn't know what happened today. Web search and tool plugins give it the ability to research, fetch live data, and interact with external services.

### Set Up Web Search

Your agent can search the web to answer questions about current events, look up facts, or research topics. The most common provider is Brave Search.

**1.** First, get a Brave Search API key:

**a.** Go to **https://brave.com/search/api/** and sign up for a free account.

**b.** Once logged in, navigate to the API dashboard and create a new API key.

**c.** Copy the key — it starts with `BSA`. Add it to your `.env` file (from Step 8):

```bash
echo 'BRAVE_API_KEY=BSA-your-key-here' >> ~/.openclaw/.env
```

**2.** Now add the web search config to your `openclaw.json`:

```bash
nano ~/.openclaw/openclaw.json
```

Add this inside your config (alongside your existing `agents`, `channels`, etc.):

```jsonc
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

> The `"fetch": { "enabled": true }` part lets your agent read full web pages — not just search snippets. This is useful when it needs to dig into an article or documentation page.

**3.** Prefer Perplexity? It gives AI-synthesized answers instead of raw search results. You can route it through OpenRouter (a service that lets you access many AI providers with one API key):

**a.** Sign up at **https://openrouter.ai** and get an API key.

**b.** Add it to your `.env` file:

```bash
echo 'OPENROUTER_API_KEY=sk-or-your-key-here' >> ~/.openclaw/.env
```

**c.** Use this config instead of the Brave one above (replace the `tools.web` block):

```jsonc
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

Skills are pre-built capability packs — like "apps" for your agent. They teach it how to do specific tasks (e.g., managing databases, writing in a particular style, running specific workflows). ClawHub is the registry where community-built skills are published.

**4.** Search for skills relevant to your use case:

```bash
clawhub search "postgres backups"
```

This returns a list of matching skill packs. Find one you want and install it:

```bash
clawhub install <skill-name>
```

> Replace `<skill-name>` with the actual name from the search results — e.g., `clawhub install pg-backup-manager`.

**5.** Keep your skills up to date:

```bash
clawhub update --all
```

**6.** Want your skills to auto-refresh when you edit their files? Add this to your config:

```jsonc
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

Plugins are more powerful than skills — they add entirely new features to your agent, like voice calls, new messaging channels, or integration with external services.

**7.** Install a plugin from the registry. For example, the voice-call plugin:

```bash
openclaw plugins install @openclaw/voice-call
```

**8.** See all your installed plugins:

```bash
openclaw plugins list
```

**9.** If a plugin isn't working, run the plugin doctor:

```bash
openclaw plugins doctor
```

Your agent can now search the web, read full pages, and use community-built skills. It's no longer limited to what it already knows.

---

## Step 10: Build Your Use Cases

Your infrastructure is solid. Now it's time to put it to work. Below are three real-world patterns you can adapt to your own workflow. You don't need to build all three — pick the one closest to what you need and start there.

### Use Case A: Automated Content Creation Pipeline

One of the most powerful setups is an automated content pipeline that flows like this: **Ideas → Planning → Script Writing → Filming → Posting → Analytics → (back to Ideas)**.

Here's how to replicate it:

**1.** Create a file where you and your agent collect content ideas throughout the week:

```bash
nano ~/.openclaw/workspace/all-ideas.md
```

Add a simple structure:

```markdown
# Content Ideas

## Raw Ideas (dump anything here)
- Tutorial on how to use OpenClaw for email automation
- Hot take: why most pitch decks are too long
- Behind-the-scenes of my content workflow

## Validated (worth making)
- (move ideas here after review)
```

**2.** Set up a weekly cron job that reads the ideas file and generates a content plan:

```bash
openclaw cron add \
  --name "Weekly content plan" \
  --cron "0 9 * * 1" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Read all-ideas.md. Create a weekly content plan based on logged ideas. Prioritize by relevance and timeliness. Save the plan to content-plan.md." \
  --announce
```

**3.** Store your past scripts, writing samples, and templates in the workspace (you already did this in Step 3). When the agent writes new scripts, it references these to match your voice and style.

**4.** Add an analytics cron job that fetches performance data and feeds learnings back into the ideas file — creating a self-improving loop:

```bash
openclaw cron add \
  --name "Weekly analytics review" \
  --cron "0 18 * * 5" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Review this week's content performance. What topics got the most engagement? Add 3 new content ideas to all-ideas.md based on what's working." \
  --announce
```

### Use Case B: Personal CRM & Follow-Up System

Imagine a CRM you can *talk to* — "Hey, who do I need to follow up with today?" — and it checks your Google Sheet, Gmail, and calendar.

**1.** Set up Gmail integration. This wizard walks you through connecting your Gmail account:

```bash
openclaw webhooks gmail setup --account your@gmail.com
```

> The wizard will ask you to authenticate with Google, set up a Pub/Sub topic, and configure the webhook endpoint. Follow each prompt — it handles the technical details for you.

**2.** After the wizard completes, it creates a `gmail.js` transform module automatically. Verify it's wired up by checking your config — it should include a hooks section:

```bash
cat ~/.openclaw/openclaw.json | grep -A 10 "hooks"
```

If you're configuring manually without the wizard, you'll need to install the Gmail hook pack first:

```bash
openclaw hooks install @openclaw/gmail-hook-pack
```

Then add this to your config:

```jsonc
{
  "hooks": {
    "enabled": true,
    "mappings": [{
      "id": "gmail-hook",
      "action": "agent",
      "transform": { "module": "gmail.js", "export": "transformGmail" }
    }],
    "gmail": {
      "account": "your@gmail.com",
      "subscription": "gog-gmail-watch-push"
    }
  }
}
```

**3.** Create a follow-up tracker in your workspace:

```bash
nano ~/.openclaw/workspace/follow-ups.md
```

```markdown
# Follow-Up Tracker

## Pending Follow-Ups
| Name | Last Contact | Channel | Due By | Notes |
|------|-------------|---------|--------|-------|
| Priya (Acme Corp) | 2026-03-18 | Email | 2026-03-25 | Waiting on proposal feedback |
| David (Investor) | 2026-03-15 | WhatsApp | 2026-03-22 | Schedule next call |
```

**4.** Store your follow-up email templates in the templates folder (from Step 3). The agent can draft emails using your templates and send them — or save as drafts for your review.

**5.** Set up a daily cron job that checks for overdue follow-ups:

```bash
openclaw cron add \
  --name "Follow-up check" \
  --cron "0 9 * * *" \
  --tz "Asia/Kolkata" \
  --session isolated \
  --message "Read follow-ups.md. List anyone whose follow-up is overdue or due today. Draft follow-up messages using the templates in templates/follow-up-email.md." \
  --announce
```

### Use Case C: Multi-Agent Setup

As your needs grow, you might want separate agents for different areas of your life — one for personal tasks, another for work. Each agent gets its own workspace (files, memory, personality) and messages are automatically routed to the right one.

**1.** Create workspaces for each agent:

```bash
mkdir -p ~/.openclaw/workspace-home
mkdir -p ~/.openclaw/workspace-work
mkdir -p ~/.openclaw/agents/home/agent
mkdir -p ~/.openclaw/agents/work/agent
```

**2.** Add the multi-agent config to your `openclaw.json`. This sets up two agents and routes WhatsApp messages to the right one based on which account they arrive from:

```jsonc
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
    { "agentId": "home", "match": { "channel": "whatsapp", "accountId": "personal" } },
    { "agentId": "work", "match": { "channel": "whatsapp", "accountId": "biz" } }
  ]
}
```

> **What are bindings?** They're routing rules. The `bindings` array tells OpenClaw: "when a message comes from my personal WhatsApp, send it to the Home agent. When it comes from my business WhatsApp, send it to the Work agent." You can bind by channel (WhatsApp, Telegram, Discord), account, or even specific group chats.

**3.** Create separate `SOUL.md` files for each agent so they have different personalities:

```bash
nano ~/.openclaw/workspace-home/SOUL.md
# Add your personal assistant personality

nano ~/.openclaw/workspace-work/SOUL.md
# Add your professional/work personality
```

**4.** Restart the gateway to activate the multi-agent setup:

```bash
openclaw gateway restart
```

You've now got real workflows running on your agent. Start with one, let it prove itself, then layer on more.

---

## Quick Reference

Here are the commands you'll use most often, all in one place. Type these in your terminal unless noted otherwise.

| What You Want To Do | Command | Where To Run |
|---------------------|---------|-------------|
| Run a health check | `openclaw doctor` | Terminal |
| Fix invalid config keys | `openclaw doctor --fix` | Terminal |
| View your config | `cat ~/.openclaw/openclaw.json` | Terminal |
| Edit your config | `nano ~/.openclaw/openclaw.json` | Terminal |
| List your models | `openclaw models list` | Terminal |
| Check model auth | `openclaw models status` | Terminal |
| Start the gateway | `openclaw gateway` | Terminal |
| Restart the gateway | `openclaw gateway restart` | Terminal |
| Stop the gateway | `openclaw gateway stop` | Terminal |
| Check channel status | `openclaw channels status --probe` | Terminal |
| List cron jobs | `openclaw cron list` | Terminal |
| List plugins | `openclaw plugins list` | Terminal |
| Compact chat history | `/compact` | Inside agent chat |
| Start a new session | `/new` | Inside agent chat |
| Create a backup | `openclaw backup create --verify` | Terminal |
| Update OpenClaw | `openclaw update` | Terminal |
| Follow live logs | `openclaw logs --follow` | Terminal |
| Set queue mode | `/queue steer` | Inside agent chat |

> Commands starting with `/` (like `/compact`, `/new`, `/queue`) are typed *inside a conversation with your agent* — not in your system terminal. Everything else runs in your regular terminal app.

---

## Full Config Reference

Throughout this guide, you've been adding blocks to `~/.openclaw/openclaw.json` one at a time. Here's what a complete config looks like with everything assembled — models, heartbeat, channels, security, and web search. Use this as a reference to compare against your own file.

> **Don't copy-paste this entire file.** It's a reference, not a starter. Your config may have different channels, models, or settings. Use it to check that your sections are nested correctly and that nothing is missing.

```jsonc
{
  // Session isolation (Step 8)
  "session": { "dmScope": "per-channel-peer" },

  // Gateway authentication (Step 8)
  "gateway": {
    "auth": { "token": "${OPENCLAW_GATEWAY_TOKEN}" },
    "bind": "loopback"
  },

  // Environment variable references (Step 8)
  "env": {
    "ANTHROPIC_API_KEY": "${ANTHROPIC_API_KEY}",
    "BRAVE_API_KEY": "${BRAVE_API_KEY}"
  },

  // Agent configuration (Steps 2, 3, 4, 5)
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-20250514",
        "fallbacks": ["openai/gpt-4o-mini"]
      },
      "memorySearch": {
        "experimental": { "sessionMemory": true },
        "sources": ["memory", "sessions"]
      },
      "heartbeat": {
        "every": "30m",
        "target": "last",
        "lightContext": true,
        "isolatedSession": true,
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "identity": {
          "name": "Atlas",
          "theme": "space lobster",
          "emoji": "🦞"
        }
      }
    ]
  },

  // Communication channels (Step 7)
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+91XXXXXXXXXX"]
    },
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    }
  },

  // Web search (Step 9)
  "tools": {
    "web": {
      "search": {
        "enabled": true,
        "provider": "brave",
        "apiKey": "${BRAVE_API_KEY}",
        "maxResults": 5
      },
      "fetch": { "enabled": true }
    }
  },

  // Cron tuning (Step 6, optional)
  "cron": {
    "sessionRetention": "12h",
    "runLog": {
      "maxBytes": "3mb",
      "keepLines": 1500
    }
  }
}
```

**Key things to check in your own config:**

- Every opening `{` has a matching closing `}`
- Every opening `[` has a matching closing `]`
- Commas separate items in arrays and objects — but no trailing comma after the last item
- All string values are wrapped in double quotes `"like this"`
- `identity` is nested inside the agent object, not at the agent level
- Secret values use `${VAR_NAME}` syntax, not actual keys

> **Broke your config?** Run `openclaw doctor --fix` to auto-repair common issues. If that doesn't help, paste your full config into the troubleshooting project from Step 1 and ask it to find the JSON error.

---

## Common Problems & Fixes

When something goes wrong, start here. These are the most frequent issues people hit during setup.

### "command not found: openclaw"

**Cause:** OpenClaw isn't installed, or your terminal can't find it.

**Fix:**

```bash
# Reinstall
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash

# Or check if it's installed but not in your PATH
which openclaw
```

If `which` returns nothing, close and reopen your terminal — the installer may have updated your PATH but the current session doesn't have it yet.

### "Gateway failed to start: gateway already running"

**Cause:** The gateway is already running as a background service. You're trying to start a second instance.

**Fix:** This is normal. Just use the existing gateway:

```bash
openclaw health
openclaw gateway restart
```

### "Config invalid / Unrecognized keys"

**Cause:** A key is in the wrong place in your config (e.g., `name` directly on the agent instead of inside `identity`).

**Fix:**

```bash
openclaw doctor --fix
```

This strips invalid keys. Then re-add them in the correct location using the Full Config Reference above.

### "Model authentication failed"

**Cause:** Your API key is missing, expired, or incorrect.

**Fix:**

```bash
openclaw models status
```

If it shows errors, refresh your credentials:

```bash
openclaw models auth setup-token --provider anthropic
```

Or check that your `.env` file has the correct key:

```bash
cat ~/.openclaw/.env | grep ANTHROPIC
```

### "Port 18789 is already in use"

**Cause:** Another process (or the gateway itself) is using the default port.

**Fix:** Check what's on the port:

```bash
# macOS:
lsof -i :18789

# Linux:
sudo ss -tlnp | grep 18789
```

If it's the OpenClaw gateway, just use `openclaw gateway restart`. If it's something else, either stop that process or use a different port:

```bash
openclaw gateway --port 18790
```

### "Cannot connect to the Docker daemon"

**Cause:** Docker isn't running (needed for sandboxing in Step 8 Advanced).

**Fix:**

```bash
# macOS: open Docker Desktop from Applications

# Linux:
sudo systemctl start docker
```

### Gateway starts but agent doesn't respond on WhatsApp/Telegram

**Cause:** Channel isn't connected, pairing expired, or allowlist is blocking you.

**Fix:**

```bash
# Check channel status
openclaw channels status --probe

# Re-pair WhatsApp if needed
openclaw channels login --channel whatsapp

# Check that your number/ID is in the allowFrom list
cat ~/.openclaw/openclaw.json | grep allowFrom
```

### Cron jobs aren't firing

**Cause:** Gateway isn't running, timezone mismatch, or the job was created with the wrong schedule.

**Fix:**

```bash
# Confirm gateway is running
openclaw health

# List your jobs and check schedules
openclaw cron list

# Follow logs to see if cron triggers appear
openclaw logs --follow
```

> Press `Ctrl + C` to stop following logs.

### Config file is broken and nothing works

**Cause:** Malformed JSON — a missing comma, bracket, or quote.

**Fix:**

```bash
# Try auto-repair first
openclaw doctor --fix

# If that doesn't work, validate the JSON manually
cat ~/.openclaw/openclaw.json | python3 -m json.tool
```

The Python command will show exactly which line has the error. Fix it, then restart:

```bash
openclaw gateway restart
```

> **Nuclear option:** If your config is beyond repair, restore from a backup: `openclaw backup create` (if you can still run it) or manually restore from your last Git commit: `cd ~/.openclaw/workspace && git checkout -- .`

### Still stuck?

Open the troubleshooting Claude/ChatGPT project you set up in Step 1, paste the full error output along with your config, and get a diagnosis. That's what it's there for.

---

## What's Next?

You've built the foundation. From here, the key is iteration. Here's a practical path forward:

**Week 1:** Start with the heartbeat (Step 5) and one cron job — maybe the morning briefing. Live with it for a week. See if the notifications are useful or annoying. Adjust the timing and the checklist.

**Week 2:** Personalize harder. Update your `SOUL.md` based on what the agent got wrong in Week 1. Add more writing samples. The agent gets noticeably better with each sample you add.

**Week 3:** Try a use case from Step 10 — the content pipeline or the CRM. Start simple, add complexity as it proves useful.

**Ongoing:** Commit your workspace to Git regularly (`git add . && git commit -m "snapshot"`). Run `openclaw backup create --verify` monthly. Keep your models and plugins updated with `openclaw update`.

**If something breaks:** Check the Common Problems section above first. Run `openclaw doctor` as your first diagnostic step — it catches most issues. For anything else, paste the error into your troubleshooting project from Step 1.

OpenClaw is still early. There will be rough edges. But the people who start experimenting now will be the ones who know how to manage AI agents when this becomes the norm. The magical moments are worth the effort.

Happy building.

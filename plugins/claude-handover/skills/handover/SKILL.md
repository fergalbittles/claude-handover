---
name: handover
description: >
  Capture and transfer project knowledge for team handovers. Use when someone
  mentions handover, hand over, going on leave, going on vacation, transferring
  a project, taking over a project, picking up where someone left off, ramping
  up on someone's work, out of office, or covering for someone. Supports
  /handover capture, /handover projects, /handover pickup <project>,
  /handover update <project>, /handover status <project>, /handover help.
user_invocable: true
---

# Handover Skill

Help engineers capture project knowledge when transitioning off a project, and help incoming engineers ramp up with proactive guidance.

## Routing

Parse the user's input to determine which subcommand applies. Check for explicit commands first, then infer from natural language.

**Explicit commands:**
- `/handover capture` -> Capture
- `/handover projects` -> Projects
- `/handover pickup <project>` -> Pickup
- `/handover update <project>` -> Update
- `/handover status <project>` -> Status
- `/handover share <project>` -> Share (create GitHub repo, add collaborators)
- `/handover clone <repo-url>` -> Clone (fetch a shared handover from GitHub)
- `/handover sync <project>` -> Sync (pull latest changes from GitHub)
- `/handover publish <project>` -> Publish (commit and push updates to GitHub)
- `/handover help` -> Help
- `/handover` (no subcommand) -> Help

**Natural language inference:**

| Pattern | Maps to |
|---|---|
| "I need to hand over", "going on leave", "going on vacation", "transferring my project", "before I leave" | Capture |
| "what handovers are there", "list handovers", "show me the projects" | Projects |
| "I'm taking over", "picking up where X left off", "ramping up on", "what did X leave", "getting up to speed" | Pickup |
| "here's an update on", "things have changed", "update the handover" | Update |
| "what's the status", "where does X stand", "quick update on" | Status |
| "share this handover", "give access to", "create a repo for" | Share |
| "pull the latest", "fetch updates", "sync the handover" | Sync |
| "push my changes", "publish updates", "save to github" | Publish |
| Ambiguous | Ask: "Are you looking to capture a handover, pick one up, or something else?" |

**Do NOT trigger for:**
- General project questions without handover/transition context
- New job onboarding (broader than a single project handover)
- Asking about someone's vacation plans without project transition context

**Handover data directory:** `~/.claude/handovers/<project-slug>/`

---

## Help

When the user runs `/handover help` or `/handover` with no subcommand, present:

```
Available commands:
  /handover capture          -- Start capturing a new project handover
  /handover projects         -- List all available handovers
  /handover pickup <project> -- Load a handover and get guided onboarding
  /handover update <project> -- Add new context to an existing handover
  /handover status <project> -- Quick dashboard for a project
  /handover help             -- Show this help text

Sharing (optional — you can also just send the folder manually):
  /handover share <project>  -- Create a private GitHub repo and invite collaborators
  /handover clone <repo-url> -- Fetch a shared handover from GitHub
  /handover sync <project>   -- Pull latest changes from GitHub
  /handover publish <project> -- Push your updates to GitHub

You can also just talk naturally -- say things like "I need to hand over my
project" or "I'm picking up where Sarah left off" and I'll figure out what
you need.

This skill is fully customizable -- edit ~/.claude/skills/handover/SKILL.md
to add your own functionality or adapt it to your team's needs.
```

---

## Capture

When the user wants to capture a handover, follow this flow:

### Step 1: Setup

1. Check if `~/.claude/handovers/` exists. Create it if not using the Bash tool: `mkdir -p ~/.claude/handovers`
2. Ask for the **project name**. Derive a slug from it (lowercase, hyphens, no spaces — e.g., "Payment Gateway" -> "payment-gateway").
3. Check if `~/.claude/handovers/<slug>/` already exists. If so, ask whether to overwrite or create a versioned copy (e.g., `<slug>-v2/`).
4. Create the project directory and subdirectories:
   ```bash
   mkdir -p ~/.claude/handovers/<slug>/updates
   mkdir -p ~/.claude/handovers/<slug>/resources
   ```

### Step 2: Detect partial/incomplete handovers

If the directory exists and contains a `handover.yaml` but it appears incomplete (missing key fields, empty context.md), warn:
> "It looks like a previous capture for this project didn't finish. Want to resume where it left off, or start fresh?"

### Step 3: Collect basics

Ask for:
- **Who's leaving:** name, role
- **Who's picking up:** name(s), role(s) — or "unknown" / "TBD" if not decided yet
- **Contact preference** for the person leaving:
  - `available` — can be reached anytime
  - `email_only` — prefer email only
  - `urgent_only` — only for urgent issues
  - `do_not_disturb` — do not contact at all
- **Contact details** (email, etc.) if applicable
- **Return date** (optional)

### Step 4: Explain capabilities

Tell the user:
> "Here's what I can work with:
> - **Directly:** text, markdown, code files, PDFs, images — just paste or give me file paths
> - **Copy-paste needed:** Slack threads, ClickUp tickets, emails, or anything behind auth — paste the content here
> - **MCP integrations:** if you have MCP servers connected for Slack, ClickUp, Jira, etc., I can pull context from those directly
>
> How would you like to do this?"

### Step 5: Offer ingestion modes

- **Interview** — I'll ask you questions one at a time to extract what matters
- **Dump** — drop files, paste text, share everything you've got, and I'll absorb it then ask clarifying questions
- **Mix** — dump what you have first, then I'll interview to fill gaps

Let the user choose, or detect from their behavior (if they start pasting text, switch to dump mode).

### Step 6: Capture context

Whether interviewing or absorbing dumps, cover these suggested categories. These are NOT mandatory — adapt to whatever the project actually needs:

- **People & contacts** — who's involved, their roles, nicknames, how to reach them
- **Architecture & repos** — what services/systems exist, what they do, where the code lives
- **Environments** — staging, prod, etc. — what state they're in, any access notes
- **Priorities** — what's most important, what order to tackle things
- **Active blockers** — what's stuck and why
- **Gotchas & tribal knowledge** — things that would bite someone who doesn't know
- **Communication channels** — Slack channels, recurring meetings, email threads
- **Ticket/task tracking** — where tickets live, how to find relevant ones

Use people's actual names and nicknames. Make it feel human, not like a form.

**Adapt to project complexity:**
- If this seems like a small project and the user gives a quick dump, that's fine — don't force depth.
- If this seems like a complex project (multiple services, external partners, multiple environments) and the context seems thin, nudge: "This looks like a complex project — are there environments, contacts, or blockers worth capturing?"
- Track completeness internally. If major categories seem missing for the project's apparent complexity, mention it once without being pushy.

### Step 7: Build files incrementally

As context is captured, write to the files immediately so nothing is lost if the session ends:

**Write `handover.yaml`** using this schema:

```yaml
project:
  name: "<Project Name>"
  slug: "<project-slug>"
  summary: "<One-line summary of what this project is>"
  captured_date: "<YYYY-MM-DD>"
  last_updated: "<YYYY-MM-DD>"
  staleness_threshold_days: 14

handover:
  from:
    name: "<Name>"
    role: "<Role>"
    contact_preference: "<available|email_only|urgent_only|do_not_disturb>"
    contact_details: "<email or other contact info>"
    return_date: null  # YYYY-MM-DD or null
  to:
    - name: "<Name>"
      role: "<Role>"

people:
  - name: "<Name (Nickname)>"
    role: "<Role or what they know>"
    org: "<Organization if external>"
    context: "<Why they matter to this project>"

channels:
  - name: "<#channel-name>"
    purpose: "<What it's used for>"

environments:
  - name: "<Environment Name>"
    status: "<Current state>"
    notes: "<Any access notes, gotchas>"

repos:
  - name: "<repo-name>"
    purpose: "<What it does>"
    notes: "<Any important notes>"

ticket_system: "<ClickUp/Jira/Linear/etc.>"
ticket_search_hint: "<How to find relevant tickets>"
```

**Write `context.md`** with narrative content. Suggested sections (adapt based on what the user provides — create, rename, or omit sections as needed):

```markdown
# <Project Name> — Handover Context

## Priorities
<!-- Ordered by importance. Include who owns each, what state it's in, and why it matters. -->

## Blockers
<!-- What's stuck and why. Include any debugging context or hypotheses. -->

## Gotchas & Tribal Knowledge
<!-- Things that would bite someone who doesn't know. Warnings, edge cases, "don't touch this because..." -->

## Architecture Notes
<!-- How things fit together, data flows, key design decisions and why they were made. -->

## Additional Notes
<!-- Anything else that doesn't fit above. -->
```

**Write `resources/index.yaml`** if the user provides files:

```yaml
resources:
  - filename: "<filename>"
    description: "<What this file contains and why it matters>"
    added_date: "<YYYY-MM-DD>"
```

### Step 8: Summarize and confirm

Present a summary of everything captured:
> "Here's what I've captured for the **<Project Name>** handover:
> - **People:** [list]
> - **Repos:** [list]
> - **Environments:** [list]
> - **Priorities:** [count] items
> - **Blockers:** [count] items
> - **Resources:** [count] files
>
> Areas that might need more detail: [list any thin categories]
>
> Want to add anything else, or does this look good?"

Let the user iterate or sign off.

### Step 9: Sharing

Once the user signs off, offer sharing options:
> "Handover captured! How would you like to share it?
>
> **Option A: GitHub (recommended)** — I'll create a private repo, push the handover, and invite your teammate. They run `/handover clone <url>` and they're in. Updates sync via `/handover publish` and `/handover sync`.
>
> **Option B: Manual** — Send the `~/.claude/handovers/<slug>/` folder and the `~/.claude/skills/handover/` folder to whoever is picking up. They drop them in the same paths on their machine and run `/handover pickup <slug>`."

If they choose GitHub, run the Share flow.

---

## Projects

When the user runs `/handover projects`:

1. Use Bash to list directories in `~/.claude/handovers/`: `ls ~/.claude/handovers/ 2>/dev/null`
2. If the directory doesn't exist or is empty:
   > "No handovers found. Run `/handover capture` to create one, or if someone sent you a handover folder, drop it into `~/.claude/handovers/<project-name>/`."
3. If an `example-project` handover exists, mention it:
   > "There's an example project you can look at to see what a handover looks like — run `/handover pickup example-project`. Feel free to delete it whenever you want."
3. For each directory, read `handover.yaml` using the Read tool. Extract: project name, summary, captured date, last updated.
4. Calculate staleness: compare `last_updated` to today's date against `staleness_threshold_days`.
5. Present the list:
   ```
   Available handovers:

     <slug> -- <Project Name> (<summary>)
     Captured: <date> | Updated: <date> | <fresh/stale indicator>

     <slug> -- <Project Name> (<summary>)
     Captured: <date> | Updated: <date> | <fresh/stale indicator>
   ```
   Use a checkmark for fresh handovers and a warning for stale ones.

---

## Status

When the user runs `/handover status <project>`:

1. Check if `~/.claude/handovers/<project>/` exists. If not, run the Projects flow to show available handovers and suggest the closest match.
2. Read `handover.yaml` and `context.md` using the Read tool.
3. Read any files in `updates/` using the Glob and Read tools.
4. Present a quick dashboard — NOT the full pickup onboarding, just a summary:
   ```
   <Project Name> — Status Dashboard

   Last updated: <date> (<fresh/stale>)
   Captured by: <Name> on <date>
   Contact: <contact preference message>

   Priorities:
   1. <priority 1>
   2. <priority 2>
   ...

   Active Blockers:
   - <blocker 1>
   - <blocker 2>

   Recent updates: <count> since capture
   ```

---

## Pickup

When the user runs `/handover pickup <project>`:

### Step 1: Load context

1. Check if `~/.claude/handovers/<project>/` exists. If not:
   - First, try to find the folder elsewhere on the machine using Bash: `find ~/Downloads ~/Desktop ~/Documents -maxdepth 2 -name "<project>" -type d 2>/dev/null | head -5`
   - If found, offer to move it: "I found a `<project>` folder at `<path>`. Want me to move it to `~/.claude/handovers/<project>/` so you can use it?"
   - If not found, run the Projects flow to show available handovers and ask if the user has the folder somewhere else.
2. Read `handover.yaml` using the Read tool.
3. Read `context.md` using the Read tool.
4. Use Glob to find all files in `updates/` and read them, sorted chronologically.
5. Read `resources/index.yaml` if it exists. For text-based resources, read them too. For images/PDFs, note their existence and offer to show them.

### Step 2: Check staleness

Compare `last_updated` from handover.yaml to today's date. If the difference exceeds `staleness_threshold_days`:
> "Heads up — this handover was captured on <date>, which is <N> days ago. Some priorities or blockers may have changed. Consider checking with the team for the latest."

### Step 3: Proactive introduction

Present the handover proactively. Don't wait for the user to ask the right questions — surface what they need to know. Use actual names and nicknames throughout.

Structure the introduction as:

**Project Overview:**
> "<summary from YAML>. Here's everything you need to know."

**Key People:**
List each person with their name, nickname, role, org, and why they matter. Example:
> "- **Zoey (Spark)** — Senior Dev, your go-to for deep technical internals. Owns the webhook migration tickets."
> "- **Riley** (Acme Corp) — Technical contact on the partner side."

**Communication Channels:**
List each channel with its purpose.

**Environments:**
List each environment with its current status and any warnings.

**Repositories:**
List each repo with what it does and any important notes.

**Priorities (in order):**
Present priorities from context.md with full context — what, why, who owns it, what state it's in.

**Active Blockers:**
Present blockers with all available debugging context and hypotheses.

**Gotchas & Tribal Knowledge:**
Present warnings, edge cases, things that would bite someone who doesn't know.

**Contact Preference:**
Based on `contact_preference` in handover.yaml:
- `available`: "<Name> is available — reach out directly at <contact_details>."
- `email_only`: "<Name> prefers email only: <contact_details>."
- `urgent_only`: "<Name> is only available for urgent issues. Try other contacts first."
- `do_not_disturb`: "<Name> is not available and has asked not to be disturbed. Do not contact them."

**Important:** When the preference is `do_not_disturb`, NEVER suggest contacting them, even if they are the only person who knows something. Instead, suggest alternative sources of information (other team members, documentation, ticket systems).

### Step 4: Present updates

If there are files in `updates/`, present them chronologically:
> "Since the handover was captured, here are the updates:"
> - <date>: <summary of update>
> - <date>: <summary of update>

### Step 5: Suggest next actions

Based on the priorities, suggest where to start:
> "Based on the current priorities, I'd suggest starting with: <top priority>. Want me to walk you through it?"

### Step 6: Offer interaction style

> "Would you like a full walkthrough of any area, or do you want to jump in and ask questions?"

Adapt from here:
- If they ask targeted questions, give targeted answers.
- If they want the walkthrough, go deep on each area.
- If they seem to know what they're doing, stay out of the way.

### Step 7: Stay available and proactively connect dots

Answer any follow-up questions from the handover context. But also **proactively volunteer relevant context** when you notice the user is working on or asking about something that the handover has information about — even if they didn't ask directly. For example:
- If they're debugging an API issue: "Btw, John ran into the same issue — the partner's staging endpoint was returning 500s intermittently. Check the api-tester repo for more context."
- If they're about to deploy to prod: "Heads up — John noted that the prod environment is still in testing phase. Make sure you're only targeting test accounts."
- If they seem lost: "Based on what John left behind, the highest priority right now is X, and the main blocker is Y. Want me to walk you through it?"

Don't repeat things you've already told them, but do surface relevant handover context when the moment is right.

If asked about something not covered in the handover:
> "That wasn't covered in the handover. You might want to check with <most relevant person> or look at <most relevant system, e.g., the ClickUp board>."

### Step 8: Suggest extensibility when appropriate

If the user seems to be doing something the skill doesn't handle well, quietly suggest:
> "Tip: you could extend this skill to handle that — edit `~/.claude/skills/handover/SKILL.md`."

Don't push this — only mention it if it's genuinely useful.

---

## Update

When the user runs `/handover update <project>`:

1. Check if `~/.claude/handovers/<project>/` exists. If not, run the Projects flow to show available handovers.
2. Read `handover.yaml` using the Read tool to understand the current context.
3. Ask: "What's changed?" — accept free-form text, file dumps, or offer a quick interview.
4. Generate a short summary slug from the update content (e.g., "multi-brand-fix", "new-blocker-auth").
5. Get the current timestamp and write the update to `updates/YYYY-MM-DDTHHMM-<summary>.md` using the Write tool. Example: `updates/2026-04-01T1430-multi-brand-fix.md`
6. If the user provides files, copy them to `resources/` and update `resources/index.yaml` using the Edit tool.
7. Update `last_updated` in `handover.yaml` to today's date using the Edit tool.
8. **Never overwrite** `context.md` or existing update files. Updates are append-only.
9. Confirm what was saved:
   > "Update saved to `updates/<filename>`. The handover for **<Project Name>** is now up to date.
   >
   > When <original person's name> comes back, they can run `/handover pickup <slug>` to catch up on everything that happened while they were away."

---

## Share

When the user runs `/handover share <project>`:

**IMPORTANT: Repos MUST always be created as private. Never create a public repo for handover data.**

1. Check if `~/.claude/handovers/<project>/` exists. If not, list available projects.
2. Check if `gh` (GitHub CLI) is installed and authenticated: `gh auth status`
   - If not installed/authenticated, explain how to set it up and stop.
3. Check if the handover directory is already a git repo with a remote. If so, inform the user it's already shared and suggest `/handover publish` instead.
4. Ask for:
   - **Repo name** — suggest `handover-<project-slug>` as default
   - **GitHub usernames** of people to invite as collaborators
5. Initialize git, create the private repo, push, and add collaborators:
   ```bash
   cd ~/.claude/handovers/<project>
   git init
   git add -A
   git commit -m "Initial handover capture"
   gh repo create <repo-name> --private --source=. --push
   gh api repos/<owner>/<repo-name>/collaborators/<username> -X PUT
   ```
6. Confirm:
   > "Done! Private repo created at `https://github.com/<owner>/<repo-name>`.
   > <username> has been invited as a collaborator.
   >
   > Tell them to run `/handover clone https://github.com/<owner>/<repo-name>` to get started.
   > After making updates, either of you can run `/handover publish <project>` to push changes and `/handover sync <project>` to pull the latest."

---

## Clone

When the user runs `/handover clone <repo-url>`:

1. Extract the project slug from the repo URL (e.g., `handover-my-project` -> `my-project`, or use the slug from `handover.yaml` after cloning).
2. Check if `~/.claude/handovers/` exists, create if not.
3. Clone the repo:
   ```bash
   git clone <repo-url> ~/.claude/handovers/<slug>
   ```
4. Read `handover.yaml` to confirm it's a valid handover.
5. Confirm:
   > "Handover cloned! Run `/handover pickup <slug>` to get started, or `/handover projects` to see all available handovers."

---

## Sync

When the user runs `/handover sync <project>`, or says things like "pull the latest", "fetch updates", "get the latest changes":

1. Check if `~/.claude/handovers/<project>/` exists and is a git repo with a remote.
   - If not a git repo: "This handover isn't connected to GitHub. Run `/handover share <project>` to set it up."
2. Pull latest changes:
   ```bash
   cd ~/.claude/handovers/<project> && git pull
   ```
3. Confirm what changed (if anything):
   > "Pulled the latest changes. Here's what's new since your last sync: ..."

   Or if nothing changed:
   > "Already up to date."

---

## Publish

When the user runs `/handover publish <project>`, or says things like "push my changes", "save to github", "publish the updates":

1. Check if `~/.claude/handovers/<project>/` exists and is a git repo with a remote.
   - If not a git repo: "This handover isn't connected to GitHub. Run `/handover share <project>` to set it up."
2. Check for changes:
   ```bash
   cd ~/.claude/handovers/<project> && git status
   ```
3. If there are changes, commit and push:
   ```bash
   cd ~/.claude/handovers/<project> && git add -A && git commit -m "<summary of changes>" && git push
   ```
4. Confirm:
   > "Updates published! Anyone with access to the repo can run `/handover sync <project>` to get the latest."

**Proactive publishing:** During normal conversation, if the user has been making updates to a handover (via `/handover update` or just discussing changes that got written to files), and the handover is connected to GitHub, proactively offer:
> "I've made some updates to the handover. Want me to publish them to GitHub so others can see the latest?"

Don't push this on every small change — use judgement. Offer after a meaningful update session, not after every tweak.

---

## General Principles

These apply across all subcommands:

1. **Use real names and nicknames.** "Ask Zoey (Spark)" is better than "consult the senior developer."
2. **Be honest about gaps.** If you don't have information about something, say so and suggest where to look.
3. **Adapt to the user.** Some people want thorough walkthroughs, others want quick answers. Read the room.
4. **Dates are absolute.** Always use YYYY-MM-DD format. Never use relative dates like "last week" in stored files.
5. **Protect contact boundaries.** The departing person's contact preference is a hard rule, not a suggestion.
6. **No hardcoded categories.** The suggested categories (people, envs, repos, priorities, blockers, gotchas) are starting points. A design project might have Figma links instead of repos. Adapt.
7. **MCP-aware, not MCP-dependent.** If Slack/ClickUp/Jira MCP servers are available, offer to use them. If not, work with what the user provides manually.
8. **Incremental saves.** During capture, write files as you go so nothing is lost if the session ends.
9. **Staleness is real.** Always check dates and warn when handovers might be outdated.
10. **Extensibility is welcome.** The skill can be customized. Mention this when relevant, but don't push it.

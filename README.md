# claude-handover

A Claude Code skill for capturing and transferring project knowledge during team handovers.

When someone goes on leave or transitions off a project, critical knowledge gets scattered across Slack threads, ticket boards, rough notes, and people's heads. This skill helps capture all of that in a structured way, and then helps the person picking up the project ramp up with proactive guidance.

## Features

- **Capture** project context through flexible ingestion (interview, file dumps, or both)
- **Pickup** a handover with proactive guidance -- Claude surfaces what you need to know, suggests next actions, and connects dots from the handover context as you work
- **Update** handovers bidirectionally -- the new person can add context so the original person can catch up when they return
- **Share** via private GitHub repos with automatic sync, or just send the folder manually
- **Staleness detection** -- warns when handover context might be outdated
- **Contact preferences** -- respects whether the departing person wants to be reached or left alone

## Installation

### Claude Code (Plugin Marketplace)

Register the marketplace and install:

```bash
/plugin marketplace add fergalbittles/claude-handover
/plugin install claude-handover@claude-handover
```

### Manual Installation

If you prefer to install manually:

```bash
mkdir -p ~/.claude/skills/handover
curl -o ~/.claude/skills/handover/SKILL.md https://raw.githubusercontent.com/fergalbittles/claude-handover/main/skills/handover/SKILL.md
```

### Try the Example Project

To see what a completed handover looks like, copy the example project:

```bash
git clone https://github.com/fergalbittles/claude-handover.git /tmp/claude-handover
cp -r /tmp/claude-handover/handovers/example-project ~/.claude/handovers/example-project
```

Then run `/handover pickup example-project` in Claude Code.

## Commands

```
/handover capture          -- Start capturing a new project handover
/handover projects         -- List all available handovers
/handover pickup <project> -- Load a handover and get guided onboarding
/handover update <project> -- Add new context to an existing handover
/handover status <project> -- Quick dashboard for a project
/handover help             -- Show this help text

Sharing (optional):
/handover share <project>  -- Create a private GitHub repo and invite collaborators
/handover clone <repo-url> -- Fetch a shared handover from GitHub
/handover sync <project>   -- Pull latest changes from GitHub
/handover publish <project> -- Push your updates to GitHub
```

You can also just talk naturally -- "I need to hand over my project" or "I'm picking up where Sarah left off" and Claude will figure out what you need.

## How It Works

### For the person leaving

Run `/handover capture` and Claude will walk you through it. You can:
- **Interview mode** -- Claude asks questions one at a time
- **Dump mode** -- drop files, paste text, share everything at once
- **Mix** -- dump what you have, then Claude fills gaps with questions

Claude adapts to your project's complexity -- quick dump for simple projects, deeper interview for complex ones.

### For the person picking up

Run `/handover pickup <project>` and Claude proactively presents:
- Project summary and current status
- Key people (with nicknames) and how to reach them
- Priorities in order, with context
- Active blockers and debugging context
- Gotchas and tribal knowledge
- Contact preferences for the person who left

As you work, Claude connects dots from the handover -- "Btw, the handover mentioned this might happen, you should check..."

### Sharing

After capturing, share via GitHub (recommended) or manually:

**GitHub:** `/handover share <project>` creates a private repo and invites collaborators. Updates sync with `/handover publish` and `/handover sync`.

**Manual:** Send the `~/.claude/handovers/<project>/` folder to whoever is picking up. They drop it in the same path and run `/handover pickup`.

## File Structure

Handover data lives in `~/.claude/handovers/<project-slug>/`:

```
~/.claude/handovers/my-project/
  handover.yaml        # Structured metadata (people, channels, envs, contact prefs)
  context.md           # Narrative context (priorities, blockers, gotchas, architecture)
  updates/             # Timestamped update files (append-only)
  resources/           # Raw files (PDFs, notes, configs) with index.yaml
```

## Customization

The skill is fully customizable -- edit `~/.claude/skills/handover/SKILL.md` to add functionality or adapt it to your team's needs.

## License

MIT

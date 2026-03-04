# AI Chief of Staff (Forked & Extended)

A fork of [Mike Murchison's claude-chief-of-staff](https://github.com/mimurchison/claude-chief-of-staff), extended with voice learning, CRM-driven follow-ups, and a draft-edit-send workflow.

I'm [Paddy Stobbs](https://linkedin.com/in/paddystobbs), Co-Founder & CEO at [Stackfix](https://www.stackfix.com). Mike's original project gave me the foundation — an AI chief of staff running on Claude Code that connects to every tool I use. I've been extending it to fit how I actually work: drafting in my voice, following up on sales pipeline, and learning from every correction I make.

Watch Mike's original walkthrough [here](https://x.com/mimurchison/status/2022368529417224480).

---

## What's New in This Fork

Three capabilities on top of Mike's original system:

### Voice Learning
Claude learns how I write by capturing before/after pairs whenever I edit a draft. Over time, first drafts get closer to send-ready. Style examples live in `voice/` and are loaded as few-shot context before any drafting.

### CRM-Driven Nudges
The `/nudge` command pulls from a Notion CRM pipeline, finds follow-ups due today, enriches them with Gmail history and meeting notes from the Document Hub, then drafts tailored outreach. Stale thread scanning runs as a secondary pass.

### Draft-Edit-Send Loop
Every draft — triage responses, nudges, outreach — goes through the same flow: draft, review, edit, send (or save as Gmail draft). When I edit before sending, the before/after pair is saved to improve future drafts.

---

## The Original Four Pillars

Built on Mike's core architecture:

**Communicate** — Triage inbox across email, Slack, and messaging. Draft responses in your voice, prioritised by who matters most.

**Learn** — Morning briefings, meeting prep, context from every source before every interaction.

**Deepen Relationships** — Contact tracking with staleness alerts. Auto-enrichment across channels.

**Achieve Goals** — Quarterly objectives filter every triage decision, scheduling call, and task prioritisation.

---

## Quick Start

### Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- Gmail MCP server
- Google Calendar MCP server

### Setup

```bash
git clone https://github.com/paddy-stobbs/claude-chief-of-staff.git
cd claude-chief-of-staff

chmod +x install.sh
./install.sh

claude
# Then type: /gm
```

---

## Commands

| Command | What It Does |
|---------|-------------|
| `/gm` | Morning briefing — calendar, tasks, urgent messages |
| `/triage` | Inbox scan with prioritised draft responses |
| `/nudge` | CRM pipeline follow-ups + stale thread nudges |
| `/my-tasks` | Task management with execution, not just tracking |
| `/enrich` | Auto-build contact profiles from all channels |

---

## What's Included

```
claude-chief-of-staff/
├── CLAUDE.md                    # Your AI operating system — customise this
├── install.sh                   # One-command setup
├── nudge-config.yaml            # Nudge tracker settings
├── contacts/
│   └── example-contact.md       # Contact file template
├── commands/
│   ├── gm.md                    # Morning briefing
│   ├── triage.md                # Inbox triage + draft-edit-send loop
│   ├── nudge.md                 # Follow-up tracker (CRM + stale threads)
│   ├── my-tasks.md              # Task management
│   └── enrich.md                # Contact enrichment
├── voice/
│   ├── seed-*.yaml              # Starter voice examples
│   └── README.md                # How voice learning works
└── docs/
    ├── setup-guide.md
    ├── mcp-servers.md
    ├── customization.md
    ├── commands/
    │   └── nudge.md             # Extended nudge spec (CRM version)
    └── plans/                   # Design docs
```

Operational files (`goals.yaml`, `my-tasks.yaml`, `nudge-log.yaml`) are created locally by `install.sh` and gitignored.

---

## MCP Servers

| Server | Required? | What It Enables |
|--------|-----------|-----------------|
| Gmail | **Yes** | Email triage, drafting, sending |
| Google Calendar | **Yes** | Scheduling, availability, meeting prep |
| Notion | Recommended | CRM pipeline, meeting notes, document hub |
| Slack | Recommended | Slack triage, channel monitoring |
| WhatsApp | Optional | WhatsApp message triage |
| iMessage | Optional | iMessage triage (macOS only) |

See [docs/mcp-servers.md](docs/mcp-servers.md) for installation instructions.

---

## Customisation

`CLAUDE.md` is the core — it defines who you are, how you write, what you care about, and how Claude should operate. The voice system learns from your edits over time.

See [docs/customization.md](docs/customization.md) for the full guide.

---

## Credits

Original project by [Mike Murchison](https://twitter.com/mimurchison) — [claude-chief-of-staff](https://github.com/mimurchison/claude-chief-of-staff). The core architecture, philosophy, and base commands are his work. This fork extends it with voice learning, CRM integration, and the draft-edit-send workflow.

---

MIT License. See [LICENSE](LICENSE) for details.

# Claude Code Skills

A collection of custom Claude Code skills for daily productivity, project management, and operational workflows. These are plain markdown files that install to `~/.claude/commands/` and are invoked with `/skill-name` in Claude Code.

## Two Tiers

Skills are organized into two tiers:

- **Core** — Work standalone with just Claude Code and a workboard file. Install and use immediately.
- **Integration** — Require external tools (Aha, Hex, Webex, Slack webhooks, custom Node.js services). Shared as templates to adapt to your own infrastructure.

## Installation

```bash
git clone https://github.com/benxmy/claude-skills.git
cd claude-skills

# Install all core skills
cp skills/morning-coffee.md skills/today.md skills/wrap-up.md \
   skills/yap.md skills/cos.md skills/note.md skills/deprioritize.md \
   skills/research.md skills/write-customer-doc.md ~/.claude/commands/

# Or install everything (core + integration)
cp skills/*.md ~/.claude/commands/
```

---

## Core Skills

These work out of the box with Claude Code and a workboard file. No external services required.

### Daily Workflow

The core daily loop — plan your day, stay on track, review at close.

| Skill | Description |
|-------|-------------|
| `/morning-coffee` | Daily focus picker. Reads workboard, pulls calendar/email/triage, helps pick 2-3 focuses, flags stalled work |
| `/today` | Quick reference for today's focus plan |
| `/wrap-up` | End-of-day review. Compare plan vs actual, capture carry-forwards, update workboard |
| `/yap` | Activity logger. Narrate what you're doing so Claude builds context over time |

**See also**: [morning-coffee](https://github.com/benxmy/morning-coffee) — a standalone package with setup scripts, provider system, and full documentation for the daily workflow.

### Operational

Chief-of-staff style skills for managing work across projects.

| Skill | Description |
|-------|-------------|
| `/cos` | Cross-project status checks, routing incoming work, follow-up tracking, resource surfacing |
| `/note` | Discrete notes for calls/meetings/topics. Tag with people and projects. Search later |
| `/deprioritize` | Park a project and draft stakeholder communications about the deferral |

### Content & Research

Skills for producing documents and structured research.

| Skill | Description |
|-------|-------------|
| `/write-customer-doc` | Draft customer-facing docs (feature guides, release notes, KB articles) |
| `/research` | Structured research framework for product/opportunity or customer/adoption research |

---

## Integration Skills

These require external tools, APIs, or custom infrastructure. **They won't work out of the box** — use them as templates and adapt the commands/paths to your own setup.

| Skill | Description | Requires |
|-------|-------------|----------|
| `/prep` | Prepare for a conversation. Pulls tracking file, message history, and email threads | Microsoft Graph plugin + Webex agent |
| `/manager-update` | Generate 2-5 topic update for weekly manager 1:1 | Microsoft Graph plugin + Webex agent |
| `/portfolio-deck` | Generate a portfolio status PPTX deck from workboard data | Custom Node.js deck generator |
| `/aha-sync` | Sync project status between workboard and Aha releases | Aha API key + custom Node.js connector |
| `/webex-recording` | Find and extract Webex recordings. `since 24h` lists new calls from both your own recordings and ones shared with you by email; then pulls transcript, summary, chapters, action items | Webex API token; Chrome with remote debugging + MCP server (shared recordings only) |
| `/slack-mode` | Route notifications and permission prompts to Slack | Slack webhook + shell hook script |
| `/rotate-hex-token` | Rotate an API token stored in macOS Keychain | Hex account + macOS Keychain |

### Adapting Integration Skills

These skills reference specific tools from the author's setup. To use them:

1. Read the skill file to understand what it does
2. Replace tool-specific commands with your equivalents:
   - `~/projects/deck-generator/` → your PPTX generation tool
   - `~/projects/aha-connector/` → your project management API wrapper
   - `~/.config/claude-graph/bin/msgraph` → your calendar/email CLI
   - Webex agent paths → your messaging platform's API
   - `~/projects/webex-agent/.webex_token.json` → wherever you store your Webex OAuth token
3. Update `allowed-tools` to match your actual commands
4. Test with `/skill-name` in Claude Code

---

## Dependencies

| Dependency | Used by | Tier |
|-----------|---------|------|
| Workboard file | morning-coffee, today, wrap-up, cos, manager-update, portfolio-deck, deprioritize | Core |
| yap-log.md | wrap-up, manager-update | Core (optional) |
| Microsoft Graph plugin | morning-coffee (optional), prep, manager-update | Integration |
| Webex agent | morning-coffee (optional), prep, manager-update | Integration |
| Webex API token (`spark:recordings_read`, `meeting:recordings_read`) | webex-recording | Integration |
| Chrome (remote debugging) | webex-recording (shared recordings only) | Integration |
| Aha API key (Keychain) | aha-sync | Integration |
| Slack webhook | slack-mode | Integration |
| Custom Node.js services | portfolio-deck, aha-sync | Integration |

## Workboard

The workboard is the backbone for most of these skills. See the [morning-coffee repo](https://github.com/benxmy/morning-coffee) for templates, examples, and a guided bootstrapping skill (`/workboard-init`).

## Customization

These are plain markdown files. Edit them directly to change behavior, tone, or workflow. Many skills reference the user by name or use specific project examples — update these to match your context. See [morning-coffee/docs/customization.md](https://github.com/benxmy/morning-coffee/blob/main/docs/customization.md) for guidance on common modifications.

## License

MIT

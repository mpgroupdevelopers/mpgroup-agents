# MP Group Payments for AI Coding Agents

Add MP Group card payments to an app by handing your coding agent one link.

## Use it

Paste this into Codex, Claude Code, Cursor, or another coding agent:

> Read https://raw.githubusercontent.com/YOUR-GITHUB-NAME/mpgroup-agents/main/payments.md and use it to add MP Group payments to this app.

Your agent builds a test-mode checkout ($25 one-time card sale) using NMI's secure payment form. No real money moves.

## Before you start

You need an MP Group **test account**. Contact:

**Christian Brown** · christian.brown@gompgroup.com · 678-740-4364

Your keys stay with you: you put them in a `.env` file yourself. Never paste keys into an AI chat.

## Prefer an installable skill?

The `skill/` folder has the same guidance packaged as an agent skill. Copy `skill/mpgroup-nmi-checkout` into `.agents/skills/` (Codex) or `.claude/skills/` (Claude Code). See `skill/mpgroup-nmi-checkout/INSTALL.md`.

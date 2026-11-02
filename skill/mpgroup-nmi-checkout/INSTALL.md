# Installing the MP Group NMI Checkout skill

This file is for people. The assistant reads `SKILL.md`.

## 1. Put the skill folder in your project

Copy the whole `mpgroup-nmi-checkout` folder (including `references/` and `agents/`) to:

| Tool | Folder |
|---|---|
| Claude Code | `.claude/skills/mpgroup-nmi-checkout/` |
| Codex | `.agents/skills/mpgroup-nmi-checkout/` |

Using both? Keep one real copy and link the other to it, for example from the project root:

```
mkdir -p .agents/skills
ln -s ../../.claude/skills/mpgroup-nmi-checkout .agents/skills/mpgroup-nmi-checkout
```

If a tool doesn't pick up the link, copy the folder to both places instead.

## 2. Connect NMI's Docs MCP (no account or key needed)

| Tool | Command |
|---|---|
| Claude Code | `claude mcp add --transport http nmi-docs https://docs.nmi.com/mcp` |
| Codex | `codex mcp add nmi-docs --url https://docs.nmi.com/mcp` |

Restart the tool afterward.

## 3. Add your keys yourself

1. Create a test merchant account in the MP Group partner portal (My Accounts → Add Merchant → Test).
2. In that test merchant account, create a public tokenization key and a private API key (Settings → Security Keys).
3. After the assistant creates `.env.example`, copy it to `.env` and fill in the keys yourself.

Never paste keys into the chat, and never give the assistant the MP Group partner API key.

## 4. Try it

Prompt: "Add a checkout that charges $25 using MP Group's NMI test account."
In Codex you can also call the skill directly with `$mpgroup-nmi-checkout`.

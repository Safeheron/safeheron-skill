# safeheron-skill

**AI Skill for Safeheron API** — Works with Claude Code and Cursor

> Use natural language to generate, debug, and troubleshoot Safeheron API integrations. No more digging through docs.

---

## ✨ What This Skill Does

- **Code Generation** — Describe your requirements in natural language; the AI generates production-ready Java SDK code
- **Full API Coverage** — Wallets, transactions, MPC signing, Web3, webhooks, whitelists, Gas Station, AML/KYT, Co-Signer

---

## 📦 Installation

### Claude Code

```bash
# Project-level
mkdir -p .claude/skills
cp -r skills/safeheron-api .claude/skills/safeheron-api

# Or user-level
mkdir -p ~/.claude/skills
cp -r skills/safeheron-api ~/.claude/skills/safeheron-api
```

Or install as a plugin:

```bash
claude plugin add safeheron/safeheron-skill
```

### Cursor

Cursor natively supports the same `SKILL.md` format and also reads from `.claude/skills/` for compatibility — so one install covers both tools.

```bash
# Option A — shared install (works for both Claude Code AND Cursor)
mkdir -p .claude/skills
cp -r skills/safeheron-api .claude/skills/safeheron-api

# Option B — Cursor native path (project-level)
mkdir -p .cursor/skills
cp -r skills/safeheron-api .cursor/skills/safeheron-api

# Option C — Cursor native path (user-level, applies to all projects)
mkdir -p ~/.cursor/skills
cp -r skills/safeheron-api ~/.cursor/skills/safeheron-api
```

> **Note:** `~/.cursor/skills-cursor/` is Cursor's built-in read-only directory — do **not** install there. Use `~/.cursor/skills/` instead.

---


## 🚀 Example Prompts

Once installed, try these in Claude Code or Cursor:

- `"Use Safeheron skill to set up my first API call"`
- `"Generate Java code to create a wallet and add ETH and USDT"`
- `"Create a transaction to send 0.01 ETH from wallet abc to address 0x1234..."`
- `"Help me set up the API Co-Signer approval callback service"`
- `"My API call returns error 1010 — what's wrong and how do I fix it?"`
- `"Write a webhook handler that processes incoming transaction events"`
- `"Generate Spring Boot configuration class for Safeheron SDK"`
---

## Resources

- [Safeheron API](https://docs.safeheron.com/api/en.html)
- [Java SDK GitHub](https://github.com/Safeheron/safeheron-api-sdk-java)
- [Safeheron Console](https://www.safeheron.com)

## License

MIT

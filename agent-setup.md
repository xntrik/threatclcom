# Threatcl Cloud — Agent Setup

You are setting up the Threatcl Cloud skill for the user. Work through the steps below, run the commands yourself where appropriate, and report concrete results — don't just recite the outcome.

## What is Threatcl Cloud?

Threatcl Cloud is a collaborative threat modeling platform. Engineers write threat models as HCL files, sync them to the cloud, and share reusable libraries of threats and controls. The `threatcl` CLI is the primary interface.

> **Desktop AI apps (Claude Desktop, ChatGPT Desktop, etc.)** — those don't use skills; they use the MCP server instead. If that's what the user wants, point them at https://help.threatcl.com/article/9-mcp-connector and stop here.

## Step 1 — Install the skill file

Ask the user whether they want the skill installed **user-wide** (available in every project) or **project-only** (scoped to this repo). Default to user-wide unless they say otherwise.

> **Heads up:** this downloads a markdown file that will be loaded into the agent's instructions. Tell the user they can review it at https://threatcl.com/threatcl.SKILL.md before or after install.

**Claude Code — user-wide:**
```bash
mkdir -p ~/.claude/skills/threatcl
curl -fsSL https://threatcl.com/threatcl.SKILL.md -o ~/.claude/skills/threatcl/SKILL.md
```

**Claude Code — project-only:**
```bash
mkdir -p .claude/skills/threatcl
curl -fsSL https://threatcl.com/threatcl.SKILL.md -o .claude/skills/threatcl/SKILL.md
```

**Codex — user-wide:**
```bash
mkdir -p ~/.codex/skills/threatcl
curl -fsSL https://threatcl.com/threatcl.SKILL.md -o ~/.codex/skills/threatcl/SKILL.md
```

Verify the download actually landed:
```bash
test -s <path>/SKILL.md && echo "SKILL.md installed" || echo "install failed"
```

If the file is empty or missing, stop and surface the curl error to the user — don't continue.

> In Claude Code, newly installed skills are picked up on next turn; in Codex, a restart may be required. If the user doesn't see the skill active shortly, suggest they restart the agent.

## Step 2 — Verify the CLI

```bash
threatcl version
```

If it's missing, suggest one of:
- macOS/Linux: `brew install threatcl`
- Go: `go install github.com/threatcl/threatcl/cmd/threatcl@latest`
- Download: https://github.com/threatcl/threatcl/releases

Wait for the user to install before continuing.

## Step 3 — Point the CLI at the cloud API

The default endpoint (`https://api.threatcl.com`) will work eventually, but today `threatcl cloud` commands need `THREATCL_API_URL=https://beta-api.threatcl.com`.

For this session, export it so the next steps work:
```bash
export THREATCL_API_URL=https://beta-api.threatcl.com
```

Then tell the user to persist it in their shell profile (`~/.zshrc`, `~/.bashrc`, etc.) so future sessions don't break.

## Step 4 — Check authentication

```bash
threatcl cloud whoami
```

If this fails, tell the user to run `threatcl cloud login` and complete the browser flow. Wait for them to finish, then re-run `whoami`.

Once it succeeds, note the **organization slug** in the output — the user will need it for the `backend` block when authoring new threat models. If there are multiple orgs, ask which one they want as the default.

## Step 5 — Smoke test

Prove setup actually works end-to-end by running:
```bash
threatcl cloud threatmodels
```

Report the result to the user concretely — e.g. "You have 7 threat models in `acme-corp`. The most recent is `payments-api`." If the org is empty, say so and offer to scaffold a starter model with `threatcl generate boilerplate`.

That's it — setup is done. From here on, the skill at `SKILL.md` takes over and documents the rest of the workflow (search, push, libraries, policies, DFDs, etc.).

## Full Documentation

For full detailed command references and HCL syntax read: https://threatcl.com/agent-docs.md

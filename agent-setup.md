# Threatcl Cloud — Agent Setup

## What is Threatcl Cloud?

Threatcl Cloud is a collaborative threat modeling platform. Engineers write
threat models as HCL (HashiCorp Configuration Language) files, sync them to
the cloud for team collaboration, and use reusable libraries of threats and
controls. The `threatcl` CLI is the primary interface.

## Installation

### Desktop AI Apps, or Coding Agents?

There are two ways to interact with Threatcl Cloud with AI tools:

1. **Coding Agents** (Claude Code, Codex, Cursor etc) - install the skill and use the `threatcl` CLI. This is best for engineers already working this way. The rest of this setup focuses on this.

2. **Desktop AI Apps** (Claude Desktop, ChatGPT Desktop, Cursor Chat etc) - Threatcl Cloud also supports an MCP Server. If the user would like to explore that, visit https://help.threatcl.com/article/9-mcp-connector

### Step 1: Install the skill

**Claude Code:**
```bash
mkdir -p .claude/skills/threatcl
curl -fsSL https://threatcl.com/threatcl.SKILL.md -o .claude/skills/threatcl/SKILL.md
```

**Codex:**
```bash
mkdir -p ~/.codex/skills/threatcl
curl -fsSL https://threatcl.com/threatcl.SKILL.md -o ~/.codex/skills/threatcl/SKILL.md
```

### Step 2: Verify the CLI

Run `threatcl version` to verify the CLI is installed. If not installed:

- macOS/Linux: `brew install threatcl`
- Go users: `go install github.com/threatcl/threatcl/cmd/threatcl@latest`
- Download: https://github.com/threatcl/threatcl/releases

#### Configure the API endpoint for `threatcl cloud`

In the future, the default endpoint of `https://api.threatcl.com` will work for `threatcl cloud` commands, but, for the timebeing, `threatcl cloud` commands will need to be prefixed with `THREATCL_API_URL=https://beta-api.threatcl.com`, or, this will need to be `export` in the current shell

### Step 3: Check authentication

Run `threatcl cloud whoami`. If not authenticated, tell the user to run
`threatcl cloud login` to authenticate via the browser.

### Step 4: Confirm setup

Tell the user: "Threatcl Cloud skill is installed. I can now help you with:
- Creating and editing HCL threat model files
- Pushing and syncing threat models to Threatcl Cloud
- Searching threats and controls across your organization
- Working with reusable threat and control libraries
- STRIDE analysis and security gap identification
- Generating data flow diagrams and dashboards

Try asking me to list your threat models or create a new one."

## Full Documentation

For detailed command reference and HCL syntax, read:
https://threatcl.com/agent-docs.md

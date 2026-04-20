# Setup Instructions — FinAlly Claude Project Environment

Complete step-by-step guide to reproduce all tools, plugins, skills, agents, and MCP servers on a fresh machine.

---

## Prerequisites

### 1. Install Node.js (v20+)

```bash
# macOS (via Homebrew)
brew install node

# Windows — download installer from https://nodejs.org
# Linux (Ubuntu/Debian)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

Verify: `node --version` should print `v20.x.x` or higher.

---

### 2. Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

Verify: `claude --version`

First-time setup — authenticate with your Anthropic account:

```bash
claude
```

Follow the prompts to log in.

---

### 3. Install Python + uv (for backend projects)

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Verify: `uv --version`

---

## Plugins

Plugins are installed via the Claude Code CLI. There are two scopes:
- `--scope user` — available in every project on this machine
- `--scope project` — available only in the current project directory (run from inside the project folder)

### User-scoped plugins (install once, available everywhere)

```bash
claude plugin install superpowers@claude-plugins-official --scope user
claude plugin install feature-dev@claude-plugins-official --scope user
claude plugin install code-review@claude-plugins-official --scope user
claude plugin install code-simplifier@claude-plugins-official --scope user
claude plugin install ralph-loop@claude-plugins-official --scope user
```

### Project-scoped plugins (run from inside your project directory)

```bash
cd /path/to/your/project

claude plugin install frontend-design@claude-plugins-official --scope project
claude plugin install context7@claude-plugins-official --scope project
claude plugin install playwright@claude-plugins-official --scope project
```

### Verify installed plugins

```bash
claude plugin list
```

### What each plugin provides

| Plugin | Skills | Agents |
|--------|--------|--------|
| `superpowers` | brainstorming, writing-plans, executing-plans, systematic-debugging, TDD, verification-before-completion, code-review flows, git worktrees, dispatching-parallel-agents, subagent-driven-development, writing-skills | `superpowers:code-reviewer` |
| `feature-dev` | feature-dev | code-architect, code-explorer, code-reviewer |
| `code-review` | code-review:code-review | — |
| `code-simplifier` | simplify | code-simplifier |
| `ralph-loop` | ralph-loop, cancel-ralph, help | — |
| `frontend-design` | frontend-design | — |
| `context7` | (MCP tool) fetch current library docs | — |
| `playwright` | (MCP tool) browser automation | — |

---

## MCP Servers

MCP servers are configured in `~/.claude/settings.json` (user-level) or `.claude/settings.json` (project-level).

### Gmail MCP

Enables Claude to read and send Gmail.

**Step 1** — Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "gmail": {
      "command": "npx",
      "args": ["-y", "@gongrzhe/server-gmail-autoauth-mcp"]
    }
  }
}
```

**Step 2** — On first use, Claude will prompt you to authenticate with Google.

### Google Drive MCP

Built into Claude Code — no installation needed. Authenticate when prompted:

```
Use the tool: mcp__claude_ai_Google_Drive__authenticate
```

### IDE MCP (VS Code / JetBrains)

Provides `executeCode` and `getDiagnostics` tools from your IDE.

**VS Code**: Install the **Claude Code** extension from the VS Code marketplace, then open a project — the IDE MCP activates automatically.

**JetBrains**: Install the **Claude Code** plugin from JetBrains Marketplace.

### context7 and playwright MCPs

These are provided automatically by their respective plugins (installed above). No manual configuration needed.

---

## Settings File

Create or update `.claude/settings.json` in your project root with the following content to enable all plugins and permissions:

```json
{
  "enabledPlugins": {
    "superpowers@claude-plugins-official": true,
    "frontend-design@claude-plugins-official": true,
    "feature-dev@claude-plugins-official": true,
    "code-review@claude-plugins-official": true,
    "code-simplifier@claude-plugins-official": true,
    "ralph-loop@claude-plugins-official": true,
    "context7@claude-plugins-official": true,
    "playwright@claude-plugins-official": true
  },
  "permissions": {
    "allow": [
      "Agent"
    ]
  }
}
```

For user-level settings (apply to all projects), put the same content in `~/.claude/settings.json`.

---

## Project Skills

Skills are Markdown files placed in `.claude/skills/` inside your project directory. Claude Code loads them automatically.

### cerebras-inference skill

Create the file `.claude/skills/cerebras/SKILL.md` in your project:

```markdown
---
name: cerebras-inference
description: Use this to write code to call an LLM using LiteLLM and OpenRouter with the Cerebras inference provider
---

# Calling an LLM via Cerebras

These instructions allow you write code to call an LLM with Cerebras specified as the inference provider.
This method uses LiteLLM and OpenRouter.

## Setup

The OPENROUTER_API_KEY must be set in the .env file and loaded in as an environment variable.

The uv project must include litellm and pydantic.
`uv add litellm pydantic`

## Code snippets

### Imports and constants

\`\`\`python
from litellm import completion
MODEL = "openrouter/openai/gpt-oss-120b"
EXTRA_BODY = {"provider": {"order": ["cerebras"]}}
\`\`\`

### Code to call via Cerebras for a text response

\`\`\`python
response = completion(model=MODEL, messages=messages, reasoning_effort="low", extra_body=EXTRA_BODY)
result = response.choices[0].message.content
\`\`\`

### Code to call via Cerebras for a Structured Outputs response

\`\`\`python
response = completion(model=MODEL, messages=messages, response_format=MyBaseModelSubclass, reasoning_effort="low", extra_body=EXTRA_BODY)
result = response.choices[0].message.content
result_as_object = MyBaseModelSubclass.model_validate_json(result)
\`\`\`
```

### doc-review skill

Create `.claude/skills/doc-review.md`:

```markdown
---
name: doc-review
description: Review the documentation file in the planning folder called $ARGUMENTS and add questions, clarifications, and suggestions
---

Review the documentation file at planning/$ARGUMENTS.
Add a section at the bottom with questions, clarifications needed, and improvement suggestions.
Be specific and actionable.
```

---

## Local Agents

Agents are Markdown files placed in `.claude/agents/` inside your project directory.

### reviewer agent

Create `.claude/agents/reviewer.md`:

```markdown
---
name: reviewer
description: This custom agent do comprehensive reviews code changes and provides feedback to the user. It can also ask clarifying questions if needed.
---
You review file PLAN.MD and write your feedback to planning/REVIEW.md. You can also ask clarifying questions if needed. Your review should be comprehensive and cover all aspects of the plan, including any potential issues or areas for improvement. You should also look for opportunities to simplify the plan and make it more efficient. Your feedback should be clear and actionable, and should provide specific suggestions for how to improve the plan. You should also be sure to highlight any areas of the plan that are particularly strong or well thought out. Overall, your goal is to help the user create a plan that is as effective and efficient as possible, while also providing constructive feedback that can help them improve their planning skills over time.
```

---

## Environment Variables

Create a `.env` file in your project root. Never commit this file to git.

```bash
# Required for LLM chat (get from https://openrouter.ai)
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: real market data (get from https://massiveapi.com)
MASSIVE_API_KEY=

# Optional: deterministic mock LLM responses for testing
LLM_MOCK=false
```

Add `.env` to `.gitignore`:

```bash
echo ".env" >> .gitignore
```

---

## Final Directory Structure

After completing all steps, your project's `.claude/` directory should look like:

```
.claude/
├── settings.json          # Plugin and permission config
├── agents/
│   └── reviewer.md        # Local reviewer agent
└── skills/
    ├── cerebras/
    │   └── SKILL.md       # Cerebras LLM inference skill
    └── doc-review.md      # Documentation review skill
```

---

## Verification Checklist

Run these checks after setup to confirm everything is working:

```bash
# 1. Claude Code is installed
claude --version

# 2. Plugins are installed
claude plugin list

# 3. Project settings exist
cat .claude/settings.json

# 4. Skills exist
ls .claude/skills/

# 5. Agents exist
ls .claude/agents/

# 6. Start a Claude session and type /skills to list available skills
claude

# 7. In the Claude session, run:
#    /agents  — should show the reviewer agent
#    /skills  — should show cerebras-inference, doc-review, and all plugin skills
#    /mcp     — should show context7 and playwright tools
```

---

## Quick Install Script (macOS/Linux)

Save as `setup_claude_env.sh` and run with `bash setup_claude_env.sh` from inside your project directory:

```bash
#!/bin/bash
set -e

echo "Installing user-scoped plugins..."
claude plugin install superpowers@claude-plugins-official --scope user
claude plugin install feature-dev@claude-plugins-official --scope user
claude plugin install code-review@claude-plugins-official --scope user
claude plugin install code-simplifier@claude-plugins-official --scope user
claude plugin install ralph-loop@claude-plugins-official --scope user

echo "Installing project-scoped plugins..."
claude plugin install frontend-design@claude-plugins-official --scope project
claude plugin install context7@claude-plugins-official --scope project
claude plugin install playwright@claude-plugins-official --scope project

echo "Creating .claude directory structure..."
mkdir -p .claude/agents
mkdir -p .claude/skills/cerebras

echo "Done! Copy agent and skill files manually (see instruction_v1.md)."
echo "Then create .claude/settings.json with the content from setting_v1.json."
```

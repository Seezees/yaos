# YAOS — Yet Another Orchestration Skills

**EN**: Universal orchestration skills for AI agents. Delegate work to Google Antigravity CLI (`agy`) and Gemini via SDK or CLI one-shot mode.

**RU**: Универсальные скиллы оркестрации для AI-агентов. Делегирование работы Google Antigravity CLI (`agy`) и Gemini через SDK или CLI one-shot режим.

## Skills

### agy-usage

Orchestration guide for delegating work to AGY as a subagent. Supports two interfaces:

- **Primary**: AGY SDK (`google-antigravity`) — real-time streaming of tokens, thoughts, and tool calls
- **Fallback**: `agy -p` — CLI one-shot non-interactive mode

**Default model**: `Gemini 3.1 Pro (High)` (maximum reasoning effort, verified live)

## Installation

### Factory Droid

```bash
# Personal skills (all projects)
cp -r agy-usage ~/.factory/skills/

# Project skills (single project)
cp -r agy-usage .factory/skills/
```

Or on Windows PowerShell:

```powershell
# Personal
Copy-Item -Recurse agy-usage "$env:USERPROFILE\.factory\skills\"

# Project
Copy-Item -Recurse agy-usage ".\.factory\skills\"
```

### Any agent that supports markdown skills

Place the `agy-usage` directory (containing `SKILL.md` and `references/`) into your agent's skills directory. The skill is a standard markdown file with YAML frontmatter — no compilation or dependencies required.

### Prerequisites

- **AGY CLI** (for fallback mode): Install from [antigravity.google](https://antigravity.google)
- **AGY SDK** (for primary mode): `pip install google-antigravity`
- **Gemini API key** (for SDK): Set `GEMINI_API_KEY` environment variable

The CLI uses browser OAuth (no API key needed). The SDK requires a Gemini API key or Vertex AI credentials.

## Usage

### As an orchestrating agent

When the user says "delegate to agy", "отдай грязную работу agy", or similar, the skill activates and the agent follows the orchestration protocol defined in `SKILL.md`.

### CLI one-shot (fallback)

```bash
agy -p "Summarize this repository" --model "Gemini 3.1 Pro (High)" --add-dir /path/to/project
```

### SDK (primary)

```python
import asyncio
from google.antigravity import Agent, LocalAgentConfig, CapabilitiesConfig

async def main():
    config = LocalAgentConfig(
        model="Gemini 3.1 Pro (High)",
        system_instructions="You are an expert assistant.",
        capabilities=CapabilitiesConfig(),
        workspaces=["/path/to/project"],
    )
    async with Agent(config) as agent:
        response = await agent.chat("Summarize this repository")
        async for token in response:
            print(token, end="")

asyncio.run(main())
```

### Plan/Build pipeline

```bash
# Phase 1 — Plan
agy -p "Create a detailed plan for <task>. Do NOT edit files." \
  --model "Gemini 3.1 Pro (High)" --add-dir /path/to/project

# Phase 2 — Build
agy -p "Implement the plan. Run tests after." \
  --model "Gemini 3.1 Pro (High)" --add-dir /path/to/project --print-timeout 15m
```

## Structure

```
agy-usage/
├── SKILL.md                       # Slim orchestration protocol (60 lines)
└── references/
    └── agy-cli-reference.md       # Full reference: SDK API, CLI flags, recovery, settings, safety
```

## License

BSD 3-Clause — see [LICENSE](LICENSE)

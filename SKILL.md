---
name: agy-usage
description: |
  Orchestration guide for delegating work to Google Antigravity CLI (`agy`) and
  Gemini via the AGY SDK or CLI one-shot mode `agy -p`.

  Trigger when the user says "работай как оркестратор AGY", "оркестратор agy",
  "используй agy как сабагента", "отдай грязную работу agy", "скинь рутину agy",
  "делегируй agy", "запусти agy", or similar wording.
  Also trigger on: "use agy as a subagent", "delegate to agy", "orchestrate agy",
  "offload to agy", "agy subagent", "run agy on", "ask agy to".
---

# AGY Orchestration
Protocol revision: 1.0.0 — Slim orchestration protocol. See Reference Routing
for detailed material. Do not read reference files for normal delegation.

## Defaults

**Model**: `"Gemini 3.1 Pro (High)"` (exact name from `agy models`).
Models with `(High)` include maximum reasoning effort.
Warning: if a model name is not in `agy models`, agy does not error — it silently
falls back to `Gemini 3.5 Flash (Medium)`. Always use exact names from `agy models`.

**Primary — AGY SDK** (`google-antigravity`):
```python
from google.antigravity import Agent, LocalAgentConfig, CapabilitiesConfig
config = LocalAgentConfig(
    model="Gemini 3.1 Pro (High)",
    system_instructions="...",
    capabilities=CapabilitiesConfig(),  # enables write tools
    workspaces=["/path/to/project"],     # equivalent to --add-dir
)
async with Agent(config) as agent:
    response = await agent.chat("task")
    async for token in response:         # text
    async for thought in response.thoughts:  # reasoning
    async for call in response.tool_calls:   # tools
```
Check: `python -m pip show google-antigravity`
Auth: SDK needs `GEMINI_API_KEY` env var or `api_key=` in config (separate from CLI OAuth).

**Fallback — CLI one-shot** (`agy -p`, non-interactive):
```text
agy -p "task" --model "Gemini 3.1 Pro (High)" --add-dir "<workspace>"
```
`--add-dir` is repeatable. `--print-timeout` controls timeout (default 5m).

## Reference Routing

| Question | Where |
|---|---|
| SDK API, streaming, MCP, policies | `references/agy-cli-reference.md` → Orchestration Interfaces → SDK |
| All `agy -p` CLI flags | `references/agy-cli-reference.md` → Orchestration Interfaces → CLI |
| TUI slash-commands | `references/agy-cli-reference.md` → TUI Slash Commands |
| Settings keys | `references/agy-cli-reference.md` → Configuration Settings |
| Recovery after timeout/crash | `references/agy-cli-reference.md` → Recovery Protocol |
| CLI subcommands (`agy models`, etc.) | `references/agy-cli-reference.md` → CLI Subcommands |
| Safety rules R1-R7 | `references/agy-cli-reference.md` → Mandatory Safety Rules |
| Dirty-work delegation matrix | `references/agy-cli-reference.md` → Safe Dirty-Work Delegation |

## Key Rules
**Do not invent**: `agy -p` is real and verified. Never invent `--stream-json`,
`--headless`, `--exec`, or similar flags unless confirmed from live `agy --help`
or official docs.
**Safety** (ref: R1-R7 in reference): Conservative defaults — `request-review`,
sandbox enabled, `allowNonWorkspaceAccess: false`.
**Dirty-work** (ref: full matrix in reference): Delegate mechanical tasks
(build/test, log summaries, repo recon). Verify results independently.
Never delegate security, auth, billing, destructive ops, or final judgment.
**Verification**: After every AGY run, independently verify results — diff
review, rerun tests, inspect artifacts/logs.
**TUI note**: Interactive `agy` (without `-p`) is for human use, not automated
orchestration. Slash commands are TUI-only; for `-p`, encode intent in prompt.

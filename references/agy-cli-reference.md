# AGY CLI Reference

Full reference for AGY orchestration. Read this only when the slim `SKILL.md`
does not contain enough detail. The orchestrating agent should consult specific
sections as directed by the Reference Routing table in `SKILL.md`.

## Entrypoint And TUI

- Run `agy` to open the Antigravity CLI.
- On first run, authenticate through the browser OAuth flow and paste the code back into the terminal.
- `Esc` closes active menus such as `/skills` or `/settings`.
- `Ctrl+D Ctrl+D`, `/exit`, or `/quit` exits the CLI.
- Up/down arrows navigate command history.
- The CLI is primarily interactive. Use `agy -p` for non-interactive orchestration (see below).

---

## Orchestration Interfaces

### SDK (Primary)

The Python SDK package `google-antigravity` is the primary interface for
programmatic orchestration. It provides real-time streaming of tokens, thoughts,
and tool calls with full conversation control.

**Installation and availability check**:
```bash
pip install google-antigravity
python -m pip show google-antigravity  # verify installed
```

**Authentication** (verified against SDK v0.1.5):
- SDK requires a **Gemini API key** — set `GEMINI_API_KEY` env var or pass
  `api_key="..."` in `LocalAgentConfig`. This is separate from CLI OAuth.
- Alternatively, use Vertex AI: `vertex=True`, `project="..."`, `location="..."`
  with Application Default Credentials (`gcloud auth application-default login`).
- CLI `agy -p` uses browser OAuth (stored in keyring) — no API key needed.

**Core classes** (verified against SDK v0.1.5):

`LocalAgentConfig` — agent configuration:
| Parameter | Type | Description |
|---|---|---|
| `model` | `str \| ModelTarget \| None` | Model ID, e.g. `"Gemini 3.1 Pro (High)"` |
| `system_instructions` | `str \| None` | System prompt for the agent |
| `workspaces` | `list[str] \| None` | Workspace directories (equivalent to CLI `--add-dir`, repeatable) |
| `capabilities` | `CapabilitiesConfig \| None` | Tool capabilities; omit for read-only |
| `tools` | `list[Callable] \| None` | Custom Python tool functions |
| `mcp_servers` | `list \| None` | MCP server configs (`McpStdioServer`, `McpStreamableHttpServer`) |
| `policies` | `Sequence[Policy] \| None` | Policy hooks (`deny`, `allow`, `ask_user`, `enforce`) |
| `hooks` | `list \| None` | Lifecycle hooks (`InspectHook`, `DecideHook`, `TransformHook`) |
| `triggers` | `list[Callable] \| None` | Background trigger functions (from `google.antigravity.triggers`) |
| `conversation_id` | `str \| None` | Resume a previous conversation by ID |
| `save_dir` | `str \| None` | Directory for persisting conversation state |
| `skills_paths` | `list[str] \| None` | Paths to skill files to load |
| `response_schema` | `dict \| None` | Structured output schema |
| `models` | `list[ModelTarget] \| None` | Multiple model configs |
| `api_key` | `str \| None` | Gemini API key override |
| `vertex` | `bool \| None` | Use Vertex AI instead of Gemini API |
| `project` | `str \| None` | GCP project ID (Vertex) |
| `location` | `str \| None` | GCP location (Vertex) |

`CapabilitiesConfig` — tool enablement (verified fields):
| Field | Type | Default | Description |
|---|---|---|---|
| `enable_subagents` | `bool` | `True` | Allow agent to spawn subagents |
| `enabled_tools` | `list[BuiltinTools] \| None` | `None` | Whitelist of tools |
| `disabled_tools` | `list[BuiltinTools] \| None` | `None` | Blacklist of tools |
| `compaction_threshold` | `int \| None` | `None` | Context compaction threshold |
| `finish_tool_schema_json` | `str \| None` | `None` | Schema for finish tool |

Without `CapabilitiesConfig()` the agent runs **read-only** (no write/edit/run tools).
Pass `CapabilitiesConfig()` to enable all standard write tools.

`Agent` — async context manager (verified methods):
```python
async with Agent(config) as agent:
    response = await agent.chat("task")     # returns ChatResponse
    agent.conversation_id                    # current conversation UUID
    agent.conversation                       # Conversation object
    agent.is_started                         # bool
```
Agent methods: `chat()`, `conversation` (property), `conversation_id` (property),
`is_started` (property). There is **no `interrupt()` method** — use
`agent.conversation.cancel()` to cancel a running task.
`run_interactive_loop(agent)` helper from `google.antigravity.utils.interactive`.

`Conversation` — persistent session (verified methods):
- `conversation.history` — list of all messages
- `conversation.turn_count` — number of completed turns
- `conversation.last_response` — last response object
- `conversation.conversation_id` — UUID for this conversation
- `conversation.chat("msg")` — convenience send + collect
- `conversation.send("msg")` — low-level send
- `conversation.receive_steps()` — receive response steps (async iterator)
- `conversation.receive_chunks()` — receive response chunks (async iterator)
- `conversation.cancel()` — cancel a running task
- `conversation.wait_for_idle()` — wait until agent is idle
- `conversation.wait_for_wakeup()` — wait for agent wakeup
- `conversation.is_idle` — bool property
- `conversation.clear_history()` — reset conversation
- `conversation.total_usage` — token usage stats
- `conversation.disconnect()` — close connection

**Streaming** (on the `ChatResponse` object from `agent.chat()`):
```python
async for token in response:            # text tokens (str)
async for thought in response.thoughts:  # reasoning deltas (str)
async for call in response.tool_calls:   # ToolCall events (name, args)
```
`ChatResponse` is defined in `google.antigravity.types`.

**Multimodal attachments**:
```python
from google.antigravity.types import from_file
attachment = from_file("screenshot.png")
response = await agent.chat(["Analyze this image", attachment])
```

**Triggers** (verified against SDK v0.1.5):
```python
from google.antigravity.triggers import every

async def check_status(ctx):
    await ctx.send("Check the deployment status.")

config = LocalAgentConfig(
    triggers=[every(60, check_status)],  # interval_seconds: float, callback: async fn
)
```
`every(interval_seconds, callback)` — `interval_seconds` is a **float** (seconds),
`callback` is an **async function** taking a `TriggerContext`.

**MCP servers** (verified against SDK v0.1.5):
```python
from google.antigravity.types import McpStdioServer, McpStreamableHttpServer

config = LocalAgentConfig(
    mcp_servers=[
        McpStdioServer(name="my_server", command="python", args=["my_mcp_server.py"]),
        McpStreamableHttpServer(name="remote", url="https://mcp.example.com/mcp"),
    ]
)
```
`McpStdioServer` requires `name` (str) and `command` (str); optional: `args`,
`env`, `timeout_seconds`, `enabled_tools`, `disabled_tools`.
`McpStreamableHttpServer` requires `name` (str) and `url` (str); optional:
`headers`, `timeout`, `sse_read_timeout`, `terminate_on_close`.
Note: there is **no `McpSseServer`** class — use `McpStreamableHttpServer` for
remote HTTP/SSE servers.

**Policies** (verified against SDK v0.1.5):
```python
from google.antigravity.hooks.policy import deny, allow, ask_user, enforce

async def my_handler(ctx):
    # custom approval logic
    return True  # or False to deny

config = LocalAgentConfig(
    policies=[
        deny("*"),                                    # block all by default
        allow("view_file"),                           # auto-approve specific tool
        ask_user("run_command", handler=my_handler),  # require confirmation (handler required)
        enforce("must run tests first"),              # enforce constraint
    ]
)
```
`ask_user(tool, *, handler, when=None, name="")` — `handler` is a **required**
keyword argument (async function returning bool). `tool` is a string or MCP
server config.

### CLI (Fallback — `agy -p`)

When the SDK is not installed, use `agy -p` for one-shot non-interactive
delegation. This is the verified CLI equivalent of `droid exec`.

**All `-p` flags**:

| Flag | Description | Default |
|---|---|---|
| `-p`, `--print`, `--prompt <text>` | Non-interactive one-shot prompt | (required) |
| `-i`, `--prompt-interactive <text>` | Interactive session with initial prompt | — |
| `--model <name>` | Model ID for the session | (from settings) |
| `--add-dir <path>` | Add directory to workspace (repeatable) | cwd |
| `--print-timeout <duration>` | Timeout for `-p` (e.g. `10m`, `300s`) | `5m` |
| `--sandbox` | Enable terminal sandbox | off |
| `--dangerously-skip-permissions` | Auto-approve all tool permissions | off |
| `-c`, `--continue` | Resume last session | — |
| `--conversation <uuid>` | Resume specific conversation by ID | — |
| `--new-project` | Create a new project | — |
| `--project <id>` | Use specific project ID | — |
| `--log-file <path>` | Custom log file path | default location |

**Verified working combinations** (tested against agy v1.0.16):
```bash
# Basic one-shot
agy -p "Say hello" --model "Gemini 3.1 Pro (High)"

# Workspace-scoped with sandbox
agy -p "Summarize this repo" --model "Gemini 3.1 Pro (High)" --add-dir "/path/to/project" --sandbox

# Long-running task with extended timeout
agy -p "Implement feature X" --model "Gemini 3.1 Pro (High)" --add-dir "/path/to/project" --print-timeout 15m

# Multiple workspaces
agy -p "Compare project A and B" --model "Gemini 3.1 Pro (High)" --add-dir "/path/A" --add-dir "/path/B"

# Resume last session
agy -c
agy --continue

# Resume specific conversation
agy --conversation <uuid>
```

**Model names** (from `agy models` output):
```
Gemini 3.5 Flash (Medium)
Gemini 3.5 Flash (High)
Gemini 3.5 Flash (Low)
Gemini 3.1 Pro (Low)
Gemini 3.1 Pro (High)          ← recommended default
Claude Sonnet 4.6 (Thinking)
Claude Opus 4.6 (Thinking)
GPT-OSS 120B (Medium)
```
Additionally, `"Gemini 3.5 Pro (High)"` is accepted by `--model` but **silently
falls back** to `Gemini 3.5 Flash (Medium)` — it does not run 3.5 Pro and does
not return an error. Any unrecognized model name produces the same silent
fallback. Always verify the model name with `agy models` before use.
Models with `(High)` already include maximum reasoning effort — no additional
flags needed.

---

## TUI Slash Commands

These commands are available inside the interactive `agy` TUI. They are NOT
available through `agy -p`; for non-interactive mode, encode the same intent
in the prompt text.

### High-Value Orchestration Commands

| Command | Use For | Orchestration Notes |
| --- | --- | --- |
| `/grill-me` | Aligning on complex plans or design choices through an interactive interview. | Use before architecture, ambiguous product behavior, API contracts, or high-cost implementation. Ask for requirements, tradeoffs, risks, and acceptance criteria. |
| `/goal` | Running a thorough, long-running agent task that persists until fully completed. | Use for implementation, repo audits, migrations, research, and validation. Include scope, constraints, tests, artifact expectations, and permission limits. |
| `/schedule` | Scheduling recurring background jobs or one-time timers. | Include timezone, recurrence or exact date/time, task text, output channel, stop condition, and safety limits. |
| `/teamwork-preview` | Orchestrating multiple autonomous subagents on large projects. | Split by independent roles, require evidence from each subagent, define conflict resolution, and ask for final synthesis. |
| `/learn` | Recording custom setups and preferences so AGY persists behavior in future sessions. | Use only for stable preferences or environment facts confirmed by the user. Include explicit scope and trigger. |

### All TUI Commands

| Command | Description |
| --- | --- |
| `/add-dir` | Add a directory path to the active project workspace. |
| `/agents` | List active subagents and custom agents. |
| `/artifact` | Open the TUI artifact viewer. |
| `/btw` | Ask a quick side question without starting a full agent run. |
| `/changelog` | Display product changelog and release notes. |
| `/clear` or `/new` | Clear the terminal screen and scrollback buffer. |
| `/config` or `/settings` | Open the TUI configuration panel. |
| `/context` | List files and symbols currently in the agent context. |
| `/copy` | Copy the last agent response to the system clipboard. |
| `/credits` | Display third-party licenses and attributions in external builds. |
| `/diff` | Display the current codebase diff made by the agent. |
| `/exit` or `/quit` | Exit the active CLI session. |
| `/fast` | Toggle fast mode for typing delays and thought visualization in external builds. |
| `/feedback` | Open product feedback and bug reporting. |
| `/fork` or `/branch` | Fork the current conversation into a new thread. |
| `/help` | Display help, commands, and keyboard shortcuts. |
| `/hooks` | List registered lifecycle hooks. |
| `/keybindings` | Configure active CLI keyboard shortcuts. |
| `/logout` | Log out of the active Google account session. |
| `/mcp` | List active MCP servers and exposed tools. |
| `/model` | Change the active Gemini model for the session. |
| `/open` | Open a file in the preferred local editor. |
| `/permissions` | Manage tool execution permissions. |
| `/planning` | Open graphical planning and task list editor in external builds. |
| `/rename` | Rename the active conversation thread. |
| `/resume`, `/switch`, or `/conversation` | Resume a past conversation by ID or name. |
| `/rewind` or `/undo` | Rewind conversation history to a previous checkpoint. |
| `/skills` | List active agent skills. |
| `/statusline` | Toggle the terminal status line. |
| `/tasks` | Display the active task list and progress. |
| `/title` | Toggle or configure the terminal title. |
| `/usage` or `/quota` | Display token usage and session cost statistics. |

### TUI → CLI Equivalents

| TUI Command | `agy -p` Equivalent |
|---|---|
| `/grill-me` | `agy -p "Interview me to align on <decision>. Focus on requirements, tradeoffs, risks, acceptance criteria. Do not edit files."` |
| `/goal` | `agy -p "Implement <task>. Stay within <scope>. Run <tests>. Report changes."` |
| `/schedule` | Not available via `-p`; use SDK `every()` trigger instead. |
| `/teamwork-preview` | `agy -p "Split <project> into independent roles. Reconciling into final plan."` |
| `/learn` | `agy -p "Remember for <scope>: <setup>. Use it when <trigger>."` |
| `/permissions` | Check `~/.gemini/antigravity-cli/settings.json` directly. |
| `/model` | Use `--model <name>` flag. |
| `/context` | Not available via `-p`; inspect workspace directly. |
| `/diff` | Not available via `-p`; run `git diff` after the run. |

---

## Configuration Settings

The CLI uses `~/.gemini/antigravity-cli/settings.json`.

| JSON Key | Type | Default | Notes |
| --- | --- | --- | --- |
| `allowNonWorkspaceAccess` | bool | `false` | Permits reading/writing outside the workspace root. Keep false unless explicitly needed. |
| `altScreenMode` | string | `default` | Alternate screen buffer usage: `default`, `always`, `never`. |
| `artifactReviewPolicy` | string | `asks-for-review` | `always-proceed`, `agent-decides`, `asks-for-review`. |
| `colorScheme` | string | `terminal` | `terminal`, `dark`, or `light`. |
| `editor` | string | `auto` | Preferred editor command. |
| `enableTelemetry` | bool | `true` | Anonymous usage/crash reporting. |
| `enableTerminalSandbox` | bool | `false` | Runs commands in a restricted sandbox. Prefer true for risky exploration. |
| `gcp` | object | `nil` | GCP project/location configuration. |
| `historySize` | int | `2000` | Persisted history entries; `-1` for unlimited. |
| `model` | string | `gemini-3.5-flash` | Active model identifier. |
| `notifications` | bool | `false` | System notifications on completion. |
| `permissions` | object | `nil` | Global allow/deny/ask rules for files, commands, URLs. |
| `runningLightSpeed` | string | `medium` | Thought visualization speed: `off`, `fast`, `medium`, `slow`. |
| `showFeedbackSurvey` | bool | `true` | Product feedback surveys. |
| `showTips` | bool | `true` | Tips and shortcuts. |
| `statusLine` | object | `nil` | TUI status line config. |
| `title` | object | `nil` | Terminal title config. |
| `toolPermission` | string | `request-review` | `always-proceed`, `request-review`, `strict`, `proceed-in-sandbox`. |
| `trustedWorkspaces` | array | `[]` | Workspace paths trusted for execution. |
| `useG1Credits` | bool | `false` | Use Google One AI premium quotas. |
| `verbosity` | string | `high` | Trace rendering detail: `high` or `low`. |

**Recommended safe defaults for orchestration**:
```json
{
  "model": "Gemini 3.1 Pro (High)",
  "toolPermission": "request-review",
  "allowNonWorkspaceAccess": false,
  "enableTerminalSandbox": true,
  "artifactReviewPolicy": "asks-for-review",
  "verbosity": "high"
}
```
Note: `settings.json` `model` key accepts display names (e.g. `"Gemini 3.1 Pro (High)"`)
as well as internal IDs (e.g. `"gemini-3.5-flash"`). The `--model` CLI flag and
SDK `model` param both accept display names from `agy models` output.

**Safety preflight** (TUI commands before file/command work):
```text
/permissions
/context
/diff
```
Confirm AGY is scoped to the intended workspace, respects deny/ask rules, and
will report approvals or permission blockers.

---

## Mandatory Safety Rules (R1-R7)

Apply these rules before asking AGY to operate on the user's behalf:

**R1 - Settings immutability**: Never modify `~/.gemini/antigravity-cli/settings.json`
without explicit user consent. If a settings change is needed, show the exact
intended keys or JSON diff first.

**R2 - Workspace trust**: Do not ask AGY to read, write, or run commands in
untrusted or unrelated directories. Prefer adding only the necessary workspace
with `--add-dir`, and keep scope explicit in the prompt.

**R3 - Permission compliance**: Check or ask AGY to respect current `/permissions`
before file, command, or URL operations. Deny rules win over allow rules.

**R4 - Non-workspace access**: Treat `allowNonWorkspaceAccess: false` as a hard
boundary. If the user requests outside-workspace work, explain the boundary and
ask before changing scope.

**R5 - Session control protection**: Do not invoke `/logout`, `/exit`, `/quit`,
`/rewind`, or destructive conversation/session actions unless the user explicitly
asks for them.

**R6 - Review transparency**: When AGY is under `request-review` or `strict`
tool permissions, require one command/action per approval and a clear statement
of expected effects.

**R7 - Sandbox integrity**: Do not silently toggle terminal sandboxing or
auto-execution. For risky commands, prefer sandboxed/reviewed execution and
report the safety posture.

---

## Safe Dirty-Work Delegation

Use AGY/Gemini for low-risk, reversible, mechanically checkable work while the
orchestrating agent remains the decision-maker.

### Safe to delegate with minimal pre-analysis

- Build/test/environment checks: `npm run build`, `npm test`, `pytest`, `cargo test`, `tsc`, lint/typecheck.
- Log and error summarization: group stack traces, repeated failures, likely root-cause candidates.
- Repo reconnaissance: relevant files, module summaries, call sites, existing patterns, TODO searches.
- Mechanical summaries: docs, changelogs, PR diffs, test output, benchmark output, session history.
- Drafts: release notes, test plans, migration checklists, issue triage, review checklists.

### Delegate only with tight constraints and post-review

- Small mechanical edits: typos, imports, comment/docstring cleanup, obvious tests.
- Bulk repetitive edits where the pattern is clear and the orchestrator can inspect the diff.
- Dependency/config suggestions.
- Refactor proposals.

### Do not blindly delegate

- Security, auth, permissions, crypto, billing/payment logic, migrations, data deletion, production deploys, or secrets.
- Ambiguous product/design decisions, frontend visual judgment, accessibility/UX polish, screenshots, or image understanding.
- Destructive filesystem operations, destructive git commands, pushes, releases, or remote infrastructure changes.
- Final bug diagnosis when evidence is incomplete.

### Example: read-only reconnaissance

```text
/goal Inspect the build/test output and summarize failures. Do not modify files.
Report commands run, failing tests, likely root causes, and recommended next steps.
```

For `-p` mode:
```bash
agy -p "Inspect build/test output. Summarize failures. Do NOT modify files. Report commands run, failing tests, likely root causes, recommended next steps." --model "Gemini 3.1 Pro (High)" --add-dir "<ws>" --sandbox
```

---

## Prompt Templates

### Complex alignment (TUI)
```text
/grill-me Interview me to align on <decision>. Focus on requirements, tradeoffs,
risks, acceptance criteria, and what evidence would change the decision. Do not
edit files.
```

### Complex alignment (`-p`)
```bash
agy -p "Interview me to align on <decision>. Ask clarifying questions. Focus on
requirements, tradeoffs, risks, acceptance criteria, and what evidence would
change the decision. Do NOT edit files. Output the final aligned specification."
--model "Gemini 3.1 Pro (High)" --add-dir "<ws>"
```

### Long-running implementation (TUI)
```text
/goal Implement <task> in <workspace>. Stay within <scope>. Before editing,
inspect existing patterns. After editing, run <tests>. Report files changed,
commands run, test results, and unresolved questions.
```

### Long-running implementation (`-p`)
```bash
agy -p "Implement <task> in <workspace>. Stay within <scope>. Before editing,
inspect existing patterns. After editing, run <tests>. Report files changed,
commands run, test results, and unresolved questions."
--model "Gemini 3.1 Pro (High)" --add-dir "<ws>" --print-timeout 15m
```

### Parallel work (TUI)
```text
/teamwork-preview Orchestrate subagents for <project>. Assign independent roles
for <area A>, <area B>, and <verification>. Require evidence from each subagent,
reconcile conflicts, and produce a final plan/diff summary.
```

### Scheduled job (TUI)
```text
/schedule Create a <recurring|one-time> task in timezone <tz>: <task>. Run at
<time/cron>. Report <format>. Stop or ask for review if <safety condition>.
```

### Durable memory (TUI)
```text
/learn Remember for <scope>: <stable preference or setup>. Use it when <trigger>.
Do not generalize beyond this scope.
```

### SDK plan/build pipeline
```python
# Phase 1 — Plan (read-only, no CapabilitiesConfig)
config = LocalAgentConfig(
    model="Gemini 3.1 Pro (High)",
    system_instructions="You are a planning assistant. Do not edit files.",
    workspaces=["/path/to/project"],
)
async with Agent(config) as agent:
    conv_id = agent.conversation_id  # save for Phase 2
    response = await agent.chat("Create a detailed plan for <task>. Do NOT edit files.")
    async for token in response:
        print(token, end="")

# Phase 2 — Build (with write capabilities, resume conversation)
config = LocalAgentConfig(
    model="Gemini 3.1 Pro (High)",
    system_instructions="Implement the plan. Edit files as needed.",
    workspaces=["/path/to/project"],
    capabilities=CapabilitiesConfig(),
    conversation_id=conv_id,  # resume from Phase 1
)
async with Agent(config) as agent:
    response = await agent.chat("Implement the plan from Phase 1. Run <tests> after.")
    async for token in response:
        print(token, end="")
```

### CLI plan/build pipeline
```bash
# Phase 1 — Plan
agy -p "Create a detailed plan for <task>. Do NOT edit files. Output the plan." \
  --model "Gemini 3.1 Pro (High)" --add-dir "<ws>"

# Phase 2 — Build
agy -p "Implement the plan from Phase 1. Edit files as needed. Run <tests> after." \
  --model "Gemini 3.1 Pro (High)" --add-dir "<ws>" --print-timeout 15m
```

---

## Visual Inputs

Antigravity CLI supports pasting rich media directly from the system clipboard
via `Ctrl+V` in the TUI.

For `agy -p` (non-interactive), images cannot be passed directly. The orchestrating
agent must:
1. Inspect the image using its own visual tools.
2. Extract relevant facts: visible text, layout, errors, colors/states, coordinates.
3. Pass the text description to AGY in the prompt.

Example:
```bash
agy -p "The UI has a red button at top-right that says 'Submit'. Change it to
blue and align it to the left. The error message below it reads 'Invalid input'."
--model "Gemini 3.1 Pro (High)" --add-dir "<ws>"
```

For SDK, use `from_file()` to attach images directly:
```python
from google.antigravity.types import from_file
attachment = from_file("screenshot.png")
response = await agent.chat(["Fix the button color as shown", attachment])
```

The orchestrating agent keeps responsibility for visual verification after AGY
suggests or makes changes.

---

## Verification Rule

After any AGY run that affects or informs real work, the orchestrating agent must
perform its own validation before reporting success. Validation may be:

- Diff review (`git diff`)
- Rerun commands / tests
- Log inspection
- Generated artifact check
- Screenshot or browser verification

Do not trust AGY output without independent verification.

---

## Recovery Protocol

### SDK Recovery

The SDK provides two recovery mechanisms (verified against SDK v0.1.5):

**1. In-process recovery** — `Conversation` preserves history within a single
process. If `agent.chat()` fails mid-stream, the conversation state is still
accessible:
```python
async with Agent(config) as agent:
    # Check state after a failure
    print(f"Completed {agent.conversation.turn_count} turns")
    print(f"History length: {len(agent.conversation.history)}")

    # Continue with a new prompt (same conversation context)
    response = await agent.chat("Continue from where you left off.")
    async for token in response:
        print(token, end="")
```

**2. Cross-process recovery** — pass `conversation_id` to resume a previous
conversation in a new process:
```python
# First run — save the conversation ID
async with Agent(config) as agent:
    conv_id = agent.conversation_id  # save this UUID
    response = await agent.chat("Start implementing feature X.")

# Later run — resume by conversation_id
resume_config = LocalAgentConfig(
    model="Gemini 3.1 Pro (High)",
    conversation_id=conv_id,  # resume previous conversation
    save_dir="/path/to/save",  # persist conversation state
)
async with Agent(resume_config) as agent:
    response = await agent.chat("Continue implementing feature X.")
```

**3. Cancel a running task**:
```python
# Cancel via conversation object
await agent.conversation.cancel()

# Wait for agent to become idle
await agent.conversation.wait_for_idle()
```

### CLI Foreground Runs

Use generous `--print-timeout` for long-running tasks:

- **Planning/investigation**: start with `10m`
- **Implementation**: start with `15m`, scale up for large tasks
- **Repository analysis**: `20m` or more for large codebases

```bash
agy -p "Implement feature X" --model "Gemini 3.1 Pro (High)" --add-dir "<ws>" --print-timeout 15m
```

### CLI Background Runs (PowerShell)

For long-running tasks where the orchestrating agent must not block:

```powershell
$task = "Implement feature X in the workspace"
$cwd = (Get-Location).ProviderPath
$stamp = Get-Date -Format "yyyyMMddHHmmssfff"
$out = Join-Path $env:TEMP "agy-run-$stamp.out"
$err = Join-Path $env:TEMP "agy-run-$stamp.err"
$manifest = Join-Path $env:TEMP "agy-run-$stamp.manifest.json"
$exe = (Get-Command agy).Source
$args = @(
    "-p", $task,
    "--model", "Gemini 3.1 Pro (High)",
    "--add-dir", $cwd,
    "--print-timeout", "15m"
)
$p = Start-Process -FilePath $exe -ArgumentList $args -WorkingDirectory $cwd `
    -RedirectStandardOutput $out -RedirectStandardError $err `
    -WindowStyle Hidden -PassThru
$run = [pscustomobject]@{
    Id        = $p.Id
    Out       = $out
    Err       = $err
    StartedAt = (Get-Date).ToString("o")
    Cwd       = $cwd
    Task      = $task
}
$run | ConvertTo-Json | Set-Content -LiteralPath $manifest -Encoding UTF8
$run
```

### CLI Background Runs (Bash)

```bash
TASK="Implement feature X in the workspace"
STAMP=$(date +%Y%m%d%H%M%S%3N)
OUT="/tmp/agy-run-$STAMP.out"
ERR="/tmp/agy-run-$STAMP.err"
MANIFEST="/tmp/agy-run-$STAMP.manifest.json"
nohup agy -p "$TASK" --model "Gemini 3.1 Pro (High)" --add-dir "$(pwd)" \
    --print-timeout 15m > "$OUT" 2> "$ERR" &
AGY_PID=$!
cat > "$MANIFEST" << EOF
{"pid":$AGY_PID,"out":"$OUT","err":"$ERR","startedAt":"$(date -Iseconds)","cwd":"$(pwd)","task":"$TASK"}
EOF
echo "AGY PID: $AGY_PID | Out: $OUT | Manifest: $MANIFEST"
```

### Polling Background Runs

```powershell
# Check if still running
$run = Get-Content $manifest | ConvertFrom-Json
$alive = Get-Process -Id $run.Id -ErrorAction SilentlyContinue
if (-not $alive) {
    Write-Host "AGY finished. Output:"
    Get-Content $run.Out
}
```

### CLI Recovery After Timeout or Crash

1. **Resume last session**:
   ```bash
   agy --continue
   agy -c
   ```

2. **Resume specific conversation**:
   ```bash
   agy --conversation <uuid>
   ```

3. **Inspect logs** at `~/.gemini/antigravity-cli/`:
   - Conversation JSON files contain full history
   - Log files contain runtime diagnostics
   - Check for the most recent conversation directory

4. **Conversation ID capture strategies**:
   - Parse stdout for conversation UUID (AGY prints it at session start)
   - Check `~/.gemini/antigravity-cli/conversations/` for session files sorted by date
   - In TUI: use `/resume` to browse and select past conversations
   - Save the UUID from the first run as part of the manifest

### Recovery Decision Tree

```
AGY run failed or timed out?
├─ Still running? → Wait or poll (check manifest.json for PID)
├─ SDK run? → Re-create Agent, use conversation.history to assess progress, continue
├─ CLI foreground run?
│   ├─ Timeout? → Re-run with larger --print-timeout using --continue
│   └─ Crash? → agy --continue or agy --conversation <uuid>
├─ CLI background run?
│   ├─ Check stdout/stderr from manifest.json paths
│   ├─ If partial output: agy --continue
│   └─ If no output: inspect ~/.gemini/antigravity-cli/ logs
└─ Output empty?
    ├─ Check stderr for errors
    ├─ Check ~/.gemini/antigravity-cli/ for session data
    └─ Re-run with --log-file <path> for diagnostics
```

### Workspace Configuration Files

AGY automatically reads workspace-level instruction files at startup:

- `GEMINI.md` or `AGENTS.md` at workspace root — codebase rules and conventions
- `~/.gemini/GEMINI.md` — global, cross-project rules

Create a `GEMINI.md` in the workspace for better AGY orchestration:
```markdown
# Project Conventions
- Use TypeScript strict mode
- Tests live next to source files as *.test.ts
- Run `npm test` before committing
- API endpoints follow RESTful conventions
```

---

## CLI Subcommands

These are standalone commands (not flags to `agy`) invoked as `agy <subcommand>`.

| Subcommand | Description |
|---|---|
| `agy models` | List available models with their identifiers |
| `agy plugin list` | List installed plugins |
| `agy plugin install <path>` | Install a plugin from a local path |
| `agy plugin uninstall <name>` | Remove an installed plugin |
| `agy plugin enable <name>` | Enable a disabled plugin |
| `agy plugin disable <name>` | Disable an enabled plugin |
| `agy changelog` | Display product changelog and release notes |
| `agy update` | Update the CLI to the latest version |
| `agy install` | Configure PATH and shell integration |
| `agy help` | Display help and usage information |

### `agy models` — Example Output

```
$ agy models
Gemini 3.5 Flash (Medium)
Gemini 3.5 Flash (High)
Gemini 3.5 Flash (Low)
Gemini 3.1 Pro (Low)
Gemini 3.1 Pro (High)
Claude Sonnet 4.6 (Thinking)
Claude Opus 4.6 (Thinking)
GPT-OSS 120B (Medium)
```

Use the exact model name from this output with `--model <name>`.

### Plugin Management

| Command | Description |
|---|---|
| `agy plugin list` | Show all installed plugins with their status (enabled/disabled) |
| `agy plugin install <path>` | Install a plugin from a local directory path |
| `agy plugin uninstall <name>` | Remove a plugin by name |
| `agy plugin enable <name>` | Enable a previously disabled plugin |
| `agy plugin disable <name>` | Disable a plugin without uninstalling |

Plugin directory structure (for plugin authors):
```
my-plugin/
├── plugin.json          # plugin manifest (name, version, description)
├── skills/              # agent skill definitions
├── agents/              # custom agent configurations
├── rules/               # policy rules and hooks
└── mcp_config.json      # MCP server configuration
```

---

## Antigravity 2.0 App Notes

Antigravity 2.0 is a desktop Electron app for launching and monitoring agents
outside an IDE.

Useful surfaces:

- **Projects**: manage and switch repositories/workspaces.
- **Scheduled Tasks**: define recurring jobs and one-time timers.
- **Skills & Customizations**: view skills, rules, plugins, and MCP servers.
- **Chat Canvas**: direct prompts, slash commands, @ mentions, and media uploads.
- **Settings**: model, permissions, sandboxing, non-workspace access, internet access, artifact review, notifications, command/browser allowlists.

Project-level settings can override global file access, internet access, sandbox
mode, auto-execution policy, artifact review, and permission grants.

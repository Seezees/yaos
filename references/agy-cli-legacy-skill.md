---
name: agy-cli
description: Orchestration and reference skill for the Antigravity CLI (`agy`). Covers TUI navigation, keyboard shortcuts, all CLI-only slash commands, settings.json configuration, auth flow, session management, and safe delegation rules. Activate when the user asks about agy, the Antigravity CLI, CLI slash commands, agy configuration, or troubleshooting TUI sessions. Do NOT activate for Antigravity IDE, Antigravity 2.0 Desktop, or Python SDK questions — route those to the antigravity-guide skill instead.
---

# AGY CLI — Codex & Orchestration Skill

This skill provides a complete operational reference for the **Antigravity CLI
(`agy`)**. It is designed for two roles:

1.  **Codex (reference)**: Look up any slash command, settings key, or keyboard
    shortcut.
2.  **Orchestration (agent behavior)**: Define safe defaults, delegation rules,
    and guardrails so an agent can operate the CLI on behalf of a user without
    violating security or workspace boundaries.

> [!NOTE]
> Canonical user documentation is maintained at
> `C:\Users\dmtrv\Desktop\аги докс\references\cli.md`. This skill distills and
> operationalises that reference for agent use. For IDE, Desktop App, or SDK
> questions, activate the `antigravity-guide` skill instead.

---

## 1. Trigger Rules

Activate this skill when the user's prompt matches **any** of:

| Trigger Pattern                                          | Example Query                                       |
| :------------------------------------------------------- | :-------------------------------------------------- |
| Mentions `agy` or "Antigravity CLI"                      | "How do I start agy?"                               |
| Asks about a CLI-only slash command                      | "What does /add-dir do?"                            |
| Asks about TUI navigation or keyboard shortcuts          | "How do I exit the agy TUI?"                        |
| Needs help with `settings.json` configuration            | "Change the default model in agy"                   |
| Asks about AGY CLI authentication or session management  | "How do I log out of agy?"                          |
| Wants to compare CLI with other Antigravity surfaces     | "CLI vs IDE — which should I use?"                  |
| Troubleshoots a TUI session, config, or permission issue | "agy says permission denied for a shell command"    |

**Negative triggers** (do NOT activate):
- Questions about the Antigravity IDE (inline edits, sidebar chat)
- Questions about Antigravity 2.0 Desktop (Electron app, left sidebar)
- Questions about the Python SDK (`google-antigravity` package)

Route those to the `antigravity-guide` skill.

---

## 2. Workflow Defaults

When acting as an orchestration agent for the AGY CLI, always apply these
defaults **unless the user explicitly overrides them**:

| Concern                | Default                                | Source / Rationale                                |
| :--------------------- | :------------------------------------- | :------------------------------------------------ |
| **Primary reference**  | `references/cli.md` (user docs)        | Authoritative offline reference                   |
| **Fallback live docs** | `https://antigravity.google/docs`      | Use when offline reference is stale or incomplete |
| **Default model**      | `gemini-3.5-flash`                     | Respect `settings.json` → `model` key             |
| **Tool permission**    | `request-review`                       | Safe-by-default; user must approve each tool call |
| **Workspace access**   | `allowNonWorkspaceAccess: false`       | Agent confined to workspace root                  |
| **Terminal sandbox**   | `enableTerminalSandbox: false`         | Default; toggle only on explicit user request     |
| **Alt screen**         | `altScreenMode: default`               | Let the terminal decide                           |
| **Artifact review**    | `artifactReviewPolicy: asks-for-review`| Agent pauses before modifying artifacts           |
| **Telemetry**          | `enableTelemetry: true`                | Respect user privacy; disclose if asked           |
| **Session start**      | `agy` → `Enter` → OAuth → paste code  | First-run auth flow                               |
| **Session end**        | `Ctrl+D Ctrl+D` or `/exit`             | Graceful shutdown                                 |

---

## 3. Safe Delegation Rules

When the agent operates the AGY CLI on a user's behalf, the following rules are
**mandatory**. Violating any of them is a safety incident.

### R1 — Settings Immutability
> **Never** modify `~/.gemini/antigravity-cli/settings.json` without explicit,
> confirmed user consent. If a change is proposed, surface the exact JSON diff
> and wait for approval.

### R2 — Workspace Trust
> **Never** execute agent commands (`/add-dir`, file writes, shell commands) in
> a directory that is not listed in `trustedWorkspaces`. If the user asks to
> work in an untrusted directory, warn them and offer to add it to the trusted
> list first.

### R3 — Permission List Compliance
> **Always** check `permissions.allow`, `permissions.deny`, and
> `permissions.ask` before executing any file, command, or URL operation. Deny
> rules take precedence over allow rules.

### R4 — Non-Workspace Access Warning
> When `allowNonWorkspaceAccess` is `false` (the default), **never** attempt to
> read or write files outside the active workspace. If the user requests an
> out-of-workspace operation, explain that this setting blocks it and ask
> whether they want to toggle it.

### R5 — Session Control Protection
> **Never** invoke `/logout` or `/exit` unless the user explicitly requests it.
> These are destructive session actions.

### R6 — Strict Mode Transparency
> When `toolPermission` is `strict` or `request-review`, **always** surface the
> full command string and its expected effects before requesting approval. Do
> not batch multiple commands into a single approval request — one command, one
> approval.

### R7 — Sandbox Integrity
> Treat `enableTerminalSandbox: false` as a conscious, user-chosen security
> posture. **Never** silently toggle it. If the user is about to run a risky
> command without the sandbox, offer to re-enable it for that operation.

---

## 4. CLI Slash Command Inventory

All commands below are **CLI-only** (not available in the IDE or Desktop App).
Commands are grouped by functional category.

### 4.1 Session Management

| Command                     | Description                                              |
| :-------------------------- | :------------------------------------------------------- |
| `/exit` or `/quit`          | Exit the active AGY CLI session.                         |
| `/clear` or `/new`          | Clear the terminal screen and scrollback buffer.         |
| `/rename <name>`            | Rename the active conversation thread.                   |
| `/resume`, `/switch`, or `/conversation` | Resume a past conversation by ID or name.   |
| `/fork` or `/branch`        | Fork current conversation into a new thread, preserving history. |
| `/rewind` or `/undo`        | Rewind conversation history to a previous checkpoint.    |
| `/logout`                   | Log out of the active Google account session.            |

### 4.2 Workspace & Context

| Command        | Description                                                |
| :------------- | :--------------------------------------------------------- |
| `/add-dir`     | Add a directory path to the active project workspace.     |
| `/context`     | List all files and symbols currently in the agent context. |
| `/diff`        | Display the codebase diff of changes made by the agent.    |
| `/open`        | Open a file in the preferred local editor.                 |
| `/copy`        | Copy the last agent response to the system clipboard.      |

### 4.3 Configuration & Preferences

| Command                      | Description                                              |
| :--------------------------- | :------------------------------------------------------- |
| `/config` or `/settings`     | Open the TUI configuration panel.                        |
| `/model`                     | Change the active Gemini model for the current session.  |
| `/permissions`               | Manage tool execution permissions (allow/deny lists).    |
| `/keybindings`               | Configure active CLI keyboard shortcuts.                 |
| `/statusline`                | Toggle the terminal status line.                         |
| `/title`                     | Toggle or configure the terminal title.                  |

### 4.4 Agent & Tooling

| Command        | Description                                                   |
| :------------- | :------------------------------------------------------------ |
| `/agents`      | List all active subagents and custom agents.                  |
| `/skills`      | List all active agent skills.                                 |
| `/hooks`       | List all registered lifecycle hooks.                          |
| `/mcp`         | List active MCP servers and their exposed tools.              |
| `/tasks`       | Display the active task list and progress.                    |
| `/planning`    | Open the interactive graphical planning and task list editor (external builds only). |
| `/btw`         | Ask a quick side-question without starting a full agent run.  |

### 4.5 Information & Help

| Command                     | Description                                               |
| :-------------------------- | :-------------------------------------------------------- |
| `/help`                     | Display the help menu with commands and keyboard shortcuts. |
| `/usage` or `/quota`        | Display token usage and session cost statistics.          |
| `/changelog`                | Display the product changelog and release notes.          |
| `/credits`                  | Display third-party software licenses (external builds).  |
| `/feedback`                 | Open the product feedback and bug reporting form.         |

### 4.6 Display & Utilities

| Command        | Description                                                   |
| :------------- | :------------------------------------------------------------ |
| `/artifact`    | Open the TUI artifact viewer.                                 |
| `/fast`        | Toggle Fast Mode for typing delays and thought visualization (external builds only). |

---

## 5. Keyboard Shortcuts

| Shortcut              | Action                                  |
| :-------------------- | :-------------------------------------- |
| `ESC`                 | Close active menus (`/skills`, `/settings`, etc.) |
| `Ctrl+D Ctrl+D`       | Exit the CLI session                    |
| `Up Arrow` / `Down Arrow` | Navigate command history            |

---

## 6. Configuration Reference (`settings.json`)

File location: **`~/.gemini/antigravity-cli/settings.json`**

| JSON Key                   | Type   | Description                                                                 | Default              |
| :------------------------- | :----- | :-------------------------------------------------------------------------- | :------------------- |
| `allowNonWorkspaceAccess`  | bool   | Permit agent to read/write files outside workspace root.                    | `false`              |
| `altScreenMode`            | string | Alternate screen buffer usage (`default`, `always`, `never`).               | `"default"`          |
| `artifactReviewPolicy`     | string | When agent asks for artifact review (`always-proceed`, `agent-decides`, `asks-for-review`). | `"asks-for-review"` |
| `colorScheme`              | string | CLI color scheme (`terminal`, `dark`, `light`).                             | `"terminal"`         |
| `editor`                   | string | Preferred editor command (`vim`, `nano`, `auto`).                           | `"auto"`             |
| `enableTelemetry`          | bool   | Toggle anonymous usage and crash reporting.                                 | `true`               |
| `enableTerminalSandbox`    | bool   | Run terminal commands inside a restricted sandbox.                          | `false`              |
| `gcp`                      | object | GCP project and location configurations for cloud-based tools.              | `null`               |
| `historySize`              | int    | Max history entries persisted to disk (`-1` for unlimited).                 | `2000`               |
| `model`                    | string | Active model identifier used by the main agent.                             | `"gemini-3.5-flash"` |
| `notifications`            | bool   | Enable system notifications on task completion.                             | `false`              |
| `permissions`              | object | Global allow/deny/ask rules for files, commands, and URLs.                  | `null`               |
| `runningLightSpeed`        | string | Artificial typing delays (`off`, `fast`, `medium`, `slow`).                 | `"medium"`           |
| `showFeedbackSurvey`       | bool   | Display periodic product feedback surveys.                                  | `true`               |
| `showTips`                 | bool   | Display helpful tips and shortcuts in the CLI.                              | `true`               |
| `statusLine`               | object | Configuration for the TUI status line.                                      | `null`               |
| `title`                    | object | Configuration for the TUI title.                                            | `null`               |
| `toolPermission`           | string | Confirmation mode for tools (`always-proceed`, `request-review`, `strict`, `proceed-in-sandbox`). | `"request-review"` |
| `trustedWorkspaces`        | array  | List of directory paths trusted for execution.                              | `[]`                 |
| `useG1Credits`             | bool   | Toggle usage of Google One AI premium quotas.                               | `false`              |
| `verbosity`                | string | Detail level of agent trace rendering (`high`, `low`).                      | `"high"`             |

---

## 7. Authentication Flow

For first-time users or session recovery:

1.  Run `agy` in the terminal.
2.  Press `Enter` to launch the browser OAuth flow.
3.  Authenticate with the Google account in the browser.
4.  Copy the authorization code displayed in the browser.
5.  Paste the code back into the terminal and press `Enter`.
6.  The session is now authenticated. Settings and history persist across
    sessions.

To log out: use `/logout`.

---

## 8. Validation Checklist

Before deploying or using this skill, verify:

- [ ] YAML frontmatter is valid (`name: agy-cli`, `description:` present)
- [ ] All 30 slash commands from the CLI reference are inventoried in Section 4
- [ ] All 21 settings.json keys are documented in Section 6 with correct types and defaults
- [ ] All 7 safe delegation rules (R1–R7) are present and unambiguous
- [ ] Keyboard shortcuts (Section 5) match the CLI reference
- [ ] Authentication flow (Section 7) is complete and accurate
- [ ] Trigger rules (Section 1) correctly distinguish CLI from IDE/Desktop/SDK
- [ ] Negative triggers route to the correct sibling skill (`antigravity-guide`)
- [ ] Workflow defaults (Section 2) are conservative and safe-by-default
- [ ] Cross-reference note points to the canonical docs location

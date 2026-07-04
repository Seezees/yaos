# Droid Skill Lessons for AGY

This note captures practical lessons from the Droid orchestration skill review so
the AGY skill can be improved later without re-learning the same failure modes.

## Keep the Entry Point Slim

- Use `SKILL.md` as the operational protocol, not as the full manual.
- Put detailed command notes, settings keys, app/IDE/SDK material, and long
  examples in `references/`.
- Make reference routing explicit: say when to stay in the slim guide and when
  to read each reference file.
- Avoid duplicating the same facts in both `SKILL.md` and references.

## Preserve Output and Recovery Paths

Droid became more reliable once the skill treated lost stdout as recoverable
instead of fatal. AGY needs the same mindset, adjusted for its actual interface.

- Do not assume an interactive CLI can behave like a headless exec API unless
  the docs prove it.
- If AGY remains TUI-first, make Codex prepare prompts and require the user or
  visible terminal session to preserve artifacts.
- For any future AGY non-interactive mode, document how to capture progress,
  session IDs, logs, artifacts, and resume handles before using it for real work.
- Treat saved chat/artifacts as the source of truth when shell output is lost.

## Be Explicit About Timeouts

The Droid issue was not only "the model is slow"; the orchestration layer hid
or killed output before Codex recovered it.

- Give long-running agent tasks generous timeouts.
- Prefer background launches plus polling when output must not be lost.
- Record enough metadata to recover a run: working directory, prompt file,
  stdout/stderr paths, process ID, session ID, and start time.
- Do not treat silence as failure until recovery paths are checked.

## Encode Default Commands and Safety

Droid improved after the skill stated exact defaults. AGY should do the same.

- State the safe default permission posture.
- Name the intended slash command for each workflow:
  `/grill-me`, `/goal`, `/schedule`, `/teamwork-preview`, `/learn`.
- Explain what Codex should delegate and what Codex must verify itself.
- Keep risky areas out of blind delegation: auth, secrets, billing, destructive
  commands, production deploys, and final judgment on ambiguous evidence.

## Match AGY's Real Shape

Do not copy Droid mechanics blindly.

- Droid has `exec`, `stream-json`, session search, and JSONL recovery.
- AGY docs currently describe an interactive Antigravity CLI/TUI, not a Droid
  style `exec -o stream-json` interface.
- Therefore AGY orchestration should focus on precise slash-command prompts,
  permission review, artifact inspection, and user-visible handoff unless a
  verified headless mode appears.

## Add These Improvements Later

- Add a `Protocol revision` line to AGY `SKILL.md`.
- Add a tighter `Reference Routing` table for `agy-cli-reference.md`, `cli.md`,
  `app.md`, `ide.md`, and `sdk.md`.
- Add Russian trigger phrases to the AGY frontmatter description.
- Add explicit recovery instructions for AGY artifacts, resumed tasks, and
  saved chat history.
- Add a small "Do not invent" rule for unverified AGY flags and headless modes.
- Keep the old `agy-cli` monolithic skill only as a legacy reference while the
  active skill remains slim.

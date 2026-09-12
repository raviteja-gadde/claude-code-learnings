# Claude Code Harness Playbook

Operational patterns for Claude Code, learned from daily use across a multi-repo engineering workspace. The primary use case: Claude Code on a different machine ingests this document and produces a gap analysis of that machine's setup.

Everything here comes from running into walls, reading docs, experimenting, and talking to other users. It covers the decisions behind the configuration and the traps that cost real time to discover.

---

## 1. Architecture: What Goes Where

### The forcing function: context cost

Every line in `CLAUDE.md`, every rule file, every line of `MEMORY.md` loads into the context window at session start and stays there for the session's life. Typical config/prose runs 8–12 tokens per line, so a 200-line CLAUDE.md costs 1,600–2,400 tokens every session. That compounds: global CLAUDE.md + project CLAUDE.md + rules + memory index + SessionStart hook output can consume 5–10K tokens before the user types anything.

The goal is to minimize this baseline while ensuring nothing critical is missing.

**Decision tree:**
- Will every session need this? → `CLAUDE.md` or rules
- Will most sessions in this project need it? → Project `CLAUDE.md`
- Is it a behavioral correction that persists across sessions? → Memory
- Is it a workflow triggered by name? → Skill or command (loaded on demand)
- Is it a multi-step pipeline? → Command that spawns agents

### File map

```
~/.claude/
├── settings.json            global: model, permissions, sandbox, hooks, status line
├── CLAUDE.md                behavior preferences for ALL projects (always loaded)
├── rules/*.md               always loaded (path-scoped with `paths` frontmatter)
├── skills/<name>/SKILL.md   loaded on-demand when invoked or judged relevant
├── hooks/*.sh               scripts called by hooks in settings.json
├── templates/               reusable file templates (zero context cost until used)
├── statusline.sh            status line script
└── projects/<path>/memory/
    ├── MEMORY.md             index (roughly first 200 lines loaded every session)
    └── <topic>.md            loaded on demand when relevant

~/.claude.json               MCP server registrations (per-project scope keys)

<project>/
├── .mcp.json                project-scoped MCP servers (committed, shared with team)
└── .claude/
    ├── settings.json            project hooks (SessionStart, PostToolUse)
    ├── settings.local.json      project-private permissions (gitignored, not committed)
    ├── agents/<name>.md         project-scoped subagents
    ├── commands/<name>.md       project-scoped slash commands
    └── rules/<name>.md          always-loaded project rules (path-scoped with `paths` frontmatter)
```

### Load behavior summary

- **Always loaded:** CLAUDE.md (global + project), rules, MEMORY.md index. These define the session's baseline context cost.
- **On demand:** Skills, commands, agents, individual memory files. Cost nothing until invoked.
- **Event-triggered:** Hooks run scripts in response to tool use or session events. SessionStart and UserPromptSubmit stdout becomes conversation context; other hook types do not by default.

**The trap:** Putting workflow instructions in `CLAUDE.md` or rules. They load every session and eat context even when irrelevant. If something isn't needed every session, make it a skill or command. The corollary: if something MUST be enforced every session (formatting rules, visibility boundaries), it goes in rules. Skills can be ignored by the model.

---

## 2. Model and Context Management

### Model configuration

The documented model selection mechanism uses the `model` field in `settings.json`:

```json
{
  "model": "opus[1m]",
  "alwaysThinkingEnabled": true
}
```

**The `[Nm]` suffix** selects the extended context window variant. `[1m]` = the 1-million-token context window (available on Opus 4.6+ and Sonnet 5). Cost per token is higher above 200K; use it for large-codebase sessions, not by default.

**Custom model picker entries** can be populated via three env vars in `settings.json`. This mechanism is undocumented as of mid-2026 and may change:

```json
{
  "env": {
    "ANTHROPIC_CUSTOM_MODEL_OPTION": "claude-opus-4-6",
    "ANTHROPIC_CUSTOM_MODEL_OPTION_NAME": "Opus 4.6",
    "ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION": "Opus 4.6, most capable for complex work"
  }
}
```

**Observed fallback behavior:** In mid-2026 builds, a `model` value Claude Code doesn't recognize has been observed to fall back to Sonnet without warning. The mitigation is the status line: always display the active model name so you can verify which model is actually running.

**`alwaysThinkingEnabled`** forces extended thinking on every turn, even simple ones. Worth the token cost for complex work (architecture, debugging, multi-file reasoning) but adds overhead to trivial questions.

**`model: opus` in agent frontmatter** is an alias that resolves to whichever Opus version that CLI build maps it to, usually but not necessarily the latest. Pin a specific model ID for stability; use the alias to stay current.

### Prompt cache and context economics

Claude Code caches the system prompt (CLAUDE.md, rules, memory index, etc.) across turns within a session. The first turn pays the full input cost; subsequent turns pay only for new content. Things that invalidate this cache:

1. **Idle time.** The cache TTL is 5 minutes by default (1 hour on some plans). Walking away for lunch re-pays the full prefix. Most common cache killer.
2. **`/clear`.** Resets conversation history. The system prompt prefix is unchanged, so API-level caching may still reduce the cost, but the session-level cache is rebuilt.
3. **Model switching** mid-session. The cache is model-specific.
4. **Tool set changes.** An MCP server connecting late or a plugin enabling mid-session shifts the prefix and invalidates the cache.

### Context window strategy

Managing the context window is an operational skill, not a configuration setting.

**When to `/clear`:** Starting a genuinely unrelated task. Shedding irrelevant conversation history is worth the prefix re-ingestion.

**When to `/compact`:** Deep in one problem, context filling up, but you need the conversation thread. The system prompt stays cached; earlier turns get summarized. Use `/compact <instructions>` for focused compaction, e.g. `/compact keep the failing test output and the diff`, to control what survives.

**When to fork:** The primary reason to fork isn't speed. It's keeping intermediate tool output (file reads, search results, git logs) out of your main context window. A fork inherits your full conversation and shares the prompt cache, so it's cheap. The fork's final report is a short summary that enters your context; the hundreds of lines of tool noise stay in the fork's thread.

**When to use a fresh agent:** When you need independent judgment without the current context's biases. Verification and review agents must be fresh. If they inherit the builder's reasoning, they'll confirm rather than challenge. A fresh agent starts with zero context, so its prompt must be self-contained.

**Practical heuristic:** Watch the context percentage in the status line (also available via `/context`). Above 50%, start thinking about what to offload. Above 75%, fork aggressively or `/compact`. Near the limit, autocompact fires on its own, but proactive management produces better results than waiting for automatic compression.

---

## 3. Permission Model: Auto + Sandbox

The permission configuration that eliminated nearly all prompts while maintaining safety:

```json
{
  "permissions": {
    "defaultMode": "auto",
    "deny": [
      "Bash(rm -rf /:*)", "Bash(rm -rf ~:*)", "Bash(sudo rm:*)",
      "Bash(git push --force:*)", "Bash(mkfs:*)", "Bash(dd:*)",
      "Read(**/.env)", "Read(**/.env.*)", "Read(**/*.key)",
      "Read(**/*.pem)", "Read(**/*secret*)", "Read(**/*credential*)",
      "Read(**/.aws/credentials)", "Read(**/.ssh/*)",
      "Read(**/*.p12)", "Read(**/*.pfx)", "Read(**/*_rsa)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "allowWrite": ["~/.claude"]
    }
  }
}
```

*Place this in `~/.claude/settings.json` (user-level; `defaultMode` has no effect in project settings). The deny list shown is abbreviated. A production list covers 40+ patterns including API keys (`*apikey*`, `*api_key*`), container credentials (`.docker/config.json`, `kubeconfig`), password files (`*password*`, `*passwd*`), and package registry tokens (`.npmrc`, `.pypirc`).*

**Why `auto` over other modes:** `default` mode (prompts before every non-read action) and explicit allow-lists (partial automation) are noisier without being safer when sandbox + deny are already in place. `auto` mode uses a classifier that approves routine actions and blocks risky ones. Combined with sandbox, normal dev work flows without prompts.

**The real safety layer is `permissions.deny`, not CLAUDE.md.** CLAUDE.md instructions are advisory. The model can be steered past them. Deny rules cannot be overridden, even under prompt injection, as long as the settings files themselves are write-protected. The sandbox's `denyWrite` on `settings.json` paths closes this loop.

**Project-level permissions** go in `.claude/settings.local.json` (gitignored) for machine-specific allows that shouldn't be committed:

```json
{
  "permissions": {
    "allow": [
      "Bash(git --version)",
      "Read(/Users/yourname/.claude/**)"
    ]
  }
}
```

---

## 4. Session Context Injection (SessionStart Hook)

Every session opens with orientation, no manual briefing needed.

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "cd \"$CLAUDE_PROJECT_DIR\" && printf '## Active work\\n' && cat journal/current.md && printf '\\n## Recent commits\\n' && git log -5 --format='%ad  %s' --date=short 2>/dev/null; exit 0"
      }]
    }]
  }
}
```

*Place this in `.claude/settings.json` (project-level). This is a simplified example. A production version adds available-commands reminders, last-checkin date, and completed-item nudges.*

**What to inject:**
- Active work items (so the model knows what's in flight)
- Recent git commits (so the model knows what just shipped)
- Available commands (the ones you won't remember)
- Actionable nudges (completed items not yet swept, stale journal)

**What NOT to inject:** Everything in SessionStart stdout enters the context window permanently. Injecting 500 lines of "helpful context" means every turn in the session carries that weight. Target under 50 lines of high-signal orientation. If something is reference material, put it in a file and let the model read it when needed.

**Design principle:** SessionStart and UserPromptSubmit are the two hook types whose stdout becomes conversation context by default. SessionStart runs once at session open; UserPromptSubmit runs before each user message is processed. Use SessionStart for orientation, UserPromptSubmit for per-message guards or context refresh.

---

## 5. Writing Effective CLAUDE.md

CLAUDE.md is the easiest config file to get wrong because there's no feedback when it's too long or too vague.

**Don't repeat the system prompt.** Claude Code's built-in system prompt already covers git safety, file handling, and many best practices. Restating them in CLAUDE.md wastes context and can conflict if the system prompt changes.

**Don't restate skills or commands.** The `SKILL.md` is the source of truth for a skill's behavior. Describing the same skill in CLAUDE.md creates a copy that drifts within weeks. Reference skills ("use `/learn` for new concepts"), don't redefine them. Use `@path` imports in CLAUDE.md to reference files without duplicating their content.

**Keep it concise.** Every line pays context cost every session. If you're past 100 lines, audit what should be a rule (path-scoped, always loaded) vs. a skill (loaded on demand).

---

## 6. Hooks and Automation

### How hooks work

Hooks are shell commands triggered by Claude Code events. They receive a JSON payload on stdin with context about the event.

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "export PATH=\"$HOME/.local/bin:$PATH\"; f=$(jq -r '.tool_input.file_path // empty'); [ -n \"$f\" ] && [[ \"$f\" == *.py ]] && ruff check --no-fix \"$f\" 2>&1; exit 0"
      }]
    }]
  }
}
```

*A PostToolUse lint hook that runs `ruff check` on every Python file edit. It reads the file path from JSON on stdin via `jq`. There is no environment variable for it. The `exit 0` ensures the hook always succeeds; PostToolUse hooks cannot block execution. The PATH export is needed because hook scripts inherit a minimal environment.*

### Hook types and their powers

This list covers the most commonly used hook types; additional types may exist in newer builds:

- **SessionStart** runs once at session open. Stdout becomes conversation context. Use for orientation injection.
- **UserPromptSubmit** runs before each user message. Stdout becomes context. Can block (exit 2).
- **PreToolUse** runs before a tool executes. Can block (exit 2). Use for safety gates.
- **PostToolUse** runs after a tool executes. Cannot block the action (it already ran), but can provide feedback to the model via JSON `additionalContext` output.
- **Stop** fires each time Claude finishes a response turn (not session end). Can block (exit 2). Use for turn-level gates like "did you run the tests before stopping."
- **SessionEnd** runs when the session ends. Use for cleanup.

### Pre-commit secret gate

A two-layer scan installed as a pre-commit hook in every repo with a remote:

1. **Path-based:** Blocks staging of `.env*`, `.pem`, `.key`, `credentials.*`, `token*`, and similar patterns
2. **Content-based:** Runs `gitleaks protect --staged` (if installed) or a fallback regex scan for common secret patterns: API keys, PATs, private key blocks

This is the last gate before secrets reach a remote. It catches things the sandbox and deny rules miss because those operate at the Claude Code level, not the git level.

### Hook gotchas

- **Input is JSON on stdin, not environment variables.** The edited file path is `.tool_input.file_path` in the JSON. There is no `$CLAUDE_FILE_PATH` env var. A script that reads one silently gets an empty string and does nothing.
- **Minimal PATH.** Hook scripts don't inherit your shell profile. Tools in `~/.local/bin` (from `uv tool install` or `pipx`) need an explicit PATH export inside the script.
- **Exit codes matter only for blocking hooks.** PreToolUse, UserPromptSubmit, and Stop can block on exit 2. PostToolUse cannot block since the tool already ran.

---

## 7. Status Line

The status line provides continuous ambient awareness without consuming conversation context.

### Setup

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  }
}
```

The script receives a JSON object on stdin. Key paths (snake_case, nested):

- `model.display_name` (active model)
- `effort.level` (reasoning effort)
- `fast_mode` (boolean)
- `context_window.used_percentage`, `context_window.context_window_size`
- `rate_limits.five_hour.used_percentage`, `rate_limits.five_hour.resets_at`
- `rate_limits.seven_day.used_percentage`
- `workspace.project_dir`, `workspace.current_dir`
- `worktree.name` (active worktree, if any)
- `pr.number`, `pr.review_state` (associated PR)
- `cost.total_duration_ms` (session duration)

The schema is not fully documented. Dump stdin once on first setup and inspect. The script runs frequently, so keep it fast: do all JSON parsing in a single `jq` call.

### What to display (prioritized)

1. **Context window percentage.** The one number you always need. Color-code it: green under 50%, yellow 50–75%, red above 75%.
2. **Rate limit proximity.** Show 5-hour and 7-day usage percentages with reset countdown when usage is high.
3. **Model name.** Confirms you're on the right model (catches fallback).
4. **Branch + dirty state.** Orientation without running `git status`.

Context window percentage and rate limit visibility prevent the two most common session-killers: running out of context mid-task and hitting rate limits without warning. The status line makes both visible before they hit.

---

## 8. Commands and Pipeline Architecture

### How commands work

A command is a markdown file at `.claude/commands/<name>.md`. When the user types `/<name> some args`, Claude reads the file and `$ARGUMENTS` is replaced with the user's input. The file contains natural-language instructions: phase descriptions, agent spawn directives, gate conditions, output format requirements.

```markdown
---
description: Research a topic and produce a verified knowledge note
---

Research `$ARGUMENTS` and produce a verified knowledge note.

## Phase 1: Intake
Parse the topic. Check knowledge/_index.md for existing coverage...

## Phase 2: Research (parallel forks)
Launch 3 forks:
- Fork 1: Primary sources (official docs, specs, RFCs)
- Fork 2: Applied patterns (real-world usage, blog posts)
- Fork 3: Failure modes (common mistakes, known issues)

## Phase 3: Verify (gate)
Spawn claim-verifier (fresh agent, NOT fork) on the draft.
If any verdict is REFUTED → stop, report, fix before proceeding.
```

**Key command-authoring patterns:**
- **Phase headers** make the workflow scannable and debuggable
- **Gate conditions** prevent bad output from flowing downstream. Explicit "stop if X" instructions.
- **Agent spawn directives** specify fresh vs fork and why. This distinction matters for bias independence.
- **`$ARGUMENTS`** passes user input. Design commands that take a topic, file path, or scope as their argument.

### Pipeline examples

**Research pipeline**, the most complex, demonstrating all patterns:

```
INTAKE → RESEARCH (3 parallel forks) → DRAFT → VERIFY (2 parallel agents) → REVISE → REGISTER → SYNTHESIZE
                                                  │                                          │
                                          claim-verifier ──┐                         BUILD (fresh agent)
                                          technical-reviewer ──┤                           │
                                                  │           │                    REVIEW (synthesis-reviewer)
                                          gate: no REFUTED ───┘                           │
                                                                                  gate: not NEEDS_REVISION
```

Design decisions that transfer to other pipelines:
- **Research phase uses forks** (parallel investigation of the same topic). Forks inherit context and share the prompt cache, so they're cheap and well-informed.
- **Verification phase uses fresh agents** (not forks). Deliberately no context from the researcher, preventing confirmation bias.
- **Draft lands in a staging area first.** Only promoted to permanent storage after passing verification gates.
- **Multiple independent gates.** Claim accuracy and pedagogical quality are checked by different agents with different evaluation criteria.

**Pattern: evidence-first commands.** If the model can see git commits, file changes, and completed items, the command should synthesize from evidence and present for confirmation. Don't ask the user to narrate what happened.

**Pattern: read-only scan, then report, then confirmed fixes.** For any structural sweep (index gaps, orphan files, config drift), the command reads first and reports findings, then asks before acting. Never auto-deletes.

---

## 9. Agent Design: Verification and the Fork/Fresh Decision

### The fork vs. fresh agent decision

**Fork** when the agent needs your current context and you want cache sharing. Forks inherit the full conversation, so they're ideal for parallel research legs that build on the same question. The trade: they also inherit biases, assumptions, and any mistakes in the current context.

**Fresh agent** when independence matters more than context. Verification agents must be fresh. If they inherit the builder's reasoning, they'll confirm rather than challenge. A fresh agent starts with zero context; its prompt must be self-contained, like briefing a colleague who just walked in.

Rule of thumb: any quality gate should use a fresh agent. Any exploration of a shared question should use forks.

**Worktrees** (`isolation: "worktree"` on the Agent tool) give an agent its own copy of the repo, so it can edit files without conflicting with parallel agents or your main session. Use worktrees when multiple agents need to edit the same codebase concurrently.

**Rate-limit cost of parallel agents:** A research pipeline with 3 forks + 2 verification agents consumes rate limits 5x faster than single-threaded work. Batch parallel work early in a rate-limit window, not late. Watch the 5-hour usage percentage in the status line before launching agent-heavy pipelines.

### Three-agent verification system

Three read-only agents that gate knowledge quality from different angles:

**claim-verifier** checks individual factual assertions against primary sources. Verdicts: CONFIRMED / REFUTED / PARTIALLY TRUE / UNVERIFIABLE. Rigor standard: CONFIRMED requires a fetched source that positively asserts the same thing. "I didn't find a contradiction" = UNVERIFIABLE, not CONFIRMED.

**technical-reviewer** evaluates a note as a learning artifact across 5 dimensions: mental model correctness, completeness, example quality, citation sufficiency, structural coherence. Catches pedagogical gaps that correct facts alone don't reveal.

**synthesis-reviewer** evaluates an HTML page as a visual communication instrument across 7 dimensions: scanability, visual hierarchy, layout, information architecture, visual elements, dark mode, template compliance. Catches presentation failures invisible in source markdown.

Why three instead of one: each catches things the others miss. claim-verifier doesn't evaluate teaching quality. technical-reviewer doesn't verify facts against sources. synthesis-reviewer catches visual failures both text-focused agents are blind to.

### Agent frontmatter

```yaml
---
name: claim-verifier
description: Verify claims in knowledge notes against primary sources
tools: Read, WebSearch, WebFetch, Bash, Grep, Glob
model: opus
effort: high
maxTurns: 25
---
```

- **`tools`**: explicit allowlist. Verification agents get read + search tools, never Edit or Write.
- **`model: opus`**: verification quality directly determines knowledge base integrity; use the strongest model.
- **`effort: high`**: forces thorough evaluation over quick passes.
- **`maxTurns: 25`**: prevents a confused agent from running indefinitely, burning cost and wall-clock time. But set it too low and the agent returns partial results that look complete. 20–25 for thorough work, 10–15 for focused tasks.

---

## 10. Template Injection Pattern

When Claude Code produces a recurring artifact type (rendered pages, config scaffolds, test files, project skeletons), generating from scratch is expensive and inconsistent. Instead: author the boilerplate once as a template file, `cp` it to the destination, then use `Edit` to fill the variable parts. `Edit` transmits only the diff, so the boilerplate costs zero output tokens after the copy.

Because the template is a real file on disk (not instructions in CLAUDE.md), it stays out of the context window until needed and doesn't drift the way inline instructions do.

### Example: HTML synthesis pages

```
1. cp ~/.claude/templates/synthesis.html <dest>.html
2. Edit: fill {{TITLE}}, {{DATE}}, {{CONTENT}}, {{SOURCES}}, {{SOURCES_MANIFEST}}
```

The template contains the full theme (CSS, dark/light toggle, navigation JS), authored once, consistent across all pages, zero output tokens per generation.

**Trade-off:** Existing pages are snapshots. They don't pick up template improvements retroactively. To re-skin an old page, regenerate it from its source. The template changes rarely and regeneration is cheap, so this works fine in practice.

**Page types:**
- **Canonical**: named after its source note, regenerated when the note changes. A render of knowledge that lives in markdown.
- **Ephemeral**: date-stamped, no source file. One-time absorption views. Swept when stale.

---

## 11. Memory System

Auto-memory persists behavioral corrections across sessions. The index (`MEMORY.md`, roughly the first 200 lines or 25KB, whichever is smaller) loads every session; individual topic files load on demand when the model judges them relevant.

### What to memorize

**Feedback memories** are behavioral corrections that prevent repeating mistakes. Examples: "use worked examples with visible data in knowledge notes, not just definitions"; "synthesize from git evidence instead of asking the user to narrate what happened."

**User profile** covers role, goals, tech stack. Changes how explanations are framed and what level of detail is appropriate.

**Project context** captures decisions, motivations, and constraints that aren't in git. Data models to use for examples, tool ideas, deadlines with absolute dates.

### What NOT to memorize

- **Code patterns, architecture, file paths** are derivable from the current project state. They become stale in memory faster than they're useful.
- **Debugging solutions**: the fix is in the code; the context is in the commit message. Memory adds a stale copy.
- **Ephemeral task state**: use tasks (`TaskCreate` / `TaskUpdate`) for in-session tracking. Memory is for things that matter next week.
- **Anything that can be `grep`'d**: if the information lives in a file, memory is a stale cache of it.

**Memory is a stale cache.** Always verify against current state before acting on a memory. A memory that names a specific function, file, or flag is a claim about what existed when the memory was written. It may have been renamed, removed, or never merged.

---

## 12. MCP Server Configuration

MCP (Model Context Protocol) servers extend Claude Code with external tool access. The parts worth knowing:

**Scoping:** Servers register per project directory in `~/.claude.json`. The server only loads when Claude Code runs from that directory. Use `-s user` for servers needed everywhere, or `.mcp.json` in the repo root for project-scoped servers that should be committed and shared with the team. Register from the narrowest scope that needs the server.

**Transport types:** `stdio` (local subprocess), `http` (streamable HTTP, newer), `sse` (Server-Sent Events, older remote servers).

**Permission naming:** MCP tool permissions follow `mcp__<server>__<tool>`. Wildcards work: allow `mcp__<server>__get_*` for all reads, set `ask` on `mcp__<server>__create_*` and `mcp__<server>__delete_*` for mutations.

**Debugging:** If tools don't appear: (1) is the server registered for the current directory? `claude mcp list` shows registrations and scopes. (2) Is the transport correct? (3) Is the process reachable? (4) Are permissions blocking it? `/permissions` shows all rules; `/mcp` shows MCP status.

**Deferred loading:** MCP tool schemas load on demand. At session start, only tool names are in context, not full parameter schemas. Adding more MCP servers has minimal baseline context cost.

---

## 13. Content Staging and Verification Gates

One pattern from workspace design that directly affects Claude Code operation: **never let agent output go directly into permanent storage.** Agent-generated content lands in a staging area (`drafts/` or equivalent) and must pass a verification gate (fresh agent, not fork; see §9) before promotion. This goes for knowledge notes, config changes, and any artifact the agent produces that will persist beyond the session.

The gate criteria should be explicit in the command that triggers the pipeline (see §8): what verdicts block promotion, which agent runs the check, what happens on failure.

---

## 14. Traps and Anti-Patterns

1. **Advisory rules for hard constraints.** Use `permissions.deny` or a PreToolUse hook, not CLAUDE.md prose. (§3)

2. **Skills described in multiple places.** The `SKILL.md` is the source of truth; copies drift within weeks. Use `@path` imports to reference without duplicating. (§5)

3. **Hook scripts reading `$CLAUDE_FILE_PATH`.** This environment variable does not exist. Hook input is JSON on stdin; the file path is `.tool_input.file_path`. A script reading a nonexistent env var silently gets an empty string and does nothing. The failure is invisible.

4. **Workflow instructions in rules.** Rules load every session; workflows belong in skills or commands (loaded on demand). (§1)

5. **Asking the user what they did.** If the model can see git commits, file changes, and completed items, it should synthesize from evidence and present for confirmation. Asking the user to narrate what's already visible wastes time and produces worse summaries.

6. **Forks for verification.** Forks inherit the builder's context and biases. Verification agents must be fresh (zero context, self-contained prompt) for independent judgment. Any quality gate should be a fresh agent. (§9)

7. **Silent model fallback.** An unrecognized model ID has been observed to fall back to a different model without warning. Always display the model name in the status line to verify. (§2, §7)

8. **Agents without `maxTurns`.** Without a turn limit, a confused agent loops indefinitely, burning cost and wall-clock time. But too-low limits cause partial results that look complete. Set explicit `maxTurns` in every agent's frontmatter. (§9)

9. **Agent prompts that delegate understanding.** "Based on your research, fix the bug" pushes synthesis onto the agent when you should do it. Write agent prompts that prove you understood: include file paths, what specifically to investigate, what you already ruled out. Fresh agents have zero context. They need a briefing, not a delegation.

10. **Rules without path scope.** A rule file without `paths` frontmatter loads in every session for every file in the project. A note-formatting rule should be scoped to `knowledge/**/*.md`, not loaded when editing Python. Use the `paths` field in frontmatter.

11. **Treating memory as documentation.** Memory is a stale cache of behavioral corrections. Architecture, file paths, code patterns: all derivable from the codebase, all become stale in memory. If it can be `grep`'d, it shouldn't be memorized. (§11)

12. **Multi-agent rate-limit burn.** A pipeline with 3 forks + 2 verification agents consumes rate limits 5x faster than single-threaded work. Check the 5-hour usage percentage before launching agent-heavy pipelines. (§9)

---

## 15. New Machine Checklist

### Tier 1: Universal (any Claude Code installation)

- [ ] `~/.claude/settings.json` exists with: model, `defaultMode: auto`, sandbox enabled with `autoAllowBashIfSandboxed`, deny list covering secret-shaped paths and destructive commands (40+ patterns)
- [ ] Global `CLAUDE.md`: output style, behavioral preferences. Keep it concise; every line earns its context cost
- [ ] Status line configured with at minimum context window %, rate limit visibility, and model name
- [ ] Pre-commit secret gate installed in every repo with a remote
- [ ] Global PostToolUse lint hook for your primary language
- [ ] Verify the deny list works: attempt to read a denied path and confirm it blocks

### Tier 2: Project infrastructure (adapt to your workflow)

- [ ] SessionStart hook injecting active work context (target under 50 lines)
- [ ] At least one verification agent (fresh, read-only) gating content quality
- [ ] Commands for recurring multi-step workflows (journal maintenance, research, deployment)
- [ ] Path-scoped rules for domain-specific formatting or linting (use `paths` frontmatter)
- [ ] `.claude/settings.local.json` for machine-specific permissions (gitignored)
- [ ] Staging area (`drafts/` or equivalent) for agent output awaiting verification
- [ ] MCP servers registered at appropriate scope (project `.mcp.json` or user-level)

### Tier 3: Knowledge and memory (for long-lived workspaces)

- [ ] Memory: user profile (role, stack, goals) + feedback corrections from sessions
- [ ] `MEMORY.md` index under 200 lines / 25KB (content beyond that is truncated)
- [ ] Workspace structure with explicit trust levels and gates between them
- [ ] Master index tracking all knowledge artifacts
- [ ] Synthesis template for consistent recurring outputs
- [ ] Naming conventions documented in project CLAUDE.md

### Tier 4: Automation (when the setup is stable)

- [ ] Sweep command for structural rot (index gaps, orphan pages, config drift)
- [ ] Batch verification of knowledge artifacts
- [ ] launchd plist (or equivalent scheduler) triggering maintenance at login/daily. Use `claude -p` (headless mode) for non-interactive scheduled runs

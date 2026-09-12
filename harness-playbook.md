# Claude Code Harness Playbook

Field-tested configuration patterns, pipeline designs, and operational learnings from daily Claude Code use across a multi-repo personal engineering workspace (knowledge base, learning system, investment tools, public projects). This captures what actually changed how sessions work — not configuration reference, but the decisions behind the configuration and the traps that cost hours to find.

**Purpose:** A portable reference for auditing and improving a Claude Code installation on any machine. Each pattern includes the rationale and failure mode that motivated it, so the reader can decide whether it applies to their own setup rather than blindly copying config.

**Scope:** All patterns are original synthesis from firsthand use. Configuration examples are illustrative, not copyrighted material. Workspace-specific implementations are presented as examples of general patterns — adapt the concepts, not the specifics.

---

## 1. Architecture: What Goes Where

### The forcing function: context cost

Every line in `CLAUDE.md`, every rule file, every line of `MEMORY.md` loads into the context window at session start and stays there for the session's life. At roughly 4 tokens per line, a 200-line CLAUDE.md costs ~800 tokens every session. That's small in isolation but compounds: global CLAUDE.md + project CLAUDE.md + rules + memory index + SessionStart hook output can consume 5-10K tokens before the user types anything.

The architecture's job is to minimize this baseline while ensuring nothing critical is missing.

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
├── rules/*.md               always loaded (path-scoped with globs: frontmatter possible)
├── skills/<name>/SKILL.md   loaded on-demand when invoked or judged relevant
├── hooks/*.sh               scripts called by hooks in settings.json
├── templates/               reusable file templates (zero context cost until used)
├── statusline.sh            status line script
└── projects/<path>/memory/
    ├── MEMORY.md             index — roughly first 200 lines loaded every session
    └── <topic>.md            loaded on demand when relevant

~/.claude.json               MCP server registrations (per-project scope keys)

<project>/.claude/
├── settings.json            project hooks (SessionStart, PostToolUse)
├── settings.local.json      project-private permissions (gitignored, not committed)
├── agents/<name>.md         project-scoped subagents
├── commands/<name>.md       project-scoped slash commands
└── rules/<name>.md          always-loaded project rules (path-scoped with globs: frontmatter)
```

### Load behavior summary

- **Always loaded** — CLAUDE.md (global + project), rules, MEMORY.md index. These define the session's baseline context cost.
- **On demand** — Skills, commands, agents, individual memory files. These cost nothing until invoked.
- **Event-triggered** — Hooks. They run scripts in response to tool use or session events. Only SessionStart and UserPromptSubmit stdout becomes conversation context.

**The trap:** Putting workflow instructions in `CLAUDE.md` or rules. They load every session and eat context even when irrelevant. If something isn't needed every session, it's a skill or command. The corollary: if something MUST be enforced every session (formatting rules, visibility boundaries), it goes in rules — skills can be ignored by the model.

---

## 2. Model and Context Management

### Model configuration

```json
{
  "model": "opus[1m]",
  "env": {
    "ANTHROPIC_CUSTOM_MODEL_OPTION": "claude-opus-4-6",
    "ANTHROPIC_CUSTOM_MODEL_OPTION_NAME": "Opus 4.6",
    "ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION": "Opus 4.6 — most capable for complex work"
  },
  "alwaysThinkingEnabled": true
}
```

**Silent fallback trap:** As of mid-2026, a `model` value Claude Code doesn't recognize falls back to Sonnet with no warning. To use a non-default model, set all three `ANTHROPIC_CUSTOM_MODEL_OPTION*` env vars in `settings.json` alongside the `model` field. Update them together — a mismatch means the model picker shows the right name but the session runs the wrong model.

**The `[Nm]` suffix** sets the thinking token budget. `[1m]` = up to 1 million thinking tokens per turn (tokens, not minutes). Higher budgets improve reasoning on hard problems but increase latency and cost. Without the suffix, the model uses its default thinking allocation.

**`alwaysThinkingEnabled`** forces extended thinking on every turn, even simple ones. Worth the token cost for complex work (architecture, debugging, multi-file reasoning) but adds overhead to trivial questions.

**`model: opus` in agent frontmatter** is an alias that resolves to the latest Opus model, not a pinned version. This means agent behavior may shift when Anthropic releases a new Opus. For stability, pin a specific model ID; for staying current, use the alias.

### Prompt cache and context economics

Claude Code caches the system prompt (CLAUDE.md, rules, memory index, etc.) across turns within a session. The first turn pays the full input cost; subsequent turns pay only for new content. Three things destroy this cache:

1. **`/clear`** — wipes the conversation entirely; the next turn re-pays for the full system prompt
2. **Model switching** mid-session — the cache is model-specific
3. **Context window overflow** — when the conversation is compressed, cached content may be evicted

**`/compact`** summarizes the conversation but preserves the system prompt cache. It's lossy — details from earlier turns may be dropped. Use it when deep in a single problem and the context bar is filling up.

### Context window strategy

The context window is a finite resource. Managing it is an operational skill, not a configuration setting.

**When to `/clear`:** Starting a genuinely unrelated task. The cache cost of rebuilding the system prompt is worth it to shed irrelevant conversation history.

**When to `/compact`:** Deep in one problem, context filling up, but you need the conversation thread. The system prompt stays cached; earlier turns get summarized.

**When to fork:** The primary reason to fork isn't speed — it's keeping intermediate tool output (file reads, search results, git logs) out of your main context window. A fork inherits your full conversation and shares the prompt cache, so it's cheap. The fork's final report is a short summary that enters your context; the hundreds of lines of tool noise stay in the fork's thread. Use forks for any open-ended question where you don't know how many files you'll need to read.

**When to use a fresh agent:** When you need independent judgment without the current context's biases. Verification and review agents must be fresh — if they inherit the builder's reasoning, they'll confirm rather than challenge. A fresh agent starts with zero context, so its prompt must be self-contained.

**Practical heuristic:** Watch the context percentage in the status line. Above 50%, start thinking about what to offload. Above 75%, fork aggressively or `/compact`. Above 90%, `/clear` or finish the task.

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

*Place this in `~/.claude/settings.json`. The deny list covers two categories: destructive system commands and secret-shaped file paths. Both are hard floors that no classifier judgment or CLAUDE.md instruction can override.*

**Why `auto` over other modes:** The alternatives — `default` mode (prompts before every non-read action) and explicit allow-lists (partial automation) — are noisier without being safer when sandbox + deny are already in place. `auto` mode uses a server-side classifier that approves routine actions and blocks risky ones. Combined with sandbox, normal dev work flows without prompts.

**The real safety layer is `permissions.deny`, not CLAUDE.md.** CLAUDE.md instructions are advisory — the model can be steered past them. Deny rules and hooks cannot be overridden, even under prompt injection. Anything that must never happen goes in `deny`, not in prose.

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

The single highest-ROI hook. Every session opens with orientation — no manual briefing needed.

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

*Place this in `.claude/settings.json` (project-level). This is a simplified example — a production version adds available-commands reminders, last-checkin date, and completed-item nudges.*

**What to inject:**
- Active work items (so the model knows what's in flight)
- Recent git commits (so the model knows what just shipped)
- Available commands (the ones you won't remember)
- Actionable nudges (completed items not yet swept, stale journal)

**What NOT to inject:** Everything in SessionStart stdout enters the context window permanently. Injecting 500 lines of "helpful context" means every turn in the session carries that weight. Target under 50 lines of high-signal orientation. If something is reference material, put it in a file and let the model read it when needed.

**Design principle:** SessionStart and UserPromptSubmit are the two hook types whose stdout becomes conversation context. SessionStart runs once at session open; UserPromptSubmit runs before each user message is processed. Use SessionStart for orientation, UserPromptSubmit for per-message guards or context refresh.

---

## 5. Writing Effective CLAUDE.md

CLAUDE.md is the single most-read config file and the easiest to get wrong.

**Keep it short.** Global CLAUDE.md should be under 30 lines. Project CLAUDE.md can be longer but every line pays context cost every session. If you're past 100 lines, audit what should be a rule (path-scoped, always loaded) vs. a skill (loaded on demand).

**Be specific and testable.** "Write clean code" is noise — the model already tries to do that. "Use early returns; no else-after-return" is actionable. Every line should be something the model can unambiguously follow or violate.

**Separate behavior from knowledge.** CLAUDE.md defines how Claude should behave (output style, formatting, safety rules). It should not contain project documentation, architecture descriptions, or workflow instructions. Those belong in rules, skills, or knowledge files that load at the right time.

**Structure for scanning.** Use headers and short bullets. The model reads the full file but attends more to structured content than prose paragraphs. Frontload the highest-priority instructions.

**Don't repeat the system prompt.** Claude Code's built-in system prompt already covers git safety, file handling, and many best practices. Restating them in CLAUDE.md wastes context and can conflict if the system prompt changes.

**Don't restate skills or commands.** The `SKILL.md` is the source of truth for a skill's behavior. Describing the same skill in CLAUDE.md creates a copy that drifts within weeks. Reference skills ("use `/learn` for new concepts"), don't redefine them.

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
        "command": "f=$(echo '$INPUT' | jq -r '.tool_input.file_path // empty'); if [ -n \"$f\" ] && [[ \"$f\" == *.py ]]; then export PATH=\"$HOME/.local/bin:$PATH\"; ruff check --no-fix \"$f\" 2>&1 || true; fi"
      }]
    }]
  }
}
```

*A PostToolUse lint hook that runs `ruff check` on every Python file edit. The `|| true` ensures exit code 0 — PostToolUse hooks cannot block; a non-zero exit is just noise. The PATH export is needed because hook scripts inherit a minimal environment.*

### Hook types and their powers

- **SessionStart** — runs once at session open. Stdout becomes conversation context. Use for orientation injection.
- **UserPromptSubmit** — runs before each user message. Stdout becomes context. Can block (exit 2).
- **PreToolUse** — runs before a tool executes. Can block (exit 2). Use for safety gates.
- **PostToolUse** — runs after a tool executes. Cannot block. Use for lint, render, side effects.
- **Stop** — runs when the session ends. Can block (exit 2).

### Pre-commit secret gate

A two-layer scan installed as a pre-commit hook in every repo with a remote:

1. **Path-based:** Blocks staging of `.env*`, `.pem`, `.key`, `credentials.*`, `token*`, and similar patterns
2. **Content-based:** Runs `gitleaks protect --staged` (if installed) or a fallback regex scan for common secret patterns — API keys, PATs, private key blocks

This is the last gate before secrets reach a remote. It catches things the sandbox and deny rules miss because those operate at the Claude Code level, not the git level.

### Hook gotchas

- **Input is JSON on stdin, not environment variables.** The edited file path is `.tool_input.file_path` in the JSON. There is no `$CLAUDE_FILE_PATH` env var — a script that reads one silently gets an empty string and does nothing.
- **Minimal PATH.** Hook scripts don't inherit your shell profile. Tools in `~/.local/bin` (from `uv tool install` or `pipx`) need an explicit PATH export inside the script.
- **Only two hook types inject context.** SessionStart and UserPromptSubmit stdout enters the conversation. PostToolUse output goes to the hook's log but not the model.
- **Exit codes matter only for blocking hooks.** PreToolUse, UserPromptSubmit, and Stop can block on exit 2. PostToolUse ignores the exit code.

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

The script receives a JSON object on stdin with fields including `model`, `thinking`, `effort`, `project`, `worktree`, `contextWindow` (with `used` and `total`), and `rateLimits`. It writes one line to stdout — that line becomes the status bar. The script runs frequently, so it must be fast; do all JSON parsing in a single `jq` call.

### What to display (prioritized)

1. **Context window percentage** — the one number you always need. Color-code it: green under 50%, yellow 50-75%, red above 75%.
2. **Rate limit proximity** — the one that ambushes you. Show 5-hour and 7-day usage percentages with reset countdown.
3. **Model name** — confirms you're on the right model (catches silent fallback).
4. **Branch + dirty state** — orientation without running `git status`.
5. **Session duration** — awareness of how long you've been in one context.

**Why it matters:** Context window percentage and rate limit visibility prevent the two most common session-killers: running out of context mid-task and hitting rate limits without warning. These are operational risks, not configuration problems — the status line makes them visible before they hit.

---

## 8. Commands and Pipeline Architecture

### How commands work

A command is a markdown file at `.claude/commands/<name>.md`. When the user types `/<name> some args`, Claude reads the file and `$ARGUMENTS` is replaced with the user's input. The file contains natural-language instructions — phase descriptions, agent spawn directives, gate conditions, output format requirements.

```markdown
---
description: Research a topic and produce a verified knowledge note
---

Research `$ARGUMENTS` and produce a verified knowledge note.

## Phase 1: Intake
Parse the topic. Check knowledge/_index.md for existing coverage...

## Phase 2: Research (parallel forks)
Launch 3 forks:
- Fork 1: Primary sources — official docs, specs, RFCs...
- Fork 2: Applied patterns — real-world usage, blog posts...
- Fork 3: Failure modes — common mistakes, known issues...

## Phase 3: Verify (gate)
Spawn claim-verifier (fresh agent, NOT fork) on the draft.
If any verdict is REFUTED → stop, report, fix before proceeding.
```

**Key command-authoring patterns:**
- **Phase headers** make the workflow scannable and debuggable
- **Gate conditions** prevent bad output from flowing downstream — explicit "stop if X" instructions
- **Agent spawn directives** specify fresh vs fork and why — this distinction matters for bias independence
- **`$ARGUMENTS`** passes user input — design commands that take a topic, file path, or scope as their argument

### Pipeline examples

**Research pipeline** — the most complex, demonstrating all patterns:

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
- **Research phase uses forks** (parallel investigation of the same topic) — forks inherit context and share the prompt cache, so they're cheap and well-informed
- **Verification phase uses fresh agents** (not forks) — deliberately no context from the researcher, preventing confirmation bias
- **Draft lands in a staging area first** — only promoted to permanent storage after passing verification gates
- **Multiple independent gates** — claim accuracy and pedagogical quality are checked by different agents with different evaluation criteria

**Evidence-first journal maintenance** (`/checkin`): Originally asked the user what they did. Rewritten to synthesize from git evidence (commits, file changes, completed items) and present for confirmation. The model can see what happened — asking the user to narrate it wastes time and produces worse summaries.

**Structural health sweep** (`/upkeep`): 9-check sweep for workspace rot — index gaps, orphan pages, stale files, config drift. Reports findings first, then asks before acting. Never auto-deletes. Design pattern: read-only scan → structured report → user-confirmed fixes.

---

## 9. Agent Design: Verification and the Fork/Fresh Decision

### The fork vs. fresh agent decision

This is the single most transferable concept from the agent system.

**Fork** when the agent needs your current context and you want cache sharing. Forks inherit the full conversation, so they're ideal for parallel research legs that build on the same question. The trade: they also inherit biases, assumptions, and any mistakes in the current context.

**Fresh agent** when independence matters more than context. Verification agents must be fresh — if they inherit the builder's reasoning, they'll confirm rather than challenge. A fresh agent starts with zero context; its prompt must be self-contained, like briefing a colleague who just walked in.

**The pattern generalizes:** Any quality gate should use a fresh agent. Any exploration of a shared question should use forks.

### Three-agent verification system

Three read-only agents that gate knowledge quality from different angles:

**claim-verifier** — checks individual factual assertions against primary sources. Verdicts: CONFIRMED / REFUTED / PARTIALLY TRUE / UNVERIFIABLE. Rigor standard: CONFIRMED requires a fetched source that positively asserts the same thing. "I didn't find a contradiction" = UNVERIFIABLE, not CONFIRMED.

**technical-reviewer** — evaluates a note as a learning artifact across 5 dimensions: mental model correctness, completeness, example quality, citation sufficiency, structural coherence. Catches pedagogical gaps that correct facts alone don't reveal.

**synthesis-reviewer** — evaluates an HTML page as a visual communication instrument across 7 dimensions: scanability, visual hierarchy, layout, information architecture, visual elements, dark mode, template compliance. Catches presentation failures invisible in source markdown.

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

- **`tools`** — explicit allowlist. Verification agents get read + search tools, never Edit or Write.
- **`model: opus`** — verification quality directly determines knowledge base integrity; use the strongest model.
- **`effort: high`** — forces thorough evaluation over quick passes.
- **`maxTurns: 25`** — explicit limit prevents a confused agent from burning through rate limits. Set this on every agent.

---

## 10. Template Injection Pattern

### The general pattern: template + Edit

When Claude Code produces a recurring artifact type (rendered pages, config scaffolds, test files, project skeletons), generating from scratch is expensive and inconsistent. Instead: author the boilerplate once as a template file, `cp` it to the destination, then use `Edit` to fill the variable parts. `Edit` transmits only the diff, so the boilerplate costs zero output tokens after the copy.

This works for any repeating structure. The key is that the template is a real file on disk (not instructions in CLAUDE.md), so it stays out of the context window until needed and doesn't drift the way inline instructions do.

### Example: HTML synthesis pages

```
1. cp ~/.claude/templates/synthesis.html <dest>.html
2. Edit: fill {{TITLE}}, {{DATE}}, {{CONTENT}}, {{SOURCES}}, {{SOURCES_MANIFEST}}
```

The template contains the full theme (CSS custom properties, dark/light toggle, scroll-spy navigation JS) — authored once, consistent across all pages, zero output tokens per generation. `Edit` transmits only the content being injected.

**Template features:** Sticky horizontal nav with IntersectionObserver scroll-spy, dark/light toggle persisted to localStorage, responsive 860px max-width layout, CSS custom properties for theming.

**Trade-off:** Existing pages are snapshots — they don't pick up template improvements retroactively. To re-skin an old page, regenerate it from its source. This is acceptable because the template changes rarely and regeneration is cheap.

**Page type discipline:**
- **Canonical** — named after its source note, regenerated when the note changes. A render of knowledge that lives in markdown.
- **Ephemeral** — date-stamped, no source file. One-time absorption views. Swept when stale.

---

## 11. Memory System

Auto-memory persists behavioral corrections across sessions. The index (`MEMORY.md`, roughly the first 200 lines) loads every session; individual topic files load on demand when the model judges them relevant.

### What to memorize

**Feedback memories** — behavioral corrections that prevent repeating mistakes:
- "Notes teach, not just record" → knowledge notes need worked examples with visible data, analogies, real-world usage
- "Expertise over utility learning" → build full-domain expertise from fundamentals; don't scope learning to "just what current work needs"
- "/checkin synthesizes from evidence" → don't ask what's already visible in git history

**User profile** — role, goals, tech stack, career targets. Changes how explanations are framed and what level of detail is appropriate.

**Project context** — decisions, motivations, and constraints that aren't in git. Data models to use for examples, tool ideas, deadlines with absolute dates.

### What NOT to memorize

- **Code patterns, architecture, file paths** — derivable from the current project state. They become stale in memory faster than they're useful.
- **Debugging solutions** — the fix is in the code; the context is in the commit message. Memory adds a stale copy.
- **Ephemeral task state** — use tasks for in-session tracking. Memory is for things that matter next week.
- **Anything that can be `grep`'d** — if the information lives in a file, memory is a stale cache of it.

**Memory is a stale cache.** Always verify against current state before acting on a memory. A memory that names a specific function, file, or flag is a claim about what existed when the memory was written — it may have been renamed, removed, or never merged.

---

## 12. MCP Server Configuration

MCP (Model Context Protocol) servers extend Claude Code with external tool access — data sources, browsers, APIs, services.

### Registration and scope

```bash
claude mcp add <name> --transport http <url>
```

Servers register per project directory in `~/.claude.json` by default — the server only loads when Claude Code runs from that directory. Use `-s user` for servers needed across all projects.

**Transport types:**
- `http` — remote servers accessible via URL
- `stdio` — local process spawned as a subprocess

**Scoping principle:** Register from the narrowest directory that needs the server. A server registered from the workspace root loads in every session launched there, even for unrelated tasks.

### Permission patterns

MCP tool permissions use the naming pattern `mcp__<server>__<tool>`:

```json
{
  "permissions": {
    "allow": ["mcp__<server>__get_*"],
    "ask": ["mcp__<server>__place_*", "mcp__<server>__cancel_*"]
  }
}
```

*Allow all read operations; prompt before any mutation. Wildcards work for bulk grants. Deny rules override allow rules at any level.*

### Debugging MCP connections

If an MCP server's tools don't appear in the session:
1. Is the server registered for the current project directory? `claude mcp list` shows registrations and scopes.
2. Is the transport correct? Local servers: `stdio`. Remote servers: `http`.
3. Is the server process reachable? For `stdio`, Claude Code spawns it — check the process exists.
4. Are permissions blocking it? `/permissions` shows all rules and their source files.

### Tool discovery and deferred loading

MCP tools load their schemas on demand (deferred loading) to save context. At session start, only tool names and server instructions are in context — not full parameter schemas. When the model needs to call a tool, it loads the schema via `ToolSearch`. This means adding more MCP servers has minimal baseline context cost.

---

## 13. Knowledge Architecture (Transferable Pattern)

### The principle: trust-level segregation

Separate content by how permanent and how verified it is. Raw scratch belongs in a different place than verified knowledge, with explicit gates between them.

**The levels matter less than the gates:**
- Unverified content never directly enters the knowledge base — it goes through a staging area first and must pass verification
- Each level has a sweep criterion (delete after days, review after weeks, graduate or kill after months)
- An index file tracks what exists at each level so orphans are detectable

### One implementation: a graduated workspace

```
journal/          TIME-BOUND — what you are doing (single source of truth for active work)
knowledge/        TIMELESS — what you know (verified, indexed, styled for 6-month recall)
sandbox/          HOURS — experiments (date-stamped, delete freely)
workbench/        WEEKS — immature projects (manifested in an index, may die)
projects/         YEARS — graduated repos (own remotes, README, license)
local/            NEVER LEAVES — personal, financial, employer-internal
drafts/           STAGING — agent output awaiting human judgment
```

**Naming discipline** removes the need for organizational guesswork: `<type>_<domain>_<topic>.md` for timeless notes, `YYYY-MM-DD_<topic>` for date-stamped snapshots. When the naming system is consistent, search replaces navigation.

**The work loop:**
```
backlog → active [ ] → work → [x] → tidy sweep → archived (by date)
                                         ↓
                               verification gate before knowledge promotion
```

---

## 14. Traps and Anti-Patterns

A quick-reference index of failure modes. Each was discovered by hitting it — not theorized.

1. **Advisory rules for hard constraints.** CLAUDE.md prose is advisory. Anything that must never happen goes in `permissions.deny` or a PreToolUse hook. A determined prompt injection can steer past prose; it cannot override a deny rule.

2. **Skills described in multiple places.** The `SKILL.md` is the source of truth. Copies in CLAUDE.md or notes drift within weeks. Reference skills, don't redefine them.

3. **Hook scripts reading `$CLAUDE_FILE_PATH`.** This environment variable does not exist. Hook input is JSON on stdin; the file path is `.tool_input.file_path`. A script reading a nonexistent env var silently gets an empty string and does nothing — the failure is invisible.

4. **Workflow instructions in rules.** Rules load every session. Workflows are skills or commands (loaded on demand). The context cost of always-loaded instructions that aren't always relevant compounds across every turn of every session.

5. **Asking the user what they did.** If the model can see git commits, file changes, and completed items, it should synthesize from evidence and present for confirmation. Asking the user to narrate what's already visible wastes time and produces worse summaries.

6. **Forks for verification.** Forks inherit the builder's context and biases. Verification agents must be fresh (zero context, self-contained prompt) to provide independent judgment. The pattern generalizes: any quality gate should be a fresh agent.

7. **Silent model fallback.** An unrecognized model ID falls back to Sonnet with no warning. The three `ANTHROPIC_CUSTOM_MODEL_OPTION` env vars must be set alongside `model` in settings.json. Update them together.

8. **Agents without `maxTurns`.** Without a turn limit, a confused agent loops until it exhausts rate limits. Set explicit `maxTurns` in every agent's frontmatter — 20-25 for thorough work, 10-15 for focused tasks.

9. **Overloading SessionStart output.** Everything SessionStart prints enters the context window permanently. 500 lines of "helpful context" means every turn carries that weight. Target under 50 lines of high-signal orientation.

10. **Agent prompts that delegate understanding.** "Based on your research, fix the bug" pushes synthesis onto the agent when you should do it. Write agent prompts that prove you understood: include file paths, what specifically to investigate, what you already ruled out. Fresh agents have zero context — they need a briefing, not a delegation.

11. **Rules without path scope.** A rule file without globs frontmatter loads in every session for every file in the project. A note-formatting rule should be scoped to `knowledge/**/*.md`, not loaded when editing Python. Use the `globs` field in frontmatter.

12. **Treating memory as documentation.** Memory is a stale cache of behavioral corrections. It's for "don't do X, do Y instead" patterns from conversation. Architecture, file paths, code patterns — these are derivable from the codebase and become stale in memory. If it can be `grep`'d, it shouldn't be memorized.

---

## 15. New Machine Checklist

### Tier 1: Universal (any Claude Code installation)

These apply regardless of what you're building.

- [ ] `~/.claude/settings.json` exists with: model, `defaultMode: auto`, sandbox enabled with `autoAllowBashIfSandboxed`, deny list for secret-shaped paths and destructive commands
- [ ] Custom model env vars set if using a non-default model (all three `ANTHROPIC_CUSTOM_MODEL_OPTION*` together)
- [ ] Global `CLAUDE.md` under 30 lines: output style, behavioral preferences, technical integrity stance
- [ ] Status line configured with at minimum context window % and rate limit visibility
- [ ] Pre-commit secret gate installed in every repo with a remote
- [ ] Global PostToolUse lint hook for your primary language

### Tier 2: Project infrastructure (adapt to your workflow)

These are patterns worth implementing, adapted to whatever you're building.

- [ ] SessionStart hook injecting active work context (under 50 lines)
- [ ] At least one verification agent (fresh, read-only) gating content quality
- [ ] Commands for recurring multi-step workflows (journal maintenance, research, deployment)
- [ ] Path-scoped rules for domain-specific formatting or linting
- [ ] `.claude/settings.local.json` for machine-specific permissions (gitignored)

### Tier 3: Knowledge and memory (for long-lived workspaces)

These compound over time in workspaces you return to daily.

- [ ] Memory: user profile (role, stack, goals) + feedback corrections from sessions
- [ ] `MEMORY.md` index under 200 lines (content beyond that is truncated)
- [ ] Workspace structure with explicit trust levels and gates between them
- [ ] Master index tracking all knowledge artifacts
- [ ] Synthesis template for consistent recurring outputs
- [ ] Naming conventions documented in project CLAUDE.md
- [ ] Scheduled maintenance (launchd or equivalent) for journal/workspace hygiene

### Tier 4: Automation (when the setup is stable)

These prevent the setup itself from rotting.

- [ ] `/upkeep` or equivalent sweep for structural rot (index gaps, orphan pages, config drift)
- [ ] `/verify-all` or equivalent batch verification of knowledge artifacts
- [ ] launchd plist (or equivalent scheduler) triggering maintenance at login/daily
- [ ] Mirror process syncing harness learnings to a portable reference

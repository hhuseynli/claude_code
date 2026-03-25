# Claude Code Notes

## 1. Context Management

### CLAUDE.md Files
**Why:** Claude has no memory between sessions. Without a `CLAUDE.md`, you're re-explaining your project structure, build commands, and conventions every time you open a new session — that's wasted tokens and wasted time. `CLAUDE.md` is persistent context that loads automatically.

Claude reads it at session start to understand your project without you having to re-explain anything. Use it to store architecture notes, common commands, and coding conventions.

| File | Scope | Use for |
|------|-------|---------|
| `CLAUDE.md` (project root) | Current project | Architecture, build commands, team conventions |
| `CLAUDE.local.md` (project root, gitignored) | Current project, personal | Your local overrides, personal preferences per-project |
| `~/.claude/CLAUDE.md` | All projects | Global style guide, personal preferences |

**Example project CLAUDE.md:**
```markdown
## Build & Test
- `npm run dev` — start dev server
- `npm test` — run test suite

## Architecture
- `src/api/` — Express routes
- `src/db/` — Prisma models

## Conventions
- Use named exports only
- No `any` in TypeScript
```

Run `/init` to auto-generate a `CLAUDE.md` from your codebase. Useful when onboarding to a team project.

---

### File Mentions (`@`)
**Why:** Without `@`, Claude has to search through your codebase to find the relevant file — burning tool calls and context on navigation. Mentioning the file directly skips that and pins exactly what you're talking about.

Use `@filename` to pull a specific file into context mid-conversation, rather than having Claude search for it.

```
@src/auth/middleware.ts — why is this throwing a 401 on valid tokens?
```

---

### Memory (`/memory`)
**Why:** Sometimes Claude learns something mid-session that should persist — a naming convention you clarify, a decision you make about architecture. Without `/memory`, that insight disappears when the session ends. This lets Claude write it down for itself.

Previously `#` (memorize mode). Use `/memory` to edit your `CLAUDE.md` files directly from within a conversation — so Claude can update its own context based on what it learns.

---

## 2. Making Changes in Code

### Input Methods
- **Screenshots**: `Ctrl+V` to paste directly. Useful for sharing UI bugs or error dialogs.
- **Files**: Drag in or use `@` mentions.

### Thinking Modes
**Why:** Default Claude Code responses are fast but shallow — fine for simple tasks, but it'll miss edge cases on hard bugs or make poor architectural decisions on broad refactors. Thinking/Planning modes trade tokens for quality, so you only use them when the task genuinely warrants it.

| Mode | When to use | Example |
|------|-------------|---------|
| **Thinking** | Deep, single-focus problems | Tricky bug, algorithmic design |
| **Planning** | Broad, multi-step tasks | Refactoring a module, adding a feature across many files |

You can chain them: plan first, then think deeply on the hardest step. Watch your token budget.

---

## 3. Controlling Context & Workflow

**Why these matter:** Claude's context window is finite. Long sessions degrade response quality as earlier context gets pushed out or diluted. These tools let you manage that window intentionally — interrupting bad directions early, recovering from mistakes, or compressing context before it overflows.

| Tool | What it does |
|------|--------------|
| `Esc` | Interrupt Claude mid-response |
| `Esc Esc` (double) | Rewind the conversation without losing new context Claude picked up |
| `/compact` | Summarize the session so far and continue — good for long sessions approaching context limits |
| `/clear` | Wipe the slate; start fresh |

**Rule of thumb**: Use `/compact` when you're deep in a task but running long. Use `/clear` between unrelated tasks.

---

## 4. Custom Commands

**Why:** Repetitive prompts — code review, writing tests, generating changelogs — waste your time and introduce variation. Custom commands encode the exact instructions once and let you trigger them consistently with a single slash command.

Automate repeatable workflows by creating custom slash commands.

**Setup:**
1. Create `.claude/commands/<command-name>.md`
2. Re-run Claude
3. Use `/<command-name>` to trigger it

**Dynamic inputs:** Include `$ARGUMENTS` in the file — Claude substitutes it with whatever you pass.

**Example — `.claude/commands/review.md`:**
```markdown
Review the following file for code quality, edge cases, and potential bugs:
$ARGUMENTS

Focus on: error handling, naming clarity, and test coverage gaps.
```

Usage: `/review @src/auth/middleware.ts`

---

## 5. MCP Servers

**Why:** Claude Code alone can only read/write files and run shell commands. MCP extends that to anything with an API — controlling a real browser for E2E testing, querying a database, calling external services. Without it, you'd have to manually run those steps yourself and paste results back in.

MCP (Model Context Protocol) lets Claude Code interact with external environments — browsers, APIs, databases, etc.

**Add a server:**
```bash
claude mcp add playwright npx @playwright/mcp@latest
```

**Avoid repeated permission prompts** by whitelisting in `.claude/settings.local.json`:
```json
{
  "permissions": {
    "allow": ["mcp__playwright"],
    "deny": []
  }
}
```

Once added, Claude auto-detects which MCP tool to use — you don't need to name it explicitly. Just describe what you want done in the browser (or whatever environment).

---

## 6. GitHub Integration

**Why:** Without this, Claude Code only works when you're actively in a session. The GitHub integration makes Claude a persistent team member — it can respond to PR comments, catch issues, and make fixes while you're doing other things. Useful for solo projects too: you open a PR, Claude reviews it, you merge. No context switching.

Run `/install-github-app` to connect Claude Code to your GitHub repo.

Once configured, Claude can act on two triggers:

| Action | What it does |
|--------|--------------|
| **Mention Action** | Tag Claude (@claude) in a PR/issue comment; it responds or makes changes |
| **PR Action** | Claude reviews or updates PRs automatically |

This enables async workflows where Claude handles review cycles without you being present.

## 7. Hooks
 
**Why:** Claude makes mistakes that only show up later — it changes a function signature but misses the call sites, or creates a duplicate query instead of reusing an existing one. By the time you notice, it's already moved on. Hooks let you catch these issues immediately and feed the error back to Claude so it self-corrects in the same session, rather than you having to chase it down manually.
 
Hooks are scripts that run automatically before or after Claude uses a tool. They can inspect what Claude is about to do, block it if needed, or run follow-up checks after the fact.
 
### Hook Types
 
| Type | When it runs | Can block? |
|------|-------------|------------|
| `PreToolUse` | Before the tool executes | Yes (exit code 2) |
| `PostToolUse` | After the tool executes | No |
 
### Configuration Locations
 
| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All projects |
| `.claude/settings.json` | Project, shared with team |
| `.claude/settings.local.json` | Project, local only (gitignored) |
 
### Hook Structure
 
```json
"PreToolUse": [
  {
    "matcher": "Read",
    "hooks": [
      {
        "type": "command",
        "command": "node /home/hooks/read_hook.js"
      }
    ]
  }
]
```
 
The `matcher` specifies which tool(s) to intercept. Use `|` to match multiple: `"Read|Grep"`.
 
### How Hooks Work
 
1. Claude attempts a tool call (e.g., `Read`)
2. If a hook matches, Claude pipes the tool call data to your script as JSON via **stdin**
3. Your script inspects it — the JSON contains `tool_name` and `input` (including `file_path`, etc.)
4. Your script exits with a code to signal intent:
 
| Exit code | Meaning |
|-----------|---------|
| `0` | Allow the tool call to proceed |
| `2` | Block it (pre-hooks only) — stderr output is sent to Claude as feedback |
 
> **Tip:** Ask Claude for a list of available tool names rather than guessing them.
 
### Example: Block `.env` Access
 
```js
// .claude/hooks/read_hook.js
process.stdin.resume();
let input = "";
process.stdin.on("data", d => input += d);
process.stdin.on("end", () => {
  const { tool_input } = JSON.parse(input);
  if (tool_input?.path?.includes(".env")) {
    console.error("Blocked: cannot read .env files.");
    process.exit(2);
  }
  process.exit(0);
});
```
> **Note:** Always restart Claude after changing hook config or scripts.
 
### Practical Use Cases
 
| Use case | Hook type | What it does |
|----------|-----------|--------------|
| **Auto-format** | PostToolUse | Run Prettier after file edits |
| **Type checking** | PostToolUse | Run `tsc --no-emit` after `.ts` edits; feed errors back to Claude |
| **Test runner** | PostToolUse | Run relevant tests after changes |
| **Access control** | PreToolUse | Block reads/writes to sensitive files |
| **Deduplication** | PostToolUse | Launch a second Claude instance to check if new code already exists |
| **Linting** | PostToolUse | Run ESLint and return violations as feedback |
 
### Real Example: TypeScript Type Checker
 
A post-hook that runs `tsc --no-emit` after every `.ts` file edit. If Claude changes a function signature but misses call sites, the type errors get fed back immediately and Claude fixes them in the same pass — instead of you catching broken code later.
 
### Real Example: Duplicate Code Detection
 
For larger projects, Claude sometimes creates a new query or utility instead of reusing an existing one — especially mid-task when it's focused on something else. A hook can watch a critical directory (e.g., `queries/`), launch a second Claude instance via the SDK to compare the new code against existing files, and block the write if it's a duplicate. Trade-off: extra time and cost per edit. Worth it only for directories where duplication is a real problem.
 
---
 
## 8. Claude Code SDK
 
**Why:** Sometimes you don't want an interactive session — you want Claude Code as a component inside a larger automated pipeline. The SDK lets you call Claude Code programmatically from TypeScript or Python, so you can build scripts, hook scripts, CI steps, or internal tools that use Claude as an intelligent sub-process.
 
The Claude Code SDK exposes the same tools as the terminal version but through a programmatic interface (CLI, TypeScript, Python).
 
**Primary use case:** integrating Claude into existing pipelines and workflows — not as a standalone assistant, but as an intelligent step inside a script or hook.
 
### Key Details
 
- **Default permissions:** read-only (files, directories, grep). Write access must be explicitly enabled.
- **Enabling writes:**
 
```ts
await query({
  prompt: "Refactor this file...",
  options: {
    allowTools: ["edit", "write"]
  }
});
```
 
- **Output format:** raw message-by-message conversation between the local Claude Code process and the model. The final message is Claude's response.
 
**Best suited for:** hook scripts, helper commands, and CI integrations within existing projects — not general standalone use.
 
---
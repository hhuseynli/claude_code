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
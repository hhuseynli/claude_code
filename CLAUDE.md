# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users chat with Claude to generate React components, which are rendered in real-time via a virtual file system and iframe preview.

## Commands

```bash
# Development
npm run setup          # First-time setup: install deps + prisma generate + migrate dev
npm run dev            # Start dev server (Turbopack)
npm run dev:daemon     # Start dev server in background, logs to logs.txt

# Production
npm run build
npm run start

# Code Quality
npm run lint           # ESLint

# Testing
npm run test           # Vitest

# Database
npm run db:reset       # Force reset Prisma database
```

All commands should be run from the `uigen/` directory.

## Architecture

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API uses Vercel AI SDK's `streamText()` with Claude model and file operation tools
3. AI responds with text and/or tool calls (create/edit files in the virtual FS)
4. Client updates `FileSystemContext` state, triggering a re-render in `PreviewFrame`
5. `PreviewFrame` uses Babel (via `src/lib/transform/jsx-transformer.ts`) to transpile JSX/TSX in-browser and render in an iframe with dynamic import maps

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`) — All files exist only in memory (no disk writes). Serialized as JSON and stored in the database on project save.

**AI Tools** (`src/lib/tools/`) — Two tool groups exposed to the AI:
- `str-replace.ts`: view, create, str_replace, insert operations on files
- `file-manager.ts`: rename and delete

**Provider** (`src/lib/provider.ts`) — Wraps `@ai-sdk/anthropic`. Falls back to `MockLanguageModel` (returns static sample components) when `ANTHROPIC_API_KEY` is absent.

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) — Runs Babel standalone in the browser to transpile component code, strips CSS imports, and builds import maps for dynamic module resolution inside the iframe.

### State Management

Two React contexts wrap the entire app:
- `FileSystemContext` — virtual file tree, selected file, file CRUD operations
- `ChatContext` — messages, streaming state, project metadata

### Authentication

JWT-based sessions (`src/lib/auth.ts`) using `jose`. Sessions stored in httpOnly cookies (7-day expiry). Passwords hashed with `bcrypt`. Server actions in `src/actions/index.ts` handle auth and project CRUD.

### Database

SQLite via Prisma. Two models:
- `User`: email, hashed password
- `Project`: name, `messages` (JSON), `data` (JSON = serialized virtual FS), userId

### UI Layout

`MainContent` (`src/app/main-content.tsx`) uses resizable panels:
- Left (35%): `ChatInterface`
- Right (65%): Tabs switching between `PreviewFrame` and `CodeEditor` + `FileTree`

## Code Style

- Minimize comments. Only add them where logic is non-obvious. Do not add docstrings or redundant inline comments.

## Environment Variables

Requires a `.env` file in `uigen/`:
```
ANTHROPIC_API_KEY=...   # Optional: falls back to mock provider if absent
SESSION_SECRET=...      # Required for JWT signing
```

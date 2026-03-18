# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-powered React component generator with live preview. Users describe components in chat, Claude generates the code, and components render in real-time.

## Commands

```bash
npm run setup          # Install deps + Prisma generate + migrations
npm run dev            # Start dev server (localhost:3000) — uses Turbopack
npm run build          # Production build
npm run test           # Run all Vitest tests
npm run test -- --reporter=verbose  # Run tests with detailed output
npm run test -- path/to/file        # Run a single test file
npm run lint           # ESLint check
npm run db:reset       # Force reset SQLite database
```

## Environment Variables

- `ANTHROPIC_API_KEY` — Optional. Falls back to `MockLanguageModel` without it.
- `JWT_SECRET` — Optional. Defaults to `"development-secret-key"`.

## Architecture

### Key Directories

- `src/app/api/chat/` — Single streaming AI endpoint (all AI interactions go here)
- `src/lib/` — Core logic: auth, virtual file system, AI provider, tools, prompts, JSX transform
- `src/lib/contexts/` — React contexts: `FileSystemContext`, `ChatContext`
- `src/lib/tools/` — AI tools: `str_replace_editor` (create/edit/view files), `file_manager` (rename/delete)
- `src/lib/prompts/` — System prompt for component generation
- `src/lib/transform/` — JSX → JS transform + import map builder for iframe preview
- `src/components/` — React UI: chat, editor (Monaco), preview (iframe), auth, shadcn/ui
- `src/actions/` — Next.js server actions for project CRUD and auth
- `prisma/` — SQLite schema: `User` and `Project` models

### Virtual File System

All component files live in an in-memory `VirtualFileSystem` (nothing written to disk). It uses a `Map` with normalized paths (all start with `/`). State serializes to JSON and persists in `Project.data` in the database. `FileSystemContext` owns the instance on the client; the API route reconstructs it from the serialized body on each request.

### AI Streaming Flow

```
User input → ChatContext (useChat hook)
  → POST /api/chat { messages, files: serialized VFS, projectId }
  → streamText() with str_replace_editor + file_manager tools (max 40 steps)
  → AI calls tools → server mutates its own VFS copy
  → streaming response to client
  → FileSystemContext.handleToolCall applies same mutations client-side
  → PreviewFrame detects refresh trigger → Babel transforms JSX → blob URL import map → iframe srcdoc
  → onFinish: saves project to DB (authenticated users only)
```

The AI tools run synchronously against the server-side VFS. The same tool calls are replayed on the client via `onToolCall` in `ChatContext` so both sides stay in sync.

### Preview Rendering

`src/lib/transform/jsx-transformer.ts` transforms each VFS file with Babel standalone, creates blob URLs, and builds an ES module import map. `@/` aliases resolve to local blob URLs; third-party packages fall back to `esm.sh`; missing imports get placeholder modules. CSS imports are collected and injected into the iframe `srcdoc`. The entrypoint is `App.jsx` with fallback to `index.jsx`.

### Mock Provider

When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` uses `MockLanguageModel`, which simulates a 4-step agentic flow and returns hardcoded components (Counter, Form, or Card based on prompt keywords). Useful for UI/UX development without incurring API costs.

### Authentication

JWT (HS256, 7-day expiry) stored in httpOnly cookies. Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Anonymous users are supported — projects have a nullable `userId`. Anonymous work is tracked in `sessionStorage` via `anon-work-tracker.ts` for recovery.

### Database Schema

- `User`: id (CUID), email (unique), password (bcrypt)
- `Project`: id, name, userId (nullable), messages (JSON string), data (JSON string = serialized VFS), timestamps

### AI System Prompt

`src/lib/prompts/generation.tsx` enforces: use Tailwind CSS (no hardcoded styles), `/App.jsx` as entrypoint, `@/` for cross-file imports. Prompt caching (Anthropic ephemeral) is applied to reduce token usage on repeat requests.

## Path Aliases

- `@/*` → `./src/*`

## Hooks (Automated on Edit)

Configured in `.claude/settings.local.json`. These run automatically — no action needed:

- **Prettier** — runs after every `Write`/`Edit` to auto-format the saved file.
- **TypeScript check** (`hooks/tsc.js`) — runs after every `Write`/`Edit` to a `.ts`/`.tsx` file. Exits with code 2 (blocking) if there are type errors; fix them before proceeding.

## Testing

Vitest with jsdom environment. Test files are co-located with source in `__tests__/` subdirectories.

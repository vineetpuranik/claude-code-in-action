# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator. Users chat with Claude, which uses tool calls to create/modify files in a virtual (in-memory) file system. Generated components render live in a preview pane.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server (Turbopack) at http://localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Run all Vitest tests
npm run db:reset    # Force reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

Environment: add `ANTHROPIC_API_KEY` to `.env`. Without it, the app uses a mock provider automatically.

## Architecture

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route serializes the virtual file system into the system prompt, calls Claude via Vercel AI SDK with streaming
3. Claude invokes tools (`str_replace_editor`, `file_manager`) to create/modify code files
4. Tool results update the virtual file system; the stream returns updated file states to the client
5. The client re-renders the live preview

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`) — All generated code lives in memory, never on disk. Serialized to JSON and stored in the `Project.data` DB column. The AI tools operate on this abstraction.

**AI Provider** (`src/lib/provider.ts`) — Wraps `@ai-sdk/anthropic`. Falls back to a mock provider when no API key is set. Uses Anthropic prompt caching (`ephemeral` cache control) on the system prompt.

**Chat Context** (`src/lib/contexts/chat-context.tsx`) — Manages streaming state, message history, and file system updates on the client.

**File System Context** (`src/lib/contexts/file-system-context.tsx`) — Client-side state for the virtual FS, shared between the editor, file tree, and preview.

### AI Tools (`src/lib/tools/`)

- `str-replace.ts` — Targeted string replacement in existing files
- `file-manager.ts` — Create and delete files

### UI Layout (`src/app/main-content.tsx`)

Three resizable panels via `react-resizable-panels`:
- **Left**: Chat (`src/components/chat/`)
- **Center**: Code editor with file tree (`src/components/editor/`)
- **Right**: Live preview (`src/components/preview/`)

### Auth & Persistence

- JWT sessions via `jose`, passwords hashed with `bcrypt` (`src/lib/auth.ts`)
- Prisma + SQLite (`prisma/schema.prisma`): `User` and `Project` models
- Projects store the full message history and virtual FS as JSON blobs
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`
- Users without accounts can still generate components; projects save anonymously

### Database

All database files are in `prisma/`: `schema.prisma` (models), `dev.db` (SQLite file), and `migrations/`. Reference these whenever understanding or modifying the database structure.

### Path Alias

`@/*` maps to `src/*`.

## Code Style

Use comments sparingly — only where the logic is genuinely non-obvious.

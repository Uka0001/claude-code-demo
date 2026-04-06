# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack, requires node-compat.cjs shim)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Run tests matching a name pattern
npx vitest run -t "should create a file"

# Reset the SQLite database
npm run db:reset
```

## Architecture

### Overview

UIGen is a Next.js 15 (App Router) application that lets users describe React components in a chat interface, then generates and live-previews them. The key insight is that **no files are ever written to disk** — all generated code lives in an in-memory `VirtualFileSystem`.

### Core Data Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Vercel AI SDK streams a response from Claude with tool calls
3. Claude uses `str_replace_editor` and `file_manager` tools to create/edit virtual files
4. Tool calls are intercepted client-side by `FileSystemContext.handleToolCall` and applied to the in-memory VFS
5. `PreviewFrame` re-renders on every VFS change: it transforms all files with Babel (in-browser), builds an import map resolving local imports as blob URLs and npm packages via `esm.sh`, and writes an HTML string into an iframe's `srcdoc`

### Virtual File System (`src/lib/file-system.ts`)

`VirtualFileSystem` is a simple in-memory tree of `FileNode` objects. It supports create/read/update/delete/rename plus serialization. The project data column in the DB stores the serialized VFS as JSON. The client deserializes it on load.

### Preview Pipeline (`src/lib/transform/jsx-transformer.ts`)

- `transformJSX` — Babel-transforms JSX/TSX to plain JS in-browser
- `createImportMap` — iterates all VFS files, transforms each, creates blob URLs, resolves `@/` aliases, and maps npm package names to `https://esm.sh/<package>`
- `createPreviewHTML` — produces a full HTML document with the import map injected and an `App.jsx`/`index.jsx` entry point loaded via `<script type="module">`

### AI Tools (`src/lib/tools/`)

Two tools are exposed to Claude at the API layer:
- `str_replace_editor` — view, create, str_replace, and insert operations on VFS files
- `file_manager` — rename and delete operations

These tools operate on a server-side `VirtualFileSystem` instance during streaming, but their effects are also replayed client-side via `onToolCall` in `ChatContext` so the UI stays in sync without waiting for the stream to finish.

### State Management

Two React contexts wrap the app:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance, exposes file operations, and provides a `refreshTrigger` counter that signals the preview to re-render
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, serializes the VFS into each request body, and forwards tool calls to `FileSystemContext`

### Authentication

Custom JWT auth using `jose`. Sessions are stored in an HTTP-only cookie (`auth-token`, 7-day expiry). `src/lib/auth.ts` is marked `server-only`. The middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes. `/api/chat` is open but project persistence is skipped if no valid session exists.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User` — email/password (bcrypt)
- `Project` — `messages` (JSON string array) and `data` (JSON string of serialized VFS nodes), nullable `userId` for anonymous sessions

Prisma client is generated into `src/generated/prisma/`.

### Mock Provider

If `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a mock model that echoes static code. The chat route caps `maxSteps` at 4 for the mock vs. 40 for real Claude.

### Node Compatibility Shim

`node-compat.cjs` is `--require`d via `NODE_OPTIONS` in all npm scripts. This is needed because Next.js with Turbopack has incompatibilities with certain Node.js built-ins used by Prisma/bcrypt.
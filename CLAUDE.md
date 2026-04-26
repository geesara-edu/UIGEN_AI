# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** is an AI-powered React component generator. Users describe components in natural language; Claude generates them in real-time with a live preview, code editor, and virtual file system — all in the browser.

## Commands

```bash
# Initial setup (install deps + Prisma generate + migrate)
npm run setup

# Development (http://localhost:3000)
npm run dev

# Build & start production
npm run build && npm start

# Lint
npm run lint

# Tests (Vitest + jsdom)
npm test

# Run a single test file
npx vitest src/path/to/file.test.ts

# Database reset (destructive)
npm run db:reset
```

The dev server requires `NODE_OPTIONS="--require ./node-compat.cjs"` (handled by npm scripts) to strip browser globals (`localStorage`, `sessionStorage`) that break SSR on Node 25+.

## Environment

Create a `.env` file with:
```
ANTHROPIC_API_KEY=your_key_here
```

If `ANTHROPIC_API_KEY` is absent, the app falls back to a mock provider that returns static placeholder components (limited to 4 tool-use steps vs. 40 for real Claude).

## Architecture

### Request Flow

1. User types in chat → `MessageInput` → `ChatContext` → `POST /api/chat`
2. `/api/chat/route.ts` calls Claude via Vercel AI SDK with tool definitions
3. Claude calls tools (`str_replace_editor`, `file_manager`) to create/modify files
4. Streaming response updates `FileSystemContext` (in-memory virtual FS)
5. `PreviewFrame` transpiles JSX in-browser via `@babel/standalone` and renders it

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — All generated files live in memory; nothing is written to disk. This class manages the in-memory file tree and is serialized to JSON for database persistence.

**`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — Client-side React context wrapping `VirtualFileSystem`. All components read/write files through this context.

**`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Owns the Vercel AI SDK `useChat` hook, handles streaming, and bridges chat messages to file system mutations.

**AI Tools** (`src/lib/tools/`) — Two tools Claude uses to generate components:
- `str_replace_editor` — targeted string replacement within existing files
- `file_manager` — create or delete files

**`PreviewFrame`** (`src/components/preview/`) — Renders the generated component by transpiling the virtual FS entry point in-browser with Babel standalone.

### Authentication

JWT tokens in HTTP-only cookies (7-day expiry). `src/lib/auth.ts` handles signing/verifying tokens with `jose`. Passwords hashed with `bcrypt`. `src/middleware.ts` guards API routes. Anonymous users can generate components; their work is tracked in `localStorage` via `src/lib/anon-work-tracker.ts` but not persisted to the database.

### Persistence

SQLite via Prisma (`prisma/schema.prisma`). Two models: `User` and `Project`. A `Project` stores the project name, the full chat history (JSON), and the serialized `VirtualFileSystem` state.

### UI Layout

Three-panel resizable layout in `src/app/main-content.tsx`:
- Left: Chat panel (35%)
- Right: Code editor / Preview tabs (65%)

## Tech Stack Highlights

- **Next.js 15** App Router with React 19 server/client components
- **Tailwind CSS v4** (PostCSS plugin, not the CLI)
- **Vercel AI SDK** (`ai` package) for streaming and tool-calling — not the raw Anthropic SDK
- **Monaco Editor** for the code editor panel
- **Radix UI + shadcn/ui** (New York style) for headless primitives; components live in `src/components/ui/`
- **Vitest** with jsdom and React Testing Library for unit tests
- **Prisma 6** with SQLite (file: `./dev.db`)

## Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).

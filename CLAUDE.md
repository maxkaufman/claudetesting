# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset the database (destructive)
npm run db:reset
```

The app runs at `http://localhost:3000`. Set `ANTHROPIC_API_KEY` in `.env` to use the real Claude API; without it, a mock provider returns static hardcoded components.

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat and see them rendered live in an iframe preview.

### Request flow

1. User types a message in `ChatInterface` → `ChatProvider` (via Vercel AI SDK `useChat`) sends a POST to `/api/chat/route.ts`
2. The API route reconstructs a `VirtualFileSystem` from the serialized `files` payload, then calls `streamText` with two AI tools: `str_replace_editor` and `file_manager`
3. Streamed tool calls come back to the client; `ChatContext` forwards each tool call to `FileSystemContext.handleToolCall`, which mutates the in-memory `VirtualFileSystem`
4. `PreviewFrame` watches `refreshTrigger` from `FileSystemContext`, calls `createImportMap` + `createPreviewHTML` (in `src/lib/transform/jsx-transformer.ts`), and writes the result into an iframe's `srcdoc`
5. The preview uses Babel Standalone to transpile JSX/TSX in-browser and an ES module import map with blob URLs; third-party packages are resolved from `esm.sh` on the fly

### Key abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — In-memory tree of `FileNode`s. Nothing is ever written to disk on the user's machine. Supports `serialize()` / `deserializeFromNodes()` for round-tripping through the database.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context wrapping `VirtualFileSystem`. All AI tool call mutations go through `handleToolCall` here, which triggers `refreshTrigger` to update the preview.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Thin wrapper around `useChat` from `@ai-sdk/react`. Passes the serialized file system in the request body on every message.
- **AI tools** — `str_replace_editor` (`src/lib/tools/str-replace.ts`) handles `create`, `str_replace`, `insert`, `view` commands on the virtual FS. `file_manager` (`src/lib/tools/file-manager.ts`) handles `rename` and `delete`.
- **JSX transformer** (`src/lib/transform/jsx-transformer.ts`) — Runs Babel in the browser to produce blob URLs for each file, builds an import map, injects Tailwind CDN, and generates the full preview HTML document.
- **`MockLanguageModel`** (`src/lib/provider.ts`) — Implements `LanguageModelV1` and returns hardcoded component code when no API key is set.

### Auth & persistence

- JWT-based sessions via `jose`, stored in an httpOnly cookie (`src/lib/auth.ts`)
- Prisma + SQLite (`prisma/dev.db`). Prisma client is generated to `src/generated/prisma/`
- `Project` model stores `messages` and `data` (serialized `VirtualFileSystem`) as JSON strings
- Anonymous users get work tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts`; on sign-up, their work is offered to be saved
- Authenticated users are redirected to their most recent project on load (`src/app/page.tsx`)

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live in `__tests__` subdirectories next to the code they test.

## Database

The database schema is defined in `prisma/schema.prisma`. Reference it anytime you need to get information from the database.

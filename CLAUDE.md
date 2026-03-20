# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and Claude generates JSX files into a virtual file system that renders in a live preview iframe.

## Commands

- `npm run setup` — Install deps, generate Prisma client, run migrations (first-time setup)
- `npm run dev` — Start dev server with Turbopack (requires `--require ./node-compat.cjs`)
- `npm run build` — Production build
- `npm run lint` — ESLint
- `npm run test` — Vitest (all tests)
- `npx vitest run src/lib/__tests__/file-system.test.ts` — Run a single test file
- `npx prisma migrate dev` — Run pending migrations
- `npm run db:reset` — Reset database (destructive)

## Architecture

### AI Generation Flow

1. User sends a message via the chat UI → `POST /api/chat` (route: `src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from the client-sent file state, prepends the system prompt from `src/lib/prompts/generation.tsx`, and calls `streamText` from the Vercel AI SDK
3. The LLM has two tools: `str_replace_editor` (create/view/edit files) and `file_manager` (rename/delete) — both operate on the `VirtualFileSystem`
4. Tool calls stream back to the client where `FileSystemContext` mirrors changes into the client-side `VirtualFileSystem`
5. The preview panel transforms files with `@babel/standalone` (`src/lib/transform/jsx-transformer.ts`), builds an import map with blob URLs and esm.sh for third-party packages, and renders in an iframe

### Virtual File System

`src/lib/file-system.ts` — In-memory tree with `FileNode` types. Exists on both server (for tool execution during generation) and client (for UI state). No files are written to disk. Serialized as JSON for persistence.

### Key Contexts (Client)

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — Owns the client-side VFS, handles tool call side-effects, manages selected file state
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — Wraps `useChat` from `@ai-sdk/react`, sends file system state with each request

### Mock Provider

`src/lib/provider.ts` — When `ANTHROPIC_API_KEY` is not set, a `MockLanguageModel` returns static responses so the app runs without an API key. The real provider uses `claude-haiku-4-5`.

### Auth & Data

- JWT sessions via `jose` (cookie-based), managed in `src/lib/auth.ts`
- Server actions in `src/actions/` handle signup/signin/signout and project CRUD
- Prisma + SQLite — schema at `prisma/schema.prisma`, Prisma client generated to `src/generated/prisma`
- Anonymous users can use the app without auth; registered users get project persistence
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes

### UI

- shadcn/ui components in `src/components/ui/` (New York style, Tailwind CSS v4)
- Main layout: resizable left chat panel + right preview/code panel (`src/app/main-content.tsx`)
- Path alias: `@/*` maps to `./src/*`

## Testing

Tests use Vitest + jsdom + React Testing Library. Test files live in `__tests__/` directories next to their source. The preview iframe uses Tailwind CDN and esm.sh for runtime dependencies.

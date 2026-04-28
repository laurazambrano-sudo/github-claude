# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + prisma generate + migrate
npm run dev          # Start dev server (Next.js with Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (jsdom environment)
npm run db:reset     # Reset and re-run all Prisma migrations
```

Requires a `.env` file with `ANTHROPIC_API_KEY` for AI features to work.

## Architecture

**UIGen** is a Next.js 15 (App Router) AI-powered React component generator. Users chat with Claude to generate components that are previewed live in an iframe.

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route calls Vercel AI SDK `streamText` with Claude via `@ai-sdk/anthropic`
3. Claude uses two tools to write/edit files:
   - `str_replace_editor` — create or patch file contents
   - `file_manager` — rename/delete files
4. Tool results update a `VirtualFileSystem` (in-memory, no disk writes) held in the request body
5. The updated file tree is streamed back to the client and displayed in Monaco Editor + preview iframe

### Key Modules

| Path | Purpose |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; owns VFS lifecycle and DB persistence |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory file store serialized as JSON |
| `src/lib/provider.ts` | Initializes the Anthropic language model |
| `src/lib/prompts/` | System prompt and tool descriptions sent to Claude |
| `src/lib/tools/` | Zod schemas + handlers for `str_replace_editor` and `file_manager` |
| `src/lib/auth.ts` | JWT session (7-day) using `jose`; cookie-based |
| `src/actions/` | Next.js server actions for user/project CRUD |
| `src/components/preview/` | Iframe preview that executes the generated component code |
| `src/components/editor/` | Monaco Editor + file tree |

### Data Model (Prisma + SQLite)

- `User` — email/bcrypt-hashed password; optional (anonymous sessions supported)
- `Project` — `messages` (JSON string) + `data` (serialized VirtualFileSystem JSON); linked to `User` optionally

Projects are persisted only when the user is authenticated; anonymous users work entirely in-memory.

### Authentication

JWT stored as an HTTP-only cookie (`session`). `src/lib/auth.ts` exports `getSession`, `setSession`, and `deleteSession`. Server actions in `src/actions/` call `getSession` to gate mutations.

### Styling

Tailwind CSS v4 with the PostCSS plugin. Component primitives come from Radix UI, assembled shadcn/ui-style in `src/components/ui/`.

### Testing

Vitest with `jsdom`. Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

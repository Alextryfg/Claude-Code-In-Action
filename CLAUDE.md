# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack + node-compat polyfill)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/editor/__tests__/file-tree.test.tsx

# Reset the database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run new migrations
npx prisma migrate dev
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix is applied automatically by all npm scripts — it patches Node.js built-ins for Next.js compatibility. Don't run `next` directly without it.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat, Claude generates them using tool calls, and a live preview renders the result in an iframe — all without writing any files to disk.

### Data flow

1. **Chat** (`src/app/api/chat/route.ts`) — receives messages + serialized virtual FS state, calls Claude (or the mock provider) via Vercel AI SDK's `streamText`, exposes two tools: `str_replace_editor` and `file_manager`.
2. **Virtual file system** (`src/lib/file-system.ts`) — an in-memory tree (`VirtualFileSystem`) that lives only client-side. It is serialized to plain JSON for each API request and reconstructed server-side; mutations happen on the server during tool calls, then the updated state is streamed back via `onToolCall` callbacks.
3. **Contexts**
   - `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the singleton `VirtualFileSystem` instance, exposes CRUD helpers and `handleToolCall` which applies incoming tool calls to the local FS.
   - `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps the AI SDK's `useChat`, wires `onToolCall` → `handleToolCall`, and sends the current serialized FS with every request.
4. **Preview** (`src/components/preview/PreviewFrame.tsx`) — on every `refreshTrigger` it transpiles all virtual files via Babel Standalone (`src/lib/transform/jsx-transformer.ts`), creates blob URLs, builds an import map, and writes the result into an `<iframe srcdoc>`. Third-party npm packages are resolved via `https://esm.sh/`.
5. **Persistence** — authenticated users can save projects; `messages` and virtual FS data are stored as JSON strings in SQLite via Prisma (`prisma/schema.prisma`). Anonymous work is tracked in `src/lib/anon-work-tracker.ts` using `localStorage`.

### Provider / model

`src/lib/provider.ts` checks for `ANTHROPIC_API_KEY`. When absent, it returns `MockLanguageModel` (a scripted stub that generates a counter/card/form component in 4 tool-call steps). When present, it uses `claude-haiku-4-5` via `@ai-sdk/anthropic`.

### Auth

JWT-based, cookie-stored (`auth-token`). `src/lib/auth.ts` signs/verifies tokens with `jose`. `src/middleware.ts` protects routes. Passwords are hashed with `bcrypt`.

### Key conventions

- All server-only code is guarded with `"server-only"` imports.
- The `@/` path alias maps to `src/` (configured in `tsconfig.json` and `vitest.config.mts`).
- Prisma client is generated to `src/generated/prisma` (not `node_modules`).
- Tests use Vitest + jsdom + React Testing Library; test files live in `__tests__/` directories alongside the code they test.
- UI components in `src/components/ui/` are shadcn/ui primitives (Radix + Tailwind). Don't modify them directly.

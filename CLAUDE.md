# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev        # Start dev server (Next.js + Turbopack) at http://localhost:3000
npm run build      # Production build
npm run setup      # Install deps + Prisma generate + migrate (first-time setup)
npm run test       # Run Vitest tests
npm run db:reset   # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.tsx
```

## Critical Dependency Versions

The project uses **`ai@4.3.16`** — do NOT upgrade to v5+. The API surface changed completely:
- v4 uses `toDataStreamResponse()`, `appendResponseMessages()`, and `parameters:` in tool definitions
- v4 uses `@ai-sdk/react@1.2.12` (streaming protocol: `"data"` with `0:`, `d:`, `f:` prefixes)
- v5 uses `toUIMessageStreamResponse()` and `inputSchema:` in tool definitions, requiring `@ai-sdk/react@2.x`

These are intentionally pinned in `package-lock.json`. Use `npm ci` not `npm install` to preserve exact versions.

## Architecture

### Core Pipeline

User prompt → `/api/chat` → Claude streams tool calls → `FileSystemContext.handleToolCall()` updates in-memory VFS → `refreshTrigger` increments → `PreviewFrame` re-renders iframe

All files exist only in memory (`VirtualFileSystem` class in `src/lib/file-system.ts`). Nothing is written to disk at runtime.

### Context Hierarchy

```
FileSystemProvider (src/lib/contexts/file-system-context.tsx)
  └─ ChatProvider (src/lib/contexts/chat-context.tsx)
       └─ UI components
```

`FileSystemContext` owns the VFS and exposes `handleToolCall()`. `ChatContext` wraps `useChat` from `@ai-sdk/react` and forwards tool calls to `handleToolCall`. The `body` sent to `/api/chat` includes a serialized snapshot of the entire VFS on every request.

### AI Tool Definitions (`src/lib/tools/`)

Two tools the model can call:
- **`str_replace_editor`** — `view`, `create`, `str_replace`, `insert`, `undo_edit` on paths
- **`file_manager`** — `rename`, `delete` on paths

Both receive a `VirtualFileSystem` instance and operate on it directly. Results stream back to the client.

### Preview Pipeline (`src/components/preview/PreviewFrame.tsx`)

On each `refreshTrigger` change:
1. All `.jsx/.tsx/.ts/.js` files are transpiled via `@babel/standalone` (JSX + TS)
2. Each file is converted to a blob URL
3. An import map is built: `@/` aliases → blob URLs, third-party packages → `esm.sh`
4. A self-contained HTML document is injected into `iframe.srcdoc`

The iframe runs in `allow-scripts allow-same-origin allow-forms` sandbox. It must have `allow-same-origin` for blob URL imports to work.

### Authentication (`src/lib/auth.ts`)

JWT signed with `JWT_SECRET` env var stored in an `httpOnly` cookie (`auth-token`, 7-day expiry). `getSession()` is server-only (imported from `server-only`). Anonymous users can use the app without signing in — projects are only saved to SQLite when a session exists.

### Database (Prisma + SQLite)

Two models: `User` (email + bcrypt password) and `Project` (name, userId?, messages JSON string, data JSON string). Both `messages` and `data` are stored as raw JSON strings, not normalized. `userId` is nullable to support anonymous projects (though anonymous projects are never actually persisted — the `onFinish` callback in `/api/chat/route.ts` only writes if `projectId` is provided and the user is authenticated).

### Language Model (`src/lib/provider.ts`)

Returns `anthropic("claude-haiku-4-5")` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that streams pre-scripted responses (useful for development without an API key). `maxSteps` is 40 for real, 4 for mock.

### System Prompt

Defined in `src/lib/prompts/generation.tsx`. Key rules the model follows:
- Always create `/App.jsx` as the entry point
- Use `@/` import alias for local files (e.g., `import Button from '@/components/Button'`)
- Tailwind CSS only, no inline styles
- `.jsx`/`.tsx` files only (no HTML)

The system message uses Anthropic's `cacheControl: { type: "ephemeral" }` to reduce token costs on repeated requests.

### Routing

- `/` — anonymous landing or redirect to most recent project if authenticated
- `/[projectId]` — load a specific project (requires auth, verifies ownership)
- `/api/chat` — streaming POST endpoint

### Anonymous Work Tracking

`src/lib/anon-work-tracker.ts` uses `sessionStorage` to remember if an anonymous user has generated components, so they can be prompted to save by signing up.

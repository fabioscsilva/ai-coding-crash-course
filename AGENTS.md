# AGENTS.md

Cadence is a course platform (a mini Udemy) and the exercise repo for the AI Coding Crash Course. Scripts live in `package.json`; setup and course navigation live in `README.md`.

## Commands

- `npm run typecheck` runs `react-router typegen` before `tsc`. Bare `tsc` reports missing `./+types/*` imports.
- Formatting is Prettier (`.prettierrc`). There is no linter.
- `data.db` is disposable: delete it and run `npm run db:seed` whenever it gets into a bad state. `scripts/seed.ts` is the source of truth for starting data.

## Architecture

### Layering

Requests flow **route → service → Drizzle**. Routes own HTTP concerns (loaders, actions, redirects, status codes) and rendering. Services export plain synchronous functions with positional parameters and use the shared `db` from `~/db` directly; tests swap it by mocking the module.

### Routing

Routing is config-based: a new page needs its module in `app/routes/` **and** an entry in `app/routes.ts`. Most pages sit inside `routes/layout.app.tsx`.

### Access control

`app/lib/access.server.ts` centralises authorisation. `getAccess(request, courseId)` returns an `Access` record and never throws, so lesson pages can render a paywall preview. The throwing guards (`requireUserId`, `requireContentAccess`, `requireStaff`, `requireInstructorOrAdmin`) are for callers that reject outright.

Loaders and actions are authorised independently: every mutating action calls a `require*` guard itself.

There is no password auth. The session holds a `userId` set from the DevUI panel (`app/components/dev-ui.tsx`); a 401 reading "Select a user from the DevUI panel" means no user is selected. The panel also overrides the viewer's country for PPP testing — a student who bought at a regional discount loses content access (`pppBlocked`) when browsing from a full-price country.

### Validation

Route actions and loaders parse input with the Zod helpers in `app/lib/validation.ts`.

### Markdown

Two renderers, not interchangeable:

- `app/lib/markdown.server.ts` — lesson content and sales copy. Trusted; raw HTML passes through.
- `app/lib/comment-markdown.server.ts` — student comments. Escapes HTML, restricts links, drops images.

Both feed `dangerouslySetInnerHTML`; treat changes to the comment renderer as security-sensitive.

### Testing

`createTestDb` in `app/test/setup.ts` builds an in-memory database from the real migrations; `seedBaseData` adds a user, instructor, category, and course. Mock `~/db` with a getter and import the module under test **after** the mock, so the service sees each fresh database:

```ts
let testDb: ReturnType<typeof createTestDb>;

vi.mock("~/db", () => ({
  get db() {
    return testDb;
  },
}));

import { getCourseById } from "~/services/courseService";

beforeEach(() => {
  testDb = createTestDb();
});
```

## Conventions

- Service and lib modules open with a `// ─── Name ───` header recording their responsibility and the reasoning behind non-obvious decisions. Read it before changing the module.
- Enums are TypeScript `enum`s from `app/db/schema.ts`, used at every insert site including `scripts/seed.ts`.
- Server-only modules use the `.server.ts` suffix.
- Lesson slugs (the commit prefixes on `live-run-through`) are a public interface for `ai-hero-cli` and recorded video. Keep them stable.

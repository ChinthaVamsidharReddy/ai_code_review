# Architecture

## Frontend Architecture

**Stack**: Next.js 14 (App Router), TypeScript, Tailwind CSS. No server-side rendering of authenticated data — the app is a client-rendered SPA-style experience served by Next.js, talking to the NestJS API over `fetch`.

**Structure** (`frontend/src`):
- `app/` — routes: `/` (public landing page — redirects to `/dashboard` automatically if already signed in), `/login`, `/register`, `/dashboard`, `/providers`, `/projects/[id]` (tabbed: Explorer, Run Review, Review History, Chat, Docs & Architecture).
- `components/` — shared UI (Toast, Spinner, EmptyState, ProtectedRoute, SeverityBadge, TopNav, ThemeToggle, MarkdownView) and `components/project/*` — the project-workspace pieces (FileTree, CodeViewer, UploadPanel, ReviewPanel, ReviewDetail, ReviewHistoryPanel, ChatPanel, DocsPanel).
- `lib/api.ts` — a single `fetch` wrapper: attaches the bearer token, unwraps the backend's `{ success, data }` envelope, normalizes errors into `ApiError`.
- `lib/auth-context.tsx` — React context holding the current user; login/register/logout persist the token + user to `localStorage`.
- `lib/theme-context.tsx` — theme preference (light/dark/system), described below.
- `types/api.ts` — shared request/response types, hand-kept in sync with the backend DTOs/entities (see "Trade-offs" below for why this isn't code-generated).

**Theming**: the app defaults to the OS/browser's light-or-dark preference and lets the user override it explicitly via a three-way toggle (Light / Dark / System) in the top nav, persisted to `localStorage`. Colors are defined as CSS custom properties on `:root` (light — a warm, cream-based palette, not a plain-white theme) and re-defined under `[data-theme='dark']`; Tailwind's color tokens (`bg-surface`, `bg-panel`, `border-border`, `text-fg`, `text-muted`, `bg-accent`, …) resolve through those variables (`rgb(var(--x) / <alpha-value>)`), so most components need no `dark:` variant at all — the same class works in both themes because the variable behind it changes. A blocking inline script in `layout.tsx`'s `<head>` sets the initial `data-theme` attribute before first paint (reading `localStorage`, falling back to `prefers-color-scheme`) to avoid a flash of the wrong theme; `ThemeProvider` takes over after hydration and also listens for live OS theme changes when the user hasn't explicitly overridden the default. A `.dark` class is toggled alongside `data-theme` specifically so `@tailwindcss/typography`'s `prose` classes (used by `MarkdownView`) have a working `dark:` hook, since that plugin's defaults aren't CSS-variable-based.

**Markdown rendering**: AI-generated Markdown (Architecture Analysis, generated docs) is rendered as real HTML — headings, lists, code blocks, tables — via `MarkdownView` (`marked` → `DOMPurify.sanitize` → Tailwind `prose` styling), not shown as raw Markdown text in a `<pre>` block. The response is treated as untrusted content and sanitized before being injected, since it originates from a third-party AI provider.

**State management**: local component state + one small auth context. No global store (Redux/Zustand) — the app's state is mostly "what does this screen need," and prop drilling stays shallow because of the tab-based project workspace.

**Auth token storage**: the JWT lives in `localStorage`, not an httpOnly cookie. This is simpler to wire up for an assessment-scale app but is readable by any script running on the page (XSS risk). A production version should move to httpOnly, `SameSite=strict` cookies plus CSRF protection on state-changing requests.

## Backend Architecture

**Stack**: NestJS (modular, dependency-injected), TypeORM, PostgreSQL.

**Module boundaries** (`backend/src`):

```
auth/            registration, login, logout, JWT issuing/validation
users/           user persistence
projects/        project CRUD + the single ownership-check chokepoint
files/           safe ZIP ingestion, file tree, file content reads
ai-providers/    provider config CRUD, encryption, and the OpenAI-compatible client
reviews/         review templates, context budgeting, the review engine itself
chat/            keyword-based context retrieval + AI chat
docs-generator/  bonus: README / setup guide / API docs generation
architecture-analysis/  bonus: AI architecture summary
common/          cross-cutting: exception filter, response envelope, JWT guard, decorators, crypto util
```

Each module owns one responsibility and depends on the modules below it (e.g. `reviews` depends on `files` and `ai-providers`, never the reverse). Controllers stay thin — they resolve ownership via `ProjectsService.getOwnedProject()` and delegate everything else to a service.

**Request flow example (running a review)**:
`ReviewsController.create` → `ProjectsService.getOwnedProject` (auth + ownership) → `ReviewsService.create` → `FilesService` (load file contents) → `context-builder.ts` (fit within a character budget, prioritizing source files) → `review-templates.ts` (mode-specific system prompt + shared JSON output contract) → `AiProvidersService.resolveClient` (decrypt key, build an `OpenAiCompatibleClient`) → parse + validate the JSON response defensively → persist `Review` + `ReviewIssue` rows → return to client.

**Error handling**: a single `AllExceptionsFilter` (`common/filters/http-exception.filter.ts`) normalizes every error response to `{ success: false, statusCode, error, message, path, timestamp }` and never leaks stack traces or driver errors to the client — unexpected errors are logged server-side with full detail and returned to the client as a generic message.

**AI provider failures are not application failures**: `ReviewsService.create` catches `AiProviderError` (a well-known, typed error the client throws for timeout / rate-limit / invalid key / invalid model / empty or malformed response / network / context-too-large) and persists a `status: 'failed'` `Review` row with a human-readable `errorMessage`, instead of bubbling a 500. Review history stays complete and honest even when a review attempt failed.

## Database Design

```
users            id, email (unique), passwordHash, displayName, timestamps
projects         id, name, description, ownerId → users, timestamps
                   index: (ownerId, createdAt) — the dashboard's main query
code_files       id, projectId → projects (cascade delete), relativePath,
                   fileName, extension, sizeBytes, isBinary, createdAt
                   unique index: (projectId, relativePath)
ai_provider_configs  id, userId, name, providerType, baseUrl,
                   apiKeyEncrypted (nullable), model, enabled, isDefault, timestamps
reviews          id, projectId → projects (cascade), requestedByUserId, mode,
                   scope, filePaths (array), summary, generalRecommendations (array),
                   status, errorMessage, aiModel, createdAt
                   index: (projectId, createdAt) — review history's main query
review_issues    id, reviewId → reviews (cascade), title, description,
                   severity, filePath, lineHint, recommendation
chat_sessions    id, projectId → projects (cascade), userId, title, timestamps
chat_messages    id, sessionId → chat_sessions (cascade), role, content,
                   contextFilePaths (array), createdAt
```

**Ownership and cascades**: every project-scoped table cascades on project delete (`onDelete: 'CASCADE'`), so removing a project cleans up its files, reviews, issues, chat sessions, and messages in one operation — no orphaned rows. `ProjectsService.getOwnedProject()` is the single place ownership is checked; every controller for files/reviews/chat/docs/architecture-analysis calls it before doing anything else, so authorization can't accidentally be skipped in one route and enforced in another.

**Pagination**: `projects` and `reviews` listings are paginated (`page`/`pageSize` query params) rather than returned in full, so both stay cheap as a user's history grows.

**Indexes**: added on the columns each module's main listing query actually filters/sorts by (`projects(ownerId, createdAt)`, `reviews(projectId, createdAt)`, `code_files(projectId, relativePath)` unique for upsert-by-path semantics).

## AI Integration Flow

1. **Provider abstraction** (`ai-providers/ai-client.interface.ts`): every AI-calling module (reviews, chat, docs-generator, architecture-analysis) depends only on the `AiClient` interface (`chat(messages, options) → { content, model, usage }`) and the typed `AiProviderError`. They never know or care whether the underlying provider is OpenAI, OpenRouter, Groq, LM Studio, or Ollama.
2. **One implementation covers all of them**: `OpenAiCompatibleClient` talks to any `/chat/completions` endpoint that follows the OpenAI request/response shape — which covers every provider in the assessment's list, cloud and local, without one class per vendor. Adding a new OpenAI-compatible provider later is a config row, not a code change.
3. **Configuration is runtime data, not code**: `AiProviderConfig` rows hold `baseUrl`, `model`, `providerType` (a label), `enabled`, `isDefault`, and an **encrypted** API key (AES-256-GCM, key derived from `JWT_SECRET` — see `common/crypto.util.ts`). The API key is never returned to the frontend in plaintext (`AiProvidersService.toSafeDto` strips it down to a `hasApiKey` boolean).
4. **Context building is tiered and adaptive, not a single fixed budget** (`reviews/context-builder.ts`, reused by chat and docs-generator; sizing tuned specifically against free-tier providers like Groq, which reject oversized requests far below what a paid OpenAI plan allows). Each AI-calling module defines a small ladder of budget tiers (e.g. review: `standard` 10k chars → `reduced` 5k → `minimal` 2.5k), largest/most-complete first. `ai-providers/with-resilience.ts` runs the ladder: on a provider's `context_too_large` error it retries with the next, smaller tier instead of failing outright; on `rate_limited` it waits briefly and retries the same tier once. Every tier still prioritizes source files over config/markdown and truncates individual files that don't fit (flagged in the prompt so the model knows its view is partial) rather than dropping them silently. Chat additionally has a zero-file "overview only" final tier (just the project name/description and file list) so a purely factual question never fails just because the app defaulted to attaching file content it didn't need.
5. **Chat's context retrieval** (`chat/context-retrieval.ts`) is a deliberately simple keyword-overlap scorer — path matches weighted higher than body matches, common stopwords filtered out, and a minimum relevance score required before a file counts as a match at all (below that, zero files are attached — see point 4's "overview only" tier — rather than a weak guess). It's isolated behind one function so it can be swapped for embeddings-based retrieval later without touching `ChatService` or anything upstream.
6. **Structured output, not free text**: each review mode's prompt (`reviews/review-templates.ts`) pairs a mode-specific focus (security / performance / quality) with one shared JSON output contract (`summary`, `issues[]` with `severity`/`filePath`/`lineHint`/`recommendation`, `generalRecommendations[]`). The client asks for `response_format: json_object` where the provider supports it.
7. **Defensive parsing**: `ReviewsService.parseAiOutput` strips accidental markdown fences, validates the top-level shape, coerces an unrecognized severity to `medium` instead of discarding the whole issue, and throws a typed `AiProviderError('malformed_response')` if the shape is unusable — which becomes a persisted, visible failed review rather than a crash.

## Security

- Passwords hashed with bcrypt (12 rounds); auth responses reveal nothing about whether an email exists on login failure.
- JWT bearer auth on every non-auth route via a global `JwtAuthGuard`; `request.user` is attached by `JwtStrategy`.
- **Every** project-scoped resource (files, reviews, chat, docs, architecture analysis) is loaded through `ProjectsService.getOwnedProject()`, which 404s on a nonexistent project and 403s on one owned by someone else — this is the one place that logic lives.
- **Path traversal / zip-slip protection** (`FilesService.safeResolve` + `ingestZip`): every ZIP entry name is normalized, checked for `..` segments, and resolved against the project's storage directory with a strict "resolved path must stay inside the base directory" assertion before anything is written to disk. The same `safeResolve` gate is used on every later read (file content, AI context, docs generation), so there's one enforcement point rather than one per code path. Covered by dedicated tests in `files/__tests__/files.service.security.spec.ts`.
- Known junk directories (`node_modules`, `.git`, `.venv`, `dist`, `build`, …) are filtered out at extraction time.
- Per-file (5MB) and per-archive (2000 files) limits guard against zip-bomb-style abuse; oversized files are skipped with a warning rather than failing the whole upload.
- AI provider API keys are encrypted at rest (AES-256-GCM) and never sent back to the frontend in plaintext.
- `class-validator` DTOs with `whitelist: true, forbidNonWhitelisted: true` reject any unexpected body fields at the boundary.
- `helmet()` for standard security headers; CORS restricted to the configured frontend origin; global rate limiting (120 req/min) via `@nestjs/throttler`.
- The global exception filter never returns stack traces or raw driver errors to the client.

## Trade-offs and Known Limitations (documented per the assessment's own guidance)

- **Originally generated without being run in the authoring sandbox** (that environment has no reliable way to `npm install` two full Node apps within a session). This is now superseded: the candidate installed both apps, ran them locally, and confirmed core functionality works — see `AI_USAGE.md` for exactly what was and wasn't exercised, and confirm the newest additions (Round 4: AI provider enable/disable, review-history severity filter) before final submission, since they postdate that confirmation.
- **JWT in `localStorage`**, not an httpOnly cookie — simpler for this scope, but see the XSS note above. Would change first in a production hardening pass.
- **Character-count context sizing** instead of a real tokenizer — a cheap, dependency-free proxy (~4 chars/token) that's good enough to keep prompts bounded without pulling in a tokenizer library for an assessment-scale project. Paired with the tiered/adaptive retry described above, so an imprecise proxy doesn't need to be perfectly accurate — a rejected request just falls back to the next smaller tier.
- **Chat retrieval is keyword-based, not embeddings/RAG** — intentionally the simplest thing that works, isolated behind `retrieveRelevantFiles()` so it's a drop-in replacement point later, per the assessment's own guidance not to over-build retrieval for this scope.
- **Generated docs/architecture analysis are not persisted** — they're generated on demand and shown to the user (with copy-to-clipboard). Given the assessment's required tables list didn't call for a docs table and scope was already large, persistence was left out; it would be a straightforward addition (one table, following the `Review` pattern) if needed.
- **`synchronize: true` in development** rather than requiring a migration on every clone — faster to get running locally; migration commands are provided and are what a production deploy should use instead.

## Change Log — Post-Submission Review Pass

A follow-up review against the assessment spec found and fixed the following. Documented here rather than silently folded into the sections above, so it's clear what changed and why:

- **Root cause of "request too large" failures (Review, Chat, Docs Generator)**: every AI-calling module shared one fixed, oversized context budget (~60k characters / ~15k tokens) regardless of what the configured provider could actually accept. Groq's free tier — and free tiers generally — reject requests far below that. Fixed by replacing the single budget with per-module tiered budgets and an adaptive retry helper (`ai-providers/with-resilience.ts`) that shrinks context and retries when a provider reports the request as too large, and backs off and retries once on rate-limiting. See "AI Integration Flow" above for the details.
- **Chat additionally had a specific bug**: when no file scored as relevant to a question, it fell back to attaching several files anyway rather than none. Combined with generic query words ("project", "name") matching almost every file's content, a trivial question like "what's the project name?" was attaching real file content it never needed. Fixed by filtering stopwords, requiring a minimum relevance score before a file counts as a match, and adding a genuinely context-free final tier (project name/description/file list only) that a purely factual question resolves against.
- **Architecture Analysis (and generated docs) rendered raw Markdown as plain text.** Fixed by rendering through `MarkdownView` (`marked` + `DOMPurify` + Tailwind `prose`) instead of a `<pre>` block.
- **No light theme, no theme switching, dark-only hardcoded colors.** Added a CSS-variable-based theme system (cream-toned light theme, existing dark theme, system-default with manual override), and replaced hardcoded Tailwind gray/white utility classes across the app with theme-aware semantic tokens.
- **No public landing page.** Added one at `/`, describing only implemented functionality.
- **Responsive gaps** in the project workspace (fixed-height two-column panels, top nav) on small screens — addressed for the highest-traffic screens (project explorer, chat, top nav); see AI_USAGE.md for what this pass did and did not cover in full.

### Round 3 — Production error reports from local testing

Two further bugs, both surfaced by running the app locally against real data (see AI_USAGE.md for the full account):

- **Every AI feature returning a bare 500 with `Unsupported state or unable to authenticate data` in the server log.** Root cause: `AiProvidersService.resolveClient()` decrypts a stored provider API key using a key derived from `JWT_SECRET`. That decrypt call happened *before* each caller's (reviews/chat/docs/architecture) own try/catch block, so a decrypt failure — which happens whenever `JWT_SECRET` differs from what it was when the key was saved — was never caught, and escaped as a raw, unhandled error. Fixed by moving the try/catch into `resolveClient` itself (`safeDecryptApiKey`), so every caller now gets a clear `BadRequestException` explaining exactly what happened and how to fix it (remove and re-add the affected provider), instead of an opaque 500. `main.ts` also now warns at startup if `JWT_SECRET` isn't set explicitly, since that's the condition most likely to cause this later.
- **Code Explorer showing the file tree but no file content.** `FilesService.getFileContent()` previously collapsed "binary file" and "read failed" into the same `content: null` response. When a project's on-disk storage doesn't match its database rows (see `.env`'s `STORAGE_ROOT` and the file-storage discussion elsewhere in this document), every read silently failed and looked identical to a binary file. Fixed by returning an explicit `missing: boolean` alongside `content`, which the frontend now renders as an actionable message rather than a blank editor. `Review`, `Docs Generator`, and `Architecture Analysis` also now detect this case up front (`context-builder.ts`'s `allContentMissing`) and fail with a clear message instead of silently running an AI request against empty file bodies. `Chat` does not need this guard — its existing zero-file "overview only" tier already degrades gracefully.

### Round 4 — Full spec cross-check before submission

A final pass comparing the implementation against every mandatory and bonus requirement line-by-line found and closed these gaps:

- **AI provider "enabled/disabled state" was configurable in the data model but not from the UI** — the spec explicitly lists it as one of the concepts a user configures. Added `PATCH /ai-providers/:id` (partial update: name, type, base URL, model, API key, enabled, default) and a working enable/disable toggle + "set default" action in the AI Providers page, instead of delete-and-recreate being the only way to change any of it.
- **Review History's severity filter existed on the backend (`ListReviewsDto.severity`) but wasn't exposed in the UI** — added the missing dropdown so search/filter is actually usable end-to-end, not just implemented server-side.
- **ZIP upload only validated file type by filename extension in the controller, not at the multer layer** — added a `fileFilter` checking both extension and declared MIME type before the upload is even accepted, as defense in depth alongside `FilesService.ingestZip()`'s real byte-level validation.
- **Test coverage gaps**: the AI review engine's structured-output parsing (severity coercion, fenced-JSON stripping, failed-review persistence) and the resilience/retry ladder introduced in Round 2 had no tests at all. Added `reviews.service.spec.ts`, `context-builder.spec.ts`, `with-resilience.spec.ts`, and `chat/context-retrieval.spec.ts` covering exactly those paths — not padding, each targets a specific behavior the assessment's testing checklist calls out (AI review, error handling) or that a previous round's bug fix introduced.

### Round 5 — Test suite failures from a real CI/local test run

Running `npm test` in `backend/` (not just having tests exist, but actually running them) surfaced two genuine bugs — one in a test fixture, one in real application logic. Both matter for different reasons:

- **`context-retrieval.spec.ts`: "how does authentication work?" matched zero files, including `auth.service.ts`.** This was a real bug in `chat/context-retrieval.ts`, not the test. The scorer used plain substring matching (`haystack.includes(keyword)`), which can only find a *shorter* string inside a *longer* one — it had no way to relate the question's "authentication" to the code's "AuthService" or "auth.service.ts", since neither contains the other as a literal substring. This is one of the single most common gaps between how people ask questions and how code abbreviates things (auth/authentication, config/configuration, db/database), so it wasn't a contrived edge case — it would have quietly degraded the Chat feature for exactly the kind of question a real user asks first. Fixed by extracting whole words from both the question and the file (path + content) and matching them bidirectionally by prefix once both words are long enough (≥4 characters) for that to be a meaningful signal rather than noise — this is what lets "auth" and "authentication" recognize each other as related in either direction, without needing a real stemmer or dictionary.
- **`files.service.security.spec.ts`: the path-traversal test itself was flawed, not the code it tested.** The fixture built its malicious ZIP using `new AdmZip().addFile('../../etc/evil.txt', ...)` — but `AdmZip`'s own `addFile()` sanitizes traversal segments out of an entry name (via its internal `zipnamefix()`) *before the name is ever written into the archive bytes*. Confirmed by direct inspection of `adm-zip`'s source and by constructing a small reproduction script. That means the fixture could never actually contain a real `..` segment by the time `ingestZip()` read it back — the test was unknowingly relying on AdmZip's own write-time protection to make its assertion pass, not exercising `FilesService`'s read-time defenses at all. **`FilesService.ingestZip()` itself was never the problem** — but the test gave a false sense of coverage for exactly the security property the assessment calls out as mandatory. Fixed by replacing the fixture with a small hand-rolled STORED-format ZIP writer (in the test file only) that writes a genuinely unsanitized entry name directly into the archive's central directory — the same way a real malicious archive, or another language's zip library that doesn't sanitize on write, would produce one. Verified empirically (both against the new fixture and against `ingestZip()`'s actual filtering logic) before committing to the fix, not just asserted. A second case — a traversal segment embedded after a legitimate-looking prefix (`src/../../../etc/evil.txt`) — was added alongside it, since that's a distinct, common zip-slip bypass attempt worth covering now that a faithful fixture exists.

### Round 6 — Uploaded data inside tool scan scope (Jest + TypeScript)

The duplicated failures in the Round 5 test output (each spec reported once from `src/...` and again from `storage/<projectId>/.../src/...`) were not a runner quirk. Jest's `rootDir` was `.`, so it discovered spec files anywhere under `backend/`, including inside `backend/storage/`, where a copy of this project had been uploaded as test data. This is the same root cause as the earlier `start:dev` problem (TypeScript's default scope also covered `storage/`): **uploaded user data must never sit inside a build/test tool's default scan scope.** Fixed at both layers, in configuration only — storage behavior is unchanged:

- `package.json` → Jest `roots: ["<rootDir>/src"]`, so tests are only ever discovered under `src/`.
- `tsconfig.json` → `rootDir: "./src"`, `include: ["src/**/*"]`, `exclude: ["node_modules", "dist", "storage", "test"]`, so uploaded projects never enter the compile or `--watch` graph, and `dist/` output layout is pinned regardless of what `storage/` contains.

The `tsconfig` fix had been diagnosed and recommended in an earlier round but was never actually written into the project; it is now.

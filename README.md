# AI-Powered Code Review Assistant

A full-stack application that lets developers upload source code, explore it, request structured AI-generated code reviews (security, performance, code quality), keep a searchable review history, and chat with their codebase — all against **any OpenAI-compatible AI provider** (OpenAI, OpenRouter, Groq, LM Studio, Ollama, or a custom endpoint), configured entirely at runtime.

Bonus features implemented: **Documentation Generator** (README / setup guide / API docs) and **Architecture Analysis**.

---

## Tech Stack

| Layer      | Choice                                            |
|------------|----------------------------------------------------|
| Frontend   | Next.js 14 (App Router), TypeScript, Tailwind CSS  |
| Backend    | NestJS, TypeScript                                 |
| Database   | PostgreSQL + TypeORM                               |
| Auth       | JWT (passport-jwt) + bcrypt                        |
| AI         | Provider-agnostic OpenAI-compatible HTTP client    |
| File proc. | `adm-zip`, with path-traversal-safe extraction     |

## Features

- **Auth**: register / login / logout, JWT-protected routes.
- **Projects**: create, list, view, delete — each project scoped to its owner.
- **Code upload**: ZIP upload → safe extraction → indexed as `CodeFile` rows tied to the project.
- **Code Explorer**: folder-tree view, file preview, syntax highlighting.
- **AI Review Engine**: review a single file, a selection of files, or the entire project; output is structured (summary, issues, severities, recommendations), not raw model text.
- **Review Templates**: Security / Performance / Code Quality, each with a focused prompt.
- **Review History**: persisted, searchable (by summary/issue text, mode, severity), paginated.
- **AI Chat With Code**: keyword-based context retrieval selects the most relevant files for each question; conversation history persisted per session.
- **Bonus — Documentation Generator**: generates README / Setup Guide / API docs from the actual uploaded code.
- **Bonus — Architecture Analysis**: AI-generated architecture summary from the file tree + a representative sample of files.
- **Configurable AI providers**: base URL / API key / model are set per-user via the UI, encrypted at rest, never hardcoded.

## Repository Structure

```
frontend/     Next.js app (TypeScript, Tailwind)
backend/      NestJS API (TypeScript, TypeORM, PostgreSQL)
README.md
ARCHITECTURE.md
AI_USAGE.md
```

## Setup Instructions

### Prerequisites
- Node.js 20+
- PostgreSQL 14+ (running locally, or via Docker)
- An AI provider you can reach: an **OpenRouter** free-tier key, a **Groq** free-tier key, or a local **LM Studio** / **Ollama** instance. No paid key is required.

### 1. Database

```bash
createdb ai_code_review
# or, with Docker:
docker run --name ai-code-review-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=ai_code_review -p 5432:5432 -d postgres:16
```

### 2. Backend

```bash
cd backend
cp .env.example .env      # edit DB_* and JWT_SECRET as needed
npm install
npm run start:dev         # http://localhost:4000/api
```

The database schema is created automatically on first run in development (`synchronize: true` when `NODE_ENV !== production`). For a production-style setup, generate and run a migration instead:

```bash
npm run migration:generate -- src/migrations/Init
npm run migration:run
```

### 3. Frontend

```bash
cd frontend
cp .env.local.example .env.local   # points at the backend API
npm install
npm run dev                        # http://localhost:3000
```

### 4. Configure an AI provider (in the app, not `.env`)

Sign up / log in, open **AI Providers** in the top nav, and add one:

| Provider   | Base URL                          | API key       | Model example |
|------------|-------------------------------------|---------------|----------------|
| OpenRouter | `https://openrouter.ai/api/v1`      | your free key | `meta-llama/llama-3.1-8b-instruct:free` |
| Groq       | `https://api.groq.com/openai/v1`    | your free key | `llama-3.1-8b-instant` |
| LM Studio  | `http://localhost:1234/v1`          | *(none)*      | whatever model you've loaded |
| Ollama     | `http://localhost:11434/v1`         | *(none)*      | e.g. `llama3.1` |

Mark one provider "default" and reviews/chat/docs will use it automatically; you can also pass a specific `providerId` per request.

## Environment Variables

**Backend** (`backend/.env`, see `.env.example`):

| Variable | Purpose |
|---|---|
| `PORT` | API port (default 4000) |
| `CORS_ORIGIN` | Allowed frontend origin |
| `JWT_SECRET` | Signs auth tokens **and** derives the key used to encrypt AI provider API keys at rest |
| `JWT_EXPIRES_IN` | Token lifetime |
| `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME` | PostgreSQL connection |
| `STORAGE_ROOT` | Where extracted project files are stored on disk |
| `MAX_UPLOAD_SIZE_MB` | Archive upload ceiling |

Note: AI provider credentials are **not** environment variables — they're configured per-user at runtime and stored encrypted in the database (see `ARCHITECTURE.md`).

**Frontend** (`frontend/.env.local`, see `.env.local.example`):

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | Backend API base URL |

## Database Setup

See `ARCHITECTURE.md` → Database Design for the schema. In development, TypeORM's `synchronize` creates tables automatically from the entities on boot. For anything beyond local development, use the TypeORM migration commands in `backend/package.json`.

## Running Tests (backend)

```bash
cd backend
npm test
```

Covers: registration/login flows, project ownership enforcement, and — importantly — ZIP upload safety (path traversal rejection, invalid/empty archive handling, ignored-directory filtering).

## Architecture Overview

See `ARCHITECTURE.md` for the full breakdown of frontend/backend architecture, database design, and the AI integration flow.

## AI Usage

See `AI_USAGE.md` for a transparent account of how AI tools were used to build this project, and the engineering decisions made.

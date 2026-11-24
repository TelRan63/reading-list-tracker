# Implementation Plan: Book Reading List

**Branch**: `001-book-reading-list` | **Date**: 2026-11-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-book-reading-list/spec.md`

## Summary

One shared reading list without sign-in. Users add a book (title, author, status), see all books
newest first, filter by status (the filter is kept in the page address), and change a book's
status. The status is one of three fixed values.

Technical approach: an npm-workspaces monorepo with three packages. The **backend** is an
Express 5 REST API in TypeScript, layered as routes → services → Mongoose models, with MongoDB as
storage, with one Mongoose model, `Book`. The **frontend** is a React 19 single-page app built with
Vite, with two pages handled by React Router (the book list at `/` and the add-book page at
`/books/new`) and Redux Toolkit for state; server data is loaded and cached with RTK Query, the
data-fetching part of Redux Toolkit. A **shared** package holds
the OpenAPI 3.1 contract, which is the single source of truth. API types for both sides are
generated from it, and the backend validates requests (and, in tests, responses) against it.
Business rules (trimming, length limits, allowed statuses, default status, filter parsing) live in
plain TypeScript modules that are unit-tested without HTTP, a database, or a browser.

## Technical Context

**Language/Version**: TypeScript 5.x (`strict: true`) on Node.js 24 LTS (frontend builds with the
same toolchain).

**Primary Dependencies**:
- Backend: Express 5, Mongoose 8, express-openapi-validator 5, dotenv.
- Frontend: React 19, Vite 7, React Router 7 (two pages, filter in `?status=`), Redux Toolkit 2
  with RTK Query and react-redux 9 (state and server data). See research R5, R6, R13.
- Shared: openapi-typescript 7 (generates API types from the contract).
- Tooling: ESLint 9 (flat config) + typescript-eslint + eslint-plugin-import-x
  (`no-default-export`), Prettier 3, husky 9 + commitlint (Conventional Commits).

**Storage**: MongoDB 8 through Mongoose only. Local development uses MongoDB in Docker
(`docker compose`). Connection string comes from `MONGODB_URI`.

**Testing**: Vitest 3 for all packages.
- Backend unit tests for services and validation rules.
- Backend API tests with Supertest against an in-memory MongoDB (mongodb-memory-server); responses
  are checked against the OpenAPI contract by express-openapi-validator in the test environment.
- Frontend unit tests for utilities, Redux slice reducers, and selectors; component and page
  tests with React Testing Library and user-event, rendered with a real Redux store and a memory
  router; the network is replaced with MSW (Mock Service Worker) handlers built from the contract
  examples (keyboard-only flows for FR-017).

**Target Platform**: Linux/macOS/Windows development machines; current desktop and mobile
browsers (latest two versions of Chrome, Firefox, Safari, Edge).

**Project Type**: web application (React frontend + REST API backend) in one repository.

**Performance Goals**: filtered list shown in under 1 second for up to 500 books (SC-003); a
status change visible in the list immediately after the server confirms it.

**Constraints**: no sign-in (FR-015); last saved change wins (FR-016); no real-time updates
(FR-013); filter kept in the page address (FR-009a); keyboard-only operation and labeled controls
(FR-017, FR-018); UI text in Russian.

**Scale/Scope**: one shared list, up to about 500 books, a small number of people using it at the
same time; two pages (list, add book); one data model (`Book`); three API endpoints.

All Technical Context items are resolved; there are no NEEDS CLARIFICATION items. The reasoning
for each choice is in [research.md](./research.md).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Constitution version 2.1.0.

| # | Principle / rule | How this plan complies | Pre-design | Post-design |
|---|------------------|------------------------|------------|-------------|
| I | Test-first (non-negotiable) | `tasks.md` will order a failing test before each implementation task. Bug fixes start with a regression test. | PASS | PASS |
| I | Business logic covered and isolated | Rules live in `backend/src/services/` and `frontend/src/lib/`, `frontend/src/hooks/`; each rule in [data-model.md](./data-model.md) has expected and error-case tests. No rules in route handlers, schemas, or component bodies. | PASS | PASS |
| I | Three test levels | Unit (services, utilities), API (Supertest, every endpoint and error), component (RTL). | PASS | PASS |
| II | TypeScript strict, no `any`, shared types | `strict: true` in a shared `tsconfig.base.json`; `@typescript-eslint/no-explicit-any` = error; API types generated once from the contract into `packages/shared`. | PASS | PASS |
| III | REST, contract first | [contracts/openapi.yaml](./contracts/openapi.yaml) written before code: `GET /api/books`, `POST /api/books`, `PATCH /api/books/{id}`. Plural nouns, no verbs, correct status codes. | PASS | PASS |
| III | Responses checked against the contract; one error format | Response validation on in tests; every error uses `{ code, message, details? }`. | PASS | PASS |
| IV | Simplicity, small increments; new dependencies justified | Three user stories, P1 first. React Router and Redux Toolkit are a team decision; their use is justified in research R5, R6, R13. RTK Query comes with Redux Toolkit, so no separate data-fetching library is added. No migration tool yet (R8). | PASS | PASS |
| V | Validate at boundary; reject with 4xx | express-openapi-validator rejects bad shape/type/unknown fields with 400; services apply trim and length rules. | PASS | PASS |
| V | Mongoose schema, strict, validators | `Book` schema with types, `required`, `maxlength`, `enum`, `strict: 'throw'`. | PASS | PASS |
| V | Versioned migrations for shape changes | No existing data and no custom indexes in this feature, so no migration is needed now (research R8). The first real shape change must add the migration runner. | PASS | PASS |
| V | Data only through Mongoose models; no native driver | Services use `BookModel` only. `mongoose.set('sanitizeFilter', true)` as a second guard. | PASS | PASS |
| V | No `$`/`.` keys, values cast | Contract has `additionalProperties: false`; ids must match the ObjectId pattern; status is an enum. | PASS | PASS |
| V | Secrets and connection settings in env | `MONGODB_URI`, `PORT` from env; `.env.example` committed, `.env` ignored. | PASS | PASS |
| V | Escape user text | React escapes text by default; `dangerouslySetInnerHTML` banned by lint rule. | PASS | PASS |
| VI | Functional components, hooks, named exports | All components and pages are functions. Redux is used only through hooks (`useAppSelector`, `useAppDispatch`, RTK Query hooks); `connect()` is not used. Routes are declared with named page components, without `React.lazy` (which needs default exports). `import-x/no-default-export` = error, with an override only for `vite.config.ts`, `vitest.config.ts`, `eslint.config.js`, `commitlint.config.js`. | PASS | PASS |
| VI | Backend layers | `routes/` → `services/` → `models/`; one model, `Book`. Routes never import models (lint rule `no-restricted-imports`). | PASS | PASS |
| I, VI | Frontend business logic outside components | Rules live in `lib/` (status labels, filter parsing, form validation) and in pure slice reducers and selectors in `store/`; pages and components only render and dispatch. | PASS | PASS |
| Stack | Node LTS, React, REST, OpenAPI 3.x, MongoDB/Mongoose, isolated test DB | As listed in Technical Context. Tests use mongodb-memory-server. | PASS | PASS |
| Tooling | One package manager; single commands | npm workspaces; root scripts `npm install`, `npm test`, `npm run lint`, `npm run typecheck`, `npm run dev`. | PASS | PASS |
| Workflow | Conventional Commits, linter in hook and CI | husky `commit-msg` hook runs commitlint; CI runs commitlint on the PR's commits. | PASS | PASS |

**Result**: all gates pass before and after design. No deviations; Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-book-reading-list/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── openapi.yaml     # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
package.json                 # npm workspaces root; scripts: dev, test, lint, typecheck, format
tsconfig.base.json           # strict TypeScript settings shared by all packages
eslint.config.js             # shared lint rules (no-default-export, no-explicit-any, layering)
commitlint.config.js
.husky/commit-msg
docker-compose.yml           # local MongoDB 8 for development only
.env.example                 # MONGODB_URI, PORT (placeholders)

packages/shared/
├── openapi.yaml             # living API contract (starts as a copy of contracts/openapi.yaml)
├── src/
│   ├── api-types.ts         # generated by openapi-typescript (do not edit by hand)
│   └── index.ts             # re-exports generated types and BOOK_STATUSES
└── package.json             # script: generate (openapi-typescript)

backend/
├── src/
│   ├── app.ts               # creates the Express app: JSON, OpenAPI validator, routes, errors
│   ├── server.ts            # reads env, connects to MongoDB, starts the HTTP server
│   ├── config/env.ts        # reads and checks environment variables
│   ├── db/connect.ts        # Mongoose connection (sanitizeFilter, strictQuery)
│   ├── models/book.model.ts # Mongoose schema and model
│   ├── services/
│   │   ├── book-rules.ts    # pure rules: normalize and validate title/author, default status
│   │   └── book.service.ts  # listBooks, createBook, updateBookStatus (uses BookModel)
│   ├── routes/books.routes.ts   # HTTP only: parse, call service, send response
│   └── errors/              # AppError, error-to-response middleware
└── tests/
    ├── unit/                # book-rules, book.service (with in-memory MongoDB)
    └── api/                 # Supertest: every endpoint and error response

frontend/
├── index.html
├── vite.config.ts           # proxies /api to the backend in development
├── src/
│   ├── main.tsx             # creates the store and router, renders <App />
│   ├── app.tsx              # <Provider> + <RouterProvider>
│   ├── router.tsx           # routes: "/" → BookListPage, "/books/new" → AddBookPage, "*" → NotFoundPage
│   ├── store/
│   │   ├── store.ts         # configureStore; typed useAppDispatch / useAppSelector hooks
│   │   ├── books-api.ts     # RTK Query API: listBooks, createBook, updateBookStatus (tag "Book")
│   │   └── add-book-form-slice.ts  # draft title/author/status kept while the user is on the form
│   ├── lib/
│   │   ├── status.ts        # status codes ↔ Russian labels, filter parsing
│   │   ├── book-form.ts     # client-side form validation (same rules as the backend)
│   │   └── api-errors.ts    # maps ErrorResponse.details to field messages
│   ├── hooks/
│   │   └── use-status-filter.ts  # filter ↔ ?status= via React Router useSearchParams
│   ├── pages/
│   │   ├── book-list-page.tsx    # filter + list + "Добавить книгу" link
│   │   ├── add-book-page.tsx     # form; on success navigates back to the list
│   │   └── not-found-page.tsx
│   └── components/
│       ├── add-book-form.tsx
│       ├── status-filter.tsx
│       ├── book-list.tsx
│       └── book-item.tsx    # includes the status selector
└── tests/
    ├── unit/                # lib/, slice reducers, selectors, use-status-filter
    ├── components/          # RTL + user-event, including keyboard-only flows
    └── msw/                 # MSW handlers based on contract examples
```

**Structure Decision**: web application layout with `backend/` and `frontend/` at the root, plus
`packages/shared/` for the contract and generated types. This matches the constitution's
"React frontend + REST API backend in one repository" and its rule that API types come from a
single source.

## Complexity Tracking

No constitution violations. Nothing to justify.

# Research: Book Reading List

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-24

The constitution fixes the main stack: TypeScript strict, Node.js LTS, React, REST, OpenAPI 3.x,
MongoDB through Mongoose. This file records the remaining choices.

## R1. Repository layout and package manager

- **Decision**: npm workspaces with `backend/`, `frontend/`, and `packages/shared/`.
- **Rationale**: npm comes with Node.js (no extra install; pnpm is not installed on the team
  machine checked). Workspaces give one `npm install` and one lock file, as the constitution
  requires.
- **Alternatives considered**: pnpm or Yarn workspaces (extra tool to install); Turborepo/Nx
  (build orchestration not needed for three small packages).

## R2. Node.js version

- **Decision**: Node.js 24 LTS. Record it in `.nvmrc` and in `engines` in the root `package.json`.
- **Rationale**: Node 24 is an LTS line supported until April 2028 and is already installed
  (v24.15.0). Moving to Node 26 LTS later is a one-line change.
- **Alternatives considered**: Node 26 (newest LTS line, but not installed on team machines yet).

## R3. Single source for API types and request validation

- **Decision**: a hand-written OpenAPI 3.1 file in `packages/shared/openapi.yaml` is the source of
  truth. `openapi-typescript` generates `api-types.ts` from it. The backend uses
  `express-openapi-validator` to validate every request against the same file, and turns on
  response validation in the test environment.
- **Rationale**: one file gives contract-first design (Principle III), shared types (Principle
  II), boundary validation (Principle V), and the "responses conform to the contract" merge gate,
  without writing the schema twice.
- **Alternatives considered**:
  - Zod schemas + `zod-to-openapi`: the contract would be generated from code, which goes against
    "contract before implementation".
  - Hand-written TypeScript types in a shared package: types and contract could drift apart.
  - A separate contract-testing tool (for example, jest-openapi): response validation in the
    validator already covers this.

## R4. Where the validation rules live

- **Decision**: three layers, each with a different job.
  1. Contract (validator): shape, types, enum, required fields, `additionalProperties: false`,
     ObjectId pattern, and a `pattern: '\S'` so that text with only spaces is rejected early.
  2. Service rules (`book-rules.ts`): trim, then check "not empty" and the length limits (200/100)
     on the trimmed text, apply the default status. This is the business logic that FR-004/FR-005
     describe and is unit-tested.
  3. Mongoose schema: `required`, `maxlength`, `enum`, `trim` as the last guard for data at rest.
- **Rationale**: the contract cannot express "length after trimming", so the rule must live in
  code; keeping it in a pure function makes it easy to test (Principle I). The frontend uses the
  same limits for instant feedback, but the backend is the authority.
- **Alternatives considered**: rules only in the Mongoose schema (Principle I forbids business
  logic in schema definitions; error messages are harder to control).

## R5. Pages and the filter in the page address

- **Decision**: React Router 7 in data-router mode (`createBrowserRouter`) with three routes:
  `/` (book list), `/books/new` (add book), and `*` (not found). A small hook `useStatusFilter`
  wraps `useSearchParams`: it reads `?status=`, maps unknown values to "Все" (all), and writes a
  new history entry when the filter changes, so Back/Forward move between filters (FR-009a).
  The filter lives only in the URL, not in Redux, so there is one source of truth.
- **Navigation**: the list page has a "Добавить книгу" link to `/books/new`. After a successful
  save, the add page navigates to `/` with the filter "Все", so the new book is visible at the top
  (US1-1). If saving fails, the user stays on the add page and the entered values are kept. The
  add page has a "Назад к списку" link. The empty-list message links to `/books/new` (US2-4).
- **Rationale**: the team chose separate pages for the list and the form; React Router is the
  standard routing library for React and already provides URL query handling, so no custom
  `history` code is needed.
- **Alternatives considered**: one page with the form above the list and a custom URL hook
  (fewer dependencies, but not the agreed page structure); TanStack Router (less familiar to the
  team).

## R6. State and loading data in the frontend

- **Decision**: Redux Toolkit 2 with react-redux 9.
  - **Server data**: RTK Query (`createApi`, part of Redux Toolkit) with three endpoints that match
    the contract: `listBooks(status?)`, `createBook`, `updateBookStatus`. Types come from the
    generated contract types. Mutations invalidate the `Book` tag, so the list is loaded again
    after each add and status change; `refetchOnMountOrArgChange: true` loads fresh data on each
    filter change and when the user returns to the list (FR-013).
  - **UI state**: one slice, `addBookForm`, holds the draft title, author, status, and the field
    errors. Its reducers are pure functions and call the rules in `lib/book-form.ts`.
  - **No optimistic updates**: the list shows a new status only after the server confirms it; on
    an error the list keeps the last saved status and an error message is shown (FR-014).
  - **Double submission**: the save button is disabled while the `createBook` mutation is pending.
- **Rationale**: the team chose Redux Toolkit for state. RTK Query is included in Redux Toolkit,
  so loading, caching, and "load again after a change" come without another library or hand-written
  thunks. Slices and selectors are plain functions, which fits the rule that business logic is
  testable without UI rendering (Principle I).
- **Alternatives considered**: `createAsyncThunk` with a hand-written books slice (more code for
  loading/error state and refetching); TanStack Query alongside Redux (two state tools); plain
  hooks and `fetch` (not the agreed state approach).

## R7. Status values in the API and in the UI

- **Decision**: the API uses stable codes `want_to_read`, `reading`, `read`. The frontend maps them
  to the Russian labels "Хочу прочитать", "Читаю", "Прочитано". The URL uses the same codes
  (`?status=reading`).
- **Rationale**: codes are ASCII, safe in URLs, and do not change if a label is reworded.
- **Alternatives considered**: storing the Russian labels (URL encoding issues; renaming a label
  would need a data migration).

## R8. Indexes and migrations

- **Decision**: no custom indexes and no migration runner in this feature. Sort by
  `createdAt` descending, then `_id` descending, as a tiebreak.
- **Rationale**: with up to about 500 documents, MongoDB sorts in memory in milliseconds. The
  constitution requires migrations for changes to existing stored data; there is no existing data
  yet, so adding a migration tool now would be unused code (Principle IV).
- **Follow-up**: the first feature that changes stored data or adds an index must add a versioned
  migration runner that works through Mongoose models.
- **Alternatives considered**: a `{ status: 1, createdAt: -1 }` index now (not needed at this scale);
  `migrate-mongo` (uses the native driver, which the constitution forbids).

## R9. Concurrent changes (last write wins)

- **Decision**: status updates use a plain `findByIdAndUpdate` with `runValidators: true` and no
  version check. `timestamps: true` keeps `createdAt` and `updatedAt`.
- **Rationale**: matches FR-016 (the change saved last is kept, no error shown). Order is by
  `createdAt`, so a status change does not move the book (FR-012).
- **Alternatives considered**: optimistic locking with a version field (would reject changes;
  rejected in clarification Q1).

## R10. Test setup

- **Decision**: Vitest everywhere. Backend: Supertest + mongodb-memory-server, one in-memory server
  per test file, collections cleared between tests. Frontend: jsdom, React Testing Library,
  user-event. Pages are rendered with a real Redux store and `createMemoryRouter`; HTTP calls are
  answered by MSW (Mock Service Worker) handlers that use the contract examples. Slice reducers,
  selectors, and `lib/` functions get plain unit tests.
- **Rationale**: one test runner for the whole repository; in-memory MongoDB satisfies the
  "isolated test database" rule and needs no Docker in CI. MSW tests the real RTK Query code path
  instead of replacing it.
- **Alternatives considered**: Jest (slower TypeScript/ESM setup); a MongoDB container in tests
  (needs Docker in CI); mocking RTK Query hooks directly (would skip the refetch behavior that
  FR-013 requires).

## R11. Local MongoDB for development

- **Decision**: `docker-compose.yml` with the official `mongo:8` image and a named volume.
- **Rationale**: MongoDB is not installed locally; Docker is. One command starts the database.
- **Alternatives considered**: MongoDB Atlas free tier (needs accounts and network access);
  installing `mongod` on each machine.

## R12. Lint and commit rules

- **Decision**: ESLint 9 flat config with typescript-eslint (strict), `eslint-plugin-import-x`
  (`no-default-export`, with an override for tool config files), `eslint-plugin-react-hooks`,
  `react/no-danger`, and `no-restricted-imports` that stops `backend/src/routes/**` from importing
  `models/`. Prettier for formatting. husky `commit-msg` hook runs commitlint with
  `@commitlint/config-conventional`.
- **Rationale**: turns constitution rules (Principles II, V, VI and the Conventional Commits rule)
  into automatic checks.
- **Alternatives considered**: Biome (faster, but no import-layering rule equal to
  `no-restricted-imports` with path patterns at the time of writing).

## R13. Backend scope: one model

- **Decision**: the backend has exactly one Mongoose model, `Book`, and one REST resource,
  `/api/books`. Reading statuses are an enum field on `Book`, not a separate collection.
- **Rationale**: the status set is fixed (spec Key Entities); a separate collection would add
  joins and code without any requirement for it (Principle IV).
- **Alternatives considered**: a `Status` collection (not needed while the set is fixed).

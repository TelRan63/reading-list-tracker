<!--
Sync Impact Report
- Version change: 2.0.0 → 2.1.0
- Bump rationale: MINOR. One principle is added and existing guidance is materially expanded.
  No principle is removed or redefined in an incompatible way.
- Modified principles:
  - I. Test-First Development (NON-NEGOTIABLE): added a mandatory business-logic coverage rule
    and a rule that business logic lives in testable modules.
  - III. Contract-First API → III. Contract-First REST API: added REST design rules.
  - V. Data Integrity and Security: data access through Mongoose models made explicit for all
    code, including migrations; native driver access forbidden.
- Added principles:
  - VI. Code Structure and Conventions (frontend functional components, named exports only,
    backend layering).
- Modified sections:
  - Technology Stack and Constraints: architecture line now says REST API.
  - Development Workflow and Quality Gates: Conventional Commits required; new merge gates for
    business-logic coverage and commit messages.
- Added sections: none
- Removed sections: none
- Follow-up TODOs: none
- Note: remove this report before committing the constitution.
-->

# Reading List Tracker Constitution

## Core Principles

### I. Test-First Development (NON-NEGOTIABLE)

- Every change to behavior MUST start with an automated test that fails for the right reason.
- The cycle is Red → Green → Refactor: write a failing test, write the minimum code to make it
  pass, then improve the code while all tests stay green.
- Implementation code without a test that was seen failing first MUST NOT be merged.
- Bug fixes MUST include a regression test that reproduces the bug before the fix.
- Business logic MUST be covered by automated tests. Business logic means the rules of the
  application: validation rules, state changes (for example, a book's reading status),
  calculations, filtering, sorting, and access rules.
  - Every business rule MUST have at least one test for the expected case and one test for each
    error or edge case the rule defines.
  - Business logic MUST live in separate modules (backend services, frontend utility functions
    or custom hooks) so that it can be tested without HTTP, a browser, or UI rendering.
  - Business logic MUST NOT be placed in route handlers, Mongoose schema definitions, or React
    component bodies, where it is hard to test in isolation.
- Tests cover three levels: unit tests for business logic, API tests for every endpoint
  (including error responses), and component tests for frontend behavior that users depend on.

Rationale: tests written first define the expected behavior before code exists, keep the design
testable, and let every team member change code without fear of silent breakage.

### II. Strict Type Safety

- All frontend and backend code MUST be written in TypeScript with `strict` mode enabled.
- The `any` type MUST NOT be used. Where a value is truly unknown, use `unknown` and narrow it.
- Type checking (`tsc --noEmit`) MUST pass with zero errors before a change is merged.
- Types that describe API requests and responses MUST be shared between frontend and backend
  from a single source, not copied by hand.

Rationale: the compiler catches a large class of errors before tests run, and shared types stop
the frontend and backend from drifting apart.

### III. Contract-First REST API

- The backend MUST expose a REST API with JSON request and response bodies.
  - URLs MUST name resources with plural nouns (for example, `/api/books`,
    `/api/books/{id}`). Verbs MUST NOT appear in URLs.
  - HTTP methods MUST match their meaning: `GET` reads, `POST` creates, `PUT` or `PATCH`
    updates, `DELETE` removes. `GET` requests MUST NOT change data.
  - Responses MUST use correct status codes, for example `200`, `201` (with the created
    resource), `204`, `400`, `404`, `409`, `500`.
- Every HTTP endpoint MUST be defined in an OpenAPI contract before it is implemented.
- The contract states paths, methods, request bodies, response bodies, status codes, and error
  formats. Implementation MUST match the contract exactly.
- API tests MUST verify that responses conform to the contract.
- Errors MUST use one consistent JSON format with a machine-readable `code` and a human-readable
  `message`.
- A change that breaks an existing contract MUST be called out in the pull request and agreed
  by the team before merge.

Rationale: a written contract lets frontend and backend work proceed in parallel, gives tests a
fixed target, and makes the API easy to review.

### IV. Simplicity and Small Increments

- Build only what the current feature specification requires (YAGNI: "You Aren't Gonna Need It").
- Each feature MUST be delivered as independently testable user stories, smallest valuable story
  first.
- New dependencies, layers, or abstractions MUST be justified in the implementation plan. If a
  simpler option works, use it.
- Pull requests SHOULD stay small enough to review in one sitting (guideline: under 400 changed
  lines, excluding generated files).

Rationale: the team is small and time is limited. Simple code and small steps keep the project
shippable at every point and easy for every member to understand.

### V. Data Integrity and Security

- All input from users or clients MUST be validated at the API boundary against the contract
  schema. Invalid input MUST be rejected with a 4xx response.
- Every MongoDB collection MUST have a Mongoose schema with field types, required fields, and
  validators. Schemas MUST use strict mode so that unknown fields are not saved.
- Changes to the shape of existing stored data (renaming fields, changing types, backfilling
  values, adding indexes) MUST go through versioned, repeatable migration scripts kept in the
  repository. Manual edits to production or shared data are not allowed.
- Secrets (passwords, keys, tokens, connection strings) MUST NOT be committed. Use environment
  variables and keep a `.env.example` file with placeholder values.
- All work with data (reads, writes, updates, deletes, aggregations, and migration scripts)
  MUST go through Mongoose models. Direct use of the native MongoDB driver (for example,
  `mongoose.connection.db` or `collection(...)` calls) MUST NOT be used.
- Raw request objects MUST NOT be passed directly into query filters or updates; values MUST
  be validated and cast to the expected types first, and keys that start with `$` or contain
  `.` MUST be rejected or removed.
- Database connection settings MUST come from environment variables.
- User-supplied text MUST be escaped when rendered in the frontend.

Rationale: a reading list holds user data. Validation, migrations, and safe query practices
prevent data loss, corruption, and common security flaws such as NoSQL injection and XSS.

### VI. Code Structure and Conventions

- **Frontend**:
  - React components MUST be functional components. Class components MUST NOT be used.
  - Components, hooks, and utilities MUST use named exports. Default exports
    (`export default`) MUST NOT be used. The only exception is a tool configuration file whose
    tool requires a default export (for example, a bundler config file).
  - State and side effects MUST use React hooks.
  - A lint rule (for example, `import/no-default-export`) MUST enforce the export rule.
- **Backend**:
  - Code is split into layers: routes/controllers (HTTP only: parse the request, call a
    service, send the response) → services (business logic) → Mongoose models (data).
  - Routes and controllers MUST NOT contain business logic and MUST NOT call Mongoose models
    directly; they call services.

Rationale: functional components and named exports give one consistent style, make imports
easy to search and rename, and let tools catch wrong imports. Clear backend layers keep
business logic in one place where it is easy to test.

## Technology Stack and Constraints

- **Language**: TypeScript (strict mode) for all application code.
- **Runtime**: Node.js, current LTS release.
- **Architecture**: a web application with a React frontend and a REST API backend (JSON over
  HTTP), kept in one repository.
- **API contract**: OpenAPI 3.x document stored in the repository.
- **Persistence**: MongoDB, accessed only through Mongoose (ODM: Object Document Mapper).
  Other databases, including SQLite, MUST NOT be used.
- **Testing database**: automated tests MUST run against an isolated MongoDB instance (for
  example, an in-memory MongoDB server or a dedicated test database), never shared or
  production data.
- **Tooling**: one package manager for the whole repository, a shared linter and formatter
  configuration, and a single command each to install, test, lint, and run the app.
- Specific frameworks and libraries (for example, the frontend UI library, the HTTP server
  framework, the test runner) are chosen in each feature's implementation plan and MUST respect
  the principles above.

## Development Workflow and Quality Gates

- All work happens on feature branches. Direct commits to the main branch are not allowed.
- Every commit message MUST follow the Conventional Commits specification
  (https://www.conventionalcommits.org): `<type>(<optional scope>): <description>`.
  - Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`,
    `chore`, `revert`.
  - Breaking changes MUST be marked with `!` after the type or scope, or with a
    `BREAKING CHANGE:` footer.
  - A commit message linter (for example, commitlint) MUST check commit messages in a Git hook
    and in CI.
- Every pull request MUST be reviewed and approved by at least one other team member before
  merge.
- A pull request can be merged only when all of these gates pass:
  1. All automated tests pass.
  2. Type checking passes with zero errors.
  3. Lint passes with zero errors.
  4. API responses conform to the OpenAPI contract.
  5. Every new or changed business rule has tests (Principle I).
  6. All commit messages follow Conventional Commits.
  7. The reviewer has confirmed compliance with this constitution.
- Features follow the Spec Kit flow: specify → (clarify) → plan → tasks → implement. The
  implementation plan MUST include a Constitution Check that lists any deviation and its
  justification.
- The main branch MUST stay in a working state: the app builds, starts, and passes all tests.

## Governance

- This constitution is the authoritative source of truth for the project. It takes precedence
  over all other practices, guides, and individual preferences.
- **Amendments**: any team member may propose an amendment through a pull request that edits
  this file. The pull request MUST explain the reason and the impact on existing work. An
  amendment takes effect only after approval by a majority of the team.
- **Versioning**: the constitution uses semantic versioning.
  - MAJOR: a principle is removed or redefined in a way that is not backward compatible.
  - MINOR: a new principle or section is added, or guidance is materially expanded.
  - PATCH: clarifications, wording, or typo fixes with no change in meaning.
- **Compliance review**: reviewers check every pull request against this constitution.
  Deviations MUST be documented and justified in the implementation plan's Complexity Tracking
  section; unjustified deviations block the merge.
- `Claude.md` in the repository root points AI agents to this file for runtime guidance.

**Version**: 2.1.0 | **Ratified**: 2026-11-24 | **Last Amended**: 2026-11-24

# Quickstart: Book Reading List

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-24

This guide shows how to run the feature and check that it works end to end. API details are in
[contracts/openapi.yaml](./contracts/openapi.yaml); field rules are in
[data-model.md](./data-model.md).

## Prerequisites

- Node.js 24 LTS and npm (`node --version`, `npm --version`).
- Docker with Compose (for the local MongoDB only; tests do not need Docker).

## Setup

```bash
npm install                    # installs all workspaces; sets up the husky commit-msg hook
cp .env.example .env           # MONGODB_URI=mongodb://localhost:27017/reading-list, PORT=3000
docker compose up -d mongo     # starts MongoDB 8 on localhost:27017
npm run generate -w packages/shared   # generates API types from openapi.yaml
```

## Run automated checks (the merge gates)

```bash
npm test             # unit, API, and component tests (in-memory MongoDB, no Docker needed)
npm run typecheck    # tsc --noEmit in every package, zero errors expected
npm run lint         # ESLint, zero errors expected
```

Expected: all three commands exit with code 0.

## Run the app

```bash
npm run dev          # backend on http://localhost:3000, frontend on http://localhost:5173
```

Open http://localhost:5173. The app has two pages: the book list at `/` and the add-book page at
`/books/new`.

## Manual validation scenarios

Each scenario maps to the spec. Start with an empty `books` collection
(`docker compose down -v && docker compose up -d mongo` resets it).

| # | Steps | Expected result | Spec |
|---|-------|-----------------|------|
| 1 | Open the app. | Message that the list is empty and a link to add a book; the link opens `/books/new`. | US2-4, FR-010 |
| 2 | On `/books/new`, add "Мастер и Маргарита" / "Михаил Булгаков", status "Хочу прочитать". | The app returns to `/` with "Все" selected; the book is at the top of the list with these values. | US1-1 |
| 3 | Add a book without choosing a status. | Status is "Хочу прочитать". | US1-2, FR-003 |
| 4 | On `/books/new`, try to save with an empty title, then with a title of only spaces. | Not saved; the page stays open; message next to the title field. Same for the author. | US1-3, FR-004 |
| 5 | Enter a 201-character title. | Not saved; message states the 200-character limit. | FR-005 |
| 5a | Stop the backend, fill in the form on `/books/new`, and save. | Error message; the page stays open and keeps the entered values. Start the backend again. | Edge Cases, FR-014 |
| 5b | Open `/books/new` directly in a new tab, then use "Назад к списку". Open `/unknown-page`. | The add page opens and the link returns to the list. The unknown page shows a "not found" page with a link to the list. | — |
| 6 | Reload the page. | All books are still there. | US1-4, FR-006 |
| 7 | Add books with each status; select each filter, then "Все". | Only matching books, then all books; newest first. | US2-1, US2-2, FR-008 |
| 8 | Select "Прочитано" when no book has that status. | Message that no books have this status. | US2-3, FR-010 |
| 9 | Select "Читаю", reload; copy the URL into a new tab. | "Читаю" is still selected in both. URL contains `?status=reading`. | US2-6, FR-009a |
| 10 | Select "Читаю", then "Прочитано", press browser Back. | "Читаю" is selected again. | US2-7 |
| 11 | Open `/?status=unknown`. | "Все" is selected; no error. | FR-009a |
| 12 | Change a book from "Хочу прочитать" to "Читаю". | New status shown; title, author, and position unchanged. | US3-1, FR-012 |
| 13 | With filter "Читаю", change a book to "Прочитано". | It leaves the current view and appears under "Прочитано". | US3-2 |
| 14 | Stop the backend, change a status. | Error message; the book shows its last saved status. | US3-4, FR-014 |
| 15 | Do scenarios 2, 7, and 12 (including moving between the two pages) using only the keyboard (Tab, Shift+Tab, Enter, Space, arrows). | All complete without a mouse; a screen reader announces every label. | FR-017, FR-018, SC-006 |

## API smoke test (optional)

With the backend running:

```bash
curl -s -X POST localhost:3000/api/books -H 'Content-Type: application/json' \
  -d '{"title":"  Дюна ","author":"Фрэнк Герберт"}'
# 201, title is "Дюна", status is "want_to_read"

curl -s 'localhost:3000/api/books?status=want_to_read'
# 200, { "items": [ ... ] }

curl -s -X POST localhost:3000/api/books -H 'Content-Type: application/json' \
  -d '{"title":"   ","author":""}'
# 400, code VALIDATION_ERROR, details for title and author

curl -s -X PATCH localhost:3000/api/books/000000000000000000000000 \
  -H 'Content-Type: application/json' -d '{"status":"read"}'
# 404, code BOOK_NOT_FOUND

curl -s -X POST localhost:3000/api/books -H 'Content-Type: application/json' \
  -d '{"title":"X","author":"Y","$where":"1"}'
# 400, unknown field rejected
```

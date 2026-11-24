# Data Model: Book Reading List

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-24

## Entity: Book

MongoDB collection `books`, Mongoose model `Book`. This is the only model in the backend
(research R13). One document is one entry in the shared reading
list (FR-015: there is no owner field).

| Field | Type | Required | Rules | Source |
|-------|------|----------|-------|--------|
| `_id` | ObjectId | yes (auto) | Exposed in the API as `id` (24 hex characters). | — |
| `title` | string | yes | Trimmed; 1–200 characters after trimming. Any Unicode text. | FR-001, FR-004, FR-005 |
| `author` | string | yes | Trimmed; 1–100 characters after trimming. Any Unicode text. | FR-001, FR-004, FR-005 |
| `status` | string enum | yes | One of `want_to_read`, `reading`, `read`. Default `want_to_read`. | FR-002, FR-003 |
| `createdAt` | Date | yes (auto) | Set once on create; never changed. Used for ordering. | FR-008, FR-012 |
| `updatedAt` | Date | yes (auto) | Set on every change. | FR-016 |

Schema options: `timestamps: true`, `strict: 'throw'` (unknown fields are an error, not silently
dropped), `versionKey: false`. `toJSON` maps `_id` → `id` and removes `_id`.

**Uniqueness**: none. Two books with the same title and author are allowed (spec Assumptions).

**Indexes**: only the default `_id` index (research R8).

## Value set: Reading Status

| Code (API, URL, database) | Label (UI, Russian) | Meaning |
|---------------------------|---------------------|---------|
| `want_to_read` | Хочу прочитать | Planned; the default. |
| `reading` | Читаю | In progress. |
| `read` | Прочитано | Finished. |

The set is fixed in this feature. The codes are defined once in the contract; the generated type
`BookStatus` and the constant `BOOK_STATUSES` in `packages/shared` come from it. Labels live only in
the frontend (`frontend/src/lib/status.ts`).

### Status transitions

Any status can change to any other status (spec Edge Cases: re-reading is allowed).

```text
want_to_read ⇄ reading ⇄ read
      ⇅_____________________⇅
```

- Setting the status a book already has is a valid no-op: the response is `200` with the
  unchanged book.
- A status change does not change `title`, `author`, or `createdAt` (FR-012).
- When two changes reach the server at about the same time, the one saved last is kept (FR-016).

## Business rules (to be unit-tested, Principle I)

Each rule needs a test for the expected case and a test for every listed error/edge case.

| ID | Rule | Expected case | Error / edge cases |
|----|------|---------------|--------------------|
| BR-1 | Normalize text: remove leading and trailing whitespace from `title` and `author`. | `"  Дюна "` → `"Дюна"` | Inner spaces are kept: `"Война  и мир"` stays as is. |
| BR-2 | `title` is required and not empty after BR-1. | `"Дюна"` is accepted. | `""`, `"   "`, missing → error on field `title`. |
| BR-3 | `title` is at most 200 characters after BR-1. | Exactly 200 characters is accepted. | 201 characters → error on `title` that states the limit; 200 characters plus outer spaces is accepted. |
| BR-4 | `author` is required and not empty after BR-1. | `"Фрэнк Герберт"` is accepted. | `""`, `"   "`, missing → error on field `author`. |
| BR-5 | `author` is at most 100 characters after BR-1. | Exactly 100 characters is accepted. | 101 characters → error on `author` that states the limit. |
| BR-6 | All field errors are reported together. | — | Empty title and empty author → two errors in one response. |
| BR-7 | `status` defaults to `want_to_read` when not given. | Missing → `want_to_read`. | Given `reading` → `reading`. |
| BR-8 | `status` must be one of the three codes. | Each code is accepted. | `"done"`, `""`, `"Читаю"` → error on `status`. |
| BR-9 | List order: `createdAt` descending, then `_id` descending. | Newer book first. | Same `createdAt` → order by `_id`; status change does not change position. |
| BR-10 | List filter: only books with the given status; no filter → all books. | `status=reading` → only `reading`. | No match → empty list (not an error). |
| BR-11 | Parse filter from the page address (frontend). | `?status=read` → `read`. | Missing, empty, or unknown value → "all". |
| BR-12 | Status change on a book that does not exist. | — | Valid id that is not found → "not found" error. |
| BR-13 | Add-book form state (frontend `addBookForm` slice): editing a field clears that field's error; a failed validation or a `VALIDATION_ERROR` response sets field errors; a successful save resets the draft. | Valid draft → no errors; after success the draft is empty. | Server `details` for `title` → error shown on `title`; a failed save keeps the draft values. |

## Frontend state (Redux store)

The frontend store is not a second data model; it holds a cached copy of server data and the
state of the add-book form.

| Slice | Contents | Notes |
|-------|----------|-------|
| `booksApi` (RTK Query) | Cached `BookList` per status filter; request status of each query and mutation. | Loaded again after each add or status change (tag `Book`) and on each filter change (FR-013). No optimistic updates (FR-014). |
| `addBookForm` | `title`, `author`, `status` (draft values), `fieldErrors: { title?, author?, status? }`. | Pure reducers; rules from `lib/book-form.ts` (BR-1…BR-8, BR-13). |

The status filter is **not** in the store: the page address (`?status=`) is its only source
(FR-009a, BR-11).

## Validation errors returned to clients

All field errors use the shared error format from the contract (`ErrorResponse`):

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Проверьте введённые данные.",
  "details": [
    { "field": "title", "message": "Укажите название книги." },
    { "field": "author", "message": "Имя автора не может быть длиннее 100 символов." }
  ]
}
```

The frontend shows each `details[].message` next to its field (FR-004, FR-018).

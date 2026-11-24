# Feature Specification: Book Reading List

**Feature Branch**: `001-book-reading-list`

**Created**: 2026-11-24

**Status**: Draft

**Input**: User description: "Пользователь может добавить книгу с названием, автором и статусом:
хочу прочитать / читаю / прочитано. Пользователь видит список книг с фильтром по статусу.
Пользователь может изменить статус книги"

## Clarifications

### Session 2026-11-24

- Q: When two people use the shared list at the same time and one adds a book or changes a
  status, what does the other person see? → A: The last saved change wins; the other person sees
  changes made by others after reloading the page or after their own next action (add, status
  change, or filter change). No real-time updates.
- Q: What level of accessibility does the interface need? → A: Basic level: every action (add,
  filter, status change) works with the keyboard only, every field and control has a clear
  label, and the status is shown as text, not only by color.
- Q: Should the selected status filter be kept after a page reload or when someone opens a link?
  → A: Yes. The filter is part of the page address: it is kept after a reload, works with the
  browser Back and Forward buttons, and a link with a filter can be shared. Without a filter in
  the address, "Все" is selected.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Add a book to the reading list (Priority: P1)

The user adds a book by entering its title and author and choosing one of three reading
statuses: "Хочу прочитать" (want to read), "Читаю" (reading), or "Прочитано" (read). After the
book is saved, it appears in the user's reading list.

**Why this priority**: without the ability to add books, the reading list is empty and no other
feature has any value. This story alone gives a usable minimum product: a place to record books.

**Independent Test**: add a book with a title, an author, and a status; confirm that the book
appears in the list with exactly those values, and that it is still there after the page is
reloaded.

**Acceptance Scenarios**:

1. **Given** the reading list is empty, **When** the user enters the title "Мастер и
   Маргарита", the author "Михаил Булгаков", selects "Хочу прочитать", and saves, **Then** the
   list shows one book with that title, author, and status.
2. **Given** the add-book form is open, **When** the user does not select a status, **Then**
   the status "Хочу прочитать" is used by default.
3. **Given** the add-book form is open, **When** the user leaves the title or the author empty
   (or enters only spaces) and tries to save, **Then** the book is not saved and the user sees
   a message that names the missing field.
4. **Given** a book was added, **When** the user reloads the page or opens the application
   later, **Then** the book is still in the list.

---

### User Story 2 - View the list and filter it by status (Priority: P2)

The user sees all saved books in one list. Each entry shows the title, the author, and the
current status. The user can choose a filter, "Все" (all), "Хочу прочитать", "Читаю", or
"Прочитано", to see only the books with that status.

**Why this priority**: as the list grows, the user needs to answer questions such as "What am I
reading now?" or "What do I want to read next?". Filtering makes the list useful beyond a small
number of books.

**Independent Test**: with books in each of the three statuses already saved, select each filter
in turn and confirm that only the books with the matching status are shown, and that "Все" shows
every book.

**Acceptance Scenarios**:

1. **Given** the list contains 2 books with status "Читаю" and 3 books with other statuses,
   **When** the user selects the filter "Читаю", **Then** exactly those 2 books are shown.
2. **Given** a status filter is selected, **When** the user selects "Все", **Then** all books
   are shown.
3. **Given** no books have the status "Прочитано", **When** the user selects that filter,
   **Then** the user sees a message that there are no books with this status, not an empty
   screen.
4. **Given** the reading list is empty, **When** the user opens the application, **Then** the
   user sees a message that the list is empty and an invitation to add the first book.
5. **Given** the list has several books, **When** the list is shown, **Then** the most recently
   added books appear first.
6. **Given** the user selected the filter "Читаю", **When** the user reloads the page or sends
   the page link to another person who opens it, **Then** the filter "Читаю" is selected and only
   books with that status are shown.
7. **Given** the user selected "Читаю" and then "Прочитано", **When** the user presses the
   browser Back button, **Then** the filter "Читаю" is selected again.

---

### User Story 3 - Change the status of a book (Priority: P3)

The user changes the status of a book already in the list, for example from "Хочу прочитать" to
"Читаю" when they start reading, or from "Читаю" to "Прочитано" when they finish.

**Why this priority**: the status reflects reading progress over time. Without this story, the
user would have to add the book again to show progress. The story depends on books already being
in the list (Story 1).

**Independent Test**: for a saved book, change its status to each of the other two statuses and
confirm that the list shows the new status, and that the change remains after the page is
reloaded.

**Acceptance Scenarios**:

1. **Given** a book has the status "Хочу прочитать", **When** the user changes it to "Читаю",
   **Then** the list shows the book with the status "Читаю", and the title and author do not
   change.
2. **Given** the filter "Читаю" is selected and a book in the list is shown, **When** the user
   changes that book's status to "Прочитано", **Then** the book disappears from the current
   filtered view and appears under the filter "Прочитано".
3. **Given** the status of a book was changed, **When** the user reloads the page, **Then** the
   new status is still shown.
4. **Given** the user changes a status, **When** the change cannot be saved (for example, because
   of a connection problem), **Then** the user sees an error message and the list shows the
   last saved status of the book.

---

### Edge Cases

- A title or author that contains only spaces is treated as empty and rejected. Leading and
  trailing spaces are removed before saving.
- A title longer than 200 characters or an author longer than 100 characters is rejected with a
  message that states the limit.
- A title or author can contain any letters (Cyrillic, Latin, and others), digits, and
  punctuation, and is shown exactly as entered (after trimming spaces).
- Two books with the same title and author can both be saved; the system does not block
  duplicates.
- Any status transition is allowed, including moving back (for example, from "Прочитано" to
  "Читаю" for a re-read). Selecting the status a book already has changes nothing.
- If saving a new book fails, the form keeps the entered values so that the user can try again,
  and the user sees an error message.
- If the user submits the add-book form twice quickly, only one book is created.
- If two people change the status of the same book at almost the same time, the change saved
  last is kept. Neither person sees an error.
- Changes made by other people are not pushed to an open page. They appear when the page is
  reloaded or when the list is loaded again after the user's own next action (add, status
  change, or filter change).
- If the page address contains an unknown filter value (for example, a mistyped status), the
  list opens with "Все" selected and no error is shown.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Users MUST be able to add a book by providing a title, an author, and a status.
- **FR-002**: The status MUST be exactly one of three values: "Хочу прочитать", "Читаю", or
  "Прочитано". No other value can be saved.
- **FR-003**: The status MUST default to "Хочу прочитать" when the user does not choose one.
- **FR-004**: The system MUST reject a book whose title or author is empty or contains only
  spaces, and MUST show a message that names each invalid field.
- **FR-005**: The system MUST limit the title to 200 characters and the author to 100 characters
  (after removing leading and trailing spaces) and MUST show the limit when it is exceeded.
- **FR-006**: The system MUST save books permanently, so that they remain after the page is
  reloaded or the application is opened again.
- **FR-007**: Users MUST be able to see a list of all saved books, with the title, the author,
  and the status of each book.
- **FR-008**: The list MUST be ordered by the date the book was added, newest first.
- **FR-009**: Users MUST be able to filter the list by one status at a time, or choose "Все" to
  see all books.
- **FR-009a**: The selected filter MUST be part of the page address. Reloading the page or
  opening a copied link MUST show the same filter. The browser Back and Forward buttons MUST
  move between previously selected filters. When the address has no filter or an unknown
  filter value, "Все" MUST be selected.
- **FR-010**: The system MUST show a clear message when the list is empty, and a different
  message when no books match the selected filter.
- **FR-011**: Users MUST be able to change the status of any book in the list to any of the
  three statuses.
- **FR-012**: Changing the status MUST NOT change the book's title, author, or position in the
  list order.
- **FR-013**: After an add, a status change, or a filter change, the list MUST be loaded again
  with the latest saved data (including changes made by other people) without the user having to
  reload the page, and MUST respect the selected filter. Real-time updates without a user action
  are not required.
- **FR-014**: When an add or a status change fails, the system MUST show an error message and
  MUST NOT show unsaved data as if it were saved.
- **FR-015**: The application MUST provide one shared reading list without sign-in. Every person
  who opens the application sees and can change the same list of books.
- **FR-016**: When the same book is changed by more than one person, the system MUST keep the
  change that was saved last.
- **FR-017**: Every action in this feature (adding a book, choosing a filter, changing a status)
  MUST be possible with the keyboard only, without a mouse or touch.
- **FR-018**: Every input field and control MUST have a clear text label that is available to
  screen readers. Validation and error messages MUST be linked to the field or action they
  describe.
- **FR-019**: A book's status MUST be shown as text. Color or icons MAY be added but MUST NOT be
  the only way the status is shown.

### Key Entities

- **Book**: one entry in the reading list. Attributes: title (text, required, up to 200
  characters), author (text, required, up to 100 characters), status (one of "Хочу прочитать",
  "Читаю", "Прочитано"), and the date and time when the book was added (used for ordering).
- **Reading Status**: the fixed set of three values that show how far the user has progressed
  with a book. The set does not change in this feature.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A first-time user can add a book in under 30 seconds without instructions.
- **SC-002**: A user can change the status of a book in no more than 2 actions (for example,
  open the status choice and select a new status).
- **SC-003**: After the user selects a filter, the matching books are shown in under 1 second
  for a list of up to 500 books.
- **SC-004**: 100% of saved books, including their latest status, are still present after the
  page is reloaded or the application is reopened.
- **SC-005**: In usability testing, at least 9 out of 10 participants complete all three tasks
  (add a book, filter by status, change a status) on the first attempt.
- **SC-006**: A tester using only the keyboard can add a book, filter the list, and change a
  status, completing all three tasks without a mouse.

## Assumptions

- The user interface is in Russian, and the status names are shown as given in the feature
  description: "Хочу прочитать", "Читаю", "Прочитано".
- Editing a book's title or author, deleting a book, search, sorting options, ratings, notes,
  and cover images are out of scope for this feature.
- There are no user accounts in this feature: no registration, sign-in, or per-user lists. Access
  control for the shared list is out of scope. Accounts can be added later as a separate feature.
- Duplicates are allowed because two editions or re-reads of the same book are valid cases.
- All status transitions are allowed; the order "want to read → reading → read" is typical but
  not enforced.
- The application is used in a modern desktop or mobile web browser with an internet
  connection. Offline use is out of scope.
- A reading list is expected to hold up to about 500 books; larger lists do not need special
  handling (for example, pagination) in this feature.

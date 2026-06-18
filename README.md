# Library Management System

A desktop library management application built with **JavaFX** and **MySQL**, covering book and member CRUD, borrowing/returning with due dates, multi-field search, and a live dashboard.

This is the **Standard** scope of the original capstone brief: full GUI + database + core operations. Reporting (popular-books analytics, exportable reports), authentication/authorization, and JAR packaging for distribution were intentionally left out of this pass — see "Extending This Project" below if you want to add them later.

## 1. Project Overview

The goals of this project are to demonstrate a complete, layered desktop application: a JavaFX user interface backed by a normalized MySQL schema, accessed through a clean DAO → Service → GUI architecture rather than mixing SQL into the UI code.

Core features:
- Add, edit, delete, and search books (by title, author, ISBN, or genre)
- Register, edit, and remove library members
- Borrow and return books with automatic due-date calculation (14-day loan period) and availability tracking
- Dashboard showing total/available books, member count, active borrowings, and overdue items
- Input validation (ISBN format, email format, phone format, year ranges, quantity bounds) and descriptive error dialogs
- Business rules: can't delete a book with copies on loan, can't delete a member with active borrowings, can't borrow when no copies are available, members capped at 5 simultaneous borrowings, inactive members can't borrow

## 2. Architecture

```
com.library
├── Main.java                  Application entry point
├── model/                     Plain data classes
│   ├── Book.java
│   ├── Member.java
│   └── Transaction.java
├── dao/                       Raw SQL, one class per table
│   ├── BookDAO.java
│   ├── MemberDAO.java
│   └── TransactionDAO.java
├── service/
│   └── LibraryService.java    Validation + business rules; the GUI's only entry point
├── util/
│   ├── DatabaseConnection.java   Singleton JDBC connection
│   └── ValidationUtils.java      Reusable field validators
└── gui/
    ├── MainApp.java            Tabbed shell, wires views together
    ├── DashboardView.java
    ├── BooksView.java
    ├── MembersView.java
    ├── TransactionsView.java
    ├── GridPaneForm.java        Small shared dialog-form layout helper
    └── app.css                  Visual theme
```

The GUI never touches a DAO or writes SQL directly — it calls `LibraryService`, which validates input, enforces business rules, and then delegates to the DAOs. This keeps validation logic in one testable place and means swapping the GUI toolkit later wouldn't require touching the data layer.

## 3. Database Design

Three tables: `books`, `members`, `transactions`. A transaction row represents one borrow event; `status` is `BORROWED` or `RETURNED`, and `available` on `books` is kept in sync via the service layer rather than computed on the fly. Foreign keys cascade on delete so removing a book or member also clears its transaction history (the service layer blocks this at the UI level when there's an active loan, so cascading is a safety net, not the primary guard).

Full schema with comments: [`database/schema.sql`](database/schema.sql).

## 4. Setup Instructions

### Prerequisites
- JDK 17 or later
- Maven 3.8+
- MySQL 8.0+ running locally (or reachable over the network)

### Step 1 — Create the database
```bash
mysql -u root -p < database/schema.sql
```
This creates the `library_db` database, all three tables, and a small set of seed rows (4 books, 3 members) so the app isn't empty on first run.

### Step 2 — Configure the connection
By default the app connects to `jdbc:mysql://localhost:3306/library_db` as user `root` with password `password` (see `DatabaseConnection.java`). Override any of these without recompiling by passing system properties:

```bash
mvn javafx:run -Ddb.url="jdbc:mysql://localhost:3306/library_db?useSSL=false&serverTimezone=UTC" \
                -Ddb.user="root" \
                -Ddb.password="yourpassword"
```

### Step 3 — Run
```bash
mvn javafx:run
```

### Step 4 — Run the tests
```bash
mvn test
```
Tests cover validation logic and model behavior (overdue calculation, availability checks) that don't require a live database connection.

### Step 5 — Package a runnable JAR (optional)
```bash
mvn clean package
java -jar target/library-management-app-1.0.0.jar
```

## 5. User Manual

**Dashboard** — at-a-glance counts and a list of currently overdue books.

**Books** — toolbar to add/edit/delete the selected row; search box with a scope dropdown (All Fields, Title, Author, ISBN, Genre) filters the table live as you type.

**Members** — same pattern as Books: register, edit, remove, and search by name or email.

**Transactions** — "Borrow Book" opens a dialog to pick a book and member (due date is set automatically, 14 days out); "Return Selected" marks the highlighted loan returned and restores the book's available count. Overdue loans are flagged in red in the Status column.

Any action that violates a business rule (no copies available, member at borrowing limit, deleting a book that's on loan, duplicate ISBN/email, malformed input) shows a plain-language error dialog instead of failing silently.

## 6. Known Limitations / Not Included in This Build

- No authentication/login screen — anyone running the app has full access
- No PDF/Excel report export or popular-books analytics dashboard
- No automated GUI tests (JavaFX UI testing requires TestFX and a display, which wasn't in scope for this pass)
- Default DB credentials are placeholders meant for local development only — change them before using this anywhere with real data

## 7. Extending This Project

Reasonable next additions, roughly in order of effort: a login dialog gating access by role (librarian vs. admin) backed by a `users` table; a Reports tab built on top of `TransactionDAO`/`BookDAO` with simple aggregate queries (most-borrowed books, busiest days) rendered as JavaFX charts; CSV or PDF export of the overdue list; and a `maven-jpackage-plugin` step to produce a native installer instead of a plain JAR.
# Library-Management-App

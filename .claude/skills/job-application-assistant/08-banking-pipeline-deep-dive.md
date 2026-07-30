# Technical Deep-Dive: Banking Data Pipeline — Data Cleaning & Audit Log

Standalone reference for technical interview rounds on this project (listed on the CV as "Banking Data Pipeline - Data Cleaning & Audit Log", 05/2026). Not part of the `/apply` skill's automatic routing (01-07) — pull this up manually when prepping for a technical deep-dive or take-home discussion. Source: `pipeline` repo on GitHub (local copy inspected: `C:\Users\LORE\Downloads\pipeline-main\testproject\`).

## One-paragraph pitch (use this to open)

"I built an end-to-end pipeline that simulates 150 bank accounts and their transactions, deliberately injects realistic data-quality problems into that data — duplicates, nulls, malformed dates, typo'd categorical values — then cleans it in SQLite through a series of staged Python/SQL scripts, and logs every single change to an audit table with the old value, new value, action taken, and reason. On top of that I added SQL triggers that keep a running account-balance table in sync automatically, and a reporting layer that turns the audit log into a summary report, a bilingual exception report, and five data-quality visualizations. The whole thing is deterministic — same random seed every run — so it's reproducible and easy to demo."

## Architecture: pipeline stages

The pipeline is 8 scripts, orchestrated by `main.py` via `subprocess.run()` in a fixed sequence (stops immediately on any non-zero exit code):

1. `create-csv.py` — generate synthetic accounts/transactions with injected errors → `accounts.csv`, `transactions.csv`
2. `sql-filltables.py` — create SQLite schema, load CSVs, snapshot originals
3. `sql-duplicates.py` — detect and quarantine duplicate transactions
4. `sql-empty.py` — handle NULLs/empty strings, impute what's safely inferable, drop what isn't
5. `sql-cleaning.py` — exploratory: dumps distinct values per categorical column (used this during development to decide which fuzzy-match targets to hardcode in step 6; doesn't mutate data)
6. `function_cleaning.py` — the core cleaner: date normalization, fuzzy-matched categorical correction, quarantining invalid/future/non-numeric rows, writes every change to `cleaning_log`
7. `balance.py` — creates the `balance` table and 3 triggers that keep it in sync with `transactions`
8. `log.py` — re-runs the same cleaning functions as step 6 (idempotent — see "known redundancy" below), then generates all reports/visualizations from `cleaning_log`

**Libraries used and why:**
| Library | Used for |
|---|---|
| `sqlite3` (stdlib) | The database engine — file-based, zero-config, perfect for a self-contained demo project |
| `csv` (stdlib) | Reading/writing the intermediate CSV files between the generation and loading stages |
| `random` | All synthetic-data randomization, seeded (`random.seed(42)`) for reproducibility |
| `datetime` / `dateutil.parser` | Date arithmetic and **flexible parsing of inconsistently-formatted date strings** — `dateutil.parser.parse()` is the key tool that lets the cleaner accept dates in 7+ different formats and normalize them |
| `pytz` | Real IANA timezone objects (`Europe/Berlin`, `America/New_York`, `Europe/London`) attached to both accounts and transactions, since banking data is genuinely timezone-sensitive |
| `difflib` (stdlib) | `get_close_matches()` — fuzzy string matching used to auto-correct typo'd categorical values (e.g. `"ACTVE"` → `"ACTIVE"`) against a known-valid list, using a similarity cutoff |
| `pandas` | Loads the `cleaning_log` SQL table into a DataFrame for aggregation (`.value_counts()`, `.mode()`, `.groupby()`) when building reports |
| `matplotlib` / `seaborn` | The 5 PNG visualizations (bar charts, pie chart, time series, heatmap) |
| `os` | Creating the `cleaning_reports/` output directory |
| `subprocess` / `sys` | `main.py`'s orchestration — runs each script as its own Python process |

---

## 1. Simulating realistic errors in the CSV (`create-csv.py`)

This is the part that makes the project convincing as a *quality* demo rather than a toy: the errors are injected deliberately and are traceable, not just "messy data I found somewhere."

**Base generation:**
- `generate_account(num_accounts=150)` builds each account with a random type (`CACC`/`SAVG`/`LOAN`), a weighted-random timezone (80% Berlin, 10% NY, 10% London — reflecting a realistic home-market skew), a status (85% `ACTIVE` / 15% `CLOSED`), and an opening date drawn from a date range that's *usually* 2023–2024 but 5% of the time deliberately set to a Dec-2024/Jan-2025 window — this 5% becomes the seed for "account opened suspiciously close to now" edge cases that later interact with transaction-date validation.
- `generate_transactions()` creates 1–3 transactions per account (`min_txn`/`max_txn`), each with an amount drawn from a weighted tier system (`generate_amount()`: 10% chance €100-199, 20% chance €200-599, 60% chance €600-1999, 10% chance €2000-2500 — modeling a realistic skew toward mid-size transactions), a currency that's usually consistent with the account's own currency but has a 10-20% chance of drifting to a different one (simulating cross-currency transactions), and a Debit/Credit flag that flips the sign of the amount (`amount = abs(amount) if DEBIT else -abs(amount)`).
- Transaction dates are constructed relative to the account's opening date with a small (5%) chance of landing *before* the account's own opening date — this is intentional: it's what `invalid_transaction_dates()` in the cleaning stage is designed to catch.

**Error-injection functions (applied in a fixed pipeline order, each wrapping the output of the previous one):**

| Function | What it does | Applied at |
|---|---|---|
| `add_row_duplicates(table, percent_duplicates)` | Picks `n * percent` existing rows at random, duplicates them, shuffles the duplicates back into the table | 5% of accounts, 10% of transactions |
| `add_partial_nulls(table, percent_nulls, min_fields, max_fields, exclude_cols)` | Picks `n * percent` rows, and for each blanks out a random 1–5 non-ID fields (sets them to `""`) | 5% of both tables (never touches the ID columns, via `exclude_cols`) |
| `randomize_date_formats(table, date_col_idx, percent_change)` | Re-parses a valid date and re-serializes it into one of **7 different string formats** (ISO8601, `dd/mm/yyyy`, `mm-dd-yyyy 12h`, `yyyy/mm/dd`, `dd-Mon-yyyy`, etc.) | 15% of date fields in both tables — this is the main driver of the "date format chaos" the cleaner has to normalize |
| `add_field_errors(table, percent_error, error_fields)` | Picks `n * percent` rows and, per targeted field, applies one of 4 corruption types at random: **typo** (via a hardcoded `typo_map`, e.g. `"BOOK"→"BOK"`, `"ACTIVE"→"ACTVE"`), **nonsense** (replaces with `"???"`/`"N/A"`/`"null"`/`"xxxx"`), **case** (random upper/lowercase), or **whitespace** (pads with leading/trailing spaces) | 9% of rows, targeted at categorical columns (Status/Currency for accounts; DebitCredit/Status/Timezone for transactions) |

Everything is seeded (`random.seed(42)` at the top of the file), so **every run produces byte-identical CSVs** — this is what makes the pipeline reproducible and safe to demo live without surprises.

---

## 2. Database schema & loading (`sql-filltables.py`)

Two core tables, normalized with a foreign key:

```sql
CREATE TABLE accounts (
    AccountId INTEGER PRIMARY KEY AUTOINCREMENT,
    Status VARCHAR(255), Currency CHAR(10), OpeningDate DATETIME,
    AccountTypeCode CHAR(10), Timezone VARCHAR(255)
);

CREATE TABLE transactions (
    TransactionId INTEGER PRIMARY KEY AUTOINCREMENT,
    AccountId INTEGER NOT NULL, Date DATETIME, Amount INTEGER,
    DebitCredit TEXT, Status TEXT, Timezone TEXT,
    FOREIGN KEY (AccountId) REFERENCES accounts(AccountId)
        ON DELETE CASCADE ON UPDATE CASCADE
);
```

`PRAGMA foreign_keys = ON` is set explicitly — SQLite disables FK enforcement by default, so this is a deliberate choice to get real referential-integrity checking (a row insert with a bad `AccountId` throws `sqlite3.IntegrityError`, which the loader catches and diverts to a `list_empty_accid` list rather than crashing the load).

**On parametric (parameterized) queries:** every single INSERT/UPDATE in the project uses `?` placeholders with values passed as a separate tuple/list argument to `cursor.execute(query, params)` — never Python f-string interpolation of *values* into SQL. For example:
```python
cursor.execute('''
    INSERT INTO accounts (Status, Currency, OpeningDate, AccountTypeCode, Timezone)
    VALUES (?, ?, ?, ?, ?)
''', row[1:])
```
This does two things: (1) it's SQL-injection-safe — the driver sends the query and the data separately to SQLite's engine rather than building one string, so a value like `"'; DROP TABLE accounts;--"` is treated as inert data, not executable SQL; (2) SQLite can reuse the same prepared statement/query plan across every row, which is why `sql-duplicates.py`'s bulk quarantine insert uses `cursor.executemany(insert_query, txns_dupdates)` — one compiled plan, many parameter sets, rather than compiling a fresh query per row.

The only place the code builds SQL with f-strings is for **table and column names** (identifiers), not user-supplied values — e.g. `f"DELETE FROM {table} WHERE {id_column} IN (...)"` in `function_cleaning.py`. Those identifiers come from hardcoded call-site arguments in the same codebase, never from the CSV data, so it's a controlled, low-risk use of dynamic SQL for genericity (letting the same cleaning function work against both `accounts` and `transactions`) — not a place where actual injection risk exists, but worth being able to explain the distinction if asked.

**Data lineage/backup pattern:** immediately after loading, the script runs `CREATE TABLE orig_accounts AS SELECT * FROM accounts` (and the same for transactions) — an untouched snapshot taken *before* any cleaning runs, so the "before" state is always recoverable for comparison or audit, independent of the `cleaning_log`.

---

## 3. Duplicate detection (`sql-duplicates.py`)

Business rule: a duplicate is "the same account transacting on the exact same timestamp" — `GROUP BY Date, AccountId HAVING COUNT(*) > 1`.

The quarantine logic uses a **self-join against a subquery** to identify every row except the one with the lowest `TransactionId` in each duplicate group (i.e., keep the first-inserted copy, quarantine the rest):
```sql
SELECT t.*
FROM transactions t
JOIN (
    SELECT Date, AccountId, MIN(TransactionId) AS min_id
    FROM transactions
    GROUP BY Date, AccountId
    HAVING COUNT(*) > 1
) sub ON t.Date = sub.Date AND t.AccountId = sub.AccountId
WHERE t.TransactionId != sub.min_id
```
Matches get copied into a dedicated `DuplicateDates` table (`cursor.executemany`) and then deleted from `transactions` by ID. This script does **not** write to `cleaning_log` — worth flagging honestly if asked, since it's a real inconsistency: duplicate removal is auditable via the `DuplicateDates` table's existence but isn't captured in the same structured log as the other cleaning actions.

---

## 4. NULL/empty-string handling & imputation (`sql-empty.py`)

Key design decision: **an empty string (`""`) is treated as equivalent to `NULL`**, not as a valid value — the script explicitly normalizes all `""` to `NULL` first (`UPDATE {table} SET {col} = NULL WHERE {col} = ""`) before doing any null analysis, because otherwise `COUNT(*) WHERE col IS NULL` would silently undercount missingness.

**Null-density mapping:** a CASE-expression trick counts, per row, how many of its columns are NULL simultaneously:
```sql
SELECT (CASE WHEN AccountId IS NULL THEN 1 ELSE 0 END) + ... AS num_nulls, COUNT(*)
FROM accounts GROUP BY num_nulls ORDER BY num_nulls DESC
```
This produces a distribution ("X rows have exactly 1 null field, Y rows have 2", etc.) used both for reporting and for the drop decision below.

**Imputation rules actually applied (only where the missing value is safely inferable from other columns — not guessed):**
- `Status` (accounts): if NULL and the account has a transaction within the last 90 days (`EXISTS (... AND t.Date >= DATE('now','-90 days'))`), infer `'OPEN'`.
- `Timezone` (both tables): if NULL, parse the row's own date string with `dateutil.parser.parse()` and pull the timezone name off the parsed object's `tzinfo` if it carries one, else default to `'UTC'`.
- `DebitCredit` (transactions): entirely re-derived from the sign of `Amount` (`CASE WHEN Amount < 0 THEN 'DEBIT' WHEN Amount > 0 THEN 'CREDIT'`), then `Amount` is normalized to always store as a positive magnitude (`SET Amount = ABS(Amount)`) — so sign is no longer overloaded to carry meaning once `DebitCredit` is trustworthy.

**Drop rule:** any `accounts` row with **more than 1** NULL field (after the above imputation) is considered unrecoverable — moved into `AccountsWithMoreThanOneNull` for audit, its transactions cascade-deleted first (to respect the FK), then the account row itself deleted. A single missing field gets a best-effort imputation attempt; two or more is treated as "not enough real information to trust this record."

---

## 5. Core cleaning logic (`function_cleaning.py`) — the heart of the project

Six reusable, parameterized cleaning functions, each called once per (table, column) it applies to — this is deliberately written to be *generic*, not hardcoded per table:

**a) `clean_nonnumeric_rows(conn, table, column, id_column)`** — checks the column's declared SQLite type via `PRAGMA table_info`; if it's not `INTEGER`/`REAL`, it skips (no-op guard). Otherwise finds rows where `typeof(column) NOT IN ('integer','real')`, copies them into a `{table}_nonnumeric_{column}` quarantine table (schema introspected dynamically from `PRAGMA table_info`, all columns cast to `TEXT` for the quarantine copy so anything can be stored), deletes them from the source table, and logs each removal.

**b) `to_iso8601_preserve_tz()` + `correct_date_column()`** — this is where **`dateutil.parser.parse()`** does the heavy lifting: it accepts virtually any of the 7 date-format variants injected in step 1 and returns a proper `datetime` object; if the parsed date has no timezone, it's defaulted to UTC before re-serializing to a strict `YYYY-MM-DDTHH:MM:SS±HH:MM` ISO8601 string. Every row where the string actually changed gets logged as `action='corrected'`; if parsing throws (unparseable garbage), the field is set to `NULL` and logged as `action='set_null'` with the exception message as the reason — nothing is ever silently dropped without an audit trail.

**c) `clean_and_flag_column(conn, table_name, column_name, correct_values, id_column)`** — the **fuzzy-matching corrector**, applied to categorical fields (`Status`, `Currency`, `DebitCredit`, `Timezone`, `AccountTypeCode`). For each distinct value actually present in the column, it runs `difflib.get_close_matches(val, correct_values, n=1, cutoff=0.6)` against the known-valid list — this uses Python's built-in Ratcliff/Obershelp sequence-matching algorithm, so `"ACTVE"` matches `"ACTIVE"` and `"  BOOK  "` (whitespace) matches `"BOOK"` at a 0.6+ similarity threshold, but something genuinely unrecoverable like `"???"` falls below the cutoff and is left unmatched. Every actual correction is logged with both the original and mapped value. Separately, it **adds a `{Column}Cleaned` flag column** to the table (`ALTER TABLE ... ADD COLUMN`) and sets it to `1` for any row whose value still isn't in the valid list even after fuzzy-matching — a durable, queryable marker for "this specific field on this specific row needed intervention and might still be suspect," independent of the log.

**d) `move_and_remove_future_dates(conn, table, date_column, id_column)`** — any row with a date later than `datetime.now()` gets copied to a `{table}_future_{column}` quarantine table and removed, logged with the cutoff timestamp used in the reason string.

**e) `invalid_transaction_dates(conn, ...)`** — a business-logic check, not just a formatting one: joins `transactions` to `accounts` on `AccountId` and flags any transaction dated **before its own account's `OpeningDate`** (`WHERE t.Date < a.OpeningDate`) — this is what catches the 5%-probability edge case deliberately seeded in `generate_account()`/`generate_transactions()`. Quarantined to `transactions_invalid_date`, logged, and removed from the working table.

All six functions funnel through the single **`log_change(conn, table_name, row_id, column_name, original_value, new_value, action, reason)`** helper, which lazily creates the `cleaning_log` table on first call (`CREATE TABLE IF NOT EXISTS`) and inserts one row per change:

```sql
CREATE TABLE cleaning_log (
    log_id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT, row_id TEXT, column_name TEXT,
    original_value TEXT, new_value TEXT, action TEXT, reason TEXT,
    cleaned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```
This is the audit trail: for every mutation, you can answer *which table, which row, which column, what it was, what it became, what kind of action (`corrected`/`removed`/`set_null`), why, and when* — exactly the shape an auditor or a regulator-facing team (like Bundesbank's AnaCredit) would expect.

---

## 6. Trigger-based balance maintenance (`balance.py`) — "how are trigger queries implemented"

A dedicated `balance` table (one row per account, `AccountId UNIQUE`, cascades on account delete/update) is kept in sync **entirely at the database layer**, not recomputed by application code, via three `AFTER` triggers on `transactions`:

```sql
CREATE TRIGGER update_balance_after_transaction_insert
AFTER INSERT ON transactions FOR EACH ROW
BEGIN
    UPDATE balance
    SET Balance = (SELECT IFNULL(SUM(Amount),0) FROM transactions WHERE AccountId = NEW.AccountId),
        DateLastUpdate = (SELECT MAX(Date) FROM transactions WHERE AccountId = NEW.AccountId)
    WHERE AccountId = NEW.AccountId;
END;
```
- **INSERT trigger** — after any new transaction, recomputes that account's balance as `SUM(Amount)` over all its transactions (wrapped in `IFNULL(...,0)` so an account with zero transactions reports 0, not NULL) and stamps `DateLastUpdate` with the latest transaction date.
- **UPDATE trigger** — handles the case where a transaction's `AccountId` itself changes (a transaction gets reassigned to a different account): recomputes balance for the *new* account (via `NEW.AccountId`), **and** separately recomputes the *old* account's balance too (via `OLD.AccountId`, guarded by `WHERE OLD.AccountId != NEW.AccountId` so it doesn't do redundant work when only the amount changed, not the owner).
- **DELETE trigger** — after a transaction is removed, recomputes the affected account's balance using `OLD.AccountId` (the only reference available post-delete).

Why this matters as a design choice to be able to defend in an interview: the balance is **never stored as something the application computes and writes** — it's a derived aggregate that SQLite itself guarantees stays consistent with the underlying `transactions` table no matter *what* touches that table (this pipeline's own scripts, a future API, a manual `sqlite3` CLI edit) — you cannot get transactions and balance out of sync as long as they go through the same connection/triggers, which is exactly the kind of invariant a bank's ledger needs. The trade-off worth being honest about: triggers recompute the **full** `SUM()` on every single insert/update/delete rather than doing an incremental `+= Amount`, which is O(n) per transaction rather than O(1) — perfectly fine at this dataset's scale (a few hundred rows), and I'd talk through switching to incremental updates (`Balance = Balance + NEW.Amount`) if asked about scaling this to millions of transactions.

The `balance` table itself is seeded once for **every** account, including those with zero transactions (`LEFT JOIN transactions ... GROUP BY a.AccountId`), so the balance table's row count always equals the account table's row count — no account is silently missing a balance row.

---

## 7. Reporting layer (`log.py`)

Reads the entire `cleaning_log` table into a `pandas.DataFrame` via `pd.read_sql_query()` and derives everything from there:

- **`summary_report.txt`** — total change count, number of distinct tables/columns touched, the most-frequently-changed table and column (`.mode()`), the most common action type, and the date range the log spans.
- **5 visualizations** (matplotlib + seaborn, saved as numbered PNGs so they sort in a logical viewing order):
  1. `1_changes_by_table.png` — horizontal bar chart, changes per table
  2. `2_changes_by_column.png` — horizontal bar chart, top 10 columns by change count
  3. `3_actions_pie.png` — pie chart of `corrected` vs `removed` vs `set_null`
  4. `4_changes_over_time.png` — line chart of cleaning activity by date (only generated if the log spans multiple days; on a single run this degenerates to one point)
  5. `5_heatmap_table_action.png` — `pandas.crosstab(table_name, action)` rendered as a seaborn heatmap, showing which tables needed which *kinds* of intervention (e.g. "transactions needed mostly date corrections, accounts needed mostly categorical corrections")
- **`exception_report.txt`** — a human-readable, **bilingual** (English headers + Spanish body — `RESUMEN EJECUTIVO`, `DETALLE POR TIPO DE ERROR`) report grouped by `reason` (`df.groupby('reason')`), showing count and one concrete before/after example per error type — this is the "hand this to someone non-technical" artifact.
- **`cleaning_log_export.csv`** — the full log as CSV, for anyone who wants to pull it into Excel/another tool rather than query SQLite directly.

**Actual numbers from a real run** (useful to quote concretely in an interview rather than speaking only in the abstract):
- **131 total logged changes**, across **2 tables** and **5 columns** (`Date`, `OpeningDate`, `Status`, `Timezone`, `Currency`)
- **65** were date-format normalizations to ISO8601 (the single largest category — consistent with 15% of all date fields being deliberately reformatted at generation time)
- **41** were transactions removed for predating their account's opening date (the business-logic check)
- The remaining ~25 were fuzzy-matched categorical corrections spread across `Status`, `Currency`, and `Timezone` (e.g. `"ACTVE"→"ACTIVE"` ×4, `"EURO"→"EUR"` ×2, `"US"→"USD"` ×2, whitespace-padded values like `"  BOOK  "→"BOOK"`, and one lowercase `"america/new_york"→"America/New_York"`)
- Most-cleaned table: `transactions`. Most common action: `corrected` (outnumbering `removed`/`set_null`).

## Known limitations / redundancies (be ready to name these yourself — it reads as maturity, not weakness)

- **`function_cleaning.py` and `log.py` duplicate the same six cleaning functions and the same call sequence at the bottom of each file.** `main.py` runs both in sequence, so cleaning genuinely executes twice per pipeline run. This happens to be *safe* because every cleaning step is idempotent (re-normalizing an already-ISO8601 date is a no-op logged as "no change" since `original == corrected`; re-matching an already-correct categorical value doesn't touch it) — but it's real duplicated code I'd refactor by importing the functions from `function_cleaning.py` into `log.py` instead of copy-pasting them, and I'd say so directly if asked "how would you improve this."
- **`sql-duplicates.py` doesn't write to `cleaning_log`** — duplicate removal is audited only via the existence of the `DuplicateDates` table, not the same structured log as everything else. I'd unify this by having it call `log_change()` too.
- **`sql-cleaning.py` is a leftover exploration script** (dumps distinct categorical values per column) that doesn't mutate any data — genuinely useful during development to decide which values to hardcode as `correct_values` in `clean_and_flag_column()`, but it's dead weight in the production pipeline now and I'd either delete it or repurpose it as a config-generation step.
- **Balance triggers recompute the full `SUM()`** on every transaction change rather than incrementing — a deliberate simplicity-over-scale trade-off at this dataset size, discussed above.

## Likely follow-up questions and how to answer them

- **"Why SQLite and not Postgres/MySQL?"** — Zero setup, file-based, perfect for a self-contained portfolio project anyone can clone and run without provisioning a server; I'd use Postgres for a production/multi-user system, and the SQL itself (aside from SQLite-specific pragmas like `PRAGMA foreign_keys=ON` and `IFNULL` vs `COALESCE`) would port over with minimal changes.
- **"How would you scale this to real transaction volumes?"** — Batch/streaming ingestion instead of one CSV load; incremental trigger updates instead of full `SUM()` recomputation; indexing `(AccountId, Date)` since that's the pair every duplicate/lookup query filters on; moving the cleaning log to append-only partitioned storage.
- **"What would you add for production use?"** — Row-level security by reporting entity, automated data-quality alerting instead of a static report, incremental/scheduled runs instead of full-table reprocessing, and probably a proper migration tool instead of `CREATE TABLE IF NOT EXISTS` scattered across scripts.
- **"How do parameterized queries protect against SQL injection here, concretely?"** — Values always travel via `?` placeholders bound separately from the query string (see section 2); the only dynamic SQL is over table/column *identifiers*, which are hardcoded in the calling code, never derived from the untrusted CSV data itself.

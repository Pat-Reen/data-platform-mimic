# Claude Code Demo — Data Platform Migration

## What this project is

This is a live demo of Claude Code's ability to explore an unknown database, understand its structure, and autonomously build a production-quality Python notebook — starting from nothing but a connection string.

The database is a Supabase/PostgreSQL instance modelling an insurance data platform (policies, policyholders, a join/reporting table, and stored procedures that refresh it).

---

## Demo flow

The demo has three acts. Each act is driven by a plain-English prompt from the presenter:

### Act 1 — Explore the database
Prompt: *"Explore the database and describe the tables and any stored procedures."*

Expected output:
- Connect to the DB using `DATABASE_URL` from `.env`
- Introspect `information_schema` and `pg_proc` to discover tables, columns, constraints, row counts, and stored procedure source code
- Write a clear summary — ideally `describe_tables.py` and `describe_functions.py` as reusable exploration scripts

### Act 2 — Build the notebook
Prompt: *"Create a Jupyter notebook that does the same job as the stored procedure, but in Python — with proper logging and an audit trail."*

Uses findings from `describe_functions.py` and `describe_tables.py` to understand what the stored procedure does, then builds the notebook.

**Important — notebook creation order:**
1. Use the `Write` tool to create an empty `refresh_policy_policyholder.ipynb` skeleton first (the `NotebookEdit` tool requires the file to already exist before it can add cells)
2. Then use `NotebookEdit` to add each cell in sequence

Expected output:
- A notebook (`refresh_policy_policyholder.ipynb`) that:
  - Reads `policies` and `policyholders` into pandas DataFrames
  - Joins them and computes `premium_multiple = sum_insured / annual_premium`
  - Runs data quality checks (nulls, zero premiums, unmatched joins)
  - Truncates `policy_policyholder` and reloads it inside an explicit transaction
  - Verifies row counts before committing; rolls back on mismatch
  - Logs every step with timestamps using Python `logging`
  - Builds an `audit_trail` list and renders it as a styled DataFrame at the end
  - Reads `DATABASE_URL` from `.env` via `python-dotenv`

### Act 3 — Suggest and plan improvements

The notebook is now complete. The presenter will show it running, then hand back to Claude for a critical review.

Prompt: *"Now that we have the notebook working, what improvements would you suggest to make this production-ready? Don't implement anything yet — just tell me what you'd change and why."*

Expected output — Claude should surface 4–5 concrete, ranked suggestions:

1. **Incremental refresh** — replace the full truncate-reload with a change-detection approach that only processes new or updated policies (reduces DB load, safer for large tables)
2. **Schema validation with pandera** — replace the ad-hoc quality checks with a declared schema (column types, value ranges, non-null rules) so failures are explicit and testable
3. **Alerting on failure** — hook the except/rollback path to send a Teams message or email notification so failures don't go unnoticed overnight
4. **Scheduling wrapper** — wrap the notebook logic in a plain Python script callable by a job scheduler, so the refresh runs automatically on a cadence
5. **Unit tests** — add a pytest suite for the join logic and `premium_multiple` calculation so regressions are caught before they reach the database

After presenting the list Claude should ask: *"Which of these would you like me to implement?"*

This is the moment that shows Claude can think critically about its own output, not just execute instructions.

---

## Key files

| File | Role |
|---|---|
| `.env` | DB connection string — the only file present at demo start |
| `describe_tables.py` | Generated in Act 1 — introspects table structure |
| `describe_functions.py` | Generated in Act 1 — lists and prints stored proc source |
| `refresh_policy_policyholder.ipynb` | Generated in Act 2 — the main demo artefact |

---

## Resetting for a repeat demo

To run the demo again from scratch, remove the generated files:

```bash
rm describe_tables.py describe_functions.py refresh_policy_policyholder.ipynb
```

Only `.env`, `.gitignore`, `README.md`, and `CLAUDE.md` should remain. Then start the conversation fresh.

## Environment

- DB: Supabase/PostgreSQL (connection string in `DATABASE_URL`)
- Python dependencies: `psycopg2`, `pandas`, `python-dotenv`
- Notebook runtime: Jupyter

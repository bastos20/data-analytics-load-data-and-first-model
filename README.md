## Context

In the previous challenge, you installed dbt Core, configured a DuckDB connection, and defined your three Jaffle Shop sources in `schema.yml`. Now you'll put that setup to use: explore the data with DuckDB, and write your first transformation.

## Objective

Explore the data with DuckDB, and write your first dbt staging model. By the end of this challenge, raw data will be transformed into a clean, documented view using dbt's `source()` macro and the staging layer pattern.

## Prerequisites

- Completed the previous challenge with a working `jaffle_shop_dbt/` directory containing:
  - `dbt_project.yml` configured with materialization and schema settings
  - `models/schema.yml` with source definitions for the three Jaffle Shop tables
  - `~/.dbt/profiles.yml` with a working DuckDB connection (`dbt debug` passes)
  - `~/.lewagon/jaffle/dev.duckdb` shared database created

## 0. Copy Your Work from Previous Challenge

Each challenge has its own directory. Copy your dbt project files from the previous challenge.

**📍 In your terminal:**

```bash
cp -rP ~/code/<user.github_nickname>/{{ local_path_to("03-Data-Transformation/09-Data-Layers-And-Intro-DBT/01-Setup-DBT") }}/jaffle_shop_dbt .
```

Then commit so you have a clean starting point for this challenge:

```bash
git add jaffle_shop_dbt
git commit -m "Copied setup from previous challenge"
git push origin master
```

## 1. Load Raw Data into DuckDB

## 1.1 Loading The Data

Normally you'd download the Jaffle Shop dataset [from dbt Labs' GitHub](https://github.com/dbt-labs/jaffle-shop-classic/tree/main/seeds) and load it into DuckDB. We have already done that for you and created the duckdb database file for you. It's the one you downloaded in the previous challenge.

So nothing to do here.

Note: If you'd ever want to see how we did it, have a look at the `LOADING_DATA.md` in your challenge folder.

## 1.2 Let's Check The Data

Let's use the DuckDB CLI to quickly check that our data is present. You could also use the web interface, but for a quick simple check like this, the terminal is great.

**📍 In your terminal**, run:

```bash
duckdb ~/.lewagon/jaffle/dev.duckdb
```

You will always use this same command to open the database. From now on, you can use your terminal history if you need to reopen it: type `duckdb` and then the arrow-up key ⬆️ on your keyboard (the one on the bottom right). This will bring this exact command back.

You will now see another prompt: `dev D`. This indicates that you're inside the DuckDB CLI. Normal terminal commands won't work in here. Only DuckDB commands, and SQL queries.

Now, after the DuckDB prompt `dev D`, type:

```bash
.tables
```

You should see this output:

```markdown
 ─────────────────────────────── dev ──────────────────────────────── 
 ─────────────────────────────── raw ──────────────────────────────── 
┌────────────────────────┐┌────────────────────┐┌────────────────────┐
│      raw_payments      ││     raw_orders     ││   raw_customers    │
│                        ││                    ││                    │
│ id             bigint  ││ id         bigint  ││ id         bigint  │
│ order_id       bigint  ││ user_id    bigint  ││ first_name varchar │
│ payment_method varchar ││ order_date date    ││ last_name  varchar │
│ amount         bigint  ││ status     varchar ││                    │
│                        ││                    ││      100 rows      │
│        113 rows        ││      99 rows       │└────────────────────┘
└────────────────────────┘└────────────────────┘
```

If not, ask a TA for help.

Back at the DuckDB prompt, type this to exit the DuckDB CLI:

```bash
.exit
```


## 2. Create Your First Model

### 2.1. Create Staging Model

<details>
<summary markdown="span">**💡 Understanding `source()` and Jinja**</summary>

**What is `{{ source() }}`?**

The `{{ source() }}` macro is part of Jinja templating in dbt. Jinja lets you write dynamic SQL.

**Why use `{{ source() }}` instead of writing table names directly?**

```sql
-- ❌ Without source macro (brittle)
SELECT * FROM raw.raw_customers

-- With source macro (flexible)
SELECT * FROM {{ source('jaffle_shop', 'raw_customers') }}
```

**Benefits:**

- If raw table moves/renames, change it once in `schema.yml`
- dbt tracks lineage (raw data → models)
- Can add freshness tests to sources
- Shows in dbt documentation automatically

**Double curly braces** tell dbt to replace expressions with actual SQL before running.

</details>

**📍 In your terminal**, create the staging directory:

```bash
# Navigate to your dbt project
cd jaffle_shop_dbt

# Create staging directory
mkdir -p models/staging
```

**📝 In VS Code**, create a new file `models/staging/stg_customers.sql` with this SQL:

```sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('jaffle_shop', 'raw_customers') }}
)

SELECT
    id AS customer_id,
    first_name,
    last_name
FROM source
```

**💡 Why include `config()` when it's already set in `dbt_project.yml`?** The `dbt_project.yml` already sets `+materialized: view` as the default for the staging layer — so technically this block is redundant. We include it here explicitly so the intent is clear inline: anyone reading this file immediately knows it materialises as a view, without having to cross-reference the project config. This is a style choice; once you're comfortable with the project structure you can omit inline `config()` blocks if the project default already matches.

**💾 Save the file** (Cmd+S or Ctrl+S).

### 2.2. Run Your First Model

Before you execute any `dbt` commands, you should always disconnect the DuckDB CLI or UI from the database. This time we have already done that for you, so things should go smoothly.

**💡 Why?** DuckDB only allows one process to write to the database at a time. The DuckDB CLI holds a lock on the database file, which prevents dbt from running. You can reconnect the DuckDB CLI any time after dbt finishes if you want to query the results directly.

**📍 In your terminal**, navigate to your dbt project if you are not already there:

```bash
cd jaffle_shop_dbt
```

Then run the staging model:

```bash
dbt run --select stg_customers
```

<details>
<summary markdown="span">**Expected output**</summary>

```bash
Completed successfully
Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

</details>

<details>
<summary markdown="span">**🔧 Troubleshooting: Model Run Issues**</summary>

### "Conflicting lock" or "Could not set lock on file" error

**Problem:** DuckDB CLI (or another process) has the database file locked

**📍 Check all your open terminals:**

- Standalone terminal windows.
- Terminals inside VS Code.

If any of them are still running the DuckDB CLI, type `.exit` to exit.

**📍 Then retry:**

```bash
dbt run --select stg_customers
```

**After dbt finishes**, you can reconnect DuckDB to query the results.

### "Compilation Error" or SQL syntax errors

**📍 In your terminal**, compile to see the error:

```bash
dbt compile --select stg_customers
```

Check the compiled SQL in `target/compiled/jaffle_shop_dbt/models/staging/stg_customers.sql`

**Common issues:**

- Missing comma in SELECT statement
- Typo in source name: must match schema.yml exactly
- Missing closing parenthesis or quote

### Model runs but produces 0 rows

**Test source data directly:**

**🗄️ In DuckDB**, run:

```sql
SELECT COUNT(*) FROM raw.raw_customers;
-- Should return 100

SELECT * FROM raw.raw_customers LIMIT 5;
-- Should show customer data
```

If source has data but model is empty, check your WHERE clauses or JOIN conditions

### "Not in correct directory" error

**Check where you are:**

```bash
pwd
# Should end with: .../jaffle_shop_dbt
```

**Navigate to correct directory (from your challenge directory):**

```bash
cd jaffle_shop_dbt
```

**Always run dbt commands from your project root** (where dbt_project.yml lives)

</details>

Now open the compiled output to see what dbt actually ran:

**📍 In your terminal:**

```bash
cat target/compiled/jaffle_shop_dbt/models/staging/stg_customers.sql
```

The `{{ source('jaffle_shop', 'raw_customers') }}` macro has been replaced with the real table reference — this is exactly what DuckDB executed. Any time you want to understand what a model actually runs, check `target/compiled/`.

## 🧪 Checkpoint 1: Push First Staging Model

**Before pushing, validate your staging model:**

**📍 In your terminal**, if you are not already in the challenge directory, navigate there now:

```bash
cd ..
```

Then run the tests:

```bash
pytest tests/test_staging_model.py -v
```

**Optional: Run all tests together:**

```bash
make
```

**If all tests pass**, commit your staging model:

```bash
# Stage the staging model
git add jaffle_shop_dbt/models/staging/

# Commit with descriptive message
git commit -m "Add first staging model: stg_customers"

# Push to GitHub (triggers automated tests)
git push origin master
```

**❌ Any failures?** → Review the 🔧 Troubleshooting dropdowns in each section above or raise a ticket with a TA!

**All checks passed?** Congratulations! You're ready to start building data transformations with dbt Core!

---

Your first staging model is running. `stg_customers` selects from the raw source using `source()` and renames columns for consistency — the foundation every other model in this unit builds on. The next challenge adds additional configuration to make those source references more precise.

---

<details>
<summary markdown="span">**Useful Tips and Resources**</summary>

**📍 Common dbt commands** you'll use in upcoming challenges:

```bash
# Run models
dbt run                          # All models
dbt run --select model_name      # One model
dbt run --select folder        # All in folder

# Test models
dbt test                         # All tests
dbt test --select model_name     # Tests for one model

# Documentation
dbt docs generate                # Build docs
dbt docs serve                   # View at localhost:8080

# Debugging
dbt compile                      # Compile without running
dbt debug                        # Test connection
dbt list                         # Show all models
```

**📝 In VS Code**, install the dbt Power User extension:

- Click Extensions icon (Cmd+Shift+X / Ctrl+Shift+X)
- Search "dbt Power User"
- Click Install
- Features: SQL syntax highlighting, model previews, lineage graphs

**💡 Tip:** Use VS Code's integrated terminal (View → Terminal) to run dbt commands

---

#### Resources

- [dbt Core Docs](https://docs.getdbt.com/docs/core/installation)
- [dbt-duckdb Adapter](https://github.com/duckdb/dbt-duckdb)
- [DuckDB SQL Reference](https://duckdb.org/docs/sql/introduction)
- [dbt Best Practices](https://docs.getdbt.com/guides/best-practices)

</details>

---

## 🎉 Challenge Complete

Raw data is in DuckDB and your first staging model is running.

**Key takeaways:**

- `source()` is a reference, not a string — update the source definition once and every model using it updates automatically
- DuckDB allows only one writer at a time — disconnect DBeaver before `dbt run`, reconnect to query results
- `target/compiled/` holds the plain SQL dbt generated — you saw this after running `stg_customers`: `{{ source() }}` is replaced with the real table reference before the query runs

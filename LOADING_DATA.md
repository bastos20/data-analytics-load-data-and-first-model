# Loading CSV files into DuckDB

For reference only - no need to do this.


### Step 1: Download Data from GitHub

**📍 In your terminal**, download the CSV files:

```bash
# Create data directory
mkdir -p ~/.lewagon/jaffle/csv

# Download Jaffle Shop CSV files from dbt Labs
curl -o ~/.lewagon/jaffle/csv/raw_customers.csv \
  https://raw.githubusercontent.com/dbt-labs/jaffle_shop/main/seeds/raw_customers.csv

curl -o ~/.lewagon/jaffle/csv/raw_orders.csv \
  https://raw.githubusercontent.com/dbt-labs/jaffle_shop/main/seeds/raw_orders.csv

curl -o ~/.lewagon/jaffle/csv/raw_payments.csv \
  https://raw.githubusercontent.com/dbt-labs/jaffle_shop/main/seeds/raw_payments.csv

# Verify downloads succeeded
ls -lh ~/.lewagon/jaffle/csv/
# Should show: raw_customers.csv, raw_orders.csv, raw_payments.csv (~6KB total)
```


### Step 2: Load Data into DuckDB

**Before you run the SQL below, let's understand what it does:**

1. `CREATE SCHEMA IF NOT EXISTS raw;`
   - Creates a "folder" for organizing tables
   - `IF NOT EXISTS` prevents errors if schema already exists

2. `read_csv_auto('path/to/file.csv')`
   - DuckDB function that reads CSV files
   - "auto" means it detects column types automatically
   - Super useful for quick CSV imports!

3. `CREATE OR REPLACE TABLE ... AS SELECT * FROM read_csv_auto(...)`
   - Creates a permanent table from the CSV data (or replaces it if it already exists)
   - `AS SELECT` pattern: create table from query results

**This pattern is useful whenever you need to load CSV files into any database!**

**In your terminal**, open the DuckDB CLI:

```bash
duckdb ~/.lewagon/jaffle/dev.duckdb
```

**Copy and paste** the SQL below into the DuckDB CLI and hit Enter (if needed):

  ```sql
  -- Create schema for raw data
  CREATE SCHEMA IF NOT EXISTS raw;

  -- Load customers (replace YOUR_FULL_PATH_HERE with your actual path)
  CREATE OR REPLACE TABLE raw.raw_customers AS
  SELECT * FROM read_csv_auto('~/.lewagon/jaffle/csv/raw_customers.csv');

  -- Load orders
  CREATE OR REPLACE TABLE raw.raw_orders AS
  SELECT * FROM read_csv_auto('~/.lewagon/jaffle/csv/raw_orders.csv');

  -- Load payments
  CREATE OR REPLACE TABLE raw.raw_payments AS
  SELECT * FROM read_csv_auto('~/.lewagon/jaffle/csv/raw_payments.csv');

  -- Verify data loaded
  SELECT 'customers' as table_name, COUNT(*) as row_count FROM raw.raw_customers
  UNION ALL
  SELECT 'orders', COUNT(*) FROM raw.raw_orders
  UNION ALL
  SELECT 'payments', COUNT(*) FROM raw.raw_payments;
  ```


Check the output for the row counts of each table.

  You should see:

  ```bash
  table_name | row_count
  -----------|----------
  customers  | 100
  orders     | 99
  payments   | 113
  ```

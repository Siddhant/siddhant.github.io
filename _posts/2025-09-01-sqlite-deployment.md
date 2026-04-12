---
layout: post
title: "Planning SQLite Deployment to Production"
date: 2025-09-01 02:28:47 +0530
categories: [database, SQLite]
---

SQLite is a simple database, but even then, its production deployment can be complex and tricky.

> This is not a SQLite tutorial — it's a deployment operations playbook. The focus is on the procedures, scripts, and directory structure you need to manage a SQLite database from first deploy through ongoing operations.

# Assumptions:
1. Overall, this guide is for simple deployments. 
2. This guide assumes your project uses a single SQLite database. If your application requires multiple SQLite databases, you will need to adjust the architecture and deployment procedures accordingly.

# Table of Contents
- [Timeline](#timeline)
- [Environments](#environments)
- [Code & Data Organization](#code--data-organization)
- [Day 0 - Production Deployment](#day-0---production-deployment)
  - [Step-by-Step Procedure](#step-by-step-procedure)
  - [Setting Up Dev and Test Environments](#setting-up-dev-and-test-environments)
  - [Deployment Checklist](#deployment-checklist)
- [Day 1 - Open for Business](#day-1---open-for-business)
- [Day N: Ongoing Database Operations](#day-n-ongoing-database-operations)
  - [SQL File Naming Convention](#sql-file-naming-convention)
  - [Schema Migrations](#schema-migrations)
  - [Manual Data Corrections](#manual-data-corrections)
  - [Bulk/Historical Data Loading](#bulkhistorical-data-loading)
- [Day B: Production Database Backup](#day-b-production-database-backup)
- [Appendix](#appendix)
  - [.gitignore](#gitignore)
  - [db-constants.sh](#db-constantssh)
  - [day-0.sh](#day-0sh)
  - [apply-sql.sh](#apply-sqlsh)
  - [install-upgrade-sqlite.sh](#install-upgrade-sqlitesh)
  - [db-maintenance.sh](#db-maintenancesh)
  - [copy-db-prod-to-dev.sh](#copy-db-prod-to-devsh)

# Timeline
- `Day 0` - when you deploy your application to production for the first time
- `Day 1` - the first day when your application is "open for business"
- `Day N` - any day after Day 1 when you need to make a database change (schema migration, data fix, bulk load)
- `Day B` - day on which a backup is done

# Environments
- **development** - for local development and manual testing by developers
- **test** - for automated unit and integration tests
- **production** - the live environment serving real users

# Code & Data Organization
Below is a comprehensive list of the database files, Bash scripts, and Python scripts you will need, along with a suggested directory structure:
```tree
.gitignore
database/
  db/
    development/
      my-app-dev.sqlite
    production/
      my-app-prod.sqlite
      my-app-prod-2025-09-01.sqlite
    test/
      my-app-test.sqlite
  scripts/
    apply-sql.sh
    copy-db-prod-to-dev.sh
    db-constants.sh
    db-maintenance.sh
    day-0.sh
    install-upgrade-sqlite.sh
  sql/
    00-schema-init.sql
    01-data-init.sql
    02-schema-add-column-abc.sql
    03-data-add-historical.sql
    04-schema-add-table5.sql
  data/
    01-reference-data.csv
    03-historical.csv
  logs/
    sql-commands.log
    database-operations.log
```

Here is the purpose of each directory/file:

### Root Files

| Path         | Description                                                                                         |
|--------------|-----------------------------------------------------------------------------------------------------|
| `.gitignore` | Specifies files and directories to be ignored by git, including the production database.            |
| `database/`  | Root directory for all database-related files: actual db files, utility scripts, and data files.    |

### Database Files (`database/db/`)
Contains environment-specific SQLite database files.

| Path                                            | Description                                                                                                              |
|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `.../development/`                            | Stores the developer-specific SQLite database files. These files should *never* be checked into version control. |
| `.../production/`                             | On the production server: stores the live database. On a developer machine: stores temporary copies for debugging. These files should *never* be checked into version control. |
| `.../production/my-app-prod.sqlite`           | The live production database file. Created on Day 0 by `day-0.sh`. |
| `.../production/my-app-prod-2025-09-01.sqlite`| A temporary copy of the prod db file. Always suffixed with the date (and time) of copy. |
| `.../test/`                                   | Stores test data for the project's unit tests. Should be checked into version control.  |
| `.../test/my-app-test.sqlite`                 | Test data for the project's unit tests. Should be checked into version control.         |

### Scripts (`database/scripts/`)
Contains utility scripts for database management and deployment.

| Path                                 | Description                                                                                   |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| `.../apply-sql.sh`           | Applies a SQL file to a target environment with logging and verification. The workhorse script for all Day N operations.|
| `.../copy-db-prod-to-dev.sh` | Copies the production database to the development environment for testing or debugging. Uses SQLite's `.backup` command for a safe, consistent copy.|
| `.../db-constants.sh`        | Common constants (paths, filenames, versions) shared across all scripts. Sourced by every other script.|
| `.../db-maintenance.sh`      | Health check script: verifies pragmas, runs integrity check, reports file sizes and table row counts.|
| `.../day-0.sh`               | Day 0 setup: creates the database, configures pragmas, runs schema init and data seed, verifies the result.|
| `.../install-upgrade-sqlite.sh` | Checks that SQLite is installed and meets the minimum required version.|

### SQL Scripts (`database/sql/`)
Contains SQL scripts for schema and data management. Monotonically increasing numbered prefix for all files. Treat files in this folder as **immutable** once they have been executed in production.

| Path                          | Description                                                                                   |
| ----------------------------- | --------------------------------------------------------------------------------------------- |
| `.../00-schema-init.sql`      | SQL script to create an empty db with all the schemas. For one-time execution (Day 0) only.            |
| `.../01-data-init.sql`        | SQL script to seed the production database with initial data. For one-time execution (Day 0) only.     |

### Data Files (`database/data/`)
Place to store any (small) data files. Sequentially numbered to indicate the order in which they were created.

| Path                        | Description                                                                                    |
| --------------------------- | ---------------------------------------------------------------------------------------------- |
| `.../01-reference-data.csv` | Reference data to seed the production db on Day-0. |
| `.../03-historical.csv`     | Historical data to seed the production db on Day-0.|

### Logs (`database/logs/`)

| Path                                                      | Description                                                                                   |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `../logs/`                                          | Directory for SQL command logs and database operation logs. Files in this folder should *never* be checked into version control. |

# Day 0 - Production Deployment

Day 0 is when you create and initialize your production database for the first time. Everything that follows depends on getting this right.

Scripts used: `install-upgrade-sqlite.sh`, `day-0.sh`, `db-maintenance.sh`

## Step-by-Step Procedure

**1. Verify SQLite version**
```bash
./database/scripts/install-upgrade-sqlite.sh
```
Confirms that `sqlite3` is installed and meets the minimum required version.

**2. Create and initialize the production database**
```bash
./database/scripts/day-0.sh production
```
This single script handles the full Day 0 sequence:
1. Creates the database file at the path defined in `db-constants.sh`
2. Configures production pragmas, if any (e.g., WAL mode, foreign keys, timeouts)
3. Runs `00-schema-init.sql` to create all tables
4. Runs `01-data-init.sql` to seed initial data (reference data, etc.)
5. Verifies the result (integrity check, table row counts)

**3. Verify with a health check**
```bash
./database/scripts/db-maintenance.sh production
```
Review the output to confirm all pragmas are set, all tables exist, and row counts match expectations.

## Setting Up Dev and Test Environments

The same scripts work for all environments. Run Day 0 for development and test:

```bash
./database/scripts/day-0.sh development
./database/scripts/day-0.sh test
```

Each environment gets its own database file (paths defined in `db-constants.sh`), initialized with the same schema and seed data. This ensures consistency across environments.

Alternatively, to populate your development database with a copy of production data (e.g., for debugging a production issue):
```bash
./database/scripts/copy-db-prod-to-dev.sh
```

The test database (`database/db/test/my-app-test.sqlite`) should be checked into version control so that all developers and CI start from a consistent baseline.

**Refreshing the test database** (e.g., after adding new migrations or between test runs):
1. Delete `database/db/test/my-app-test.sqlite`
2. Re-run `./database/scripts/day-0.sh test`
3. Apply any migrations beyond the initial setup:
   ```bash
   ./database/scripts/apply-sql.sh test database/sql/02-schema-add-column-abc.sql
   ./database/scripts/apply-sql.sh test database/sql/03-data-add-historical.sql
   # ... and so on for each subsequent migration
   ```

## Deployment Checklist

- [ ] SQLite version verified (`install-upgrade-sqlite.sh`)
- [ ] Production database created and initialized (`day-0.sh production`)
- [ ] Integrity check passed
- [ ] Row counts validated for seeded tables
- [ ] Dev database created (`day-0.sh development`)
- [ ] Test database created (`day-0.sh test`)
- [ ] File permissions set correctly on production database
- [ ] All scripts tested on dev environment before running on prod
- [ ] Application configured with correct database path

# Day 1 - Open for Business

A brief checklist for the first day your application is live:

- [ ] Application connects to the database and reads/writes successfully
- [ ] No lock errors in application logs
- [ ] Run `db-maintenance.sh production` for a health snapshot

# Day N: Ongoing Database Operations

After Day 1, you will need to make changes to the database: adding columns, creating tables, fixing data, loading historical records. This section covers the procedures for each.

## SQL File Naming Convention

All SQL changes — schema or data — go in `database/sql/` as numbered, immutable files:

**Format:** `NN-{schema|data}-description.sql`

- `NN` — two-digit monotonically increasing number (`00`, `01`, `02`, ...)
- `schema` — for DDL changes (CREATE TABLE, ALTER TABLE, etc.)
- `data` — for DML changes (INSERT, UPDATE, DELETE, etc.)
- `description` — brief description of the change

**Examples:**
```
00-schema-init.sql
01-data-init.sql
02-schema-add-column-abc.sql
03-data-add-historical.sql
04-schema-add-table5.sql
05-data-fix-region-codes.sql
```

**The cardinal rule: files are immutable once executed in production.** Never edit a SQL file that has already been run in prod. If you need to undo or modify a previous change, create a new numbered file.

## Schema Migrations

**Procedure:**

1. **Create a new numbered SQL file:**
   ```
   database/sql/05-schema-add-users-table.sql
   ```

2. **Test on development:**
   ```bash
   ./database/scripts/apply-sql.sh development database/sql/05-schema-add-users-table.sql
   ```

3. **Run your application's test suite** against the updated development database to confirm nothing breaks.

4. **Apply to production:**
   ```bash
   ./database/scripts/apply-sql.sh production database/sql/05-schema-add-users-table.sql
   ```

5. **Verify:**
   ```bash
   ./database/scripts/db-maintenance.sh production
   ```

## Manual Data Corrections

When you need to fix data in production (e.g., correcting a miscoded region, updating a bad record):

**Procedure:**

1. **Always create a numbered SQL file** — never run ad-hoc SQL directly against production. This ensures auditability, reproducibility, and the ability to replay the fix on other environments.
   ```
   database/sql/06-data-fix-region-codes.sql
   ```

2. **Test on development first:**
   ```bash
   ./database/scripts/apply-sql.sh development database/sql/06-data-fix-region-codes.sql
   ```

3. **Verify the fix** on development (check affected rows, run queries to confirm correctness).

4. **Apply to production:**
   ```bash
   ./database/scripts/apply-sql.sh production database/sql/06-data-fix-region-codes.sql
   ```

5. **Verify affected rows in production:**
   ```bash
   ./database/scripts/db-maintenance.sh production
   ```

## Bulk/Historical Data Loading

When you need to load a CSV or bulk dataset (e.g., historical records, reference data updates):

**Procedure:**

1. **Place the data file** in `database/data/` with the correct sequence number:
   ```
   database/data/07-historical-sales.csv
   ```

2. **Create a corresponding numbered SQL file** that loads it:
   ```
   database/sql/07-data-load-historical-sales.sql
   ```
   This SQL file should handle the import (e.g., creating a staging table, inserting from CSV, validating counts).

3. **Test on development:**
   ```bash
   ./database/scripts/apply-sql.sh development database/sql/07-data-load-historical-sales.sql
   ```

4. **Verify row counts and data integrity on development.**

5. **Apply to production:**
   ```bash
   ./database/scripts/apply-sql.sh production database/sql/07-data-load-historical-sales.sql
   ```

6. **Verify:**
   ```bash
   ./database/scripts/db-maintenance.sh production
   ```

# Day B: Production Database Backup

TODO

# Appendix

## .gitignore
```
# Ignore any actual database files in dev and prod folders, (but do check in the test db).
# Though we do want to check in these as empty folders.
database/db/development/*
database/db/production/*

# Ignore log files but keep the logs directory
database/logs/*
```

## db-constants.sh
```bash
#!/bin/bash

# Database Constants - Shared across all database scripts
# This file should be sourced by other scripts: source db-constants.sh

# Database Configuration
APP_NAME="my-app"
DB_NAME_PROD="${APP_NAME}-prod"
DB_NAME_DEV="${APP_NAME}-dev"
DB_NAME_TEST="${APP_NAME}-test"

# Directory Paths (relative to project root)
DB_ROOT_DIR="database/db"
DB_PROD_DIR="$DB_ROOT_DIR/production"
DB_DEV_DIR="$DB_ROOT_DIR/development"
DB_TEST_DIR="$DB_ROOT_DIR/test"
SQL_DIR="database/sql"
DATA_DIR="database/data"
LOGS_DIR="database/logs"

# File Extensions
DB_EXTENSION=".sqlite"

# Database Files
DB_PROD_FILE="$DB_PROD_DIR/${DB_NAME_PROD}${DB_EXTENSION}"
DB_DEV_FILE="$DB_DEV_DIR/${DB_NAME_DEV}${DB_EXTENSION}"
DB_TEST_FILE="$DB_TEST_DIR/${DB_NAME_TEST}${DB_EXTENSION}"

# SQL Scripts (Day 0)
SCHEMA_INIT_SCRIPT="$SQL_DIR/00-schema-init.sql"
DATA_INIT_SCRIPT="$SQL_DIR/01-data-init.sql"

# SQLite Version
REQUIRED_SQLITE_VERSION="3.35.0"

# Backup Configuration
BACKUP_RETENTION_DAYS=30
```

## day-0.sh
```bash
#!/bin/bash

# Day 0 - Database Setup Script
# Creates and initializes a database for the target environment.
# Usage: day-0.sh [environment]
#   environment: production (default) | development | test

set -e

source "$(dirname "$0")/db-constants.sh"

ENVIRONMENT="${1:-production}"

# Resolve database file and directory
case "$ENVIRONMENT" in
  production)   DB_FILE="$DB_PROD_FILE"; DB_DIR="$DB_PROD_DIR" ;;
  development)  DB_FILE="$DB_DEV_FILE";  DB_DIR="$DB_DEV_DIR" ;;
  test)         DB_FILE="$DB_TEST_FILE"; DB_DIR="$DB_TEST_DIR" ;;
  *) echo "Error: Unknown environment '$ENVIRONMENT'"; exit 1 ;;
esac

TIMESTAMP=$(date +"%Y-%m-%d-%H%M%S")

echo "Starting Day 0 database setup for: $ENVIRONMENT"
echo "Database file: $DB_FILE"

# Create directories
mkdir -p "$DB_DIR"
mkdir -p "$LOGS_DIR"

# Abort if database already exists
if [ -f "$DB_FILE" ]; then
  echo "Error: Database already exists at $DB_FILE"
  echo "Day 0 is for initial setup only. To recreate, delete the file first."
  exit 1
fi

# Set pragmas, if any:
echo "Creating database and configuring pragmas..."
sqlite3 "$DB_FILE" <<SQL
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
PRAGMA busy_timeout=5000;
PRAGMA synchronous=NORMAL;
SQL

# Execute schema initialization
echo "Initializing schema..."
sqlite3 "$DB_FILE" < "$SCHEMA_INIT_SCRIPT"

# Execute initial data seeding
echo "Seeding initial data..."
sqlite3 "$DB_FILE" < "$DATA_INIT_SCRIPT"

# Verify: integrity check
echo "Running integrity check..."
INTEGRITY=$(sqlite3 "$DB_FILE" "PRAGMA integrity_check;")
if [ "$INTEGRITY" != "ok" ]; then
  echo "FATAL: Integrity check failed!"
  exit 1
fi
echo "Integrity check: $INTEGRITY"

# Verify: list tables and row counts
echo "Table row counts:"
sqlite3 "$DB_FILE" "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;" | while read table; do
  COUNT=$(sqlite3 "$DB_FILE" "SELECT COUNT(*) FROM \"$table\";")
  echo "  $table: $COUNT rows"
done

# Verify: pragma values
echo "Pragma values:"
echo "  journal_mode: $(sqlite3 "$DB_FILE" 'PRAGMA journal_mode;')"
echo "  foreign_keys: $(sqlite3 "$DB_FILE" 'PRAGMA foreign_keys;')"
echo "  busy_timeout: $(sqlite3 "$DB_FILE" 'PRAGMA busy_timeout;')"
echo "  synchronous:  $(sqlite3 "$DB_FILE" 'PRAGMA synchronous;')"

# Log the operation
echo "[$TIMESTAMP] DAY-0 $ENVIRONMENT $DB_FILE" >> "$LOGS_DIR/database-operations.log"

echo ""
echo "Day 0 setup complete for $ENVIRONMENT."
echo "Database: $DB_FILE"
```

## apply-sql.sh
```bash
#!/bin/bash

# Apply a SQL script to a target database environment.
# Logs the operation and verifies integrity after applying.
# Usage: apply-sql.sh <environment> <sql-file>
#   environment: production | development | test
# Example: apply-sql.sh production database/sql/02-schema-add-column-abc.sql

set -e

source "$(dirname "$0")/db-constants.sh"

ENVIRONMENT="$1"
SQL_FILE="$2"

if [ -z "$ENVIRONMENT" ] || [ -z "$SQL_FILE" ]; then
  echo "Usage: apply-sql.sh <environment> <sql-file>"
  echo "  environment: development | test | production"
  exit 1
fi

# Resolve database file
case "$ENVIRONMENT" in
  production)   DB_FILE="$DB_PROD_FILE" ;;
  development)  DB_FILE="$DB_DEV_FILE" ;;
  test)         DB_FILE="$DB_TEST_FILE" ;;
  *) echo "Error: Unknown environment '$ENVIRONMENT'"; exit 1 ;;
esac

if [ ! -f "$DB_FILE" ]; then
  echo "Error: Database file not found: $DB_FILE"
  exit 1
fi

if [ ! -f "$SQL_FILE" ]; then
  echo "Error: SQL file not found: $SQL_FILE"
  exit 1
fi

TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
mkdir -p "$LOGS_DIR"

echo "[$TIMESTAMP] Applying $SQL_FILE to $ENVIRONMENT ($DB_FILE)"

# Log the operation
echo "[$TIMESTAMP] APPLY $SQL_FILE -> $ENVIRONMENT ($DB_FILE)" >> "$LOGS_DIR/database-operations.log"

# Apply the SQL file
sqlite3 "$DB_FILE" < "$SQL_FILE" 2>&1 | tee -a "$LOGS_DIR/sql-commands.log"

# Verify integrity after applying
echo "Verifying database integrity..."
INTEGRITY=$(sqlite3 "$DB_FILE" "PRAGMA integrity_check;")
if [ "$INTEGRITY" != "ok" ]; then
  echo "WARNING: Integrity check failed after applying $SQL_FILE"
  echo "[$TIMESTAMP] INTEGRITY_FAILED after $SQL_FILE on $ENVIRONMENT" >> "$LOGS_DIR/database-operations.log"
  exit 1
fi

echo "Integrity check: $INTEGRITY"
echo "[$TIMESTAMP] SUCCESS $SQL_FILE -> $ENVIRONMENT" >> "$LOGS_DIR/database-operations.log"
echo "Successfully applied $SQL_FILE to $ENVIRONMENT."
```

## install-upgrade-sqlite.sh
```bash
#!/bin/bash

# Check SQLite installation and version.
# Usage: install-upgrade-sqlite.sh

set -e

source "$(dirname "$0")/db-constants.sh"

echo "Checking SQLite installation..."

if ! command -v sqlite3 &> /dev/null; then
  echo "Error: sqlite3 is not installed."
  echo "Install it using your system's package manager:"
  echo "  macOS:  brew install sqlite"
  echo "  Ubuntu: sudo apt install sqlite3"
  echo "  RHEL:   sudo yum install sqlite"
  exit 1
fi

INSTALLED_VERSION=$(sqlite3 --version | awk '{print $1}')
echo "Installed version:        $INSTALLED_VERSION"
echo "Required minimum version: $REQUIRED_SQLITE_VERSION"

# Compare versions (works on macOS and Linux)
version_gte() {
  [ "$(printf '%s\n' "$1" "$2" | sort -V | head -n1)" = "$2" ]
}

if version_gte "$INSTALLED_VERSION" "$REQUIRED_SQLITE_VERSION"; then
  echo "SQLite version is sufficient."
else
  echo "WARNING: SQLite version $INSTALLED_VERSION is below the required $REQUIRED_SQLITE_VERSION."
  echo "Please upgrade using your system's package manager."
  exit 1
fi
```

## db-maintenance.sh
```bash
#!/bin/bash

# Database health check and maintenance report.
# Usage: db-maintenance.sh [environment]
#   environment: production (default) | development | test

set -e

source "$(dirname "$0")/db-constants.sh"

ENVIRONMENT="${1:-production}"

# Resolve database file
case "$ENVIRONMENT" in
  production)   DB_FILE="$DB_PROD_FILE" ;;
  development)  DB_FILE="$DB_DEV_FILE" ;;
  test)         DB_FILE="$DB_TEST_FILE" ;;
  *) echo "Error: Unknown environment '$ENVIRONMENT'"; exit 1 ;;
esac

if [ ! -f "$DB_FILE" ]; then
  echo "Error: Database file not found: $DB_FILE"
  exit 1
fi

echo "=== Database Health Check: $ENVIRONMENT ==="
echo "Database file: $DB_FILE"
echo ""

# File sizes
echo "--- File Sizes ---"
ls -lh "$DB_FILE"
[ -f "${DB_FILE}-wal" ] && ls -lh "${DB_FILE}-wal" || echo "No WAL file found"
[ -f "${DB_FILE}-shm" ] && ls -lh "${DB_FILE}-shm" || echo "No SHM file found"
echo ""

# Pragma values
echo "--- Pragma Values ---"
echo "journal_mode: $(sqlite3 "$DB_FILE" 'PRAGMA journal_mode;')"
echo "foreign_keys: $(sqlite3 "$DB_FILE" 'PRAGMA foreign_keys;')"
echo "busy_timeout: $(sqlite3 "$DB_FILE" 'PRAGMA busy_timeout;')"
echo "synchronous:  $(sqlite3 "$DB_FILE" 'PRAGMA synchronous;')"
echo ""

# Integrity check
echo "--- Integrity Check ---"
sqlite3 "$DB_FILE" "PRAGMA integrity_check;"
echo ""

# Table row counts
echo "--- Table Row Counts ---"
sqlite3 "$DB_FILE" "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;" | while read table; do
  COUNT=$(sqlite3 "$DB_FILE" "SELECT COUNT(*) FROM \"$table\";")
  echo "  $table: $COUNT rows"
done
echo ""

echo "=== Health check complete ==="
```

## copy-db-prod-to-dev.sh
```bash
#!/bin/bash

# Copy the production database to the development environment.
# Uses SQLite's .backup command for a safe, consistent copy
# (works even if the database is in use).
# Usage: copy-db-prod-to-dev.sh

set -e

source "$(dirname "$0")/db-constants.sh"

if [ ! -f "$DB_PROD_FILE" ]; then
  echo "Error: Production database not found: $DB_PROD_FILE"
  exit 1
fi

mkdir -p "$DB_DEV_DIR"

TIMESTAMP=$(date +"%Y-%m-%d-%H%M%S")

echo "Copying production database to development..."
echo "  Source: $DB_PROD_FILE"
echo "  Target: $DB_DEV_FILE"

# Use .backup for a safe copy
sqlite3 "$DB_PROD_FILE" ".backup '$DB_DEV_FILE'"

# Log the operation
mkdir -p "$LOGS_DIR"
echo "[$TIMESTAMP] COPY prod -> dev ($DB_PROD_FILE -> $DB_DEV_FILE)" >> "$LOGS_DIR/database-operations.log"

echo "Successfully copied production database to development."
```

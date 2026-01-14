---
layout: post
title: "Planning SQLite Deployment to Production"
date: 2025-09-01 02:28:47 +0530
categories: [database, SQLite]
---

SQLite is a simple database, but even then, its production deployment can be complex and tricky.
Here's how to plan your SQLite deployment strategy, including testing, development and maintenance.

# Assumptions:
1. Overall, this guide is for simple deployments. 
2. This guide assumes your project uses a single SQLite database. If your application requires multiple SQLite databases, you will need to adjust the architecture and deployment procedures accordingly.

# Table of Contents
- [Timeline](#timeline)
- [Environments](#environments)
- [Code & Data organization](#code--data-organization)
- [Day 0 - Production Deployment](#day-0---production-deployment)
- [Testing Data Management](#testing-data-management)
- [Production Data Considerations](#production-data-considerations)
- [Deployment Checklist](#deployment-checklist)
- [Appendix](#appendix)
  - [.gitignore](#gitignore)
  - [db-constants.sh](#db-constantssh)
  - [day-0.sh](#day-0sh)

# Timeline
- `Day 0` - when you deploy your application to production for the first time
- `Day 1` - the first day when your application is "open for business"
- `Day B` - day on which a backup is done

# Environments
- **development** - for local development and manual testing by developers
- **test** - for automated unit and integration tests
- **production** - the live environment serving real users

# Code & Data organization
Below is a comprehensive list of the database files, Bash scripts, and Python scripts you will need, along with a suggested directory structure:
```tree
.gitignore
database/
  db/
    development/
      my-app-dev.sqlite
    production/
      my-app-prod-2025-09-01.sqlite
    test/
      my-app-test.sqlite
  scripts/
    install-upgrade-sqlite.sh
    copy-db-prod-to-dev.sh
    db-constants.sh
    db-maintenance.sh
    day-0.sh
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
| `.../development/`                            | If required, stores the developer-specific SQLite database files. These files should *never* be checked into version control. |
| `.../production/`                             | If required, stores any temporary copy of production SQLite database file. These files should *never* be checked into version control. |
| `.../production/my-app-prod-2025-09-01.sqlite`| A temporary copy of the prod db file. Always suffixed with the date (and time) of copy. |
| `.../test/`                                   | Stores test data for the project's unit tests. Should be checked into version control.  |
| `.../test/my-app-test.sqlite`                 | Test data for the project's unit tests. Should be checked into version control.         |

### Scripts (`database/scripts/`)
Contains utility scripts for database management and deployment.

| Path                                 | Description                                                                                   |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| `.../copy-db-prod-to-dev.sh`  | Script to copy the production database to the development environment, for testing or debugging.|
| `.../db-constants.sh`         | Script containing common constants related to our db. To be used by all other scripts.|

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
Script used: `day-0.sh`

1. Create db - Before you open for business, you'll of course want to create an empty db file with schema.
2. Data Seeding - We often have data which we want to be present in our db even before we are "open for business".
For example, reference data tables, historical data, etc.

**Development Environment:**
- Use sample data that mimics production structure
- Include edge cases and boundary conditions
- Keep datasets small but representative

**Staging Environment:**
- Use anonymized production data when possible
- Ensure data volume matches production expectations

# Testing Data Management

**Test Data Isolation:**
- Use separate SQLite files for each test suite
- Implement database factories for consistent test data

**Data Validation:**
- Test with various data types and sizes
- Verify constraints and triggers work correctly
- Ensure proper error handling for invalid data

# Further Production Data Considerations
Below we mention some further considerations when deploying a SQLite db in produciton. These are howver beyond the scope of this 
current blog post.
**Backup Strategy:**
- Implement automated daily backups
- Use WAL mode for better concurrency
- Consider point-in-time recovery options

**Data Migration:**
- Plan schema changes carefully
- Test migrations on staging first
- Have rollback procedures ready

# Deployment Checklist

- [ ] Database file permissions set correctly
- [ ] Backup scripts configured and tested
- [ ] Monitoring and alerting in place
- [ ] Performance baseline established
- [ ] Disaster recovery plan documented

SQLite can handle production workloads effectively when properly planned and maintained. The key is understanding your data patterns and implementing the right operational procedures.

# Appendix

## .gitignore
```
# Ignore any actual database files in dev and prod folders.
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

# Directory Paths
DB_ROOT_DIR="database/db"
DB_PROD_DIR="$DB_ROOT_DIR/production"
DB_DEV_DIR="$DB_ROOT_DIR/development"
DB_TEST_DIR="$DB_ROOT_DIR/test"
SQL_DIR="database/sql"
LOGS_DIR="database/logs"

# File Extensions
DB_EXTENSION=".sqlite"

# Database Files
DB_PROD_FILE="$DB_PROD_DIR/${DB_NAME_PROD}${DB_EXTENSION}"
DB_DEV_FILE="$DB_DEV_DIR/${DB_NAME_DEV}${DB_EXTENSION}"
DB_TEST_FILE="$DB_TEST_DIR/${DB_NAME_TEST}${DB_EXTENSION}"

# SQL Scripts
SCHEMA_INIT_SCRIPT="$SQL_DIR/00-schema-init.sql"
DATA_INIT_SCRIPT="$SQL_DIR/01-data-init.sql"

# Backup Configuration
BACKUP_RETENTION_DAYS=30
```

## day-0.sh
```bash
#!/bin/bash

# Day 0 - Production Database Setup Script
# This script creates and initializes the production database

set -e  # Exit on any error

# Source common constants
source "$(dirname "$0")/db-constants.sh"

# Get current timestamp
TIMESTAMP=$(date +"%Y-%m-%d-%H%M%S")

echo "🚀 Starting Day 0 database setup..."

# Create production database directory if it doesn't exist
mkdir -p "$DB_PROD_DIR"

# Create logs directory if it doesn't exist
mkdir -p "$LOGS_DIR"

# Create empty database file
echo "📁 Creating database: $DB_PROD_FILE"
sqlite3 "$DB_PROD_FILE" ".databases"

# Enable WAL mode for better concurrency
echo "⚙️  Enabling WAL mode..."
sqlite3 "$DB_PROD_FILE" "PRAGMA journal_mode=WAL;"

# Execute schema initialization
echo "🔧 Initializing schema..."
sqlite3 "$DB_PROD_FILE" < "$SCHEMA_INIT_SCRIPT"

# Execute initial data seeding
echo "🌱 Seeding initial data..."
sqlite3 "$DB_PROD_FILE" < "$DATA_INIT_SCRIPT"

# Verify database setup
echo "✅ Verifying database setup..."
sqlite3 "$DB_PROD_FILE" "SELECT name FROM sqlite_master WHERE type='table';"

# Create backup copy with timestamp
BACKUP_FILE="$DB_PROD_DIR/${DB_NAME_PROD}-${TIMESTAMP}${DB_EXTENSION}"
echo "💾 Creating backup: $BACKUP_FILE"
cp "$DB_PROD_FILE" "$BACKUP_FILE"

echo "🎉 Day 0 database setup completed successfully!"
echo "📊 Database file: $DB_PROD_FILE"
echo "💾 Backup file: $BACKUP_FILE"
```
# PostgreSQL Data Model Analyzer - Additional Resources

## Quick Connect Scripts

### Connection Aliases

Add these to your `.bashrc` or `.zshrc`:

```bash
# Quick connection to common databases
alias db-prod='pgcli -h prod-db.example.com -U admin -d production'
alias db-staging='pgcli -h staging-db.example.com -U admin -d staging'
alias db-local='pgcli -h localhost -U postgres -d myapp'

# With password from environment variable
alias db-secure='PGPASSWORD=$DB_PASSWORD pgcli -h $DB_HOST -U $DB_USER -d $DB_NAME'
```

### Connection String Format

```bash
# PostgreSQL connection string format
postgresql://username:password@hostname:port/database?sslmode=require

# Example
pgcli "postgresql://admin:secret@localhost:5432/myapp?sslmode=require"
```

## Automated Schema Documentation

### Generate Schema Report

Save as `generate_schema_report.sh`:

```bash
#!/bin/bash
DB_NAME=${1:-myapp}
OUTPUT_DIR=${2:-./schema_docs}

mkdir -p $OUTPUT_DIR

# Database overview
pgcli -d $DB_NAME -c "SELECT version();" > $OUTPUT_DIR/00_database_info.txt

# Tables list
pgcli -d $DB_NAME -c "
SELECT schemaname, relname, n_live_tup, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
" > $OUTPUT_DIR/01_tables.txt

# Foreign keys
pgcli -d $DB_NAME -c "
SELECT 
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column,
    tc.constraint_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY' AND tc.table_schema = 'public';
" > $OUTPUT_DIR/02_foreign_keys.txt

# Views
pgcli -d $DB_NAME -c "
SELECT schemaname, viewname, viewowner
FROM pg_views
WHERE schemaname = 'public';
" > $OUTPUT_DIR/03_views.txt

# Functions
pgcli -d $DB_NAME -c "
SELECT p.proname, pg_get_function_arguments(p.oid), pg_get_function_result(p.oid)
FROM pg_proc p
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname = 'public';
" > $OUTPUT_DIR/04_functions.txt

echo "Schema documentation generated in $OUTPUT_DIR/"
```

## Common Table Analysis Queries

### Find Tables Without Primary Keys

```sql
SELECT t.table_schema, t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints tc 
    ON t.table_name = tc.table_name 
    AND t.table_schema = tc.table_schema
    AND tc.constraint_type = 'PRIMARY KEY'
WHERE t.table_schema = 'public'
    AND t.table_type = 'BASE TABLE'
    AND tc.constraint_name IS NULL;
```

### Find Tables Without Indexes (Except PK)

```sql
SELECT t.schemaname, t.relname as table_name
FROM pg_stat_user_tables t
LEFT JOIN pg_indexes i ON t.relname = i.tablename AND t.schemaname = i.schemaname
WHERE t.schemaname = 'public'
GROUP BY t.schemaname, t.relname
HAVING COUNT(i.indexname) = 0;
```

### Find Redundant Indexes

```sql
SELECT 
    tablename, 
    indexname, 
    indexdef
FROM pg_indexes
WHERE indexdef LIKE '%(id)%' 
    AND indexdef NOT LIKE '%UNIQUE%'
    AND schemaname = 'public'
ORDER BY tablename, indexname;
```

### Large Tables Without Indexes

```sql
SELECT 
    schemaname,
    relname,
    n_live_tup,
    pg_size_pretty(pg_total_relation_size(relid)) as size,
    (SELECT COUNT(*) FROM pg_indexes WHERE tablename = relname AND schemaname = pg_stat_user_tables.schemaname) as index_count
FROM pg_stat_user_tables
WHERE n_live_tup > 100000
    AND (SELECT COUNT(*) FROM pg_indexes WHERE tablename = relname AND schemaname = pg_stat_user_tables.schemaname) < 3
ORDER BY n_live_tup DESC;
```

## Data Profiling Queries

### Column Statistics Overview

```sql
-- Get column statistics for a table
SELECT 
    schemaname,
    tablename,
    attname as column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation
FROM pg_stats
WHERE tablename = 'users'
    AND schemaname = 'public'
ORDER BY attname;
```

### Data Distribution by Type

```sql
-- Detect skewed distributions
SELECT 
    column_name,
    data_type,
    COUNT(*) as total_rows,
    COUNT(DISTINCT column_name) as unique_values,
    MIN(column_name) as min_value,
    MAX(column_name) as max_value
FROM information_schema.columns c
JOIN your_table t ON true
WHERE c.table_name = 'users'
    AND c.table_schema = 'public'
GROUP BY column_name, data_type;
```

### Identify Enum Columns

```sql
-- Find columns that might be enums (few distinct values)
WITH column_stats AS (
    SELECT 
        tablename,
        attname,
        n_distinct,
        most_common_vals::text as common_values
    FROM pg_stats
    WHERE schemaname = 'public'
        AND n_distinct > 0
        AND n_distinct < 50
)
SELECT * FROM column_stats
ORDER BY n_distinct;
```

## Migration & Schema Comparison

### Compare Schemas Between Databases

```bash
# Export schema from two databases and compare
pg_dump --schema-only --no-owner --no-privileges database1 > schema1.sql
pg_dump --schema-only --no-owner --no-privileges database2 > schema2.sql
diff schema1.sql schema2.sql > schema_diff.txt
```

### Check Schema Drift

```sql
-- Query to run on both databases and compare results
SELECT 
    table_name,
    column_name,
    data_type,
    character_maximum_length,
    numeric_precision,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;
```

## pgcli Configuration

### Config File Location

```
~/.config/pgcli/config      # Linux/Mac
%APPDATA%\pgcli\config       # Windows
```

### Recommended Settings

```ini
[main]
multi_line = True
multi_line_mode = psql
destructive_warning = True
# Confirm before executing destructive commands

[bindings]
# Custom key bindings
F2 = select_all
F3 = deselect_all

[colors]
# Syntax highlighting
token.Keyword = #ansiyellow
```

### Useful pgcli Commands

```
\?           # Show help
\dt          # List tables
\d+ table    # Describe table with details
\dn          # List schemas
\df          # List functions
\dv          # List views
\di          # List indexes
\timing      # Toggle query timing
\copy        # Export to CSV
\pset format csv    # Set output format to CSV
\pset format table  # Set output format to table (default)
\x auto      # Toggle expanded display automatically
```

## Troubleshooting

### Common Issues

**Connection refused:**
```bash
# Check PostgreSQL is running
pg_isready -h localhost

# Check pg_hba.conf for access rules
# Location: /etc/postgresql/[version]/main/pg_hba.conf
```

**SSL issues:**
```bash
# Disable SSL (not for production!)
pgcli "postgresql://user:pass@host/db?sslmode=disable"

# Or accept any SSL certificate
pgcli "postgresql://user:pass@host/db?sslmode=require"
```

**Timeout on large queries:**
```sql
-- Set statement timeout
SET statement_timeout = '60s';
```

### Performance Monitoring

```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check max connections setting
SHOW max_connections;

-- Check database size growth
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) as size,
    pg_database_size(datname) / 1024 / 1024 as size_mb
FROM pg_database
WHERE datistemplate = false;
```

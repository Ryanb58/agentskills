---
name: pgcli-data-modeler
description: PostgreSQL database model analysis using pgcli - read-only exploration of schemas, tables, relationships, views, and stored procedures
version: 1.0.0
author: agentskills
tags: [postgresql, database, data-model, pgcli, schema, read-only]
---

# PostgreSQL Data Model Analyzer

Analyze and document PostgreSQL database schemas using `pgcli` (or `psql`). This skill provides read-only commands for exploring database structures, understanding relationships, and documenting data models.

## Agent Usage Workflow

When using this skill to analyze a database:

1. **Get connection details from the user** - Ask for:
   - Hostname (e.g., `localhost`, `db.example.com`)
   - Port (default: `5432`)
   - Username (e.g., `postgres`, `analyst`)
   - Database name (e.g., `myapp`, `production`)
   - Password (if not using key-based auth)
   - SSL mode preference

2. **Establish connection** using one of these methods:
   - Environment variables (recommended for automation)
   - Connection string
   - Direct command with parameters

3. **Execute read-only queries** from this skill:
   - Start with database overview
   - Map tables and relationships
   - Analyze indexes and constraints
   - Document views and functions

4. **Never perform write operations** - All queries in this skill are SELECT statements only

5. **Report findings** - Summarize the data model structure, relationships, and any issues found

## Prerequisites

```bash
# Install pgcli (recommended) or use psql
brew install pgcli                    # macOS
apt-get install pgcli                 # Ubuntu/Debian
pip install pgcli                     # Python pip

# Or use standard psql
brew install postgresql               # macOS
apt-get install postgresql-client     # Ubuntu/Debian

# Connect to database
pgcli -h hostname -U username -d database_name
# Or with psql
psql -h hostname -U username -d database_name
```

## Connection Setup & Credential Management

### Option 1: Environment Variables (Recommended)

Set environment variables before running commands:

```bash
# Standard PostgreSQL env vars
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGPASSWORD=your_password
export PGDATABASE=myapp

# Then connect without parameters
pgcli
# or
psql
```

### Option 2: Connection String / URL

Use a connection string with all details:

```bash
# Full connection string
pgcli "postgresql://username:password@hostname:5432/database?sslmode=require"

# Using a .pgpass file for passwords (more secure)
# Create ~/.pgpass with format: hostname:port:database:username:password
chmod 600 ~/.pgpass
pgcli -h hostname -U username -d database
```

### Option 3: pgcli Config / Alias

Set up persistent connections in shell config:

```bash
# Add to ~/.bashrc or ~/.zshrc
alias db-local='pgcli -h localhost -U postgres -d myapp'
alias db-prod='pgcli -h prod.example.com -U admin -d production'

# With password from environment
alias db-secure='PGPASSWORD=$PROD_DB_PASSWORD pgcli -h $PROD_DB_HOST -U $PROD_DB_USER -d $PROD_DB_NAME'
```

### Option 4: Docker Connection

If connecting to PostgreSQL in Docker:

```bash
# Connect to PostgreSQL running in Docker
pgcli -h localhost -U postgres -d myapp

# Or via Docker exec (if pgcli not installed locally)
docker exec -it postgres_container psql -U postgres -d myapp
```

### Secure Credential Handling

**Never commit credentials to git!**

```bash
# Use a .env file (add to .gitignore!)
# .env
db_host=localhost
db_port=5432
db_user=postgres
db_password=secret
db_name=myapp

# Load in terminal
set -a && source .env && set -a
pgcli -h $db_host -U $db_user -d $db_name
```

### For AI Agents / Automated Use

When using this skill programmatically:

1. **Ask the user** for connection details when starting
2. **Store temporarily** in environment for the session
3. **Never log or echo** passwords in output
4. **Use read-only user** if available:
   ```sql
   -- Create read-only user for analysis
   CREATE USER analyst WITH PASSWORD 'password';
   GRANT USAGE ON SCHEMA public TO analyst;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;
   ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO analyst;
   ```

### Example Agent Session

```
User: "Analyze my local PostgreSQL database"

Agent: "I'll help you analyze your PostgreSQL database. First, I need the connection details:

1. What is the hostname? (e.g., localhost, 127.0.0.1)
2. What port? (default is 5432)
3. What is the database name?
4. What is the username?
5. What is the password?"

User: "localhost, 5432, myapp_db, postgres, secret123"

Agent: "Great! Let me connect and analyze your database structure.

[Agent sets environment and connects]
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=myapp_db
export PGUSER=postgres
export PGPASSWORD=secret123

[Then runs queries from this skill...]

📊 Database Overview:
- PostgreSQL 15.2
- Database: myapp_db (256 MB total size)
- 3 schemas, 47 tables

📋 Tables by Size:
1. users (1.2M rows, 89 MB)
2. orders (856K rows, 67 MB)
3. products (12K rows, 8 MB)

🔗 Key Relationships Found:
- orders.user_id → users.id
- order_items.order_id → orders.id
- order_items.product_id → products.id

⚠️ Issues Detected:
- 3 tables missing primary keys
- 7 unused indexes consuming 23 MB
- 2 tables with no indexes on foreign key columns

Would you like me to explore any specific area in more detail?"
```

## Quick Database Overview

### Database Metadata

```sql
-- Database version and settings
SELECT version();

-- Current database name
SELECT current_database();

-- Database size
SELECT pg_size_pretty(pg_database_size(current_database()));

-- All databases in the cluster
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

### Connection Info

```sql
-- Current user and session info
SELECT current_user, session_user, inet_client_addr(), inet_client_port();
```

## Schema Exploration

### List Schemas

```sql
-- All schemas in current database
SELECT schema_name, schema_owner,
       pg_size_pretty(pg_total_relation_size(schema_name || '.*')) as total_size
FROM information_schema.schemata
WHERE schema_name NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
ORDER BY schema_name;

-- Schemas with table counts
SELECT n.nspname as schema_name,
       count(c.relname) as table_count,
       pg_size_pretty(pg_total_relation_size(n.nspname || '.*')) as total_size
FROM pg_namespace n
JOIN pg_class c ON c.relnamespace = n.oid AND c.relkind = 'r'
WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
GROUP BY n.nspname
ORDER BY count(c.relname) DESC;
```

### Schema Details

```sql
-- Tables in a specific schema
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;

-- Schema privileges
SELECT grantee, privilege_type
FROM information_schema.table_privileges
WHERE table_schema = 'public'
GROUP BY grantee, privilege_type;
```

## Table Analysis

### List All Tables

```sql
-- Tables with row counts and sizes
SELECT schemaname, relname as table_name,
       n_live_tup as row_count,
       pg_size_pretty(pg_total_relation_size(relid)) as total_size,
       pg_size_pretty(pg_relation_size(relid)) as table_size,
       pg_size_pretty(pg_indexes_size(relid)) as indexes_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

### Table Structure

```sql
-- Detailed column information
SELECT column_name, data_type, character_maximum_length,
       numeric_precision, numeric_scale,
       is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = 'users'
ORDER BY ordinal_position;

-- Column info with constraints
SELECT c.column_name, c.data_type, c.is_nullable,
       CASE WHEN pk.column_name IS NOT NULL THEN 'PK' ELSE '' END as primary_key,
       c.column_default
FROM information_schema.columns c
LEFT JOIN (
    SELECT kcu.column_name, kcu.table_name
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu 
        ON tc.constraint_name = kcu.constraint_name
    WHERE tc.constraint_type = 'PRIMARY KEY'
) pk ON c.table_name = pk.table_name AND c.column_name = pk.column_name
WHERE c.table_schema = 'public' AND c.table_name = 'users'
ORDER BY c.ordinal_position;
```

### Table Constraints

```sql
-- All constraints on a table
SELECT tc.constraint_name, tc.constraint_type,
       kcu.column_name,
       ccu.table_name AS foreign_table,
       ccu.column_name AS foreign_column,
       pg_get_constraintdef(pgc.oid, true) as constraint_definition
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
LEFT JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
LEFT JOIN pg_constraint pgc
    ON pgc.conname = tc.constraint_name
WHERE tc.table_schema = 'public'
    AND tc.table_name = 'users'
ORDER BY tc.constraint_type, tc.constraint_name;
```

## Relationships & Foreign Keys

### Foreign Key Analysis

```sql
-- All foreign keys in the database
SELECT tc.table_schema,
       tc.table_name,
       kcu.column_name,
       ccu.table_schema AS foreign_table_schema,
       ccu.table_name AS foreign_table_name,
       ccu.column_name AS foreign_column_name,
       tc.constraint_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
ORDER BY tc.table_schema, tc.table_name;

-- Foreign keys from a specific table
SELECT tc.constraint_name,
       kcu.column_name,
       ccu.table_name AS references_table,
       ccu.column_name AS references_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND tc.table_schema = 'public'
    AND tc.table_name = 'orders';

-- Tables referencing a specific table
SELECT tc.table_schema,
       tc.table_name,
       kcu.column_name,
       tc.constraint_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND tc.table_schema = 'public'
    AND ccu.table_name = 'users';
```

### Relationship Diagram Query

```sql
-- Generate relationship summary for documentation
SELECT 
    format('%s.%s', tc.table_schema, tc.table_name) as table_name,
    string_agg(kcu.column_name, ', ') as columns,
    format('%s.%s', ccu.table_schema, ccu.table_name) as references_table,
    string_agg(ccu.column_name, ', ') as references_columns,
    tc.constraint_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
    AND tc.table_schema = 'public'
GROUP BY tc.table_schema, tc.table_name, ccu.table_schema, ccu.table_name, tc.constraint_name
ORDER BY tc.table_schema, tc.table_name;
```

## Indexes

### List Indexes

```sql
-- All indexes in the database
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
       indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Indexes on a specific table
SELECT indexname, indexdef, pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_indexes
WHERE schemaname = 'public' AND tablename = 'users'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Index Analysis

```sql
-- Unused indexes (check before dropping in production)
SELECT schemaname, relname as table_name, indexrelname as index_name,
       idx_scan as times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Most used indexes
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC
LIMIT 20;
```

## Views

### List Views

```sql
-- All views
SELECT table_schema, table_name, view_definition
FROM information_schema.views
WHERE table_schema NOT IN ('information_schema', 'pg_catalog')
ORDER BY table_schema, table_name;

-- Views with dependencies
SELECT v.schemaname, v.viewname, v.viewowner,
       pg_size_pretty(pg_relation_size(c.oid)) as size
FROM pg_views v
JOIN pg_class c ON c.relname = v.viewname
WHERE v.schemaname = 'public'
ORDER BY pg_relation_size(c.oid) DESC;
```

### View Definitions

```sql
-- View definition
SELECT pg_get_viewdef('public.view_name', true);

-- All view definitions in schema
SELECT viewname, pg_get_viewdef((schemaname || '.' || viewname)::regclass, true) as definition
FROM pg_views
WHERE schemaname = 'public';
```

## Functions & Stored Procedures

### List Functions

```sql
-- All functions in database
SELECT n.nspname as schema_name,
       p.proname as function_name,
       pg_get_function_arguments(p.oid) as arguments,
       pg_get_function_result(p.oid) as return_type,
       pg_get_functiondef(p.oid) as definition
FROM pg_proc p
LEFT JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
ORDER BY n.nspname, p.proname;

-- Functions in specific schema
SELECT p.proname,
       pg_get_function_arguments(p.oid) as arguments,
       pg_get_function_result(p.oid) as return_type,
       pg_get_functiondef(p.oid) as definition
FROM pg_proc p
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname = 'public'
ORDER BY p.proname;
```

### Trigger Functions

```sql
-- Triggers in the database
SELECT trigger_schema, trigger_name, event_manipulation, 
       event_object_table, action_statement
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY event_object_table, trigger_name;

-- Trigger details
SELECT tgname, tgenabled, pg_get_triggerdef(oid, true)
FROM pg_trigger
WHERE tgrelid = 'users'::regclass;
```

## Sequences

### List Sequences

```sql
-- All sequences
SELECT sequence_schema, sequence_name,
       data_type, start_value, increment,
       minimum_value, maximum_value
FROM information_schema.sequences
WHERE sequence_schema = 'public'
ORDER BY sequence_name;

-- Current sequence values
SELECT sequencename, last_value
FROM pg_sequences
WHERE schemaname = 'public';
```

## Data Model Documentation

### Generate ER Diagram Data

```sql
-- Complete table structure for documentation
WITH table_info AS (
    SELECT 
        t.table_name,
        c.column_name,
        c.data_type,
        c.is_nullable,
        CASE WHEN pk.column_name IS NOT NULL THEN 'PK' ELSE '' END as primary_key,
        CASE WHEN fk.column_name IS NOT NULL 
            THEN fk.foreign_table || '.' || fk.foreign_column 
            ELSE '' 
        END as foreign_key
    FROM information_schema.tables t
    JOIN information_schema.columns c 
        ON t.table_name = c.table_name 
        AND t.table_schema = c.table_schema
    LEFT JOIN (
        SELECT kcu.column_name, kcu.table_name
        FROM information_schema.table_constraints tc
        JOIN information_schema.key_column_usage kcu 
            ON tc.constraint_name = kcu.constraint_name
        WHERE tc.constraint_type = 'PRIMARY KEY'
            AND tc.table_schema = 'public'
    ) pk ON c.table_name = pk.table_name AND c.column_name = pk.column_name
    LEFT JOIN (
        SELECT kcu.column_name, kcu.table_name,
               ccu.table_name as foreign_table, ccu.column_name as foreign_column
        FROM information_schema.table_constraints tc
        JOIN information_schema.key_column_usage kcu 
            ON tc.constraint_name = kcu.constraint_name
        JOIN information_schema.constraint_column_usage ccu
            ON tc.constraint_name = ccu.constraint_name
        WHERE tc.constraint_type = 'FOREIGN KEY'
            AND tc.table_schema = 'public'
    ) fk ON c.table_name = fk.table_name AND c.column_name = fk.column_name
    WHERE t.table_schema = 'public' AND t.table_type = 'BASE TABLE'
)
SELECT table_name, 
       column_name, 
       data_type, 
       is_nullable,
       primary_key,
       foreign_key
FROM table_info
ORDER BY table_name, ordinal_position;
```

### Table Statistics Summary

```sql
-- Comprehensive table statistics
SELECT 
    schemaname || '.' || relname as full_table_name,
    n_live_tup as live_rows,
    n_dead_tup as dead_rows,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) as dead_row_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_live_tup DESC;
```

## Sample Data Inspection

### Quick Data Overview

```sql
-- Row count and sample data
SELECT COUNT(*) FROM users;
SELECT * FROM users LIMIT 5;

-- Column value distribution
SELECT 
    status,
    COUNT(*) as count,
    ROUND(COUNT(*)::numeric / SUM(COUNT(*)) OVER() * 100, 2) as percentage
FROM users
GROUP BY status
ORDER BY count DESC;

-- Date range in tables
SELECT 
    MIN(created_at) as earliest,
    MAX(created_at) as latest,
    MAX(created_at) - MIN(created_at) as date_range
FROM users;

-- Check for NULL values
SELECT 
    SUM(CASE WHEN column1 IS NULL THEN 1 ELSE 0 END) as null_column1,
    SUM(CASE WHEN column2 IS NULL THEN 1 ELSE 0 END) as null_column2
FROM your_table;
```

### Data Quality Checks

```sql
-- Duplicate primary keys
SELECT id, COUNT(*) as count
FROM users
GROUP BY id
HAVING COUNT(*) > 1;

-- Orphaned foreign keys (no matching parent)
SELECT o.*
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;

-- Invalid date ranges
SELECT *
FROM subscriptions
WHERE end_date < start_date;

-- Empty required fields
SELECT *
FROM users
WHERE email IS NULL OR email = '';
```

## Database Security

### User & Permission Analysis

```sql
-- Database users
SELECT usename, usesuper, usecreatedb, valuntil
FROM pg_shadow
ORDER BY usename;

-- Table permissions
SELECT grantee, table_name, string_agg(privilege_type, ', ' ORDER BY privilege_type) as permissions
FROM information_schema.table_privileges
WHERE table_schema = 'public'
GROUP BY grantee, table_name
ORDER BY table_name, grantee;

-- Column-level permissions
SELECT grantee, table_name, column_name, privilege_type
FROM information_schema.column_privileges
WHERE table_schema = 'public'
ORDER BY table_name, column_name, grantee;
```

## Performance Insights

### Query Statistics

```sql
-- Most frequent queries
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- Slowest queries
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Long running queries (currently running)
SELECT pid, usename, application_name, state, 
       query_start, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active' AND query_start IS NOT NULL
ORDER BY duration DESC;
```

### Lock Analysis

```sql
-- Current locks
SELECT l.locktype, l.relation::regclass, l.mode, l.granted,
       a.usename, a.query, a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY l.relation;
```

## Advanced Analysis

### Partitioned Tables

```sql
-- Check for partitioned tables
SELECT parent.relname as parent_table,
       child.relname as partition_name,
       pg_get_expr(child.relpartbound, child.oid) as partition_bounds
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relnamespace = 'public'::regnamespace;
```

### Extensions

```sql
-- Installed extensions
SELECT name, default_version, installed_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
ORDER BY name;
```

## Tips

1. **Always use read-only operations** - Never run UPDATE, DELETE, or INSERT when exploring
2. **Use LIMIT** - When sampling data, always add LIMIT to avoid large result sets
3. **Check production load** - Use pg_stat_activity before running intensive queries
4. **Save queries** - Build a library of queries for your specific database structure
5. **Document findings** - Export results to markdown or CSV for documentation
6. **Use \timing on** - Enable timing in pgcli/psql to see query execution times
7. **Check locks first** - Before heavy analysis, verify no blocking locks exist
8. **Export schema** - Use pg_dump --schema-only for complete schema backup

## Resources

- [pgcli Documentation](https://www.pgcli.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL Catalog Reference](https://www.postgresql.org/docs/current/catalogs.html)
- [PostgreSQL System Information Functions](https://www.postgresql.org/docs/current/functions-info.html)
- [pg_stat views](https://www.postgresql.org/docs/current/monitoring-stats.html)

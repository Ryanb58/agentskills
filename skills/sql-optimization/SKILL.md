---
name: sql-optimization
description: Expert-level SQL query and data model optimization based on use-the-index-luke.com principles
tags:
  - sql
  - database
  - performance
  - indexing
  - optimization
  - query-tuning
  - database-design
---

# SQL Query & Data Model Optimization

Expert-level guidance for optimizing SQL queries and database schemas based on the principles from use-the-index-luke.com. This skill focuses on practical, proven techniques for achieving optimal database performance through proper indexing strategies, query optimization, and data model design.

## Core Principles

### The Three Powers of Indexing

1. **B-Tree Traversal (Tree Search)**
   - Provides logarithmic scalability - tree depth grows very slowly compared to data volume
   - Real-world indexes with millions of records typically have tree depths of 4-5 levels
   - The tree is always balanced - databases maintain balance automatically
   - **Myth Debunked**: Indexes do NOT "degenerate" over time. Rebuilding indexes rarely improves performance (0-2% improvement for typical queries)

2. **Data Clustering**
   - Indexes cluster rows with similar values together
   - Reduces I/O operations by storing consecutently accessed data close together
   - Key technique: Index-Only Scan (Covering Index) - include all queried columns in the index to avoid table access entirely

3. **Pipelined ORDER BY**
   - Indexes can eliminate explicit sort operations
   - Returns first results immediately without processing all data
   - Critical for Top-N queries and pagination

### Index Structure Fundamentals

**How Indexes Work:**
- **Leaf Nodes**: Doubly linked list maintaining logical order (independent of physical storage)
- **Branch/Root Nodes**: B-tree structure enabling fast traversal (logarithmic time)
- **Index Entries**: Contain indexed column values + ROWID/RID pointing to table row

**Index Operations:**
- **INDEX UNIQUE SCAN**: Tree traversal only (for unique constraints, returns ≤1 row)
- **INDEX RANGE SCAN**: Tree traversal + leaf node chain (for range queries, returns multiple rows)
- **TABLE ACCESS BY INDEX ROWID**: Fetch actual row data from table

## Indexing Strategies

### WHERE Clause Optimization

#### 1. Equality Operator (=)

**Rule: Column Order in Concatenated Indexes**
```sql
CREATE INDEX idx ON table (col1, col2, col3)
```
- Can be used when searching: col1 alone, col1+col2, or col1+col2+col3
- **CANNOT** be used when searching only col2 or col3 alone
- **Analogy**: Like a telephone directory - sorted by last name, then first name. You can't search by first name alone.

**Rule: Index for Equality First, Then for Ranges**
- Place equality columns before range columns in index definition
- **WRONG**: INDEX(date_of_birth, subsidiary_id) for `WHERE subsidiary_id = ? AND date_of_birth BETWEEN ? AND ?`
- **CORRECT**: INDEX(subsidiary_id, date_of_birth)

#### 2. Range Operators (<, >, BETWEEN, LIKE)

**Golden Rule**: Keep the scanned index range as small as possible

**LIKE Filter Optimization**:
- **GOOD**: `LIKE 'WIN%'` - uses "WIN" as access predicate (can use index)
- **BAD**: `LIKE '%WIN'` - leading wildcard prevents index usage (must scan entire table)
- Only characters **before the first wildcard** serve as access predicates

**BETWEEN Operator**:
- Includes boundary values (equivalent to >= and <=)
- Consider using explicit range conditions for date/time types

#### 3. Functions in WHERE Clause

**The Problem**: Functions on columns prevent index usage

**Solutions**:
1. **Function-Based Indexes** (Oracle, PostgreSQL, SQL Server):
   ```sql
   CREATE INDEX idx ON employees (UPPER(last_name))
   ```
   
2. **Computed/Virtual Columns** (alternative approach)

3. **Avoid functions when possible**:
   ```sql
   -- Instead of: WHERE TRUNC(sale_date) = TRUNC(sysdate - 1)
   -- Use range:
   WHERE sale_date >= TRUNC(sysdate - 1)
     AND sale_date <  TRUNC(sysdate)
   ```

#### 4. Access Predicates vs Filter Predicates

**Critical Distinction**:
- **Access Predicates**: Define start/stop conditions for index scan (limit scanned range)
- **Filter Predicates**: Applied during leaf node traversal but don't narrow scanned range

**How to Identify in Execution Plans**:
- Oracle: "access()" vs "filter()" in Predicate Information
- PostgreSQL: Check column order in "Index Cond"
- SQL Server: "Seek Predicates" vs "Predicate"
- Db2: START/STOP vs SARG

## Anti-Patterns to Avoid

### 1. Date/Time Handling
**Anti-Pattern**: Using TRUNC() or DATE_FORMAT() on date columns
```sql
-- BAD - prevents index usage:
WHERE TRUNC(sale_date) = TRUNC(sysdate - INTERVAL '1' DAY)
```

**Solution**: Use explicit range conditions
```sql
-- GOOD:
WHERE sale_date >= TRUNC(sysdate - INTERVAL '1' DAY)
  AND sale_date <  TRUNC(sysdate)
```

### 2. "Smart Logic" (Dynamic WHERE Clauses)
**Anti-Pattern**: Using OR logic with NULL checks for optional filters
```sql
-- WORST ANTI-PATTERN - forces full table scan:
WHERE (subsidiary_id = :sub_id OR :sub_id IS NULL)
  AND (employee_id = :emp_id OR :emp_id IS NULL)
  AND (UPPER(last_name) = :name OR :name IS NULL)
```

**Why it's bad**: Database creates generic execution plan for worst case (all filters disabled = full table scan)

**Solution**: Use **Dynamic SQL** with proper bind parameters
- Build query string with only the filters you actually need
- Still use bind parameters for security and plan caching
- Most reliable method for optimal execution plans

### 3. Type Mixing
- Don't compare strings to numeric columns (or vice versa)
- Implicit conversions often prevent index usage
- Use proper bind parameters with correct data types

### 4. Column Concatenation
**Anti-Pattern**: Combining columns in WHERE clause
```sql
-- BAD:
WHERE first_name || ' ' || last_name = 'John Doe'
```

**Solution**: Use separate conditions
```sql
-- GOOD:
WHERE first_name = 'John' AND last_name = 'Doe'
```

## Join Optimization

### Three Join Algorithms

**1. Nested Loops Join**
- For each row from outer table, search inner table
- Best for: Small outer result sets
- **Indexing**: Index needed on join columns of inner table
- **N+1 Problem**: ORM tools often generate this inefficiently - use JOINs instead of nested selects

**2. Hash Join**
- Loads smaller table into hash table, probes with larger table
- Best for: Large result sets, no suitable index
- **Indexing**: **DO NOT** index join columns - index independent WHERE predicates only
- Minimize hash table size: select fewer columns, add filters

**3. Sort-Merge Join**
- Sorts both inputs, merges like a zipper
- Best for: Sorted inputs, range join conditions
- Requires indexes that provide sorted output

### Join Optimization Tips
- Execute joins in the database (not in application)
- Take control of ORM join behavior
- Enable SQL logging in development to catch N+1 problems
- Use appropriate fetch strategies (eager vs lazy loading)

## Advanced Indexing Techniques

### 1. Index-Only Scans (Covering Indexes)

**The Most Powerful Optimization**
When an index contains **all columns** needed by the query:
- No table access required
- Performance gain depends on: (1) number of rows accessed, (2) clustering factor

**Creating Covering Indexes**:
```sql
-- Index for: WHERE subsidiary_id = ? (returning eur_value)
CREATE INDEX sales_sub_eur ON sales (subsidiary_id, eur_value)
```

**INCLUDE Clause** (SQL Server, PostgreSQL 11+):
```sql
CREATE INDEX idx ON employees (subsidiary_id, last_name)
INCLUDE (phone_number, first_name)
```

**Warnings**:
- Don't create covering indexes "just in case"
- Increases storage and maintenance overhead
- Document index-only scan dependencies in comments
- Function-based indexes can't support index-only scans for the original column

### 2. Partial/Filtered Indexes

Index only specific rows to save space and improve performance:

```sql
-- Only index unprocessed messages (queue pattern)
CREATE INDEX messages_todo ON messages (receiver)
WHERE processed = 'N'
```

**Benefits**:
- Much smaller index size
- Index stays small even as table grows
- Perfect for queue-like patterns

**Database Support**:
- PostgreSQL: Partial indexes (full support)
- SQL Server: Filtered indexes (limited - no functions or OR)
- Oracle: Emulated via function-based indexes + NULL handling
- Db2 (LUW): Emulated via EXCLUDE NULL KEYS

### 3. Sorting & Grouping Optimization

**Indexed ORDER BY**:
- Index eliminates explicit SORT operation
- Index order must match ORDER BY within the scanned range
- Example: `WHERE sale_date = ? ORDER BY product_id` can use INDEX(sale_date, product_id)

**Indexed GROUP BY**:
- Works similarly to ORDER BY
- Database can pipeline groups without sorting
- Requires index to match GROUP BY columns after WHERE clause

### 4. Pagination (Partial Results)

**Top-N Queries**:
Use database-specific syntax to inform optimizer you don't need all rows:

```sql
SELECT * FROM sales ORDER BY sale_date DESC
FETCH FIRST 10 ROWS ONLY
```

**Benefits**:
- Database can use pipelined execution
- Response time becomes independent of table size
- Without pipelining, response time grows linearly with data volume

**Key Insight**: Response time of pipelined top-N query grows with number of rows **fetched**, not with table size.

## Write Performance Considerations

### Index Maintenance Costs
- INSERT: Must add entry to every index (more indexes = slower)
- DELETE: Uses indexes for WHERE clause, then removes from all indexes  
- UPDATE: Only affects indexes on changed columns
- **Rule**: Fewer indexes = better write performance

### Index-Only Scan Trade-offs
- Benefits read performance
- Costs: More storage, slower updates
- Only create when read benefit justifies write overhead

## Performance Testing & Scalability

### Data Volume Impact
- Queries that work fine in development may fail in production
- Test with production-like data volumes
- Performance difference between good/bad indexing grows exponentially with data volume

### Scalability Chart
Single performance value = one data point on scalability curve
Test performance across range of data volumes to understand true scalability

### System Load
- Production load affects response time due to resource contention
- Two dimensions of performance:
  - **Response Time**: Latency (user experience)
  - **Throughput**: Bandwidth (system capacity)

## Myths Debunked

1. **"Indexes Can Degenerate"** → FALSE. B-trees stay balanced. Rebuilding provides 0-2% benefit for typical queries.

2. **"Most Selective Column First"** → FALSE. Column order should support query patterns, not just selectivity.

3. **"Dynamic SQL is Slow"** → FALSE. Static SQL with "smart logic" is slower due to generic execution plans.

4. **"SELECT * is Bad"** → MISLEADING. It's about selecting only needed columns, not the star itself.

## Execution Plan Analysis

### Essential Components
1. Operations (what the database does)
2. Predicate Information (how it uses each condition)
3. Cost estimates (relative effort)
4. Row estimates (cardinality)

### What to Look For
- Access vs Filter predicates
- INDEX RANGE SCAN vs TABLE ACCESS FULL
- SORT operations
- JOIN methods (NESTED LOOPS, HASH JOIN, SORT-MERGE)

## Best Practices Summary

1. **Index for equality first, then for ranges**
2. **Choose concatenated index column order carefully** - match query patterns
3. **Never use functions on indexed columns** in WHERE clause
4. **Use explicit range conditions** for dates/times instead of TRUNC()
5. **Avoid leading wildcards** in LIKE patterns
6. **Use dynamic SQL** for dynamic filters (not "smart logic")
7. **Create covering indexes** only when justified by read volume
8. **Use partial indexes** for queue-like patterns
9. **Select only needed columns** (helps hash joins, index-only scans)
10. **Use bind parameters** always (security + performance)
11. **Check execution plans** with predicate information
12. **Test with production-like data volumes**
13. **Minimize number of indexes** to balance read vs write performance
14. **Execute joins in the database**, not in application
15. **Take control of ORM** - don't let it generate N+1 queries

## When to Apply These Techniques

- **Query is slow**: Check for missing indexes, function usage, leading wildcards
- **High read load**: Consider covering indexes, partial indexes
- **Write performance issues**: Reduce number of indexes, review which columns are indexed
- **Pagination queries**: Ensure pipelined ORDER BY with proper index
- **Join performance**: Analyze join algorithm, ensure proper indexing
- **Queue-like tables**: Use partial indexes for active items only

## Common Performance Issues and Solutions

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Full table scan | Missing index or function on column | Add index, remove functions |
| Slow date queries | Using TRUNC() or DATE_FORMAT() | Use explicit range conditions |
| Slow pagination | No pipelined ORDER BY | Add index matching ORDER BY |
| Slow joins | Wrong join algorithm or missing index | Analyze join type, add index to inner table for nested loops |
| Slow writes | Too many indexes | Remove unused indexes, review coverage |
| ORM performance | N+1 queries | Use proper fetch strategies, JOINs |

## Resources

Based on principles from: https://use-the-index-luke.com
- Comprehensive SQL indexing and optimization guide
- Free online resource covering all major databases
- Practical examples and explanations of database internals
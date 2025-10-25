---
name: sql-anti-patterns
description: Guide for writing maintainable, performant SQL queries by avoiding common anti-patterns like CASE WHEN abuse, function usage on indexed columns, SELECT *, DISTINCT overuse, excessive view nesting, and deep subquery nesting. Use when writing or reviewing SQL queries, database migrations, views, or optimizing query performance.
---

# SQL Anti-Patterns

## Overview

This skill provides guidance for writing clean, maintainable, and performant SQL queries by identifying and avoiding common anti-patterns that lead to performance degradation, maintenance difficulties, and data integrity issues.

Most SQL anti-patterns emerge from time pressure and short-term workarounds that accumulate technical debt. This guide helps identify these patterns early and provides better alternatives.

## Common Anti-Patterns to Avoid

### 1. CASE WHEN Abuse

**Problem:**
Scattering complex CASE WHEN statements across multiple queries leads to:
- Logic duplication and copy-paste errors
- Inconsistent interpretations of the same data
- Maintenance nightmare when business logic changes

**Example of Anti-Pattern:**
```sql
-- Scattered across multiple views and queries
SELECT
  CASE
    WHEN status_code = 1 THEN 'Out of Stock'
    WHEN status_code = 2 THEN 'In Stock'
    WHEN status_code = 3 THEN 'Discontinued'
    ELSE 'Unknown'
  END as status_text
FROM products;
```

**Better Approach:**
```sql
-- Create a dimension table or shared view
CREATE TABLE status_codes (
  code INT PRIMARY KEY,
  display_text VARCHAR(50)
);

-- Or create a shared view
CREATE VIEW vw_status_mapping AS
SELECT
  status_code,
  CASE
    WHEN status_code = 1 THEN 'Out of Stock'
    WHEN status_code = 2 THEN 'In Stock'
    WHEN status_code = 3 THEN 'Discontinued'
    ELSE 'Unknown'
  END as status_text
FROM (SELECT DISTINCT status_code FROM products) codes;

-- Then join consistently everywhere
SELECT p.*, s.status_text
FROM products p
LEFT JOIN vw_status_mapping s ON p.status_code = s.status_code;
```

**Key Principle:** Centralize business logic translation in dimension tables or shared views.

### 2. Functions on Indexed Columns (Non-Sargable Queries)

**Problem:**
Applying functions to indexed columns in WHERE clauses prevents index usage, causing full table scans.

**Example of Anti-Pattern:**
```sql
-- Index on 'name' column is not used
SELECT * FROM users WHERE UPPER(name) = 'JOHN DOE';

-- Index on 'created_at' is not used
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';
```

**Better Approach:**
```sql
-- Option 1: Apply function to the comparison value
SELECT * FROM users WHERE name = UPPER('john doe');

-- Option 2: Use case-insensitive collation
SELECT * FROM users WHERE name COLLATE Latin1_General_CI_AS = 'John Doe';

-- Option 3: Create a computed column with index (SQL Server example)
ALTER TABLE users ADD name_upper AS UPPER(name) PERSISTED;
CREATE INDEX idx_name_upper ON users(name_upper);
SELECT * FROM users WHERE name_upper = 'JOHN DOE';

-- Option 4: Use range comparisons for dates
SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2024-01-02';
```

**Key Principle:** Keep indexed columns "naked" in WHERE clauses. Transform the comparison value instead.

**Related Concept:** "Sargable" = Search ARGument ABLE (queries that can use indexes effectively)

### 3. SELECT * in Views

**Problem:**
Using SELECT * in views creates:
- Hidden dependencies on source table structure
- Performance issues from unnecessary columns
- Views breaking when source schema changes
- Ambiguity about what data is actually needed

**Example of Anti-Pattern:**
```sql
CREATE VIEW vw_user_orders AS
SELECT *
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**Better Approach:**
```sql
CREATE VIEW vw_user_orders AS
SELECT
  u.id as user_id,
  u.name,
  u.email,
  o.id as order_id,
  o.order_date,
  o.total_amount
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**Key Principle:** Always explicitly list columns in views. Be intentional about what data flows through.

### 4. DISTINCT as a "Fix" for Duplicates

**Problem:**
Using DISTINCT to hide duplicate results from incorrect joins masks data integrity issues and doesn't address root causes.

**Example of Anti-Pattern:**
```sql
-- Using DISTINCT to hide join problems
SELECT DISTINCT u.name, u.email
FROM users u
JOIN orders o ON u.id = o.user_id;  -- Creates duplicates for users with multiple orders
```

**Better Approach:**
```sql
-- Option 1: Fix the join logic if you want one row per user
SELECT u.name, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name, u.email;

-- Option 2: Use window functions for deduplication with deterministic ordering
SELECT *
FROM (
  SELECT
    u.*,
    ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.order_date DESC) as rn
  FROM users u
  JOIN orders o ON u.id = o.user_id
) ranked
WHERE rn = 1;

-- Some databases support QUALIFY for cleaner syntax
SELECT u.*
FROM users u
JOIN orders o ON u.id = o.user_id
QUALIFY ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.order_date DESC) = 1;
```

**When DISTINCT is Appropriate:**
- Getting unique values for a lookup list
- Explicit set operations where duplicates are semantically meaningful
- After proper relationship modeling confirms duplicates are expected

**Key Principle:** Understand and fix the relationship causing duplicates. Use GROUP BY or window functions with explicit ordering for controlled deduplication.

### 5. Excessive View Nesting

**Problem:**
Stacking views on top of views creates:
- Complex dependency chains that are hard to trace
- Severe performance degradation
- "Archaeological dig" debugging experiences
- Unclear data lineage

**Example of Anti-Pattern:**
```sql
-- Layer 1
CREATE VIEW vw_base AS SELECT * FROM raw_table;

-- Layer 2
CREATE VIEW vw_transform AS SELECT * FROM vw_base WHERE condition1 = true;

-- Layer 3
CREATE VIEW vw_aggregate AS SELECT col1, COUNT(*) FROM vw_transform GROUP BY col1;

-- Layer 4
CREATE VIEW vw_final AS SELECT * FROM vw_aggregate JOIN vw_transform ON ...;
```

**Better Approach:**
```sql
-- Option 1: Flatten into fewer layers
CREATE VIEW vw_user_metrics AS
WITH base_data AS (
  SELECT * FROM raw_table WHERE condition1 = true
),
aggregated AS (
  SELECT col1, COUNT(*) as count FROM base_data GROUP BY col1
)
SELECT * FROM aggregated;

-- Option 2: Materialize complex intermediate results
CREATE TABLE tbl_base_transformed AS
SELECT * FROM raw_table WHERE condition1 = true;

CREATE INDEX idx_col1 ON tbl_base_transformed(col1);

-- Then create simpler views on top
CREATE VIEW vw_metrics AS
SELECT col1, COUNT(*) FROM tbl_base_transformed GROUP BY col1;
```

**Key Principle:** Regularly flatten view hierarchies. Materialize complex transformations into tables with proper indexing.

### 6. Deep Subquery Nesting

**Problem:**
Deeply nested subqueries (3-4+ levels) create:
- Unreadable, unmaintainable code
- Difficult debugging and testing
- Poor query optimizer performance
- Context-switching mental overhead

**Example of Anti-Pattern:**
```sql
SELECT *
FROM (
  SELECT *
  FROM (
    SELECT *
    FROM (
      SELECT * FROM users WHERE active = 1
    ) active_users
    WHERE created_at > '2024-01-01'
  ) recent_users
  WHERE email LIKE '%@company.com'
) company_users;
```

**Better Approach:**
```sql
-- Use CTEs (Common Table Expressions) for readability
WITH active_users AS (
  SELECT * FROM users WHERE active = 1
),
recent_users AS (
  SELECT * FROM active_users WHERE created_at > '2024-01-01'
),
company_users AS (
  SELECT * FROM recent_users WHERE email LIKE '%@company.com'
)
SELECT * FROM company_users;
```

**Key Principle:** Use CTEs to break complex queries into logical, named steps. Each CTE should represent one clear transformation.

## Best Practices

### Write SQL as Production Code

Treat SQL with the same rigor as application code:
- Use consistent formatting and indentation
- Add meaningful comments explaining business logic
- Implement code review processes
- Version control all SQL artifacts
- Document table relationships and dependencies

### Query Formatting Standards

```sql
-- Good: Readable, consistent formatting
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(o.id) as order_count,
  SUM(o.total_amount) as total_spent
FROM users u
LEFT JOIN orders o
  ON u.id = o.user_id
  AND o.status = 'completed'
WHERE u.active = 1
  AND u.created_at >= '2024-01-01'
GROUP BY u.id, u.name, u.email
HAVING COUNT(o.id) > 0
ORDER BY total_spent DESC;
```

### Join Clarity

Always explicitly define join types and conditions:
```sql
-- Good: Clear join intent
FROM users u
INNER JOIN orders o ON u.id = o.user_id
LEFT JOIN addresses a ON u.id = a.user_id AND a.is_primary = 1

-- Avoid: Implicit joins
FROM users u, orders o
WHERE u.id = o.user_id
```

### Performance Checklist

Before deploying SQL changes:
- [ ] All WHERE clause columns are indexed or sargable
- [ ] No functions applied to indexed columns in predicates
- [ ] JOIN conditions are explicit and use indexed columns
- [ ] DISTINCT usage is justified (not masking join issues)
- [ ] View nesting depth is reasonable (<3 layers)
- [ ] CTEs used for complex logic instead of deep subqueries
- [ ] Query plan reviewed for full table scans
- [ ] Column list is explicit (no SELECT * in production views)

## When to Use This Skill

Apply these principles when:
- Writing new SQL queries, views, or stored procedures
- Reviewing SQL code in pull requests
- Debugging slow query performance
- Refactoring legacy SQL code
- Designing database schemas and views
- Creating database migration scripts
- Optimizing data pipelines
- Resolving data integrity issues from incorrect joins

## Additional Considerations

### NOT IN and != Operators

Be cautious with negative conditions:
```sql
-- Often inefficient: requires scanning everything that doesn't match
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM banned_users);

-- Better: Use LEFT JOIN with NULL check
SELECT u.*
FROM users u
LEFT JOIN banned_users b ON u.id = b.user_id
WHERE b.user_id IS NULL;
```

### NULL Handling

Understand your database's NULL behavior:
- NULL != NULL evaluates to NULL (not TRUE or FALSE)
- NULL values may or may not be indexed (database-specific)
- Use IS NULL / IS NOT NULL for null checks
- Consider adding boolean flags for "nullable" columns that need indexing

### Expression Indexes

Some databases support indexing computed expressions:
```sql
-- PostgreSQL example
CREATE INDEX idx_upper_email ON users (UPPER(email));

-- Now this query can use the index
SELECT * FROM users WHERE UPPER(email) = 'USER@EXAMPLE.COM';
```

## Conclusion

Most SQL anti-patterns result from time pressure and "quick fixes" that accumulate over time. Investing a few extra minutes in proper design—clear joins, centralized logic, explicit columns, and flat structures—saves hours of debugging and refactoring later.

Remember: SQL is not just scripting—it's production code that requires careful design, testing, and maintenance.

# Chapter 5: Indexing & Performance

## What is an Index?

An **index** is a data structure that improves the speed of data retrieval operations on a database table at the cost of additional storage space and slower writes.

**Analogy:** Like a book's index - instead of reading every page to find "Database", you check the index which says "Database: page 45, 67, 89".

```
WITHOUT INDEX:                    WITH INDEX:
┌─────────────────┐              ┌─────────────────┐
│ Search all rows │              │ Search index    │
│ Row 1 → Check   │              │ Jump to exact   │
│ Row 2 → Check   │              │ row location    │
│ Row 3 → Check   │              └─────────────────┘
│ ...             │                     │
│ Row 1M → Check  │                     ▼
└─────────────────┘              ┌─────────────────┐
Time: O(n)                       │ Return result   │
                                 └─────────────────┘
                                 Time: O(log n)
```

---

## Why Do We Need Indexes?

### Problem Without Index

```sql
-- Table with 10 million users
SELECT * FROM users WHERE email = 'john@example.com';

-- Without index: Scans ALL 10 million rows
-- Time: 5-10 seconds ❌
```

### Solution With Index

```sql
-- Create index on email column
CREATE INDEX idx_email ON users(email);

-- Same query now
SELECT * FROM users WHERE email = 'john@example.com';

-- With index: Directly jumps to matching row
-- Time: 0.001 seconds ✅
```

---

## How Index Works Internally

### B+ Tree Structure (Most Common)

```
                        ROOT NODE
                    ┌─────┬─────┐
                    │ 50  │ 100 │
                    └──┬──┴──┬──┘
                       │     │
         ┌─────────────┘     └─────────────┐
         ▼                                 ▼
    INTERNAL NODE                    INTERNAL NODE
   ┌────┬────┬────┐                ┌────┬────┬────┐
   │ 20 │ 35 │ 45 │                │ 70 │ 85 │ 95 │
   └─┬──┴─┬──┴─┬──┘                └─┬──┴─┬──┴─┬──┘
     │    │    │                     │    │    │
     ▼    ▼    ▼                     ▼    ▼    ▼
   LEAF NODES (Actual Data Pointers)
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │10,15,18  │→│22,28,33  │→│36,40,44  │→ ...
   └──────────┘ └──────────┘ └──────────┘
```

**Key Points:**
- **Balanced tree** - All leaf nodes at same level
- **Sorted order** - Easy to find range queries
- **Leaf nodes linked** - Fast sequential access
- **Height = log(n)** - 10M rows = ~4 levels only!

---

## Types of Indexes

### 1. Primary Index (Clustered Index)

**Definition:** Index on primary key. Data rows are physically stored in index order.

```
┌─────────────────────────────────────────────────────┐
│ PRIMARY INDEX (Clustered)                           │
├─────────────────────────────────────────────────────┤
│ • Only ONE per table                                │
│ • Data rows stored in sorted order                  │
│ • Created automatically on PRIMARY KEY              │
│ • Fastest for primary key lookups                   │
└─────────────────────────────────────────────────────┘

Example:
┌────────┬──────────┬─────────────┐
│ emp_id │ name     │ salary      │  ← Rows stored in emp_id order
├────────┼──────────┼─────────────┤
│ 1      │ Alice    │ 50000       │
│ 2      │ Bob      │ 60000       │
│ 3      │ Carol    │ 55000       │
│ 4      │ David    │ 70000       │
└────────┴──────────┴─────────────┘
```

### 2. Secondary Index (Non-Clustered Index)

**Definition:** Index on non-primary key columns. Contains pointers to actual data rows.

```
┌─────────────────────────────────────────────────────┐
│ SECONDARY INDEX (Non-Clustered)                     │
├─────────────────────────────────────────────────────┤
│ • Multiple per table allowed                        │
│ • Stores column value + pointer to row              │
│ • Extra lookup needed (index → row)                 │
│ • Good for frequently searched columns              │
└─────────────────────────────────────────────────────┘

Example: Index on 'name' column
┌──────────┬─────────────┐
│ name     │ Row Pointer │
├──────────┼─────────────┤
│ Alice    │ → Row 1     │
│ Bob      │ → Row 2     │
│ Carol    │ → Row 3     │
│ David    │ → Row 4     │
└──────────┴─────────────┘
```

### 3. Composite Index (Multi-Column Index)

**Definition:** Index on multiple columns together.

```sql
CREATE INDEX idx_name_dept ON employees(department, name);
```

```
┌─────────────────────────────────────────────────────┐
│ COMPOSITE INDEX                                     │
├─────────────────────────────────────────────────────┤
│ Index on (department, name)                         │
├──────────────┬──────────┬─────────────┐             │
│ department   │ name     │ Row Pointer │             │
├──────────────┼──────────┼─────────────┤             │
│ Engineering  │ Alice    │ → Row 3     │             │
│ Engineering  │ Bob      │ → Row 1     │             │
│ Marketing    │ Carol    │ → Row 2     │             │
│ Marketing    │ David    │ → Row 4     │             │
└──────────────┴──────────┴─────────────┘             │
└─────────────────────────────────────────────────────┘
```

**Important: Column Order Matters!**

```
Index: (department, name)

✅ Uses Index:
WHERE department = 'Engineering'
WHERE department = 'Engineering' AND name = 'Alice'

❌ Cannot Use Index:
WHERE name = 'Alice'  -- First column not used!
```

### 4. Unique Index

**Definition:** Ensures all values in indexed column(s) are unique.

```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

```
┌─────────────────────────────────────────────────────┐
│ UNIQUE INDEX                                        │
├─────────────────────────────────────────────────────┤
│ • Enforces uniqueness constraint                    │
│ • Automatically created for PRIMARY KEY & UNIQUE   │
│ • Rejects duplicate values on insert               │
└─────────────────────────────────────────────────────┘
```

### 5. Full-Text Index

**Definition:** For searching text content (articles, descriptions).

```sql
CREATE FULLTEXT INDEX idx_content ON articles(title, body);

-- Usage
SELECT * FROM articles 
WHERE MATCH(title, body) AGAINST('database optimization');
```

### 6. Partial Index (Filtered Index)

**Definition:** Index on subset of rows based on condition.

```sql
-- Only index active users (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) 
WHERE is_active = true;
```

```
┌─────────────────────────────────────────────────────┐
│ PARTIAL INDEX                                       │
├─────────────────────────────────────────────────────┤
│ • Smaller index size                                │
│ • Faster for specific queries                       │
│ • Good when querying subset frequently              │
└─────────────────────────────────────────────────────┘
```

---

## Index Types Comparison

| Index Type | Use Case | Pros | Cons |
|------------|----------|------|------|
| **Primary** | Primary key lookups | Fastest, auto-created | Only one per table |
| **Secondary** | Non-PK column search | Multiple allowed | Extra lookup needed |
| **Composite** | Multi-column WHERE | Single index for multiple cols | Column order matters |
| **Unique** | Enforce uniqueness | Data integrity | Slower inserts |
| **Full-Text** | Text search | Natural language search | Large index size |
| **Partial** | Filtered queries | Smaller, faster | Limited use cases |

---

## When to Create Index

### ✅ Create Index When:

```
┌─────────────────────────────────────────────────────┐
│ 1. Column used frequently in WHERE clause           │
│ 2. Column used in JOIN conditions                   │
│ 3. Column used in ORDER BY                          │
│ 4. Column used in GROUP BY                          │
│ 5. Column with high selectivity (many unique values)│
│ 6. Foreign key columns                              │
└─────────────────────────────────────────────────────┘
```

### ❌ Avoid Index When:

```
┌─────────────────────────────────────────────────────┐
│ 1. Small tables (< 1000 rows)                       │
│ 2. Columns with low selectivity (gender, boolean)   │
│ 3. Frequently updated columns                       │
│ 4. Columns rarely used in queries                   │
│ 5. Tables with heavy INSERT/UPDATE/DELETE           │
└─────────────────────────────────────────────────────┘
```

---

## Query Execution Plan (EXPLAIN)

### What is EXPLAIN?

Shows how database executes a query - helps identify performance issues.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';
```

### Reading EXPLAIN Output

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EXPLAIN OUTPUT                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ id │ select_type │ table │ type  │ key      │ rows    │ Extra              │
├────┼─────────────┼───────┼───────┼──────────┼─────────┼────────────────────┤
│ 1  │ SIMPLE      │ users │ ref   │ idx_email│ 1       │ Using index        │
└────┴─────────────┴───────┴───────┴──────────┴─────────┴────────────────────┘
```

### Key Fields to Check

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ FIELD        │ MEANING                                                      │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ type         │ Access method (best to worst):                               │
│              │ const > eq_ref > ref > range > index > ALL                   │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ key          │ Index used (NULL = no index used)                            │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ rows         │ Estimated rows to examine (lower = better)                   │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Extra        │ Additional info:                                             │
│              │ "Using index" = Good (index-only scan)                       │
│              │ "Using filesort" = Bad (extra sorting needed)                │
│              │ "Using temporary" = Bad (temp table created)                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Access Types (Best to Worst)

```
BEST ──────────────────────────────────────────────────────────────► WORST

const    eq_ref    ref      range     index      ALL
  │         │       │         │         │         │
  ▼         ▼       ▼         ▼         ▼         ▼
Single   Unique   Non-     Range    Full      Full
row      index    unique   scan     index     table
lookup   lookup   index    (>,<)    scan      scan
                  lookup
```

---

## Common Performance Problems & Solutions

### Problem 1: Full Table Scan

```sql
-- Bad: No index on email
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';
-- type: ALL, rows: 10000000 ❌

-- Solution: Add index
CREATE INDEX idx_email ON users(email);
-- type: ref, rows: 1 ✅
```

### Problem 2: Index Not Used

```sql
-- Bad: Function on indexed column
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
-- Index on email NOT used! ❌

-- Solution: Use functional index or fix query
CREATE INDEX idx_email_lower ON users(LOWER(email));
-- OR
SELECT * FROM users WHERE email = 'john@example.com';
```

### Problem 3: Wrong Index Order

```sql
-- Index: (department, name)

-- Bad: Searching by second column only
SELECT * FROM users WHERE name = 'John';
-- Index NOT used! ❌

-- Good: Include first column
SELECT * FROM users WHERE department = 'Engineering' AND name = 'John';
-- Index used ✅
```

### Problem 4: Too Many Indexes

```
┌─────────────────────────────────────────────────────┐
│ PROBLEM: Over-indexing                              │
├─────────────────────────────────────────────────────┤
│ • Each INSERT updates ALL indexes                   │
│ • Each UPDATE on indexed column updates index       │
│ • More storage space needed                         │
│ • Index maintenance overhead                        │
├─────────────────────────────────────────────────────┤
│ SOLUTION: Only index what you query                 │
│ • Audit unused indexes periodically                 │
│ • Use composite index instead of multiple single    │
└─────────────────────────────────────────────────────┘
```

---

## Index Best Practices

### 1. Selectivity Rule

```
HIGH SELECTIVITY (Good for index):
┌─────────────────────────────────────────────────────┐
│ email        → Unique values      → Index ✅        │
│ phone        → Unique values      → Index ✅        │
│ user_id      → Unique values      → Index ✅        │
└─────────────────────────────────────────────────────┘

LOW SELECTIVITY (Bad for index):
┌─────────────────────────────────────────────────────┐
│ gender       → 2-3 values         → No Index ❌     │
│ is_active    → true/false         → No Index ❌     │
│ status       → 3-5 values         → Maybe Partial   │
└─────────────────────────────────────────────────────┘
```

### 2. Covering Index

```sql
-- Query needs: name, email
SELECT name, email FROM users WHERE department = 'Engineering';

-- Covering index: Include all needed columns
CREATE INDEX idx_dept_covering ON users(department, name, email);

-- Result: "Using index" - No table lookup needed!
```

### 3. Index Column Order

```
┌─────────────────────────────────────────────────────┐
│ COMPOSITE INDEX ORDER RULES                         │
├─────────────────────────────────────────────────────┤
│ 1. Equality columns first (WHERE col = value)       │
│ 2. Range columns last (WHERE col > value)           │
│ 3. Most selective column first                      │
└─────────────────────────────────────────────────────┘

Example:
-- Query: WHERE status = 'active' AND created_at > '2024-01-01'
-- Best index: (status, created_at)
--             equality↑    range↑
```

---

## Real-World Examples

### Example 1: E-commerce Product Search

```sql
-- Table: products (10 million rows)
-- Common queries:
-- 1. Search by category
-- 2. Filter by price range
-- 3. Sort by rating

-- Optimal indexes:
CREATE INDEX idx_category ON products(category_id);
CREATE INDEX idx_category_price ON products(category_id, price);
CREATE INDEX idx_category_rating ON products(category_id, rating DESC);

-- Query using composite index
SELECT * FROM products 
WHERE category_id = 5 
  AND price BETWEEN 100 AND 500
ORDER BY rating DESC
LIMIT 20;
```

### Example 2: Social Media Feed

```sql
-- Table: posts (100 million rows)
-- Query: Get user's feed (posts from followed users)

-- Without proper index: 30+ seconds ❌
SELECT p.* FROM posts p
JOIN follows f ON p.user_id = f.following_id
WHERE f.follower_id = 12345
ORDER BY p.created_at DESC
LIMIT 20;

-- Optimal indexes:
CREATE INDEX idx_follows_follower ON follows(follower_id, following_id);
CREATE INDEX idx_posts_user_time ON posts(user_id, created_at DESC);

-- With indexes: < 50ms ✅
```

### Example 3: User Authentication

```sql
-- Table: users (50 million rows)
-- Query: Login by email

-- Critical index for login performance
CREATE UNIQUE INDEX idx_email ON users(email);

-- Query
SELECT user_id, password_hash, is_active 
FROM users 
WHERE email = 'user@example.com';

-- Result: O(log n) lookup = ~0.001 seconds
```

---

## Index Maintenance

### Monitor Index Usage

```sql
-- MySQL: Check index usage
SELECT 
    table_name,
    index_name,
    stat_value as pages_read
FROM mysql.innodb_index_stats
WHERE stat_name = 'n_leaf_pages'
ORDER BY stat_value DESC;

-- PostgreSQL: Check unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

### Remove Unused Indexes

```sql
-- Identify and drop unused indexes
DROP INDEX idx_unused ON table_name;
```

### Rebuild Fragmented Indexes

```sql
-- MySQL
ALTER TABLE users ENGINE=InnoDB;

-- PostgreSQL
REINDEX INDEX idx_email;
```

---

## Quick Reference Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        INDEX CHEAT SHEET                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│ CREATE INDEX idx_name ON table(column);        -- Basic index               │
│ CREATE UNIQUE INDEX idx_name ON table(column); -- Unique index              │
│ CREATE INDEX idx_name ON table(col1, col2);    -- Composite index           │
│ DROP INDEX idx_name ON table;                  -- Remove index              │
│ SHOW INDEX FROM table;                         -- List indexes (MySQL)      │
│ EXPLAIN SELECT ...;                            -- Check query plan          │
├─────────────────────────────────────────────────────────────────────────────┤
│ GOOD INDEX CANDIDATES:                                                       │
│ • WHERE clause columns                                                       │
│ • JOIN columns                                                               │
│ • ORDER BY columns                                                           │
│ • Foreign keys                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│ AVOID INDEXING:                                                              │
│ • Small tables                                                               │
│ • Low selectivity columns                                                    │
│ • Frequently updated columns                                                 │
│ • Columns rarely in queries                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ EXPLAIN TYPE (Best → Worst):                                                 │
│ const → eq_ref → ref → range → index → ALL                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Beginner Level

**Q1: What is an index and why is it used?**
> Index is a data structure that improves query speed by allowing direct access to rows instead of scanning entire table. Like a book's index.

**Q2: What is the difference between clustered and non-clustered index?**
> Clustered: Data rows stored in index order. Only one per table.
> Non-clustered: Separate structure with pointers to data. Multiple allowed.

**Q3: When should you NOT create an index?**
> Small tables, low selectivity columns (gender, boolean), frequently updated columns, rarely queried columns.

### Intermediate Level

**Q4: What is a composite index and how does column order matter?**
> Index on multiple columns. Order matters because index can only be used left-to-right. Index(A,B) works for WHERE A=x or WHERE A=x AND B=y, but NOT for WHERE B=y alone.

**Q5: How do you identify if a query is using an index?**
> Use EXPLAIN command. Check 'type' field (should not be ALL), 'key' field (should show index name), 'rows' field (should be low).

**Q6: What is a covering index?**
> Index that contains all columns needed by query. Avoids table lookup. Shows "Using index" in EXPLAIN.

### Senior Level

**Q7: Design indexing strategy for an e-commerce search with filters.**
> Create composite indexes based on common filter combinations. Put equality filters first, range filters last. Consider covering indexes for frequently accessed columns. Use partial indexes for status filters.

**Q8: How would you handle index maintenance for a high-traffic table?**
> Monitor index usage, remove unused indexes, schedule rebuilds during low traffic, use online index operations, consider partitioning for very large tables.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        KEY TAKEAWAYS                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Index = Fast lookup structure (B+ Tree)                                  │
│ 2. Primary Index = One per table, data in sorted order                      │
│ 3. Secondary Index = Multiple allowed, pointers to data                     │
│ 4. Composite Index = Multi-column, order matters (left-to-right)            │
│ 5. Use EXPLAIN to verify index usage                                        │
│ 6. Index columns in WHERE, JOIN, ORDER BY                                   │
│ 7. Avoid over-indexing (slows writes)                                       │
│ 8. High selectivity = Good index candidate                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

**Estimated Reading Time**: 30-35 minutes  
**Next Chapter**: [Chapter 6: Transactions & Concurrency](../Chapter-6-Transactions-Concurrency/)

*[← Back to Chapter 4: Advanced Relationships](../Chapter-4-Advanced-Relationships/)*

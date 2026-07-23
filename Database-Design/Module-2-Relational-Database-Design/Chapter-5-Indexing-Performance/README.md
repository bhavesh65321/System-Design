# Chapter 5: Indexing & Performance

## 1. Understanding Data Storage and Representation

### Logical vs Physical Representation

**Logical Representation (User's View):**
```
┌─────────────────────────────────────────────────────┐
│                    employees                         │
├──────────┬──────────┬─────────────┬─────────────────┤
│ emp_id   │ name     │ address     │ salary          │
├──────────┼──────────┼─────────────┼─────────────────┤
│ 1        │ Alice    │ NYC         │ 50000           │
│ 2        │ Bob      │ LA          │ 60000           │
│ 3        │ Carol    │ Chicago     │ 55000           │
└──────────┴──────────┴─────────────┴─────────────────┘

User sees: Clean tables with rows and columns
```

**Physical Representation (DBMS View):**
```
┌─────────────────────────────────────────────────────┐
│              DATA PAGES (8KB each)                   │
├─────────────────────────────────────────────────────┤
│ Page 1: Contains rows 1-125 in data blocks          │
│ Page 2: Contains rows 126-250 in data blocks        │
│ Page 3: Contains rows 251-375 in data blocks        │
└─────────────────────────────────────────────────────┘

DBMS sees: Data stored in pages on disk
```

### Data Pages Structure

**A data page is typically 8KB in size:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATA PAGE (8KB)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ PAGE HEADER (96 bytes)                                   │   │
│  │ • Page ID                                                │   │
│  │ • Free space available                                   │   │
│  │ • Checksum                                               │   │
│  │ • Page type (data/index)                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ DATA RECORDS AREA (~8060 bytes)                          │   │
│  │ • Row 1: 64 bytes                                        │   │
│  │ • Row 2: 64 bytes                                        │   │
│  │ • Row 3: 64 bytes                                        │   │
│  │ • ...                                                    │   │
│  │ • Row 125: 64 bytes                                      │   │
│  │ (Can fit ~125 rows of 64 bytes each)                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OFFSET ARRAY (36 bytes)                                  │   │
│  │ • Pointers to row locations                              │   │
│  │ • Ensures logical sequence                               │   │
│  │ • Maps physical to logical order                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Total: 96 + 8060 + 36 = 8192 bytes (8KB)
```

### Row Storage in Data Pages

**How rows are stored:**

```
INSERTION ORDER:
INSERT emp_id=5, name='Eve'
INSERT emp_id=2, name='Bob'
INSERT emp_id=8, name='Frank'
INSERT emp_id=1, name='Alice'

PHYSICAL STORAGE (in page):
┌─────────────────────────────────────────────────────┐
│ Position 1: emp_id=5, name='Eve'   (inserted 1st)   │
│ Position 2: emp_id=2, name='Bob'   (inserted 2nd)   │
│ Position 3: emp_id=8, name='Frank' (inserted 3rd)   │
│ Position 4: emp_id=1, name='Alice' (inserted 4th)   │
└─────────────────────────────────────────────────────┘

OFFSET ARRAY (Logical Order):
┌─────────────────────────────────────────────────────┐
│ Logical 1 → Points to Position 4 (emp_id=1)         │
│ Logical 2 → Points to Position 2 (emp_id=2)         │
│ Logical 3 → Points to Position 1 (emp_id=5)         │
│ Logical 4 → Points to Position 3 (emp_id=8)         │
└─────────────────────────────────────────────────────┘

Result: Rows appear sorted by emp_id even though 
        physically stored in insertion order!
```

**Capacity Calculation:**
```
Page Size: 8KB = 8192 bytes
Header: 96 bytes
Offset Array: 36 bytes
Available for data: 8192 - 96 - 36 = 8060 bytes

If each row is 64 bytes:
Rows per page = 8060 / 64 ≈ 125 rows

If table has 1 million rows:
Pages needed = 1,000,000 / 125 = 8,000 pages
```

### Data Blocks

**Physical storage units on disk:**

```
┌─────────────────────────────────────────────────────┐
│              DISK STORAGE HIERARCHY                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│  DISK                                                │
│  ├── Data Block 1 (4KB)                              │
│  ├── Data Block 2 (4KB)                              │
│  ├── Data Block 3 (4KB)                              │
│  └── Data Block 4 (4KB)                              │
│      ↓                                               │
│  DATA PAGE (8KB) = 2 Data Blocks                     │
│      ↓                                               │
│  MEMORY (Buffer Pool)                                │
│                                                      │
└─────────────────────────────────────────────────────┘

Key Points:
• Data blocks = Minimum I/O unit (4KB typically)
• Data page = Logical unit (8KB)
• DBMS maps pages to blocks for disk I/O
• When query needs data, entire page loaded to memory
```

---

## 2. Role of Indexing in Search Optimization

### Purpose of Indexing

**Without Index (Sequential Scan):**
```
Query: SELECT * FROM users WHERE emp_id = 5;

┌─────────────────────────────────────────────────────┐
│ SEQUENTIAL SCAN (O(n))                              │
├─────────────────────────────────────────────────────┤
│ Check Page 1: emp_id 1,2,3,4 → Not found           │
│ Check Page 2: emp_id 5,6,7,8 → FOUND! ✓            │
│ Check Page 3: emp_id 9,10,11,12 → (unnecessary)    │
│ ...                                                 │
│ Check Page 8000: emp_id 999996-1000000             │
│                                                     │
│ Time: Must read ~4000 pages (50% of table)          │
│ Complexity: O(n)                                    │
└─────────────────────────────────────────────────────┘
```

**With Index (B+ Tree Search):**
```
Query: SELECT * FROM users WHERE emp_id = 5;

┌─────────────────────────────────────────────────────┐
│ B+ TREE SEARCH (O(log n))                           │
├─────────────────────────────────────────────────────┤
│ Root: Is 5 < 500? Yes, go left                      │
│ Level 1: Is 5 < 250? Yes, go left                   │
│ Level 2: Is 5 < 125? Yes, go left                   │
│ Leaf: Found emp_id=5 → Page pointer                 │
│                                                     │
│ Time: Read ~4 pages (index pages + data page)       │
│ Complexity: O(log n)                                │
│ Speedup: 1000x faster!                              │
└─────────────────────────────────────────────────────┘
```

### How Indexing Works

**B+ Tree for emp_id column:**

```
                        ROOT
                    ┌───────────┐
                    │ 500       │
                    └─────┬─────┘
                          │
            ┌─────────────┴─────────────┐
            ▼                           ▼
        LEVEL 1                     LEVEL 1
    ┌─────────────┐             ┌─────────────┐
    │ 250         │             │ 750         │
    └──┬──────┬───┘             └──┬──────┬───┘
       │      │                    │      │
    ┌──▼──┐ ┌─▼──┐             ┌──▼──┐ ┌─▼──┐
    │125  │ │375 │             │625  │ │875 │
    └──┬──┘ └─┬──┘             └──┬──┘ └─┬──┘
       │      │                   │      │
    LEAF NODES (Actual Values)
    ┌──────────────────────────────────────────┐
    │ 1,2,3,4,5 → Page 1                       │
    │ 6,7,8,9,10 → Page 2                      │
    │ ...                                      │
    │ 995,996,997,998,999,1000 → Page 8000    │
    └──────────────────────────────────────────┘

Search for emp_id=5:
1. Start at root (500)
2. 5 < 500? Yes → Go left
3. 5 < 250? Yes → Go left
4. 5 < 125? No → Go right
5. Found in leaf: 5 → Page 1
6. Load Page 1 from disk
```

---

## 3. How DBMS Manages Data Pages, Indexing, and Rows

### Data Page Selection During Insertions

**Scenario: Insert new row with emp_id=150**

```
STEP 1: Use B+ Tree to find correct page
┌─────────────────────────────────────────────────────┐
│ Search index for emp_id=150                         │
│ Navigate B+ Tree → Find Page 2 (contains 125-250)   │
└─────────────────────────────────────────────────────┘

STEP 2: Check if page has space
┌─────────────────────────────────────────────────────┐
│ Page 2 Status:                                      │
│ • Contains 125 rows (64 bytes each)                 │
│ • Used space: 125 × 64 = 8000 bytes                │
│ • Available: 8060 - 8000 = 60 bytes                │
│ • New row: 64 bytes                                │
│ • Result: NOT ENOUGH SPACE! ❌                      │
└─────────────────────────────────────────────────────┘

STEP 3: Page Splitting
┌─────────────────────────────────────────────────────┐
│ BEFORE SPLIT:                                       │
│ Page 2: [125,126,...,249] (125 rows)                │
│                                                     │
│ AFTER SPLIT:                                        │
│ Page 2: [125,126,...,187] (63 rows)                 │
│ Page 2b: [188,189,...,249,150] (63 rows + new)      │
│                                                     │
│ UPDATE INDEX:                                       │
│ • Update B+ Tree pointers                           │
│ • Page 2 now points to [125-187]                    │
│ • Page 2b now points to [150,188-249]               │
└─────────────────────────────────────────────────────┘
```

### Offset Array Role

**Ensures logical ordering independent of physical order:**

```
PHYSICAL STORAGE (Insertion order):
┌─────────────────────────────────────────────────────┐
│ Pos 1: emp_id=150 (inserted last)                   │
│ Pos 2: emp_id=125 (inserted 1st)                    │
│ Pos 3: emp_id=200 (inserted 2nd)                    │
│ Pos 4: emp_id=175 (inserted 3rd)                    │
└─────────────────────────────────────────────────────┘

OFFSET ARRAY (Logical order):
┌─────────────────────────────────────────────────────┐
│ Logical 1 → Physical Pos 2 (emp_id=125)             │
│ Logical 2 → Physical Pos 4 (emp_id=175)             │
│ Logical 3 → Physical Pos 1 (emp_id=150)             │
│ Logical 4 → Physical Pos 3 (emp_id=200)             │
└─────────────────────────────────────────────────────┘

RESULT: Rows appear sorted [125,150,175,200]
        even though physically stored differently!
```

### Steps for Query Execution

**Query: SELECT * FROM employees WHERE emp_id = 150;**

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Load Index Page                                         │
├─────────────────────────────────────────────────────────────────┤
│ • Read B+ Tree root from disk                                   │
│ • Load into memory (buffer pool)                                │
│ • Time: 1 disk I/O                                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: Traverse Index                                          │
├─────────────────────────────────────────────────────────────────┤
│ • Navigate B+ Tree: 150 < 500? Yes → Left                       │
│ • Navigate B+ Tree: 150 < 250? Yes → Left                       │
│ • Navigate B+ Tree: 150 < 125? No → Right                       │
│ • Found leaf node: emp_id=150 → Page 2b                         │
│ • Time: In-memory operations (fast)                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: Load Data Block                                         │
├─────────────────────────────────────────────────────────────────┤
│ • Index says: emp_id=150 is in Page 2b                          │
│ • Load Page 2b from disk (2 data blocks)                        │
│ • Load into buffer pool                                         │
│ • Time: 1 disk I/O                                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: Access Row Using Offset Array                           │
├─────────────────────────────────────────────────────────────────┤
│ • Page 2b offset array: Logical 3 → Physical Pos 1              │
│ • Jump to position 1 in page                                    │
│ • Read row: emp_id=150, name='Carol', salary=55000              │
│ • Time: In-memory operation (microseconds)                      │
└─────────────────────────────────────────────────────────────────┘

TOTAL TIME: ~2 disk I/Os + in-memory navigation
WITHOUT INDEX: ~4000 disk I/Os (sequential scan)
SPEEDUP: 2000x faster!
```

---

## 4. Types of Indexing: Clustered vs Non-Clustered

### Clustered Indexing

**Definition:** Rows are physically ordered in data pages to match index order.

```
┌─────────────────────────────────────────────────────┐
│ CLUSTERED INDEX (on emp_id)                         │
├─────────────────────────────────────────────────────┤
│ • Only ONE per table                                │
│ • Determines physical row order                      │
│ • Usually on PRIMARY KEY                            │
│ • Data pages sorted by index column                 │
└─────────────────────────────────────────────────────┘

PHYSICAL DATA PAGES:
┌──────────────────────────────────────────────────────┐
│ Page 1: emp_id [1,2,3,4,5]                           │
│ Page 2: emp_id [6,7,8,9,10]                          │
│ Page 3: emp_id [11,12,13,14,15]                      │
│ ...                                                  │
│ Page 8000: emp_id [999996,999997,999998,999999,1000]│
└──────────────────────────────────────────────────────┘

OFFSET ARRAY in each page:
┌──────────────────────────────────────────────────────┐
│ Page 1 Offset Array:                                 │
│ Logical 1 → Pos 1 (emp_id=1)                         │
│ Logical 2 → Pos 2 (emp_id=2)                         │
│ Logical 3 → Pos 3 (emp_id=3)                         │
│ Logical 4 → Pos 4 (emp_id=4)                         │
│ Logical 5 → Pos 5 (emp_id=5)                         │
└──────────────────────────────────────────────────────┘

BENEFIT: Range queries are VERY fast
Query: SELECT * FROM employees WHERE emp_id BETWEEN 100 AND 200;
• Find Page containing emp_id=100
• Read sequentially until emp_id=200
• No random page jumps needed!
```

### Non-Clustered Indexing

**Definition:** Separate B+ Tree on non-primary key columns with pointers to data.

```
┌─────────────────────────────────────────────────────┐
│ NON-CLUSTERED INDEX (on name)                       │
├─────────────────────────────────────────────────────┤
│ • Multiple allowed per table                        │
│ • Does NOT affect physical row order                │
│ • Maintains separate B+ Tree                        │
│ • Leaf nodes contain pointers to data pages         │
└─────────────────────────────────────────────────────┘

PHYSICAL DATA PAGES (unchanged):
┌──────────────────────────────────────────────────────┐
│ Page 1: emp_id [1,2,3,4,5]                           │
│ Page 2: emp_id [6,7,8,9,10]                          │
│ Page 3: emp_id [11,12,13,14,15]                      │
└──────────────────────────────────────────────────────┘

SEPARATE NON-CLUSTERED INDEX (on name):
┌──────────────────────────────────────────────────────┐
│ B+ Tree Leaf Nodes:                                  │
│ Alice → Page 1, Pos 1                                │
│ Bob → Page 2, Pos 3                                  │
│ Carol → Page 3, Pos 2                                │
│ David → Page 1, Pos 5                                │
│ Eve → Page 2, Pos 1                                  │
└──────────────────────────────────────────────────────┘

QUERY: SELECT * FROM employees WHERE name = 'Carol';
1. Search name index B+ Tree → Find 'Carol'
2. Get pointer: Page 3, Pos 2
3. Load Page 3 from disk
4. Use offset array to jump to Pos 2
5. Return row

COST: 2 disk I/Os (index page + data page)
```

### Comparison: Clustered vs Non-Clustered

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLUSTERED vs NON-CLUSTERED                    │
├──────────────────────┬──────────────────┬──────────────────────┤
│ Aspect               │ Clustered        │ Non-Clustered        │
├──────────────────────┼──────────────────┼──────────────────────┤
│ Per Table            │ Only 1           │ Multiple (up to 999) │
│ Physical Order       │ Determines       │ Does not affect      │
│ Storage              │ Data pages       │ Separate B+ Tree     │
│ Range Queries        │ Very fast        │ Slower (jumps)       │
│ Lookup Speed         │ Fast             │ Slower (2 lookups)   │
│ Insert/Update        │ Expensive        │ Less expensive       │
│ Space Overhead       │ None (data)      │ Extra B+ Tree        │
├──────────────────────┼──────────────────┼──────────────────────┤
│ Best For             │ Primary key      │ Secondary columns    │
│                      │ Range queries    │ Frequent searches    │
└──────────────────────┴──────────────────┴──────────────────────┘
```

---

## 5. Why Avoid Too Many Indexes?

### Storage and Memory Costs

```
┌─────────────────────────────────────────────────────┐
│ STORAGE IMPACT OF INDEXES                           │
├─────────────────────────────────────────────────────┤
│ Table: employees (1 million rows)                   │
│ Row size: 64 bytes                                  │
│                                                     │
│ Base table size: 1M × 64 = 64 MB                    │
│                                                     │
│ + Clustered index (emp_id): ~64 MB                  │
│ + Non-clustered index (name): ~64 MB                │
│ + Non-clustered index (email): ~64 MB               │
│ + Non-clustered index (dept): ~64 MB                │
│ + Non-clustered index (salary): ~64 MB              │
│                                                     │
│ TOTAL: 64 + 64 + 64 + 64 + 64 + 64 = 384 MB        │
│                                                     │
│ OVERHEAD: 6x the original table size!               │
│ MEMORY: All indexes must fit in buffer pool         │
└─────────────────────────────────────────────────────┘
```

### Update Overhead

```
┌─────────────────────────────────────────────────────┐
│ COST OF INSERTING ONE ROW                           │
├─────────────────────────────────────────────────────┤
│ INSERT INTO employees VALUES (1001, 'Frank', ...);  │
│                                                     │
│ OPERATIONS REQUIRED:                                │
│ 1. Insert into data page                            │
│ 2. Update clustered index (emp_id)                  │
│ 3. Update non-clustered index (name)                │
│ 4. Update non-clustered index (email)               │
│ 5. Update non-clustered index (dept)                │
│ 6. Update non-clustered index (salary)              │
│                                                     │
│ TOTAL: 6 B+ Tree updates!                           │
│                                                     │
│ WITH 5 INDEXES:                                     │
│ • 1 INSERT becomes 6 operations                     │
│ • 1 million inserts = 6 million operations          │
│ • Time: 10x slower than no indexes                  │
└─────────────────────────────────────────────────────┘
```

### Practical Indexing Guidelines

```
┌─────────────────────────────────────────────────────┐
│ WHEN TO CREATE INDEXES                              │
├─────────────────────────────────────────────────────┤
│ ✅ DO INDEX:                                         │
│ • Primary key (automatic)                           │
│ • Foreign keys (for joins)                          │
│ • Columns in WHERE clause (frequently)              │
│ • Columns in ORDER BY                               │
│ • Columns in GROUP BY                               │
│ • High selectivity columns (unique values)          │
│                                                     │
│ ❌ DON'T INDEX:                                      │
│ • Small tables (< 1000 rows)                        │
│ • Low selectivity (gender, boolean, status)         │
│ • Frequently updated columns                        │
│ • Columns rarely in queries                         │
│ • Columns with many NULL values                     │
│                                                     │
│ ⚠️ INDEX SPARINGLY:                                  │
│ • Aim for 3-5 indexes per table                     │
│ • Monitor unused indexes                            │
│ • Remove indexes not used in 30 days                │
└─────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE HIERARCHY                             │
├─────────────────────────────────────────────────────────────────┤
│ Logical View (User)                                              │
│ ↓                                                                │
│ Tables with rows and columns                                    │
│ ↓                                                                │
│ Physical View (DBMS)                                             │
│ ↓                                                                │
│ Data Pages (8KB) with offset arrays                              │
│ ↓                                                                │
│ Data Blocks (4KB) on disk                                        │
│ ↓                                                                │
│ Disk Storage                                                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    INDEXING MECHANISM                            │
├─────────────────────────────────────────────────────────────────┤
│ 1. B+ Tree navigates to correct data page (O(log n))            │
│ 2. Offset array maps logical to physical row order              │
│ 3. Clustered index: Determines physical row order               │
│ 4. Non-clustered: Separate B+ Tree with pointers                │
│ 5. Each index adds storage and update overhead                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    KEY TAKEAWAYS                                 │
├─────────────────────────────────────────────────────────────────┤
│ • Index reduces search from O(n) to O(log n)                    │
│ • Data pages = 8KB with header, data, offset array              │
│ • Offset array enables logical ordering                         │
│ • Clustered index: Physical row order (1 per table)             │
│ • Non-clustered: Separate B+ Tree (multiple allowed)            │
│ • Index sparingly: 3-5 per table is optimal                     │
│ • Monitor storage and update overhead                           │
└─────────────────────────────────────────────────────────────────┘
```

---

**Estimated Reading Time**: 40-45 minutes  
**Mastery Level**: Ready for senior database internals interviews

*[← Back to Chapter 4: Advanced Relationships](../Chapter-4-Advanced-Relationships/) | [Continue to Chapter 6: Transactions & Concurrency →](../Chapter-6-Transactions-Concurrency/)*

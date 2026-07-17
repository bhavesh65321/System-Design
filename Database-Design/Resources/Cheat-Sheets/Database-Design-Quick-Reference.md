# Database Design Quick Reference

> Essential concepts for interviews and daily work

## 🔑 Key Types Cheat Sheet

| Key Type | Purpose | Example | Notes |
|----------|---------|---------|-------|
| **Primary Key** | Unique row identifier | `user_id BIGINT PRIMARY KEY` | One per table, never NULL |
| **Foreign Key** | References another table | `FOREIGN KEY (user_id) REFERENCES users(user_id)` | Maintains relationships |
| **Unique Key** | Prevents duplicates | `email VARCHAR(255) UNIQUE` | Multiple per table allowed |
| **Composite Key** | Multiple columns as key | `PRIMARY KEY (order_id, product_id)` | Common in junction tables |

## 📊 Normal Forms Summary

### 1NF (First Normal Form)
- ✅ **Atomic values**: Each cell has single value
- ✅ **No repeating groups**: No comma-separated lists
- ❌ **Bad**: `colors: "red,blue,green"`
- ✅ **Good**: Separate rows for each color

### 2NF (Second Normal Form) 
- ✅ **Must be in 1NF**
- ✅ **No partial dependencies**: All non-key columns depend on entire primary key
- 📝 **Only applies to tables with composite primary keys**

### 3NF (Third Normal Form)
- ✅ **Must be in 2NF** 
- ✅ **No transitive dependencies**: Non-key columns don't depend on other non-key columns
- 📝 **Most common target for business applications**

## 🔗 Relationship Types

### One-to-One (1:1)
```sql
-- User has one profile
CREATE TABLE users (user_id INT PRIMARY KEY, email VARCHAR(255));
CREATE TABLE profiles (user_id INT PRIMARY KEY, bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id));
```

### One-to-Many (1:N)  
```sql
-- Customer has many orders
CREATE TABLE customers (customer_id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE orders (order_id INT PRIMARY KEY, customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id));
```

### Many-to-Many (M:N)
```sql
-- Students take multiple courses, courses have multiple students
CREATE TABLE students (student_id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE courses (course_id INT PRIMARY KEY, title VARCHAR(200));
CREATE TABLE enrollments (
    student_id INT, course_id INT,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

## ⚖️ Constraints Quick Reference

| Constraint | Purpose | Example |
|------------|---------|---------|
| `NOT NULL` | Require value | `name VARCHAR(100) NOT NULL` |
| `UNIQUE` | Prevent duplicates | `email VARCHAR(255) UNIQUE` |
| `CHECK` | Validate data | `CHECK (price > 0)` |
| `DEFAULT` | Set default value | `status VARCHAR(20) DEFAULT 'active'` |
| `FOREIGN KEY` | Reference integrity | `FOREIGN KEY (user_id) REFERENCES users(user_id)` |

## 🚀 Performance Tips

### Indexing Strategy
```sql
-- Primary keys automatically indexed
-- Add indexes for frequent WHERE clauses
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);

-- Composite index for multi-column queries
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

### Data Type Selection
| Use Case | Recommended Type | Why |
|----------|------------------|-----|
| **Primary Keys** | `BIGINT AUTO_INCREMENT` | Handles billions of records |
| **Prices** | `DECIMAL(10,2)` | Exact precision for money |
| **Dates** | `DATE` or `TIMESTAMP` | Built-in date functions |
| **Status** | `ENUM('active','inactive')` | Restricts to valid values |
| **Text** | `VARCHAR(n)` for limited, `TEXT` for unlimited | Performance vs flexibility |

## 🎤 Interview Quick Answers

### "What is database normalization?"
"Normalization is the process of organizing data to eliminate redundancy and ensure data integrity. 1NF requires atomic values, 2NF eliminates partial dependencies, and 3NF eliminates transitive dependencies."

### "When would you denormalize?"
"For read-heavy applications where query performance is critical. Examples include storing calculated values like user post counts or product ratings to avoid complex joins."

### "Explain ACID properties"
- **Atomicity**: All or nothing transactions
- **Consistency**: Database remains valid after transactions  
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed changes persist permanently

## 🔧 Common Patterns

### Audit Columns
```sql
-- Add to every table for tracking
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
created_by BIGINT,
updated_by BIGINT
```

### Soft Delete
```sql
-- Instead of DELETE, mark as deleted
deleted_at TIMESTAMP NULL,
is_deleted BOOLEAN DEFAULT FALSE

-- Queries exclude soft-deleted records
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Status Tracking
```sql
-- Use enums for limited, known values
status ENUM('draft', 'published', 'archived') DEFAULT 'draft',
-- Or separate status table for complex workflows
status_id INT REFERENCES statuses(status_id)
```

---

**💡 Pro Tip**: Always design for the queries you'll run most frequently, not just for perfect normalization.

*Part of Database Design Master Course*
# Chapter 1: Database Keys

> **Master the Foundation**: Understanding keys is the cornerstone of professional database design

## 🎯 Learning Objectives

By the end of this chapter, you'll be able to:
- [ ] Explain why keys are essential for data integrity in interview-ready terms
- [ ] Identify and implement all 8 types of database keys correctly
- [ ] Choose appropriate key strategies for production systems
- [ ] Handle advanced scenarios like billion-user platforms
- [ ] Answer key-related questions from junior to staff engineer level

---

## 🤔 Why Do We Need Keys? (The Fundamental Problem)

### Real-World Scenario: Customer Management Crisis

**Imagine you're building an e-commerce platform with this design:**

```sql
-- PROBLEMATIC: Table without proper identification
CREATE TABLE customers (
    name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(15),
    city VARCHAR(100)
);

-- Sample data that creates chaos:
INSERT INTO customers VALUES 
('John Smith', 'john@gmail.com', '9999999999', 'Mumbai'),
('John Smith', 'johnsmith@yahoo.com', '8888888888', 'Delhi'),
('Jane Doe', 'jane@gmail.com', '7777777777', 'Pune');
```

### The Business Problems This Creates:

**Problem 1: Update Confusion** 
- Boss: "Update John Smith's phone number to 5555555555"
- You: "Which John Smith?" 
- Result: Wrong customer gets updated → Angry customer calls

**Problem 2: Delete Disaster**
- Boss: "Remove the John Smith from Delhi" 
- You accidentally delete John from Mumbai
- Result: Lost customer data → Revenue loss

**Problem 3: Business Logic Failure**
- Can't track individual customer order history
- Can't send personalized emails (which John gets what?)
- Reports show incorrect customer counts
- Foreign keys impossible to implement

**Problem 4: Data Integrity Collapse**
- No way to ensure data consistency
- Duplicate customer records pile up
- Customer service can't help users
- System becomes unreliable

### The Solution: Database Keys

> **Definition**: A key is one or more columns that uniquely identify a row in a table or establish relationships between tables, ensuring data integrity and enabling efficient data management.

**Keys solve the fundamental challenge**: **"How do we uniquely and reliably identify each piece of data?"**

---

## 🔑 The Complete Guide to Database Keys

### 1. Primary Key - The Chosen Identifier

**Definition**: The most appropriate key chosen to uniquely identify each row in a table. It cannot be NULL, cannot have duplicates, and serves as the main reference point for relationships.

**Complete Rules**:
- **UNIQUE** - No two rows can share the same primary key value
- **NOT NULL** - Every row must have a primary key value  
- **IMMUTABLE** - Should never change once assigned
- **ONE PER TABLE** - Each table has exactly one primary key
- **REFERENCED** - Other tables use this for foreign key relationships

**Production Example**:
```sql
-- Professional customer table design
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Surrogate Primary Key
    email VARCHAR(255) UNIQUE NOT NULL,            -- Natural Alternate Key
    phone VARCHAR(15) UNIQUE,                      -- Natural Alternate Key
    username VARCHAR(50) UNIQUE NOT NULL,          -- Natural Alternate Key
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Why This Design Works**:
- `customer_id` uniquely identifies each customer (never changes)
- System-generated, so guaranteed unique
- Efficient for joins and indexing (8-byte integer)
- Business keys (email, phone) preserved as alternate keys
- Audit trail with timestamps

### 2. Foreign Key - The Relationship Enforcer

**Definition**: An attribute or set of attributes in one table that references the primary key of another table (or the same table) to establish relationships and ensure referential integrity.

**Referential Integrity Explained**:
> **Referential Integrity** ensures that relationships between tables remain valid and consistent. If a foreign key points to something, that "something" must actually exist!

**Real-World Analogy**: Like address validation - if your delivery address says "House #123, ABC Street", that street and house number must actually exist!

**Production Example**:
```sql
-- Parent table
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Child table with Foreign Key
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,           -- Foreign Key
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'confirmed', 'shipped', 'delivered') DEFAULT 'pending',
    
    -- Foreign Key with business rules
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT                 -- Prevent deletion if orders exist
        ON UPDATE CASCADE                  -- Update orders if customer_id changes
);
```

**Foreign Key Actions Explained**:
- **CASCADE**: Automatically delete/update related records
- **RESTRICT**: Prevent action if related records exist (default, safest)
- **SET NULL**: Set foreign key to NULL (if column allows NULL)
- **SET DEFAULT**: Set foreign key to default value

**What Referential Integrity Prevents**:
```sql
-- This will FAIL - customer 999 doesn't exist
INSERT INTO orders (order_id, customer_id, total_amount) 
VALUES (101, 999, 250.00);
-- ❌ ERROR: Cannot add - referential integrity violation!

-- This will FAIL - customer 1 has orders
DELETE FROM customers WHERE customer_id = 1;
-- ❌ ERROR: Cannot delete - foreign key constraint fails!
```

### 3. Unique Key - The Duplicate Preventer

**Definition**: Ensures no duplicate values in a column or combination of columns, but unlike Primary Key, allows NULL values and multiple Unique keys per table.

**Key Differences from Primary Key**:
- Can have **multiple Unique keys** per table
- Can contain **NULL values** (only one NULL allowed per column)
- Not automatically the **clustered index**
- Used for **alternate identification methods**

**Example**:
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,     -- Primary Key
    username VARCHAR(50) UNIQUE NOT NULL,          -- Unique Key 1
    email VARCHAR(255) UNIQUE NOT NULL,            -- Unique Key 2
    phone VARCHAR(15) UNIQUE,                      -- Unique Key 3 (can be NULL)
    ssn VARCHAR(11) UNIQUE,                        -- Unique Key 4 (can be NULL)
    name VARCHAR(100) NOT NULL
);
```

**Business Value**: Users can login with username OR email OR phone - all guaranteed unique!

### 4. Composite Key - When Multiple Columns Unite

**Definition**: A Primary Key composed of multiple columns, used when no single column can uniquely identify a row.

**When to Use**: 
- Junction tables (M:N relationships)
- Time-series data with natural groupings
- When business logic requires combination uniqueness

**Classic Example - Order Items**:
```sql
CREATE TABLE order_items (
    order_id BIGINT,                               -- Part 1 of composite key
    product_id BIGINT,                            -- Part 2 of composite key
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(10,2) NOT NULL,
    
    PRIMARY KEY (order_id, product_id),           -- Composite Primary Key
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT
);
```

**Why This Works**:
- Neither `order_id` nor `product_id` alone is unique in this table
- The combination `(order_id, product_id)` is guaranteed unique
- Prevents duplicate entries: same product can't be added twice to same order
- Natural representation of the business relationship

**Advanced Example - Social Media Platform**:
```sql
-- User follows relationship
CREATE TABLE follows (
    follower_id BIGINT,                           -- Who is following
    following_id BIGINT,                          -- Who is being followed
    followed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notification_enabled BOOLEAN DEFAULT TRUE,
    
    PRIMARY KEY (follower_id, following_id),      -- Composite Key
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- Business rule: prevent self-follows
    CHECK (follower_id != following_id)
);
```

### 5. Super Key - The Foundation Concept

**Definition**: Any set of one or more attributes that can uniquely identify each row in a table (may contain extra/unnecessary attributes).

**Key Insight**: Super Key = Any combination that **guarantees uniqueness** (even with extra columns)

**Example Analysis**:
```sql
CREATE TABLE employees (
    emp_id INT,
    ssn VARCHAR(11),
    email VARCHAR(255), 
    phone VARCHAR(15),
    name VARCHAR(100)
);
```

**Super Keys in this table**:
- `{emp_id}` ✅ Super Key (minimal)
- `{ssn}` ✅ Super Key (minimal)
- `{email}` ✅ Super Key (minimal)
- `{emp_id, name}` ✅ Super Key (emp_id alone works, name is extra)
- `{ssn, email, phone}` ✅ Super Key (all are unique individually)
- `{emp_id, ssn, email, phone, name}` ✅ Super Key (all attributes)
- `{name}` ❌ NOT Super Key (names can repeat)

**Mental Model**: Think of Super Key as "any key that works for identification, even if it's overkill"

### 6. Candidate Key - The Minimal Contenders

**Definition**: A Super Key with no unnecessary attributes - the "minimal" Super Keys that could potentially be chosen as Primary Key.

**Rule**: **Candidate Key = Super Key with NO extra columns**

**From the employee example above**:
- **Candidate Keys**: `{emp_id}`, `{ssn}`, `{email}` 
- **NOT Candidate Keys**: `{emp_id, name}` (name is extra), `{ssn, email}` (both are individually unique)

**Why This Matters**: You choose your Primary Key from the Candidate Keys!

### 7. Alternate Key - The Roads Not Taken

**Definition**: Any Candidate Key that was NOT chosen as the Primary Key.

**Implementation**: Usually implemented as UNIQUE constraints.

**Strategic Example**:
```sql
CREATE TABLE products (
    product_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Chosen Primary Key
    sku VARCHAR(50) UNIQUE NOT NULL,               -- Alternate Key (business)
    barcode VARCHAR(50) UNIQUE,                    -- Alternate Key (global)
    upc VARCHAR(12) UNIQUE,                        -- Alternate Key (retail)
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);
```

**Business Value**:
- Customers search by SKU: `SELECT * FROM products WHERE sku = 'IPHONE15-128-BLK'`
- Barcode scanners use barcode: `SELECT * FROM products WHERE barcode = '123456789012'`
- Point-of-sale systems use UPC: `SELECT * FROM products WHERE upc = '012345678905'`

### 8. Natural vs Surrogate Keys - The Great Debate

#### Natural Key
**Definition**: A key that has business meaning and exists naturally in the data.

**Examples**: 
- Social Security Number
- ISBN for books  
- Email address
- License plate number
- Product SKU

#### Surrogate Key
**Definition**: An artificial key created specifically for the database with no business meaning.

**Examples**:
- Auto-incrementing integers (1, 2, 3, 4...)
- UUIDs (550e8400-e29b-41d4-a716-446655440000)

#### The Professional Decision Framework

```sql
-- Industry Best Practice: Hybrid Approach
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Surrogate Key (stability)
    email VARCHAR(255) UNIQUE NOT NULL,             -- Natural Key (business value)
    ssn VARCHAR(11) UNIQUE,                         -- Natural Key (government ID)
    phone VARCHAR(15) UNIQUE,                       -- Natural Key (contact)
    name VARCHAR(100) NOT NULL
);
```

#### Comparison Matrix

| Aspect | Natural Key | Surrogate Key |
|--------|-------------|---------------|
| **Business Meaning** | ✅ Meaningful to users | ❌ No meaning |
| **Stability** | ❌ Can change | ✅ Never changes |
| **Performance** | ❌ Variable (strings) | ✅ Optimized (integers) |
| **Size** | ❌ Variable/Large | ✅ Fixed/Small |
| **Global Uniqueness** | ❌ Limited scope | ✅ Can be globally unique |
| **Debugging** | ✅ Human readable | ❌ Abstract numbers |

#### Industry Recommendation: BIGINT vs UUID

**BIGINT Auto-increment** (Most Common):
```sql
user_id BIGINT AUTO_INCREMENT PRIMARY KEY
```
**Pros**: Fastest performance, smallest storage, sequential, cache-friendly
**Cons**: Not globally unique, reveals business information
**Use When**: Single database, performance critical, traditional architecture

**UUID** (Distributed Systems):
```sql
user_id CHAR(36) PRIMARY KEY DEFAULT (UUID())
```
**Pros**: Globally unique, no coordination needed, distributed-friendly
**Cons**: Larger storage, random access, slower joins
**Use When**: Microservices, distributed systems, global uniqueness required

---

## 🚀 Advanced Production Scenarios

### Senior Engineer Challenge: Social Media Platform (1 Billion Users)

**Business Requirements**:
- 1 billion users (Facebook/Instagram scale)
- Real-time messaging across global data centers
- Social features (follows, likes, comments)
- High availability (99.99% uptime)
- Global distribution

**Professional Key Strategy**:

#### Core Entities: BIGINT for Performance
```sql
-- Users table optimized for scale
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,     -- BIGINT for performance
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Sharding key for horizontal scaling
    INDEX idx_user_shard (user_id)
);

-- Posts table with co-location strategy  
CREATE TABLE posts (
    post_id BIGINT AUTO_INCREMENT PRIMARY KEY,     -- BIGINT for performance
    user_id BIGINT NOT NULL,                       -- Co-locate with user shard
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    -- Compound index for efficient user post queries
    INDEX idx_user_posts (user_id, created_at DESC)
);
```

**Why BIGINT for Core Entities?**
- **Performance**: 8 bytes vs 36 bytes (UUID) = 4.5x storage savings
- **Join Speed**: Integer joins 3-5x faster than string joins  
- **Cache Efficiency**: More IDs fit in CPU cache
- **Scale Math**: BIGINT max = 9.2 quintillion (handles 1 billion users + 1000 posts each easily)

#### Distributed Features: UUID for Global Coordination
```sql
-- Messages table for real-time cross-datacenter communication
CREATE TABLE messages (
    message_id UUID PRIMARY KEY DEFAULT (UUID()),  -- UUID for global uniqueness
    sender_id BIGINT NOT NULL,
    receiver_id BIGINT NOT NULL,
    content TEXT,
    sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (sender_id) REFERENCES users(user_id),
    FOREIGN KEY (receiver_id) REFERENCES users(user_id),
    
    -- Optimize for sender and receiver queries
    INDEX idx_sender_messages (sender_id, sent_at DESC),
    INDEX idx_receiver_messages (receiver_id, sent_at DESC)
);

-- Notifications for real-time updates
CREATE TABLE notifications (
    notification_id UUID PRIMARY KEY DEFAULT (UUID()),
    user_id BIGINT NOT NULL,
    type ENUM('like', 'comment', 'follow', 'message'),
    reference_id BIGINT,                           -- Points to post, comment, etc.
    content JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    read_at TIMESTAMP NULL,
    
    INDEX idx_user_notifications (user_id, created_at DESC, read_at)
);
```

**Why UUID for Distributed Features?**
- **No Coordination**: Each datacenter generates UUIDs independently
- **Conflict-Free**: Impossible for collisions across regions
- **Real-time**: No database round-trip for ID generation
- **Merge-Friendly**: Easy to replicate across datacenters

#### Relationships: Composite Keys for Efficiency
```sql
-- Follows relationship (M:N with composite key)
CREATE TABLE follows (
    follower_id BIGINT,
    following_id BIGINT,
    followed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notification_enabled BOOLEAN DEFAULT TRUE,
    
    PRIMARY KEY (follower_id, following_id),       -- Natural composite key
    FOREIGN KEY (follower_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(user_id) ON DELETE CASCADE,
    
    -- Business constraint
    CHECK (follower_id != following_id),
    
    -- Performance indexes
    INDEX idx_following_list (following_id, followed_at DESC)  -- "Who follows me?"
);

-- Likes relationship optimized for social media patterns
CREATE TABLE likes (
    user_id BIGINT,
    post_id BIGINT,
    liked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (user_id, post_id),                -- Prevents duplicate likes
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
    
    -- Performance indexes for social features
    INDEX idx_post_likes (post_id, liked_at DESC), -- "Who liked this post?"
    INDEX idx_user_likes (user_id, liked_at DESC)  -- "What did I like?"
);
```

#### Horizontal Scaling Strategy
```sql
-- Sharding function
-- Shard users by user_id: user_id % 1000 determines shard
-- Co-locate user's content on same shard

-- Modified posts table for sharding
CREATE TABLE posts (
    post_id BIGINT,
    user_id BIGINT,                                -- Determines shard placement
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Compound primary key for efficient sharding
    PRIMARY KEY (user_id, post_id),                -- user_id first for shard routing
    
    -- Global post lookup when needed
    UNIQUE KEY uk_post_global (post_id)
);
```

**Scaling Phases**:
1. **Phase 1** (0-1M users): Single database
2. **Phase 2** (1M-10M users): Master-slave replication  
3. **Phase 3** (10M-100M users): Functional sharding (users, posts, messages)
4. **Phase 4** (100M+ users): Horizontal sharding within each function

---

## ⚡ Performance Optimization Strategies

### Indexing Strategy for Keys
```sql
-- Primary keys automatically get clustered index
-- Add strategic secondary indexes for foreign keys

CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date DESC);
CREATE INDEX idx_orders_status_date ON orders(status, order_date DESC);
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Covering indexes for frequent queries
CREATE INDEX idx_orders_summary 
ON orders(customer_id, status) 
INCLUDE (order_date, total_amount);  -- Covers entire query
```

### Query Performance Impact Analysis
```sql
-- FAST: Uses primary key (clustered index) - O(1) to O(log n)
SELECT * FROM customers WHERE customer_id = 12345;

-- FAST: Uses foreign key index - O(log n)  
SELECT * FROM orders WHERE customer_id = 789 ORDER BY order_date DESC;

-- SLOW: Full table scan - O(n)
SELECT * FROM customers WHERE first_name = 'John';  -- No index on first_name

-- OPTIMIZED: Add index for frequent searches
CREATE INDEX idx_customers_name ON customers(first_name, last_name);
```

---

## 🚫 Common Mistakes & Professional Solutions

### Mistake 1: Using Business Data as Primary Key
```sql
-- ❌ WRONG: Email can change, breaking referential integrity
CREATE TABLE customers (
    email VARCHAR(255) PRIMARY KEY,
    name VARCHAR(100)
);

-- What happens when email changes?
-- Must update ALL related tables - expensive and error-prone!
```

```sql
-- ✅ RIGHT: Surrogate key + business key as alternate
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Stable surrogate key
    email VARCHAR(255) UNIQUE NOT NULL,            -- Business alternate key
    name VARCHAR(100) NOT NULL
);

-- Email changes? Only one row update needed!
UPDATE customers SET email = 'newemail@domain.com' WHERE customer_id = 123;
```

### Mistake 2: Missing Foreign Key Constraints
```sql
-- ❌ WRONG: No referential integrity
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT  -- No FK constraint = orphaned orders possible!
);
```

```sql
-- ✅ RIGHT: Proper foreign key with business rules
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT                         -- Prevent orphaned orders
        ON UPDATE CASCADE,                         -- Handle ID updates
        
    -- Named constraint for better error messages
    CONSTRAINT fk_orders_customer 
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### Mistake 3: Composite Keys with Too Many Columns
```sql
-- ❌ WRONG: Overly complex composite key
CREATE TABLE addresses (
    country VARCHAR(50),
    state VARCHAR(50), 
    city VARCHAR(50),
    street VARCHAR(100),
    building_number VARCHAR(10),
    apartment VARCHAR(10),
    
    PRIMARY KEY (country, state, city, street, building_number, apartment) -- Too complex!
);
```

```sql
-- ✅ RIGHT: Surrogate key for complex entities
CREATE TABLE addresses (
    address_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Simple, efficient
    country VARCHAR(50) NOT NULL,
    state VARCHAR(50) NOT NULL,
    city VARCHAR(50) NOT NULL,
    street VARCHAR(100) NOT NULL,
    building_number VARCHAR(10),
    apartment VARCHAR(10),
    
    -- Add unique constraint only if business requires it
    UNIQUE KEY uk_full_address (country, state, city, street, building_number, apartment)
);
```

---

## 🎤 Interview Questions by Level

### Junior Level (0-2 years)

**Q1**: "What's the difference between Primary Key and Unique Key?"
**A**: "Primary Key uniquely identifies each row, cannot be NULL, and there's only one per table. It automatically creates a clustered index. Unique Key prevents duplicates but can have NULL values (only one NULL per column), allows multiple per table, and creates a non-clustered index."

**Q2**: "Why can't Primary Keys be NULL?"
**A**: "Primary Keys must uniquely identify each row. If NULL were allowed, you couldn't distinguish between rows with NULL primary keys, violating the uniqueness requirement and making referential integrity impossible."

**Q3**: "What is a Foreign Key?"
**A**: "A Foreign Key is a column that references the Primary Key of another table to establish relationships and maintain referential integrity. It ensures that related data remains consistent - you can't have orphaned records pointing to non-existent data."

### Intermediate Level (2-5 years)

**Q4**: "Explain the difference between Natural and Surrogate keys with an example."
**A**: "Natural keys have business meaning (like SSN or email), while Surrogate keys are artificial (like auto-increment IDs). I prefer surrogate keys as primary keys because they're stable - if a customer changes their email, I only update one row instead of cascading changes throughout related tables. I keep natural keys as unique alternate keys for business functionality."

**Q5**: "When would you use a Composite Primary Key?"
**A**: "For junction tables in many-to-many relationships, like OrderItems with PRIMARY KEY (order_id, product_id). This prevents duplicate entries and naturally represents the relationship. I avoid composite keys for regular entities because they're harder to reference and less performant for joins."

**Q6**: "How do Foreign Key constraints help with data integrity?"
**A**: "Foreign Keys prevent orphaned records and maintain referential integrity. They stop you from inserting orders for non-existent customers (INSERT prevention) and deleting customers who have orders (DELETE prevention). I use CASCADE for dependent data and RESTRICT for important business entities."

### Senior Level (5+ years)

**Q7**: "Design a key strategy for a multi-tenant SaaS application with millions of records per tenant."
**A**: "I'd use a compound key strategy: tenant_id + entity_id as the primary key for row-level security and efficient sharding. Each tenant's data co-locates on the same shard. For global entities like users, I'd use UUIDs to avoid cross-tenant ID collisions. I'd implement row-level security policies to ensure tenant isolation."

**Q8**: "When would you choose UUID over BIGINT for primary keys?"
**A**: "UUIDs for distributed systems where you need globally unique IDs without coordination - microservices, offline-capable apps, or multi-datacenter setups. BIGINT for single-database systems where performance is critical. The trade-off is storage size (36 vs 8 bytes) and join performance (3-5x slower) against global uniqueness and distribution friendliness."

### Staff/Principal Level (10+ years)

**Q9**: "Design primary key strategy for a social media platform with 1 billion users."
**A**: "Hybrid approach: BIGINT for core entities (users, posts) optimized for performance and cache efficiency. UUIDs for distributed features (messages, notifications) to enable cross-datacenter replication. Composite keys for relationships (follows, likes) to prevent duplicates and optimize social queries. Implement horizontal sharding with user_id as shard key, co-locating user content. Plan migration paths for scaling phases."

**Q10**: "How would you handle primary key strategy migration in a system processing 100K transactions per second?"
**A**: "Zero-downtime migration using shadow tables and dual-write pattern. Phase 1: Add new UUID column, populate via background job. Phase 2: Dual-write to both keys. Phase 3: Update application to read from UUID, verify consistency. Phase 4: Switch writes to UUID primary key. Phase 5: Remove old key after full migration. Use feature flags for rollback capability and monitor performance throughout."

---

## 🧪 Practice Exercises

### Exercise 1: Key Identification Challenge
Analyze this table and identify ALL key types:
```sql
CREATE TABLE employees (
    emp_id INT AUTO_INCREMENT PRIMARY KEY,
    employee_number VARCHAR(10) UNIQUE NOT NULL,
    ssn VARCHAR(11) UNIQUE,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(15),
    department_id INT,
    manager_id INT,
    name VARCHAR(100) NOT NULL,
    
    FOREIGN KEY (department_id) REFERENCES departments(dept_id),
    FOREIGN KEY (manager_id) REFERENCES employees(emp_id)
);
```

**Your Task**: List all Super Keys, Candidate Keys, Primary Key, Alternate Keys, Natural Keys, Surrogate Keys, and Foreign Keys.

### Exercise 2: Design Challenge - Library Management System
**Requirements**:
- Books can have multiple authors, authors can write multiple books
- Members can borrow multiple books, books can be borrowed by multiple members (over time)
- Track loan history with dates and return status
- Handle book copies (multiple copies of same book)

**Your Task**: Design complete schema with appropriate key strategy for each table.

### Exercise 3: Performance Analysis
Compare these two designs for an order system handling 1M+ orders:

**Design A**: Natural Key Approach
```sql
CREATE TABLE orders (
    order_number VARCHAR(20) PRIMARY KEY,
    customer_email VARCHAR(255) NOT NULL,
    order_date DATE,
    total DECIMAL(10,2)
);
```

**Design B**: Surrogate Key Approach  
```sql
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id BIGINT NOT NULL,
    order_date DATE,
    total DECIMAL(10,2),
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

**Your Analysis**: Compare performance, maintainability, and scalability aspects.

---

## 🎯 Quick Quiz

**1.** Which statement about Primary Keys is FALSE?
a) Can be NULL in special cases
b) Must be unique across all rows  
c) Only one per table allowed
d) Automatically creates an index

**2.** What happens with `ON DELETE CASCADE`?
a) Prevents deletion of referenced row
b) Sets foreign key to NULL
c) Automatically deletes related child rows
d) Raises an error message

**3.** Which is the best Primary Key strategy for a distributed microservices architecture?
a) Natural keys (email addresses)
b) BIGINT auto-increment  
c) UUID
d) Composite keys only

**4.** What makes a Super Key different from a Candidate Key?
a) Super Keys can be NULL
b) Super Keys may contain unnecessary columns
c) Super Keys are always composite
d) Super Keys cannot be Primary Keys

**5.** In a junction table for Many-to-Many relationships, what's the recommended Primary Key strategy?
a) Single surrogate key column
b) Composite key from both foreign keys
c) UUID for global uniqueness
d) Natural key from business data

**Answers**: 1-a, 2-c, 3-c, 4-b, 5-b

---

## 📋 Chapter Summary

### Core Concepts Mastered
1. **Primary Keys** - Chosen unique identifiers that anchor table relationships
2. **Foreign Keys** - Relationship enforcers that maintain referential integrity  
3. **Unique Keys** - Duplicate preventers for alternate identification methods
4. **Composite Keys** - Multi-column keys for natural relationships
5. **Key Hierarchies** - Super Key → Candidate Key → Primary Key progression
6. **Natural vs Surrogate** - Business meaningful vs system-generated keys

### Professional Decision Framework
- **Use surrogate keys** (BIGINT/UUID) as Primary Keys for stability
- **Keep natural keys** as Alternate Keys (UNIQUE constraints) for business value
- **Choose BIGINT** for performance, **UUID** for distribution
- **Use composite keys** only for junction tables and natural relationships
- **Always implement** Foreign Key constraints for data integrity
- **Plan for scale** from day one with appropriate key strategies

### Performance Guidelines
- Primary Keys create clustered indexes automatically
- Add secondary indexes for all Foreign Keys
- BIGINT keys outperform UUID keys by 3-5x in joins
- Composite keys should be ordered by selectivity (most selective first)
- Consider covering indexes for frequently accessed key combinations

### Next Chapter Preview
**Chapter 2: Three-Level Architecture** - Understanding how databases separate conceptual design from physical implementation, enabling the key strategies we've learned to work efficiently at scale.

---

*[← Back to Module 2 Overview](../README.md) | [Continue to Chapter 2: Three-Level Architecture →](../Chapter-2-Three-Level-Architecture/)*
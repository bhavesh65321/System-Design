# Module 2: Relational Database Design

> **Goal**: Learn how to design relational databases like a Senior Software Engineer

## 🎯 Learning Objectives

By the end of this module, you'll be able to:
- Design production-ready relational databases
- Choose the correct keys and understand their importance
- Apply normalization principles
- Convert ER diagrams into SQL tables
- Explain your design decisions in interviews
- Build a complete E-commerce database schema

## 📋 Module Roadmap

| Chapter | Topic | Importance | Status |
|---------|-------|------------|--------|
| 1 | Keys | ⭐⭐⭐⭐⭐ | ✅ |
| 2 | Three-Level Architecture | ⭐⭐⭐⭐⭐ | ✅ |
| 3 | Relationships Deep Dive | ⭐⭐⭐⭐⭐ | 📝 |
| 4 | Normalization | ⭐⭐⭐⭐⭐ | 📝 |
| 5 | Data Types | ⭐⭐⭐⭐ | 📝 |
| 6 | Constraints | ⭐⭐⭐⭐ | 📝 |
| 7 | Naming Conventions | ⭐⭐⭐ | 📝 |
| 8 | ERD → SQL Tables | ⭐⭐⭐⭐⭐ | 📝 |
| 9 | Production Design Review | ⭐⭐⭐⭐⭐ | 📝 |

---

# Chapter 1: Database Keys

## 🤔 Why Do We Need Keys?

**Problem**: Imagine you have a User table:

| User |
|------|
| John |
| Rahul |
| Amit |
| John |
| Priya |

**Questions**:
1. How do we identify John? (There are two Johns)
2. HR says: "Update John's phone number" - Which John?
3. How do we ensure each user is unique?

**Answer**: We need a **unique identifier** - this is why keys exist.

## 🔑 What is a Key?

> **Definition**: A Key is one or more columns that uniquely identify a row in a table or establish relationships between tables.

Think of a key as a person's Aadhaar number:
- Many people can have the same name
- Only one Aadhaar number belongs to one person

Similarly:
- Multiple users can have the same name
- Each user has a unique ID

## 📊 Types of Keys

```
Keys
│
├── Super Key
├── Candidate Key  
├── Primary Key
├── Alternate Key
├── Composite Key
├── Foreign Key
├── Natural Key
└── Surrogate Key
```

---

## 1️⃣ Super Key

### Definition
> A **Super Key** is any set of one or more columns that can uniquely identify a row.

**Key Point**: It may contain extra columns that aren't necessary for uniqueness.

### Example
Consider the User table:

| user_id | email | phone | name |
|---------|-------|-------|------|
| 101 | john@gmail.com | 9991111111 | John |
| 102 | amit@gmail.com | 9992222222 | Amit |
| 103 | priya@gmail.com | 9993333333 | Priya |

**Assumptions**: 
- `user_id` is unique
- `email` is unique  
- `phone` is unique

### Super Key Examples:

✅ **Valid Super Keys**:
- `(user_id)` - Can identify one user uniquely
- `(email)` - Can identify one user uniquely
- `(phone)` - Can identify one user uniquely
- `(user_id, name)` - Still unique because user_id alone is unique
- `(user_id, email)` - Still unique
- `(user_id, email, phone, name)` - Still unique

❌ **Not Super Keys**:
- `(name)` - Names can repeat (two Johns exist)

### Mental Model
Like identifying a student:
- Roll Number ✅
- Roll Number + Name ✅
- Roll Number + Name + Class ✅

The Roll Number alone is enough, everything else is extra. All are Super Keys.

---

## 2️⃣ Candidate Key

### Definition
> A **Candidate Key** is a minimal Super Key - no column can be removed without losing uniqueness.

### Example
From our User table Super Keys:

| Super Key | Minimal? | Candidate Key? |
|-----------|----------|----------------|
| `(user_id)` | ✅ Yes | ✅ Yes |
| `(email)` | ✅ Yes | ✅ Yes |
| `(phone)` | ✅ Yes | ✅ Yes |
| `(user_id, name)` | ❌ No (name is extra) | ❌ No |
| `(user_id, email)` | ❌ No (email is extra) | ❌ No |

**Candidate Keys**: `{user_id}`, `{email}`, `{phone}`

### Key Rules:
1. Must be a Super Key (uniquely identifies rows)
2. Must be minimal (no extra columns)
3. A table can have multiple Candidate Keys

---

## 3️⃣ Primary Key

### Definition
> A **Primary Key** is the chosen Candidate Key that will be the main identifier for the table.

### Rules:
1. **Unique**: No duplicates allowed
2. **Not NULL**: Every row must have a value
3. **Immutable**: Should not change once assigned
4. **One per table**: Only one Primary Key per table

### Example
From Candidate Keys `{user_id}`, `{email}`, `{phone}`:

**Choose `user_id` as Primary Key because**:
- ✅ Never changes (unlike email/phone)
- ✅ System-generated (reliable)
- ✅ Short and efficient for indexing
- ✅ No business logic dependency

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,  -- Primary Key chosen
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(15) UNIQUE,
    name VARCHAR(100)
);
```

---

## 4️⃣ Alternate Key

### Definition
> An **Alternate Key** is any Candidate Key that was not chosen as the Primary Key.

### Example
- **Primary Key**: `user_id`
- **Alternate Keys**: `email`, `phone`

### Implementation
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE,    -- Alternate Key
    phone VARCHAR(15) UNIQUE,     -- Alternate Key  
    name VARCHAR(100)
);
```

**Why keep Alternate Keys?**
- Prevent duplicates (UNIQUE constraint)
- Alternative ways to find records
- Business requirements (unique email/phone)

---

## 5️⃣ Composite Key

### Definition
> A **Composite Key** is a key made up of multiple columns that together uniquely identify a row.

### When to Use:
When no single column can uniquely identify a row.

### Example: OrderItem Table
| order_id | product_id | quantity | price |
|----------|------------|----------|-------|
| 1001 | 501 | 2 | 1000 |
| 1001 | 502 | 1 | 500 |
| 1002 | 501 | 3 | 1500 |

**Neither `order_id` nor `product_id` alone is unique**
- Same order can have multiple products  
- Same product can be in multiple orders

**Composite Key**: `(order_id, product_id)`

```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)  -- Composite Key
);
```

---

## 6️⃣ Foreign Key

### Definition
> A **Foreign Key** is a column that references the Primary Key of another table, establishing relationships.

### Purpose:
- **Link tables together**
- **Maintain referential integrity**
- **Prevent orphaned records**

### Example: E-commerce Schema

```sql
-- Parent Table
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);

-- Child Table  
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,                    -- Foreign Key
    order_date DATE,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

### Referential Integrity Rules:
1. **Insert**: Can't create order for non-existent user
2. **Update**: Can't change user_id to non-existent user  
3. **Delete**: Can't delete user with existing orders (by default)

### Foreign Key Actions:
```sql
FOREIGN KEY (user_id) REFERENCES users(user_id)
    ON DELETE CASCADE        -- Delete orders when user deleted
    ON UPDATE CASCADE        -- Update orders when user_id changes
```

---

## 7️⃣ Natural Key vs Surrogate Key

### Natural Key
> A **Natural Key** is a key that has business meaning and exists naturally in the data.

**Examples**:
- Social Security Number
- Email Address  
- Phone Number
- ISBN (for books)
- License Plate Number

### Surrogate Key
> A **Surrogate Key** is an artificial key created specifically for the database, with no business meaning.

**Examples**:
- Auto-incrementing integers (1, 2, 3...)
- UUIDs (550e8400-e29b-41d4-a716-446655440000)
- Sequential IDs

### Comparison:

| Aspect | Natural Key | Surrogate Key |
|--------|-------------|---------------|
| **Business Meaning** | Yes (SSN, Email) | No (ID: 12345) |
| **Stability** | Can change | Never changes |
| **Size** | Variable | Fixed |
| **Uniqueness** | Business rules | System guaranteed |
| **Performance** | Variable | Optimized |

### Real-World Decision:

**Most companies use Surrogate Keys because**:
- ✅ **Stable**: Never need to change
- ✅ **Performance**: Optimized for indexing
- ✅ **Simple**: No complex business rules
- ✅ **Control**: Database controls uniqueness

**Keep Natural Keys as Alternate Keys**:
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,           -- Surrogate Key
    email VARCHAR(255) UNIQUE NOT NULL,   -- Natural Alternate Key
    ssn VARCHAR(11) UNIQUE,               -- Natural Alternate Key (if applicable)
    name VARCHAR(100)
);
```

---

## 🎯 Interview Questions & Answers

### Beginner Level

**Q1: What is the difference between Primary Key and Unique Key?**

**Answer:**
- **Primary Key**: Only one per table, cannot be NULL, automatically creates clustered index
- **Unique Key**: Multiple allowed per table, can be NULL (only one NULL value), creates non-clustered index

**Q2: Can a table exist without a Primary Key?**

**Answer:** 
- **Technically**: Yes in most databases
- **Best Practice**: No - every table should have a Primary Key
- **Why**: Without PK, you can't uniquely identify rows, replication fails, performance suffers

**Q3: What is a Composite Key?**

**Answer:** A Primary Key made of multiple columns. Used when no single column can uniquely identify a row.

### Intermediate Level

**Q4: Why do companies prefer BIGINT IDs over Natural Keys like email?**

**Answer:**
1. **Stability**: Emails can change, IDs don't
2. **Performance**: Integer comparison is faster than string comparison  
3. **Size**: BIGINT (8 bytes) vs VARCHAR(255) (up to 255 bytes)
4. **Joins**: Faster joins with smaller key size
5. **Privacy**: Don't expose business data in URLs

**Q5: When would you use UUID vs BIGINT as Primary Key?**

**Answer:**
- **UUID**: Distributed systems, need globally unique IDs, microservices
- **BIGINT**: Single database, sequential access patterns, better performance

### Senior Level

**Q6: Design a database for a social media platform. What keys would you choose and why?**

**Answer:**
```sql
-- Users: Surrogate key for stability
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(30) UNIQUE,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP
);

-- Posts: UUID for distributed system
CREATE TABLE posts (
    post_id UUID PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Follows: Composite key (natural relationship)
CREATE TABLE follows (
    follower_id BIGINT,
    following_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (following_id) REFERENCES users(user_id)
);
```

**Design Decisions:**
- **Users**: BIGINT for performance, username/email as alternate keys
- **Posts**: UUID for horizontal scaling across data centers
- **Follows**: Composite key represents natural M:N relationship

---

## 💡 Key Selection Best Practices

### 1. Primary Key Selection Priority:
1. **Surrogate Key** (Auto-increment BIGINT or UUID)
2. **Single Natural Key** (if stable and efficient)
3. **Composite Key** (only when necessary)

### 2. When to Use Each Key Type:

#### BIGINT (Auto-increment)
```sql
user_id BIGINT AUTO_INCREMENT PRIMARY KEY
```
**Use When:**
- Single database instance
- High performance needed
- Sequential access patterns

#### UUID
```sql
post_id UUID PRIMARY KEY DEFAULT (UUID())
```
**Use When:**  
- Distributed systems
- Microservices architecture
- Need globally unique IDs
- Data replication across regions

#### Composite Key
```sql
PRIMARY KEY (order_id, product_id)
```
**Use When:**
- Junction tables (M:N relationships)
- Natural business relationships
- Time-series data with partitioning

### 3. Foreign Key Best Practices:

```sql
-- Always name your constraints
CONSTRAINT fk_orders_user_id 
FOREIGN KEY (user_id) REFERENCES users(user_id)
ON DELETE RESTRICT      -- Prevent deleting users with orders
ON UPDATE CASCADE       -- Update orders if user_id changes
```

---

## 🏗️ Building Our E-commerce Schema

Let's apply what we learned to build a production-quality e-commerce database:

```sql
-- Users table with surrogate key
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,    -- Surrogate Key
    email VARCHAR(255) UNIQUE NOT NULL,           -- Alternate Key
    phone VARCHAR(15) UNIQUE,                     -- Alternate Key  
    username VARCHAR(30) UNIQUE,                  -- Alternate Key
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Categories with hierarchy support
CREATE TABLE categories (
    category_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    parent_category_id BIGINT NULL,              -- Self-referencing FK
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,           -- Alternate Key
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- Products table  
CREATE TABLE products (
    product_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    category_id BIGINT NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,             -- Alternate Key (business)
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,           -- Alternate Key (URLs)
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2) NOT NULL,
    weight DECIMAL(8,3),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- Orders table
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    order_number VARCHAR(20) UNIQUE NOT NULL,    -- Alternate Key (business)
    status ENUM('pending', 'confirmed', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL,
    shipping_amount DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Order Items (Junction table with composite key)
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),          -- Composite Key
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Key Decisions Made:**
1. **Surrogate Keys**: All entity tables use BIGINT auto-increment
2. **Alternate Keys**: Business-meaningful unique fields (email, sku, slug)
3. **Composite Key**: OrderItems uses natural composite key
4. **Foreign Keys**: Proper referential integrity with named constraints

---

## 🧪 Practical Exercises

### Exercise 1: Identify Key Types
Given this table, identify all key types:

```sql
CREATE TABLE employees (
    emp_id INT AUTO_INCREMENT PRIMARY KEY,
    employee_number VARCHAR(10) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    ssn VARCHAR(11) UNIQUE,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    department_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Your Task**: List all Super Keys, Candidate Keys, Primary Key, Alternate Keys, Natural Keys, and Surrogate Keys.

### Exercise 2: Design Decision
You're designing a library system. Choose the appropriate key strategy for:

1. **Books Table**: What should be the Primary Key?
2. **Authors Table**: How to handle authors?
3. **BookAuthors Table**: How to represent M:N relationship?

### Exercise 3: Performance Analysis
Compare these two approaches for an Orders table:

**Approach 1**: Email as Primary Key
```sql
CREATE TABLE orders (
    customer_email VARCHAR(255) PRIMARY KEY,
    order_date DATE,
    amount DECIMAL(10,2)
);
```

**Approach 2**: Surrogate Key
```sql  
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_email VARCHAR(255),
    order_date DATE, 
    amount DECIMAL(10,2)
);
```

**Analyze**: Performance, Scalability, Maintainability

---

## 📚 Summary

### Key Takeaways:
1. **Keys ensure uniqueness** and establish relationships
2. **Super Key → Candidate Key → Primary Key** progression
3. **Surrogate Keys** are preferred in production for stability and performance
4. **Foreign Keys** maintain data integrity and relationships
5. **Composite Keys** are used for junction tables and natural relationships

### Best Practices:
- Always use surrogate keys for entity tables
- Keep natural keys as alternate keys (UNIQUE constraints)
- Use composite keys only for junction tables
- Name all constraints for better maintainability
- Consider performance impact of key choices

### Next Chapter Preview:
**Chapter 2: Three-Level Architecture** - Understanding how databases separate concerns through abstraction layers.

---

*Continue to [Chapter 2: Three-Level Architecture →](./Module-2-Chapter-2-Three-Level-Architecture.md)*
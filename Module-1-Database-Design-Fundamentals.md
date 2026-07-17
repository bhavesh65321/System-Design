# Module 1: Database Design Fundamentals

> **Foundation Module**: Everything in database design builds on these concepts

## 🎯 Learning Objectives

By the end of this module, you'll be able to:
- Explain database design principles in interviews
- Design databases from business requirements  
- Identify entities, attributes, and relationships
- Create ER diagrams for complex systems
- Apply constraints and keys effectively
- Recognize good vs bad database designs

---

## Chapter 1: What is Database Design?

### Definition (Interview Ready)

> **Database Design** is the process of modeling data, defining entities, relationships, constraints, and storage structures so that the database efficiently supports application requirements while maintaining data integrity and performance.

### Why Database Design Matters

**Without Proper Design:**
- Data inconsistency
- Poor performance
- Difficult maintenance
- Scalability issues
- Security vulnerabilities

**With Proper Design:**
- Data integrity guaranteed
- Optimal performance
- Easy to maintain and extend
- Scales with business growth
- Secure by design

### Real-World Example

**Bad Design (Beginner Mistake):**
```sql
-- Everything in one table
CREATE TABLE user_orders (
    id INT,
    username VARCHAR(50),
    email VARCHAR(100),
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    quantity INT,
    order_date DATE
);
```

**Problems:**
- Data duplication (user info repeated)
- Update anomalies (change email everywhere)
- Delete anomalies (delete user loses order history)
- Waste of storage space

**Good Design (Senior Approach):**
```sql
-- Separate entities with relationships
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100) UNIQUE
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATE,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

## Chapter 2: Database Design Process

### The 6-Step Process

```
Requirements → Entities → Attributes → Relationships → Constraints → ER Diagram
```

#### Step 1: Understand Requirements
- Read business requirements carefully
- Identify what data needs to be stored
- Understand user workflows
- Note performance requirements

#### Step 2: Identify Entities
- Find "things" or "concepts" in requirements
- Entities become tables
- Each entity should represent one concept

#### Step 3: Identify Attributes  
- Properties of each entity
- Attributes become columns
- Include data types and constraints

#### Step 4: Define Relationships
- How entities connect to each other
- One-to-One, One-to-Many, Many-to-Many
- Determines foreign keys and junction tables

#### Step 5: Apply Constraints
- Business rules as database constraints
- Primary keys, foreign keys, unique constraints
- Check constraints for data validation

#### Step 6: Create ER Diagram
- Visual representation of design
- Validate design with stakeholders
- Document for future reference

---

## Chapter 3: Entities

### What is an Entity?

> An **Entity** is a thing or concept about which we want to store information.

### Identifying Entities

**Look for nouns in requirements:**
- "Users can register" → **User** entity
- "Products have prices" → **Product** entity  
- "Orders contain items" → **Order** entity
- "Categories group products" → **Category** entity

### Entity Rules

1. **One concept per entity** - Don't mix user and order data
2. **Meaningful name** - Use business terminology
3. **Independent existence** - Can exist without other entities
4. **Multiple instances** - There will be many users, many products

### Example: E-commerce Entities

**Requirements**: Build an online store where customers can browse products, add to cart, and place orders.

**Entities Identified:**
- **Customer** - People who shop
- **Product** - Items for sale
- **Category** - Product groupings
- **Cart** - Shopping cart
- **Order** - Purchase transactions
- **OrderItem** - Products in an order

---

## Chapter 4: Attributes

### What is an Attribute?

> An **Attribute** is a property or characteristic of an entity.

### Types of Attributes

#### 1. Simple Attributes
Single, indivisible values
```sql
-- Customer entity
name VARCHAR(100)
email VARCHAR(255)
age INT
```

#### 2. Composite Attributes
Can be divided into smaller parts
```sql
-- Instead of: address VARCHAR(500)
-- Better:
street_address VARCHAR(200)
city VARCHAR(100)
postal_code VARCHAR(20)
country VARCHAR(100)
```

#### 3. Derived Attributes
Calculated from other attributes
```sql
-- Don't store age, calculate from birth_date
birth_date DATE
-- age = YEAR(CURDATE()) - YEAR(birth_date)
```

#### 4. Multi-valued Attributes
Can have multiple values (avoid in relational design)
```sql
-- Wrong: phone_numbers VARCHAR(500) -- "123,456,789"
-- Right: Separate table
CREATE TABLE customer_phones (
    customer_id INT,
    phone_number VARCHAR(15),
    phone_type ENUM('mobile', 'home', 'work')
);
```

### Choosing Attributes

**Guidelines:**
1. **Atomic** - One value per attribute
2. **Relevant** - Needed for business requirements
3. **Non-redundant** - Don't duplicate data
4. **Consistent** - Same data type and format

### Example: Product Entity Attributes

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,          -- Surrogate key
    sku VARCHAR(50) UNIQUE NOT NULL,        -- Business key
    name VARCHAR(200) NOT NULL,             -- Required
    description TEXT,                       -- Optional
    price DECIMAL(10,2) NOT NULL,          -- Monetary
    cost DECIMAL(10,2) NOT NULL,           -- Internal cost
    weight DECIMAL(8,3),                   -- Optional physical
    category_id BIGINT NOT NULL,           -- Foreign key
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Chapter 5: Relationships

### What is a Relationship?

> A **Relationship** describes how entities are connected to each other.

### Types of Relationships

#### 1. One-to-One (1:1)
One instance relates to exactly one instance

**Example**: User ↔ Profile
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    bio TEXT,
    avatar_url VARCHAR(500),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

#### 2. One-to-Many (1:N)
One instance relates to many instances

**Example**: Customer → Orders (One customer, many orders)
```sql
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,           -- Foreign key
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

#### 3. Many-to-Many (M:N)
Many instances relate to many instances

**Example**: Orders ↔ Products (Orders have many products, products in many orders)
```sql
-- Junction table required
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Relationship Guidelines

1. **Identify cardinality** - How many on each side?
2. **Use foreign keys** - Maintain referential integrity
3. **Junction tables** - Required for M:N relationships
4. **Meaningful names** - Clear relationship purpose

---

## Chapter 6: Constraints

### What are Constraints?

> **Constraints** are rules that ensure data integrity and enforce business logic.

### Types of Constraints

#### 1. PRIMARY KEY
- Uniquely identifies each row
- Cannot be NULL
- Only one per table

```sql
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    -- Auto-increment alternative:
    -- customer_id BIGINT AUTO_INCREMENT PRIMARY KEY
);
```

#### 2. FOREIGN KEY
- Links to another table's PRIMARY KEY
- Maintains referential integrity

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT    -- Prevent deletion if orders exist
        ON UPDATE CASCADE     -- Update if customer_id changes
);
```

#### 3. UNIQUE
- Ensures no duplicate values
- Can have multiple UNIQUE constraints per table

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL
);
```

#### 4. NOT NULL
- Ensures column always has a value

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,        -- Required field
    price DECIMAL(10,2) NOT NULL,      -- Required field
    description TEXT                   -- Optional field
);
```

#### 5. CHECK
- Custom validation rules

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    quantity INT NOT NULL CHECK (quantity >= 0),
    status ENUM('active', 'inactive', 'discontinued') DEFAULT 'active'
);
```

#### 6. DEFAULT
- Provides default values

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Chapter 7: ER Diagrams

### What is an ER Diagram?

> An **ER Diagram** (Entity-Relationship Diagram) is a visual representation of entities, attributes, and relationships.

### ER Diagram Components

#### Entities (Rectangles)
```
┌─────────────┐
│   Customer  │
└─────────────┘
```

#### Attributes (Ovals)
```
customer_id ○─── ┌─────────────┐
name        ○─── │   Customer  │ ───○ email
            ○─── └─────────────┘ ───○ phone
```

#### Relationships (Diamonds)
```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Customer  │────────○│   Places    │○────────│    Order    │
└─────────────┘    1    └─────────────┘    N    └─────────────┘
```

### Cardinality Notation

- **1:1** - One line each side
- **1:N** - One line, crow's foot
- **M:N** - Crow's foot both sides

### Example: E-commerce ER Diagram

```
Customer (1) ──places──→ (N) Order
Order (N) ──contains──→ (M) Product [via OrderItem]
Product (N) ──belongs_to──→ (1) Category
Customer (1) ──has──→ (1) Profile
```

---

## Chapter 8: Mini Project - Movie Booking System

### Requirements
Design a database for a movie ticket booking system:

1. **Customers** can register and book tickets
2. **Movies** have showtimes at different theaters
3. **Theaters** have multiple screens with different capacities
4. **Bookings** contain multiple seats for a specific showtime
5. **Payments** are recorded for each booking

### Solution

#### Step 1: Identify Entities
- Customer
- Movie  
- Theater
- Screen
- Showtime
- Booking
- Seat
- Payment

#### Step 2: Design Tables

```sql
-- Customers
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(15) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Movies
CREATE TABLE movies (
    movie_id BIGINT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    genre VARCHAR(50),
    duration_minutes INT NOT NULL,
    rating VARCHAR(10),
    release_date DATE
);

-- Theaters
CREATE TABLE theaters (
    theater_id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(300) NOT NULL,
    city VARCHAR(100) NOT NULL
);

-- Screens within theaters
CREATE TABLE screens (
    screen_id BIGINT PRIMARY KEY,
    theater_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    capacity INT NOT NULL,
    FOREIGN KEY (theater_id) REFERENCES theaters(theater_id)
);

-- Movie showtimes
CREATE TABLE showtimes (
    showtime_id BIGINT PRIMARY KEY,
    movie_id BIGINT NOT NULL,
    screen_id BIGINT NOT NULL,
    show_date DATE NOT NULL,
    show_time TIME NOT NULL,
    price DECIMAL(8,2) NOT NULL,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id),
    FOREIGN KEY (screen_id) REFERENCES screens(screen_id),
    UNIQUE KEY unique_showtime (screen_id, show_date, show_time)
);

-- Bookings
CREATE TABLE bookings (
    booking_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    showtime_id BIGINT NOT NULL,
    booking_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('confirmed', 'cancelled') DEFAULT 'confirmed',
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (showtime_id) REFERENCES showtimes(showtime_id)
);

-- Individual seat bookings (M:N relationship)
CREATE TABLE booking_seats (
    booking_id BIGINT,
    seat_number VARCHAR(10),
    PRIMARY KEY (booking_id, seat_number),
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id)
);

-- Payments
CREATE TABLE payments (
    payment_id BIGINT PRIMARY KEY,
    booking_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    payment_method ENUM('card', 'upi', 'wallet') NOT NULL,
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('success', 'failed', 'pending') DEFAULT 'pending',
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id)
);
```

#### Step 3: Key Design Decisions

1. **Surrogate Keys** - All entities use BIGINT auto-increment
2. **Business Keys** - Email, phone as alternate keys
3. **Composite Keys** - booking_seats uses (booking_id, seat_number)
4. **Constraints** - Prevent double booking with unique constraints
5. **Enums** - Limited value sets for status fields
6. **Timestamps** - Track creation and modification times

---

## 🎯 Interview Questions

### Beginner
**Q1: What is database design?**
**A:** The process of organizing data into tables, defining relationships, and applying constraints to ensure data integrity and performance.

**Q2: What's the difference between an entity and an attribute?**
**A:** Entity is a "thing" we store data about (Customer). Attribute is a property of that entity (customer name, email).

**Q3: Why do we separate data into multiple tables?**
**A:** To eliminate redundancy, ensure data integrity, and make the system easier to maintain and scale.

### Intermediate
**Q4: How do you handle a many-to-many relationship?**
**A:** Create a junction table with foreign keys to both entities, often with additional attributes specific to the relationship.

**Q5: Why use surrogate keys instead of natural keys?**
**A:** Surrogate keys are stable (don't change), efficient (integers), and don't expose business data.

### Senior
**Q6: Design a database for a social media platform.**
**A:** Focus on users, posts, relationships (followers), and consider scalability challenges like sharding strategies.

---

## 📋 Summary

### Key Principles
1. **One concept per table** - Clear entity separation
2. **Avoid redundancy** - Normalize data appropriately
3. **Use constraints** - Enforce business rules at database level
4. **Choose keys wisely** - Surrogate for entities, composite for relationships
5. **Plan for growth** - Consider future requirements

### Design Checklist
- [ ] All entities identified from requirements
- [ ] Attributes are atomic and relevant
- [ ] Relationships clearly defined with proper cardinality  
- [ ] Primary keys chosen for all tables
- [ ] Foreign keys maintain referential integrity
- [ ] Constraints enforce business rules
- [ ] ER diagram validates design

### Next Module Preview
**Module 2: Relational Database Design** - Deep dive into keys, normalization, and advanced relationship patterns.

---

*[Continue to Module 2: Relational Database Design →](./Module-2-Relational-Database-Design.md)*
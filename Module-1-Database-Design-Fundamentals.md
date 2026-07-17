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

Let's apply the complete 6-step database design process to build a real movie booking system.

---

### Step 1: Understand Requirements

**Business Requirements:**
1. **Customers** can register with email and phone
2. **Customers** can browse movies and showtimes
3. **Movies** are shown at different **theaters**
4. **Theaters** have multiple **screens** with different capacities
5. **Movies** have multiple **showtimes** on different screens
6. **Customers** can book tickets for specific showtimes
7. **Bookings** can include multiple seats
8. **Payments** are processed for each booking
9. **Customers** can view their booking history

**Data to Store:**
- Customer information (name, contact details)
- Movie details (title, genre, duration)
- Theater and screen information
- Showtime schedules with pricing
- Booking details with seat selections
- Payment transactions

**User Workflows:**
- Customer registration → Browse movies → Select showtime → Choose seats → Make payment → Get booking confirmation

---

### Step 2: Identify Entities

**Look for nouns in requirements:**

| Requirement | Entity Identified |
|-------------|-------------------|
| "**Customers** can register" | Customer |
| "Browse **movies** and showtimes" | Movie |
| "Movies are shown at **theaters**" | Theater |
| "Theaters have **screens**" | Screen |
| "Movies have **showtimes**" | Showtime |
| "Can book **tickets**" | Booking |
| "Multiple **seats**" | Seat (or BookingSeat) |
| "**Payments** are processed" | Payment |

**Final Entity List:**
1. **Customer** - People who book tickets
2. **Movie** - Films being shown
3. **Theater** - Cinema locations
4. **Screen** - Individual screens within theaters
5. **Showtime** - Movie screenings at specific times
6. **Booking** - Ticket reservations
7. **BookingSeat** - Individual seat reservations
8. **Payment** - Financial transactions

---

### Step 3: Identify Attributes

**For each entity, identify properties:**

#### Customer Entity
```
Customer
├── customer_id (Primary Key)
├── email (Unique, Required)
├── phone (Unique, Required)
├── first_name (Required)
├── last_name (Required)
├── date_of_birth (Optional)
└── created_at (Auto-generated)
```

#### Movie Entity
```
Movie
├── movie_id (Primary Key)
├── title (Required)
├── genre (Optional)
├── duration_minutes (Required)
├── rating (Optional)
├── release_date (Optional)
├── description (Optional)
└── poster_url (Optional)
```

#### Theater Entity
```
Theater
├── theater_id (Primary Key)
├── name (Required)
├── address (Required)
├── city (Required)
├── postal_code (Optional)
└── phone (Optional)
```

#### Screen Entity
```
Screen
├── screen_id (Primary Key)
├── theater_id (Foreign Key)
├── name (Required) -- "Screen 1", "IMAX"
├── capacity (Required)
└── screen_type (Optional) -- "Regular", "IMAX", "4DX"
```

#### Showtime Entity
```
Showtime
├── showtime_id (Primary Key)
├── movie_id (Foreign Key)
├── screen_id (Foreign Key)
├── show_date (Required)
├── show_time (Required)
├── price (Required)
└── available_seats (Calculated/Derived)
```

#### Booking Entity
```
Booking
├── booking_id (Primary Key)
├── customer_id (Foreign Key)
├── showtime_id (Foreign Key)
├── booking_date (Auto-generated)
├── total_amount (Required)
├── booking_status (Required) -- "confirmed", "cancelled"
└── booking_reference (Unique code)
```

#### BookingSeat Entity
```
BookingSeat
├── booking_id (Foreign Key, Composite Primary Key)
├── seat_number (Required, Composite Primary Key)
└── seat_price (Required)
```

#### Payment Entity
```
Payment
├── payment_id (Primary Key)
├── booking_id (Foreign Key)
├── amount (Required)
├── payment_method (Required) -- "card", "upi", "wallet"
├── payment_date (Auto-generated)
├── transaction_id (External reference)
└── status (Required) -- "success", "failed", "pending"
```

---

### Step 4: Define Relationships

**Analyze how entities connect:**

#### One-to-Many (1:N) Relationships
1. **Theater → Screens**: One theater has many screens
2. **Customer → Bookings**: One customer can make many bookings
3. **Movie → Showtimes**: One movie can have many showtimes
4. **Screen → Showtimes**: One screen can show many movies
5. **Showtime → Bookings**: One showtime can have many bookings
6. **Booking → Payments**: One booking can have multiple payments (partial payments)

#### Many-to-Many (M:N) Relationships
1. **Booking ↔ Seats**: One booking can have multiple seats, one seat can be booked multiple times (different showtimes)
   - **Solution**: BookingSeat junction table

#### Relationship Details
```
Theater (1) ──has──→ (N) Screen
Customer (1) ──makes──→ (N) Booking  
Movie (1) ──shown_in──→ (N) Showtime
Screen (1) ──hosts──→ (N) Showtime
Showtime (1) ──receives──→ (N) Booking
Booking (1) ──includes──→ (N) BookingSeat
Booking (1) ──processed_by──→ (N) Payment
```

---

### Step 5: Apply Constraints

**Business rules as database constraints:**

#### Primary Keys
- Every table must have a unique identifier
- Use BIGINT AUTO_INCREMENT for all primary keys

#### Foreign Keys
- Maintain referential integrity
- Prevent orphaned records

#### Unique Constraints
- Customer email and phone must be unique
- No double booking of same seat for same showtime
- Booking reference codes must be unique

#### Check Constraints
- Movie duration must be positive
- Screen capacity must be positive  
- Showtime price must be positive
- Booking amount must be positive
- Show date cannot be in the past

#### Not Null Constraints
- Essential fields cannot be empty
- Names, prices, dates are required

#### Default Values
- Booking status defaults to 'confirmed'
- Payment status defaults to 'pending'
- Timestamps auto-generate current time

---

### Step 6: Create ER Diagram

```
                    ER DIAGRAM: Movie Booking System

┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Theater   │    1:N  │   Screen    │    1:N  │  Showtime   │
│             │─────────│             │─────────│             │
│ theater_id  │         │ screen_id   │         │showtime_id  │
│ name        │         │ theater_id  │         │ movie_id    │
│ address     │         │ name        │         │ screen_id   │
│ city        │         │ capacity    │         │ show_date   │
└─────────────┘         └─────────────┘         │ show_time   │
                                               │ price       │
                        ┌─────────────┐         └─────────────┘
                        │    Movie    │                │
                        │             │           1:N  │
                        │ movie_id    │──────────────────
                        │ title       │
                        │ genre       │
                        │ duration    │
                        │ rating      │
                        └─────────────┘

┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│  Customer   │    1:N  │   Booking   │    1:N  │BookingSeat  │
│             │─────────│             │─────────│             │
│customer_id  │         │ booking_id  │         │ booking_id  │
│ email       │         │customer_id  │         │seat_number  │
│ phone       │         │showtime_id  │         │ seat_price  │
│ first_name  │         │booking_date │         └─────────────┘
│ last_name   │         │total_amount │
└─────────────┘         │ status      │
                        └─────────────┘
                               │
                               │ 1:N
                               ▼
                        ┌─────────────┐
                        │   Payment   │
                        │             │
                        │ payment_id  │
                        │ booking_id  │
                        │ amount      │
                        │ method      │
                        │ status      │
                        └─────────────┘
```

---

### Complete Database Implementation

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

### Complete Database Implementation

**Now let's convert our design into SQL tables:**

```sql
-- Step 1: Create Theater table
CREATE TABLE theaters (
    theater_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(300) NOT NULL,
    city VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20),
    phone VARCHAR(15),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Step 2: Create Screen table (depends on Theater)
CREATE TABLE screens (
    screen_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    theater_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    capacity INT NOT NULL CHECK (capacity > 0),
    screen_type ENUM('Regular', 'IMAX', '4DX') DEFAULT 'Regular',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (theater_id) REFERENCES theaters(theater_id),
    UNIQUE KEY unique_screen_per_theater (theater_id, name)
);

-- Step 3: Create Movie table (independent)
CREATE TABLE movies (
    movie_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    genre VARCHAR(50),
    duration_minutes INT NOT NULL CHECK (duration_minutes > 0),
    rating VARCHAR(10),
    release_date DATE,
    description TEXT,
    poster_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Step 4: Create Customer table (independent)
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(15) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    date_of_birth DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Step 5: Create Showtime table (depends on Movie and Screen)
CREATE TABLE showtimes (
    showtime_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    movie_id BIGINT NOT NULL,
    screen_id BIGINT NOT NULL,
    show_date DATE NOT NULL,
    show_time TIME NOT NULL,
    price DECIMAL(8,2) NOT NULL CHECK (price > 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (movie_id) REFERENCES movies(movie_id),
    FOREIGN KEY (screen_id) REFERENCES screens(screen_id),
    UNIQUE KEY unique_showtime (screen_id, show_date, show_time),
    CHECK (show_date >= CURDATE()) -- Cannot schedule shows in the past
);

-- Step 6: Create Booking table (depends on Customer and Showtime)
CREATE TABLE bookings (
    booking_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    showtime_id BIGINT NOT NULL,
    booking_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount > 0),
    booking_status ENUM('confirmed', 'cancelled') DEFAULT 'confirmed',
    booking_reference VARCHAR(20) UNIQUE NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (showtime_id) REFERENCES showtimes(showtime_id)
);

-- Step 7: Create BookingSeat table (junction table)
CREATE TABLE booking_seats (
    booking_id BIGINT,
    seat_number VARCHAR(10) NOT NULL,
    seat_price DECIMAL(8,2) NOT NULL CHECK (seat_price > 0),
    PRIMARY KEY (booking_id, seat_number),
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id) ON DELETE CASCADE
);

-- Step 8: Create Payment table (depends on Booking)
CREATE TABLE payments (
    payment_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    booking_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL CHECK (amount > 0),
    payment_method ENUM('card', 'upi', 'wallet', 'cash') NOT NULL,
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    transaction_id VARCHAR(100),
    status ENUM('success', 'failed', 'pending') DEFAULT 'pending',
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id)
);

-- Step 9: Add indexes for performance
CREATE INDEX idx_showtimes_movie ON showtimes(movie_id);
CREATE INDEX idx_showtimes_screen_date ON showtimes(screen_id, show_date);
CREATE INDEX idx_bookings_customer ON bookings(customer_id);
CREATE INDEX idx_bookings_showtime ON bookings(showtime_id);
CREATE INDEX idx_payments_booking ON payments(booking_id);
```

### Sample Data and Queries

**Insert Sample Data:**
```sql
-- Sample theater
INSERT INTO theaters (name, address, city) 
VALUES ('PVR Cinemas', '123 Mall Road', 'Mumbai');

-- Sample screen
INSERT INTO screens (theater_id, name, capacity) 
VALUES (1, 'Screen 1', 100);

-- Sample movie
INSERT INTO movies (title, genre, duration_minutes, rating) 
VALUES ('Avengers: Endgame', 'Action', 180, 'PG-13');

-- Sample customer
INSERT INTO customers (email, phone, first_name, last_name) 
VALUES ('john@example.com', '9999999999', 'John', 'Doe');

-- Sample showtime
INSERT INTO showtimes (movie_id, screen_id, show_date, show_time, price) 
VALUES (1, 1, '2026-07-20', '18:00:00', 250.00);
```

**Common Queries:**
```sql
-- Find available showtimes for a movie
SELECT 
    s.showtime_id,
    m.title,
    t.name as theater_name,
    sc.name as screen_name,
    s.show_date,
    s.show_time,
    s.price,
    (sc.capacity - COALESCE(booked_seats.seat_count, 0)) as available_seats
FROM showtimes s
JOIN movies m ON s.movie_id = m.movie_id
JOIN screens sc ON s.screen_id = sc.screen_id
JOIN theaters t ON sc.theater_id = t.theater_id
LEFT JOIN (
    SELECT 
        b.showtime_id,
        COUNT(bs.seat_number) as seat_count
    FROM bookings b
    JOIN booking_seats bs ON b.booking_id = bs.booking_id
    WHERE b.booking_status = 'confirmed'
    GROUP BY b.showtime_id
) booked_seats ON s.showtime_id = booked_seats.showtime_id
WHERE m.movie_id = 1 
  AND s.show_date >= CURDATE()
ORDER BY s.show_date, s.show_time;

-- Customer booking history
SELECT 
    b.booking_id,
    b.booking_reference,
    m.title,
    s.show_date,
    s.show_time,
    b.total_amount,
    b.booking_status,
    GROUP_CONCAT(bs.seat_number) as seats
FROM bookings b
JOIN showtimes s ON b.showtime_id = s.showtime_id
JOIN movies m ON s.movie_id = m.movie_id
JOIN booking_seats bs ON b.booking_id = bs.booking_id
WHERE b.customer_id = 1
GROUP BY b.booking_id
ORDER BY b.booking_date DESC;
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
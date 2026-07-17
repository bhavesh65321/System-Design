# Chapter 2: Database Design Process

> **Methodology Chapter**: Learn the 6-step systematic approach used by professional database architects

## 🎯 Learning Objectives

By the end of this chapter, you'll be able to:
- [ ] Apply the 6-step database design methodology
- [ ] Analyze business requirements systematically  
- [ ] Transform requirements into database entities
- [ ] Follow a repeatable design process for any project

---

## 🔄 The 6-Step Database Design Process

Professional database designers follow a systematic approach:

```
Requirements Analysis → Entity Identification → Attribute Definition → 
Relationship Modeling → Constraint Application → ER Diagram Creation
```

### Why This Process Matters

**Without Process** (Common Mistakes):
- Jump straight to creating tables
- Miss important entities or relationships
- Forget constraints and business rules  
- End up redesigning multiple times

**With Process** (Professional Approach):
- Thorough analysis prevents redesign
- Systematic approach catches everything
- Clear documentation for stakeholders
- Predictable, high-quality results

---

## Step 1: Requirements Analysis 📋

### What to Look For

**Business Entities** (Nouns):
- "**Customers** can place **orders**"
- "**Products** belong to **categories**"  
- "**Users** write **reviews** for **products**"

**Business Actions** (Verbs):
- Customers **place** orders
- Users **write** reviews
- Products **belong to** categories

**Business Rules** (Constraints):
- "Order must have at least one item"
- "Customer email must be unique"
- "Product price cannot be negative"

### Practical Example: Online Learning Platform

**Requirements**:
> Build a platform where **instructors** create **courses** with multiple **lessons**. **Students** can **enroll** in courses, watch lessons, and submit **assignments**. The system tracks **progress** and **certificates**.

**Analysis Output**:
- **Entities**: Instructor, Course, Lesson, Student, Assignment, Progress, Certificate
- **Actions**: Create, Enroll, Watch, Submit, Track, Issue
- **Rules**: Student must enroll before accessing, Assignment deadlines, Certificate criteria

---

## Step 2: Entity Identification 🎯

### Entity Selection Rules

1. **Represents Important Business Concept**
   - ✅ Customer (central to business)
   - ❌ Customer's favorite color (not business-critical)

2. **Has Multiple Instances**
   - ✅ Product (many products exist)
   - ❌ Company name (only one company)

3. **Has Relevant Attributes**
   - ✅ Order (has date, total, status)
   - ❌ "Process" (too abstract)

4. **Independent Existence**
   - ✅ Customer (exists without orders)
   - ❌ Order item (cannot exist without order)

### Learning Platform Example

**Identified Entities**:
```
Primary Entities:
├── Instructor (teachers who create content)
├── Student (learners who take courses)  
├── Course (learning programs)
└── Lesson (individual learning units)

Secondary Entities:
├── Enrollment (student-course relationship)
├── Assignment (tasks within courses)
├── Submission (student assignment responses)
├── Certificate (completion credentials)
└── Progress (tracking student advancement)
```

---

## Step 3: Attribute Definition 📝

### Attribute Categories

**Identifying Attributes** (Primary Keys):
- `instructor_id`, `student_id`, `course_id`

**Descriptive Attributes**:
- `instructor_name`, `course_title`, `lesson_content`

**Relationship Attributes** (Foreign Keys):
- `course.instructor_id`, `lesson.course_id`

**Derived Attributes** (Calculated):
- `course.total_duration` (sum of lesson durations)
- `student.completion_percentage`

### Example: Course Entity

```sql
CREATE TABLE courses (
    -- Identifying
    course_id BIGINT PRIMARY KEY,
    
    -- Descriptive  
    title VARCHAR(200) NOT NULL,
    description TEXT,
    level ENUM('beginner', 'intermediate', 'advanced'),
    price DECIMAL(8,2) NOT NULL,
    
    -- Relationship
    instructor_id BIGINT NOT NULL,
    
    -- System
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Constraints will be added in Step 5
);
```

---

## Step 4: Relationship Modeling 🔗

### Relationship Analysis Framework

For each entity pair, ask:
1. **Do they relate?** (Business connection exists?)
2. **What's the cardinality?** (1:1, 1:N, M:N)
3. **Is it mandatory?** (Can one exist without the other?)

### Learning Platform Relationships

**1:N Relationships**:
```
Instructor (1) ──creates──→ (N) Course
Course (1) ──contains──→ (N) Lesson  
Course (1) ──has──→ (N) Assignment
```

**M:N Relationships**:
```
Student (M) ──enrolls_in──→ (N) Course
Student (M) ──submits──→ (N) Assignment
```

**1:1 Relationships**:
```
Student (1) ──earns──→ (1) Certificate [per course]
```

### Junction Tables for M:N

```sql
-- Student enrolls in Course (M:N)
CREATE TABLE enrollments (
    student_id BIGINT,
    course_id BIGINT,
    enrollment_date DATE NOT NULL,
    completion_status ENUM('active', 'completed', 'dropped'),
    PRIMARY KEY (student_id, course_id)
);
```

---

## Step 5: Constraint Application ⚖️

### Constraint Categories

**Entity Integrity**:
- Primary keys (unique, not null)
- Unique constraints (email, username)

**Referential Integrity**:  
- Foreign key relationships
- Cascade rules (ON DELETE, ON UPDATE)

**Domain Integrity**:
- Data type constraints
- Check constraints (price > 0)
- Default values

**Business Rules**:
- Complex constraints
- Triggers for advanced logic

### Example Implementation

```sql
CREATE TABLE courses (
    course_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    price DECIMAL(8,2) NOT NULL CHECK (price >= 0),
    instructor_id BIGINT NOT NULL,
    
    -- Foreign key with business rules
    FOREIGN KEY (instructor_id) REFERENCES instructors(instructor_id)
        ON DELETE RESTRICT,  -- Can't delete instructor with courses
    
    -- Business constraint
    CHECK (LENGTH(title) >= 10)  -- Title must be descriptive
);
```

---

## Step 6: ER Diagram Creation 📊

### Visual Documentation

```
ER Diagram: Online Learning Platform

┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ Instructor  │    1:N  │   Course    │    1:N  │   Lesson    │
│             │─────────│             │─────────│             │
│instructor_id│         │ course_id   │         │ lesson_id   │
│ name        │         │ title       │         │ title       │
│ email       │         │ description │         │ content     │
│ bio         │         │ price       │         │ duration    │
└─────────────┘         │instructor_id│         │ course_id   │
                        └─────────────┘         └─────────────┘
                              │
                              │ M:N
                              │
                        ┌─────────────┐
                        │ Enrollment  │
                        │             │
                        │ student_id  │
                        │ course_id   │
                        │ enroll_date │
                        │ status      │
                        └─────────────┘
                              │
                              │ N:1
                              │
┌─────────────┐         ┌─────────────┐
│  Student    │    1:N  │ Assignment  │
│             │─────────│ Submission  │
│ student_id  │         │             │
│ name        │         │ student_id  │
│ email       │         │assignment_id│
│ join_date   │         │ content     │
└─────────────┘         │ submitted_at│
                        └─────────────┘
```

---

## 🏢 Real-World Process Example

### Case Study: E-commerce Marketplace

**Step 1: Requirements**
> Multi-vendor marketplace where **vendors** list **products**, **customers** place **orders**, products have **reviews**, and the system handles **payments** and **shipping**.

**Step 2: Entities**  
Vendor, Product, Customer, Order, OrderItem, Review, Payment, ShippingAddress

**Step 3: Key Attributes**
- Vendor: vendor_id, company_name, email, commission_rate
- Product: product_id, name, price, vendor_id, category_id  
- Customer: customer_id, name, email, phone

**Step 4: Relationships**
- Vendor (1) → (N) Product
- Customer (M) → (N) Product [through Order]
- Customer (1) → (N) Review

**Step 5: Constraints**
- Commission rate between 0-30%
- Product price > 0
- Order must have at least one item

**Step 6: ER Diagram**
[Complex diagram with all entities and relationships]

---

## 🎯 Process Best Practices

### Do's ✅
- **Start simple, add complexity gradually**
- **Validate each step with stakeholders**  
- **Document decisions and reasoning**
- **Consider future requirements**
- **Plan for performance from the start**

### Don'ts ❌
- **Skip requirements analysis** 
- **Mix entities with attributes**
- **Ignore business rules**
- **Create overly complex initial design**
- **Forget about data access patterns**

---

## 🎤 Interview Application

### Common Interview Question
**"Walk me through how you would design a database for [X system]"**

**Professional Response Framework**:

1. **"First, I'd analyze the requirements to understand..."**
   - Key business entities
   - User workflows  
   - Business rules and constraints

2. **"Next, I'd identify the main entities like..."**
   - List 4-6 core entities
   - Briefly explain each

3. **"Then I'd define relationships between entities..."**
   - Identify 1:N and M:N relationships
   - Mention junction tables for M:N

4. **"Finally, I'd add constraints to enforce business rules..."**
   - Primary/foreign keys
   - Business-specific constraints

5. **"I'd validate this with an ER diagram and stakeholders"**

---

## 📋 Chapter Summary

### The 6-Step Process
1. **Requirements Analysis** - Understand the business
2. **Entity Identification** - Find the "things"  
3. **Attribute Definition** - Define properties
4. **Relationship Modeling** - Connect entities
5. **Constraint Application** - Enforce rules
6. **ER Diagram** - Visual validation

### Key Principles
- **Systematic approach prevents mistakes**
- **Each step builds on the previous**  
- **Documentation is crucial for communication**
- **Validation with stakeholders is essential**

---

## 🚀 Next Chapter

You now understand the systematic process. Let's dive deep into **identifying entities** - the building blocks of your database.

**Next**: [Chapter 3: Entities](../Chapter-3-Entities/)

---

*Part of Module 1: Database Design Fundamentals*
# Chapter 7: Database Security

## 1. Authentication & Authorization

### Authentication (Who Are You?)

**Definition:** Verifying user identity - confirming they are who they claim to be.

```
┌─────────────────────────────────────────────────────┐
│ AUTHENTICATION PROCESS                               │
├─────────────────────────────────────────────────────┤
│                                                      │
│ User: "I'm Alice"                                    │
│ System: "Prove it!"                                  │
│                                                      │
│ User: "My username is alice_smith"                  │
│ User: "My password is SecurePass123"                │
│                                                      │
│ System: Checks credentials against database         │
│ System: "✓ Verified! You are Alice"                 │
│                                                      │
│ Result: User is AUTHENTICATED                       │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Authorization (What Can You Do?)

**Definition:** Determining what authenticated user can access - permissions and privileges.

```
┌─────────────────────────────────────────────────────┐
│ AUTHORIZATION PROCESS                                │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Alice (Authenticated): "Can I see customer data?"   │
│ System: Checks Alice's role and permissions         │
│ System: "✓ You have 'customer_viewer' role"         │
│ System: "✓ You can SELECT from customers table"     │
│ System: "✗ You CANNOT DELETE from customers"        │
│                                                      │
│ Result: User is AUTHORIZED for specific actions    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Authentication vs Authorization

```
┌──────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION vs AUTHORIZATION                │
├──────────────────────────┬──────────────────────────────────────┤
│ Authentication           │ Authorization                        │
├──────────────────────────┼──────────────────────────────────────┤
│ WHO are you?             │ WHAT can you do?                     │
│ Verify identity          │ Grant permissions                    │
│ Username + Password      │ Roles + Privileges                   │
│ One-time check           │ Checked for each action              │
│ Example: Login           │ Example: Access control              │
├──────────────────────────┼──────────────────────────────────────┤
│ ✓ Authenticated          │ ✓ Authorized                         │
│ ✗ Not authenticated      │ ✗ Not authorized                     │
│ (Can't login)            │ (Can login but can't access)         │
└──────────────────────────┴──────────────────────────────────────┘
```

### User Roles and Permissions

**Role-Based Access Control (RBAC):**

```
┌─────────────────────────────────────────────────────┐
│ ROLE HIERARCHY                                       │
├─────────────────────────────────────────────────────┤
│                                                      │
│ ADMIN ROLE                                           │
│ ├── SELECT (all tables)                             │
│ ├── INSERT (all tables)                             │
│ ├── UPDATE (all tables)                             │
│ ├── DELETE (all tables)                             │
│ └── CREATE/DROP tables                              │
│                                                      │
│ MANAGER ROLE                                         │
│ ├── SELECT (customers, orders)                      │
│ ├── UPDATE (orders)                                 │
│ └── INSERT (orders)                                 │
│                                                      │
│ VIEWER ROLE                                          │
│ └── SELECT (customers, orders) - READ ONLY          │
│                                                      │
│ GUEST ROLE                                           │
│ └── SELECT (public_data only)                       │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**SQL Implementation:**

```sql
-- Create roles
CREATE ROLE admin_role;
CREATE ROLE manager_role;
CREATE ROLE viewer_role;

-- Grant permissions to roles
GRANT ALL PRIVILEGES ON database.* TO admin_role;
GRANT SELECT, INSERT, UPDATE ON database.orders TO manager_role;
GRANT SELECT ON database.customers TO viewer_role;

-- Create user and assign role
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'SecurePass123';
GRANT admin_role TO 'alice'@'localhost';

-- Verify permissions
SHOW GRANTS FOR 'alice'@'localhost';
```

---

## 2. SQL Injection & Prevention

### What is SQL Injection?

**Definition:** Attacker inserts malicious SQL code into input fields to manipulate database queries.

```
┌─────────────────────────────────────────────────────┐
│ SQL INJECTION ATTACK                                 │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Vulnerable Code:                                     │
│ username = request.get('username')                  │
│ password = request.get('password')                  │
│ query = "SELECT * FROM users WHERE                  │
│          username='" + username + "' AND            │
│          password='" + password + "'"               │
│                                                      │
│ Normal Input:                                        │
│ username = "alice"                                   │
│ password = "pass123"                                │
│ Query: SELECT * FROM users WHERE                    │
│        username='alice' AND password='pass123'      │
│ Result: ✓ Works correctly                           │
│                                                      │
│ MALICIOUS INPUT (SQL Injection):                    │
│ username = "admin' --"                              │
│ password = "anything"                               │
│ Query: SELECT * FROM users WHERE                    │
│        username='admin' --' AND password='anything' │
│        ↑                  ↑                          │
│        Closes string      Comments out rest         │
│ Result: ✗ Returns admin user without password check!│
│                                                      │
└─────────────────────────────────────────────────────┘
```

### SQL Injection Examples

**Example 1: Authentication Bypass**

```
Input: username = "' OR '1'='1"
Query: SELECT * FROM users WHERE username='' OR '1'='1'
Result: Returns ALL users (authentication bypassed!)
```

**Example 2: Data Extraction**

```
Input: username = "' UNION SELECT password FROM users --"
Query: SELECT * FROM users WHERE username='' 
       UNION SELECT password FROM users --'
Result: Extracts all passwords!
```

**Example 3: Data Deletion**

```
Input: username = "'; DROP TABLE users; --"
Query: SELECT * FROM users WHERE username=''; 
       DROP TABLE users; --'
Result: Deletes entire users table!
```

### How to Prevent SQL Injection

**Solution 1: Parameterized Queries (BEST)**

```sql
-- VULNERABLE (DON'T DO THIS):
query = "SELECT * FROM users WHERE username='" + username + "'"

-- SAFE (USE THIS):
query = "SELECT * FROM users WHERE username = ?"
statement = connection.prepare(query)
statement.bind(1, username)  -- Username treated as data, not code
result = statement.execute()

-- Even if username = "' OR '1'='1", it's treated as literal string
-- Query becomes: SELECT * FROM users WHERE username = "' OR '1'='1'"
-- Result: Searches for user with that exact name (safe!)
```

**Solution 2: Input Validation**

```sql
-- Whitelist allowed characters
username = username.replaceAll("[^a-zA-Z0-9_]", "")

-- Validate length
if (username.length() > 50) {
    throw new Exception("Username too long")
}

-- Check against allowed values
if (!["admin", "user", "guest"].contains(role)) {
    throw new Exception("Invalid role")
}
```

**Solution 3: Stored Procedures**

```sql
-- Create stored procedure
CREATE PROCEDURE authenticate_user(
    IN p_username VARCHAR(50),
    IN p_password VARCHAR(255)
)
BEGIN
    SELECT * FROM users 
    WHERE username = p_username 
    AND password = SHA2(p_password, 256);
END;

-- Call from application
CALL authenticate_user('alice', 'pass123');

-- Attacker input is passed as parameter, not SQL code
```

**Solution 4: Least Privilege**

```sql
-- Create limited user for application
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'AppPass123';

-- Grant only necessary permissions
GRANT SELECT, INSERT, UPDATE ON database.orders TO 'app_user'@'localhost';

-- DO NOT grant:
-- - DROP, CREATE, ALTER (can't modify schema)
-- - DELETE (can't delete data)
-- - GRANT (can't create new users)

-- If attacker gains access, damage is limited
```

---

## 3. Encryption

### Data at Rest (Stored Data)

**Definition:** Encrypting data stored in database files.

```
┌─────────────────────────────────────────────────────┐
│ ENCRYPTION AT REST                                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│ UNENCRYPTED (Vulnerable):                            │
│ Database file on disk:                               │
│ credit_card: 4532-1234-5678-9010                    │
│ ssn: 123-45-6789                                     │
│ (If disk stolen, data exposed!)                     │
│                                                      │
│ ENCRYPTED (Secure):                                  │
│ Database file on disk:                               │
│ credit_card: 7x9k2m@#$%^&*()_+                      │
│ ssn: 9q8w7e6r5t4y3u2i1o                             │
│ (If disk stolen, data is gibberish!)                │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Implementation:**

```sql
-- Encrypt sensitive columns
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(255),
    credit_card VARCHAR(255),  -- Will be encrypted
    ssn VARCHAR(255)           -- Will be encrypted
);

-- Encrypt data before storing
INSERT INTO users VALUES (
    1,
    'alice',
    'alice@example.com',
    AES_ENCRYPT('4532-1234-5678-9010', 'encryption_key'),
    AES_ENCRYPT('123-45-6789', 'encryption_key')
);

-- Decrypt when retrieving
SELECT 
    user_id,
    username,
    AES_DECRYPT(credit_card, 'encryption_key') as credit_card,
    AES_DECRYPT(ssn, 'encryption_key') as ssn
FROM users
WHERE user_id = 1;
```

### Data in Transit (Network)

**Definition:** Encrypting data sent between client and database.

```
┌─────────────────────────────────────────────────────┐
│ ENCRYPTION IN TRANSIT                                │
├─────────────────────────────────────────────────────┤
│                                                      │
│ UNENCRYPTED (Vulnerable):                            │
│ Client ──────────────────────────────► Database     │
│ "SELECT * FROM users"                               │
│ (Attacker can intercept on network!)                │
│                                                      │
│ ENCRYPTED (Secure):                                  │
│ Client ──────────────────────────────► Database     │
│ "7x9k2m@#$%^&*()_+9q8w7e6r5t4y3u2i1o"             │
│ (Encrypted with SSL/TLS)                            │
│ (Attacker sees gibberish)                           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Implementation:**

```sql
-- Enable SSL/TLS for database connections
-- MySQL Configuration:
[mysqld]
ssl-ca=/path/to/ca.pem
ssl-cert=/path/to/server-cert.pem
ssl-key=/path/to/server-key.pem

-- Require SSL for user connections
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'password'
REQUIRE SSL;

-- Connection string with SSL
mysql -u app_user -p --ssl-mode=REQUIRED -h database.example.com
```

### Encryption Algorithms

```
┌──────────────────────────────────────────────────────────────────┐
│                    ENCRYPTION ALGORITHMS                          │
├──────────────────┬──────────────┬──────────────┬─────────────────┤
│ Algorithm        │ Type         │ Security     │ Use Case        │
├──────────────────┼──────────────┼──────────────┼─────────────────┤
│ AES-256          │ Symmetric    │ ⭐⭐⭐⭐⭐   │ Data at rest    │
│ RSA-2048         │ Asymmetric   │ ⭐⭐⭐⭐    │ Key exchange    │
│ SHA-256          │ Hash         │ ⭐⭐⭐⭐⭐   │ Passwords       │
│ TLS 1.3          │ Protocol     │ ⭐⭐⭐⭐⭐   │ Data in transit │
└──────────────────┴──────────────┴──────────────┴─────────────────┘
```

---

## 4. Auditing & Logging

**Definition:** Recording all database activities for security monitoring and compliance.

```
┌─────────────────────────────────────────────────────┐
│ AUDIT LOG EXAMPLE                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Timestamp: 2024-01-15 10:30:45                      │
│ User: alice                                          │
│ Action: SELECT                                       │
│ Table: customers                                     │
│ Rows Affected: 1000                                  │
│ Status: SUCCESS                                      │
│                                                      │
│ Timestamp: 2024-01-15 10:31:12                      │
│ User: bob                                            │
│ Action: DELETE                                       │
│ Table: orders                                        │
│ Rows Affected: 50                                    │
│ Status: SUCCESS                                      │
│                                                      │
│ Timestamp: 2024-01-15 10:32:00                      │
│ User: attacker                                       │
│ Action: DROP TABLE                                   │
│ Table: users                                         │
│ Status: FAILED (Permission denied)                  │
│ Alert: ⚠️ SUSPICIOUS ACTIVITY DETECTED              │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Implementation:**

```sql
-- Enable audit logging
SET GLOBAL general_log = 'ON';
SET GLOBAL log_output = 'TABLE';

-- View audit logs
SELECT * FROM mysql.general_log
WHERE command_type = 'Query'
ORDER BY event_time DESC
LIMIT 100;

-- Create custom audit table
CREATE TABLE audit_log (
    log_id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    user VARCHAR(50),
    action VARCHAR(50),
    table_name VARCHAR(50),
    rows_affected INT,
    status VARCHAR(20),
    query_text TEXT
);

-- Trigger to log all updates
CREATE TRIGGER audit_user_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (user, action, table_name, rows_affected, status)
    VALUES (USER(), 'UPDATE', 'users', 1, 'SUCCESS');
END;
```

---

## 5. Compliance & Regulations

### GDPR (General Data Protection Regulation)

**Applies to:** Any company handling EU citizen data

```
┌─────────────────────────────────────────────────────┐
│ GDPR REQUIREMENTS                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│ 1. DATA MINIMIZATION                                 │
│    • Collect only necessary data                     │
│    • Don't store unnecessary information             │
│                                                      │
│ 2. CONSENT                                           │
│    • Get explicit user consent before collecting     │
│    • Easy opt-out mechanism                          │
│                                                      │
│ 3. RIGHT TO BE FORGOTTEN                             │
│    • User can request data deletion                  │
│    • Must delete within 30 days                      │
│                                                      │
│ 4. DATA PORTABILITY                                  │
│    • User can export their data                      │
│    • In machine-readable format                      │
│                                                      │
│ 5. ENCRYPTION & SECURITY                             │
│    • Encrypt sensitive data                          │
│    • Implement access controls                       │
│                                                      │
│ PENALTY: Up to €20 million or 4% of revenue         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### PCI-DSS (Payment Card Industry Data Security Standard)

**Applies to:** Any company handling credit card data

```
┌─────────────────────────────────────────────────────┐
│ PCI-DSS REQUIREMENTS                                 │
├─────────────────────────────────────────────────────┤
│                                                      │
│ 1. FIREWALL PROTECTION                               │
│    • Restrict network access to database             │
│                                                      │
│ 2. ENCRYPTION                                        │
│    • Encrypt credit card data at rest                │
│    • Encrypt data in transit (SSL/TLS)               │
│                                                      │
│ 3. ACCESS CONTROL                                    │
│    • Limit access to card data                       │
│    • Use strong authentication                       │
│                                                      │
│ 4. MONITORING & LOGGING                              │
│    • Log all access to card data                     │
│    • Monitor for suspicious activity                 │
│                                                      │
│ 5. VULNERABILITY MANAGEMENT                          │
│    • Regular security testing                        │
│    • Patch management                                │
│                                                      │
│ PENALTY: Up to $100,000 per month                    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### HIPAA (Health Insurance Portability and Accountability Act)

**Applies to:** Healthcare organizations handling patient data

```
┌─────────────────────────────────────────────────────┐
│ HIPAA REQUIREMENTS                                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│ 1. PATIENT PRIVACY                                   │
│    • Protect patient medical records                 │
│    • Limit access to authorized personnel           │
│                                                      │
│ 2. ENCRYPTION                                        │
│    • Encrypt patient data at rest                    │
│    • Encrypt data in transit                         │
│                                                      │
│ 3. AUDIT CONTROLS                                    │
│    • Log all access to patient records               │
│    • Monitor for unauthorized access                 │
│                                                      │
│ 4. BREACH NOTIFICATION                               │
│    • Notify patients of data breaches                │
│    • Within 60 days                                  │
│                                                      │
│ PENALTY: Up to $1.5 million per violation            │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 6. Security Best Practices

```
┌──────────────────────────────────────────────────────────────────┐
│                    SECURITY CHECKLIST                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ✓ AUTHENTICATION                                                  │
│   □ Use strong password requirements                              │
│   □ Implement multi-factor authentication (MFA)                   │
│   □ Use OAuth/SSO for enterprise                                  │
│                                                                   │
│ ✓ AUTHORIZATION                                                   │
│   □ Implement role-based access control (RBAC)                    │
│   □ Follow principle of least privilege                           │
│   □ Regularly audit user permissions                              │
│                                                                   │
│ ✓ SQL INJECTION PREVENTION                                        │
│   □ Use parameterized queries                                     │
│   □ Validate all user input                                       │
│   □ Use stored procedures                                         │
│   □ Implement least privilege for app user                        │
│                                                                   │
│ ✓ ENCRYPTION                                                      │
│   □ Encrypt sensitive data at rest (AES-256)                      │
│   □ Encrypt data in transit (TLS 1.3)                             │
│   □ Use strong key management                                     │
│   □ Rotate encryption keys regularly                              │
│                                                                   │
│ ✓ AUDITING & LOGGING                                              │
│   □ Enable database audit logging                                 │
│   □ Log all administrative actions                                │
│   □ Monitor for suspicious activity                               │
│   □ Retain logs for compliance period                             │
│                                                                   │
│ ✓ NETWORK SECURITY                                                │
│   □ Use firewall to restrict database access                      │
│   □ Disable remote root access                                    │
│   □ Use VPN for remote connections                                │
│   □ Implement network segmentation                                │
│                                                                   │
│ ✓ BACKUP & RECOVERY                                               │
│   □ Regular encrypted backups                                     │
│   □ Test backup restoration                                       │
│   □ Store backups securely                                        │
│   □ Implement disaster recovery plan                              │
│                                                                   │
│ ✓ PATCH MANAGEMENT                                                │
│   □ Keep database software updated                                │
│   □ Apply security patches promptly                               │
│   □ Test patches before production                                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Real-World Example: Banking System

**Scenario:** Secure online banking application

```sql
-- 1. AUTHENTICATION
CREATE USER 'bank_app'@'db.bank.com' IDENTIFIED BY 'StrongPassword123!';
REQUIRE SSL;
REQUIRE X509;

-- 2. AUTHORIZATION - Role-based access
CREATE ROLE customer_service;
CREATE ROLE fraud_detection;
CREATE ROLE admin;

GRANT SELECT ON bank.accounts TO customer_service;
GRANT SELECT ON bank.transactions TO customer_service;

GRANT SELECT ON bank.accounts TO fraud_detection;
GRANT SELECT ON bank.transactions TO fraud_detection;
GRANT SELECT ON bank.audit_log TO fraud_detection;

GRANT ALL PRIVILEGES ON bank.* TO admin;

-- 3. ENCRYPTION - Sensitive data
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    customer_id INT,
    account_number VARCHAR(255),  -- Encrypted
    balance DECIMAL(15,2),         -- Encrypted
    ssn VARCHAR(255),              -- Encrypted
    created_at TIMESTAMP
);

-- Insert with encryption
INSERT INTO accounts VALUES (
    1,
    100,
    AES_ENCRYPT('1234567890123456', 'bank_key'),
    AES_ENCRYPT('50000.00', 'bank_key'),
    AES_ENCRYPT('123-45-6789', 'bank_key'),
    NOW()
);

-- 4. SQL INJECTION PREVENTION - Parameterized query
-- Application code (Java example):
String query = "SELECT * FROM accounts WHERE account_id = ? AND customer_id = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setInt(1, accountId);
stmt.setInt(2, customerId);
ResultSet rs = stmt.executeQuery();

-- 5. AUDITING - Log all transactions
CREATE TABLE transaction_audit (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    user VARCHAR(50),
    action VARCHAR(50),
    account_id INT,
    amount DECIMAL(15,2),
    status VARCHAR(20),
    ip_address VARCHAR(45)
);

-- 6. COMPLIANCE - GDPR & PCI-DSS
-- Data retention policy
DELETE FROM transaction_audit 
WHERE timestamp < DATE_SUB(NOW(), INTERVAL 7 YEARS);

-- Right to be forgotten
DELETE FROM accounts WHERE customer_id = ?;
DELETE FROM transaction_audit WHERE account_id = ?;
```

---

## 8. Interview Questions

### Beginner Level

**Q1: What's the difference between authentication and authorization?**
> Authentication verifies WHO you are (username/password). Authorization determines WHAT you can do (permissions/roles).

**Q2: What is SQL injection?**
> Attacker inserts malicious SQL code into input fields to manipulate queries and access/modify data.

**Q3: How do you prevent SQL injection?**
> Use parameterized queries, validate input, use stored procedures, implement least privilege.

### Intermediate Level

**Q4: Explain encryption at rest vs in transit.**
> At rest: Encrypts data stored in database files. In transit: Encrypts data sent over network (SSL/TLS).

**Q5: What is GDPR and why does it matter?**
> EU regulation protecting personal data. Requires consent, encryption, right to deletion. Penalties up to €20M.

**Q6: How would you design a secure banking database?**
> Use strong authentication (MFA), role-based access, encrypt sensitive data (AES-256), SSL/TLS for transit, comprehensive auditing, PCI-DSS compliance.

### Senior Level

**Q7: Design a multi-tenant SaaS database with security isolation.**
> Separate schemas per tenant, row-level security (RLS), encryption with tenant-specific keys, audit logging per tenant, compliance with GDPR/SOC2.

**Q8: How would you handle a data breach in production?**
> Immediate: Isolate affected systems, preserve logs. Short-term: Notify affected users, rotate credentials, patch vulnerability. Long-term: Post-mortem, improve security, update incident response plan.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                    KEY TAKEAWAYS                                  │
├──────────────────────────────────────────────────────────────────┤
│ 1. Authentication = WHO (verify identity)                        │
│ 2. Authorization = WHAT (grant permissions)                      │
│ 3. SQL Injection = Malicious SQL code in input                   │
│ 4. Prevention = Parameterized queries, input validation           │
│ 5. Encryption at rest = Protect stored data                      │
│ 6. Encryption in transit = Protect network data (SSL/TLS)        │
│ 7. Auditing = Log all database activities                        │
│ 8. Compliance = GDPR, PCI-DSS, HIPAA requirements                │
│ 9. Least privilege = Minimal necessary permissions               │
│ 10. Defense in depth = Multiple security layers                  │
└──────────────────────────────────────────────────────────────────┘
```

---

**Estimated Reading Time**: 45-50 minutes  
**Next Chapter**: [Chapter 8: Production Design Review](../Chapter-8-Production-Design-Review/)

*[← Back to Chapter 6: Transactions & Concurrency](../Chapter-6-Transactions-Concurrency/)*

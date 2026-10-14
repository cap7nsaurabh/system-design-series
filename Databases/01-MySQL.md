# MySQL: The Web's Relational Workhorse

## Overview
MySQL is an open-source, fast, and battle-tested Relational Database Management System (RDBMS) that underpins a vast percentage of the world's web infrastructure (the "M" in the LAMP stack). Highly optimized for read-heavy and transactional OLTP workloads, MySQL powers platforms ranging from WordPress to Meta, GitHub, Uber, and Shopify.

---

## Architecture & Storage Engine Internals
```
+-------------------------------------------------------------+
|                      Client Connection                      |
|                 (Connection Pool, Auth, SSL)                |
+-------------------------------------------------------------+
                              │
                              ▼
+-------------------------------------------------------------+
|                     MySQL Server Layer                      |
|  - SQL Parser / Preprocessor                                |
|  - Query Optimizer (Cost-based optimizer)                   |
|  - Query Cache (Removed in 8.0)                             |
|  - Buffer / Caches (Sort buffer, Join buffer)               |
+-------------------------------------------------------------+
                              │
                              ▼
+-------------------------------------------------------------+
|             Pluggable Storage Engine Interface              |
|   ┌────────────────┬────────────────┬───────────────────┐   |
|   │     InnoDB     │     MyISAM     │      Memory       │   |
|   │ (ACID default) │ (Non-tx, fast) │ (In-RAM hash/b-tr)│   |
|   └────────────────┴────────────────┴───────────────────┘   |
+-------------------------------------------------------------+
                              │
                              ▼
+-------------------------------------------------------------+
|                Operating System & File System               |
|      (Data files .ibd, Redo Logs, Undo Logs, Binlog)        |
+-------------------------------------------------------------+
```

### 1. InnoDB Storage Engine (The Modern Standard)
- **Clustered Index:** Primary key rows are physically stored directly in the leaf pages of the B+ Tree. Secondary indexes store the Primary Key value as the row pointer (requiring a double lookup unless covering).
- **Buffer Pool:** In-memory caching area for pages, dirty page flushing via LRU algorithm.
- **Write-Ahead Logging (WAL):**
  - **Redo Log:** Ensures Durability (crash recovery). Writes changes sequentially to disk before writing dirty pages.
  - **Undo Log:** Ensures Atomicity and enables MVCC (Multi-Version Concurrency Control) for rollback and non-blocking reads.
  - **Doublewrite Buffer:** Prevents partial page write corruption during OS-level torn writes.
  - **Binary Log (Binlog):** Server-level statement/row-based replication log used for read replicas and point-in-time recovery.

---

## Real World Stories & Incidents

### 1. The GitHub MySQL Replication Split-Brain (2018)
GitHub experienced an extended outage when a 43-second network partition occurred between their primary US East Coast datacenter and their secondary datacenter.
- The automated failover tool promoted a replica in the secondary datacenter to master.
- When the network healed, both databases received writes simultaneously (**split-brain**).
- Reconciling foreign key collisions, auto-increment primary key divergences, and out-of-order binlog streams required engineering teams to spend over 24 hours manually de-conflicting records using custom binlog parsing scripts.

### 2. Uber's Migration from Postgres to Schemaless MySQL
In 2016, Uber published their decision to migrate their core trips architecture from PostgreSQL to a sharded MySQL system called **Schemaless**:
- In PostgreSQL, updating an existing row creates a new physical tuple (Write Amplification), requiring every single secondary index on that table to be updated with the new tuple pointer.
- In MySQL InnoDB, secondary indexes point to the Primary Key, not physical tuple offsets. When an update occurs that doesn't modify indexed columns, secondary indexes remain untouched. This predictable write throughput allowed Uber to scale to millions of concurrent rides.

### 3. YouTube and Vitess
As YouTube's video database exploded in 2010, no single MySQL instance could hold the view counts and video metadata. YouTube built **Vitess**, a database clustering system that orchestrates thousands of sharded MySQL instances while presenting a single logical database abstraction to application servers. Today, Vitess powers YouTube, Slack, GitHub, and Cash App.

---

## Comprehensive MySQL Queries & Supported Methods Reference

### 1. Data Definition Language (DDL)

```sql
-- Create Database with modern UTF-8 encoding
CREATE DATABASE IF NOT EXISTS ecommerce_db 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

USE ecommerce_db;

-- Create Table with Primary Key, Foreign Key, Enum, and Constraints
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    status ENUM('active', 'suspended', 'pending') DEFAULT 'pending',
    balance DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    profile_data JSON NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT chk_balance CHECK (balance >= 0.00)
) ENGINE=InnoDB;

-- Create Table with Composite Primary Key & Foreign Keys
CREATE TABLE orders (
    order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    order_status VARCHAR(20) DEFAULT 'CREATED',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB;

-- Alter Table operations
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20) AFTER email;
ALTER TABLE users MODIFY COLUMN username VARCHAR(80) NOT NULL;
ALTER TABLE users RENAME COLUMN phone_number TO mobile_number;
ALTER TABLE users DROP COLUMN mobile_number;

-- Creating Indexes
CREATE INDEX idx_orders_user_status ON orders(user_id, order_status);
CREATE UNIQUE INDEX idx_user_email ON users(email);

-- Drop & Truncate
TRUNCATE TABLE orders; -- Fast reset: deallocates data pages, resets auto-increment
DROP TABLE IF EXISTS orders;
```

---

### 2. Data Manipulation Language (DML)

```sql
-- Single and Multi-Row Insert
INSERT INTO users (username, email, password_hash, status, balance)
VALUES ('alice', 'alice@test.com', 'hash_123', 'active', 500.00),
       ('bob', 'bob@test.com', 'hash_456', 'active', 250.00);

-- INSERT ... ON DUPLICATE KEY UPDATE (MySQL Upsert)
INSERT INTO users (user_id, username, email, password_hash, balance)
VALUES (1, 'alice_updated', 'alice@test.com', 'hash_new', 600.00)
ON DUPLICATE KEY UPDATE 
    username = VALUES(username),
    balance = balance + VALUES(balance);

-- INSERT ... SELECT
INSERT INTO archived_users (user_id, username, email)
SELECT user_id, username, email FROM users WHERE status = 'suspended';

-- Update with JOIN
UPDATE orders o
JOIN users u ON o.user_id = u.user_id
SET o.order_status = 'VIP_PRIORITY'
WHERE u.balance > 10000.00;

-- Delete with Limit to prevent runaway table locking
DELETE FROM orders 
WHERE order_status = 'CANCELLED' AND created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
LIMIT 1000;
```

---

### 3. Data Query Language (DQL) & Advanced Querying

#### Joins & Set Operations
```sql
-- INNER, LEFT, RIGHT, and FULL OUTER (Emulated via UNION)
SELECT u.username, o.order_id, o.total_amount
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
WHERE o.order_status = 'COMPLETED';

-- Full Outer Join (Emulated in MySQL using LEFT + RIGHT + UNION)
SELECT u.user_id, o.order_id
FROM users u LEFT JOIN orders o ON u.user_id = o.user_id
UNION
SELECT u.user_id, o.order_id
FROM users u RIGHT JOIN orders o ON u.user_id = o.user_id;
```

#### Window Functions (MySQL 8.0+)
```sql
SELECT 
    order_id,
    user_id,
    total_amount,
    -- Rank orders per user by amount
    RANK() OVER (PARTITION BY user_id ORDER BY total_amount DESC) as user_rank,
    -- Continuous row number
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at ASC) as order_sequence,
    -- Running total spend per user
    SUM(total_amount) OVER (PARTITION BY user_id ORDER BY created_at) as running_total,
    -- Lag: Previous order value
    LAG(total_amount, 1, 0.00) OVER (PARTITION BY user_id ORDER BY created_at) as prev_order_amount,
    -- Lead: Next order value
    LEAD(total_amount, 1, 0.00) OVER (PARTITION BY user_id ORDER BY created_at) as next_order_amount
FROM orders;
```

#### Common Table Expressions (CTEs) & Recursive Queries
```sql
-- Non-Recursive CTE
WITH HighValueUsers AS (
    SELECT user_id, SUM(total_amount) as total_spent
    FROM orders
    GROUP BY user_id
    HAVING total_spent > 1000.00
)
SELECT u.username, h.total_spent
FROM users u
JOIN HighValueUsers h ON u.user_id = h.user_id;

-- Recursive CTE: Organizational Hierarchy / Tree Traversal
WITH RECURSIVE OrgChart AS (
    -- Anchor member: CEO
    SELECT employee_id, name, manager_id, 1 as depth
    FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    -- Recursive member: Direct reports
    SELECT e.employee_id, e.name, e.manager_id, o.depth + 1
    FROM employees e
    INNER JOIN OrgChart o ON e.manager_id = o.employee_id
)
SELECT * FROM OrgChart ORDER BY depth, manager_id;
```

---

### 4. JSON Functions (Native MySQL 5.7+ / 8.0+)

```sql
-- Storing and Querying JSON
INSERT INTO users (username, email, password_hash, profile_data)
VALUES ('charlie', 'charlie@test.com', 'pwd_789', 
        '{"theme": "dark", "tags": ["tech", "gaming"], "settings": {"notifications": {"email": true, "sms": false}}}');

-- Extracting values (-> returns quoted JSON, ->> returns unquoted string)
SELECT 
    username,
    profile_data->>'$.theme' AS theme_setting,
    profile_data->'$.tags[0]' AS primary_tag,
    JSON_EXTRACT(profile_data, '$.settings.notifications.email') AS email_notify
FROM users;

-- JSON Modification Functions
UPDATE users 
SET profile_data = JSON_SET(profile_data, '$.theme', 'light', '$.settings.two_factor', true)
WHERE username = 'charlie';

-- Removing keys
UPDATE users 
SET profile_data = JSON_REMOVE(profile_data, '$.tags[1]')
WHERE username = 'charlie';

-- Check JSON Membership
SELECT * FROM users 
WHERE JSON_CONTAINS(profile_data, '"gaming"', '$.tags');
```

---

### 5. Transaction Control Language (TCL) & Locking

```sql
-- Explicit Transaction with Pessimistic Locking
START TRANSACTION;

-- Exclusive lock on row (Blocks concurrent writes and reads with locks)
SELECT balance FROM users WHERE user_id = 1 FOR UPDATE;

-- Shared lock (Allows concurrent reads, blocks writes)
SELECT balance FROM users WHERE user_id = 2 LOCK IN SHARE MODE;

-- Perform business logic atomically
UPDATE users SET balance = balance - 100.00 WHERE user_id = 1;
UPDATE users SET balance = balance + 100.00 WHERE user_id = 2;

-- Savepoints within transaction
SAVEPOINT debit_completed;

-- Rollback to savepoint if needed
-- ROLLBACK TO SAVEPOINT debit_completed;

-- Finalize
COMMIT;
```

---

### 6. Built-in String, Date, and Aggregate Functions

```sql
-- String Functions
SELECT 
    CONCAT(first_name, ' ', last_name) AS full_name,
    SUBSTRING(email, 1, LOCATE('@', email) - 1) AS email_handle,
    UPPER(status),
    LENGTH(password_hash),
    REPLACE(username, '_', ' ')
FROM users;

-- Date & Time Functions
SELECT 
    NOW() AS current_datetime,
    CURDATE() AS current_date,
    DATE_ADD(NOW(), INTERVAL 7 DAY) AS expires_at,
    DATEDIFF(NOW(), created_at) AS account_age_days,
    DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:%s') AS formatted_date
FROM users;

-- Advanced Aggregations
SELECT 
    order_status,
    COUNT(*) as total_orders,
    SUM(total_amount) as gross_revenue,
    AVG(total_amount) as aov,
    -- Concatenate values into comma-separated string
    GROUP_CONCAT(order_id ORDER BY total_amount DESC SEPARATOR '; ') as top_order_ids
FROM orders
GROUP BY order_status;
```

---

### 7. Administrative, Performance, & EXPLAIN Profiling

```sql
-- Query Profiling with EXPLAIN and EXPLAIN ANALYZE (MySQL 8.0)
EXPLAIN ANALYZE
SELECT u.username, SUM(o.total_amount)
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.created_at >= '2024-01-01'
GROUP BY u.username;

-- Process Inspection & Connection Management
SHOW FULL PROCESSLIST; -- View currently running queries and thread states
KILL QUERY 1042;       -- Kill runaway query without terminating connection
KILL 1042;             -- Kill connection thread

-- Engine and Status Metrics
SHOW ENGINE INNODB STATUS; -- Detailed deadlock information, buffer pool hit rates
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Slow_queries';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

---

## MySQL Performance Tuning & Best Practices

1. **Size InnoDB Buffer Pool Appropriately:** Set `innodb_buffer_pool_size` to $60\%\text{--}75\%$ of total system RAM on dedicated database servers.
2. **Avoid `SELECT *`:** Querying unneeded columns forces MySQL to do full row lookups instead of utilizing **Covering Indexes**.
3. **Beware of Implicit Type Conversion:** Comparing a `VARCHAR` column to an integer literal disables index usage completely (`WHERE phone_number = 12345` does a full table scan; use `'12345'`).
4. **Partitioning vs Sharding:** Native MySQL table partitioning works well for archival (dropping old range partitions instantaneously via `ALTER TABLE DROP PARTITION`), but horizontal sharding (via Vitess) is required for write scaling beyond a single host.

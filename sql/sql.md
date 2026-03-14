# PostgreSQL — Complete SDE Interview Guide

A comprehensive reference covering all core PostgreSQL concepts with real-world examples, designed for software engineering interviews.

---

## Table of Contents

1. [SQL Basics — DDL, DML, DCL](#1-sql-basics--ddl-dml-dcl)
2. [Data Types in PostgreSQL](#2-data-types-in-postgresql)
3. [Filtering and Sorting — WHERE, ORDER BY, LIMIT](#3-filtering-and-sorting--where-order-by-limit)
4. [Joins — INNER, LEFT, RIGHT, FULL, SELF, CROSS](#4-joins--inner-left-right-full-self-cross)
5. [Aggregations — GROUP BY and HAVING](#5-aggregations--group-by-and-having)
6. [Subqueries and Nested Queries](#6-subqueries-and-nested-queries)
7. [Window Functions — ROW_NUMBER, RANK, LAG, LEAD](#7-window-functions--row_number-rank-lag-lead)
8. [Constraints, Keys and Normalization](#8-constraints-keys-and-normalization)
9. [Stored Procedures and Functions](#9-stored-procedures-and-functions)
10. [Transactions and ACID Properties](#10-transactions-and-acid-properties)
11. [Indexes and Query Optimization](#11-indexes-and-query-optimization)

---

## 1. SQL Basics — DDL, DML, DCL

SQL commands are grouped into three categories based on what they operate on.

| Category | Full Form | Purpose |
|----------|-----------|---------|
| DDL | Data Definition Language | Create, modify, and delete database structure (tables, indexes, schemas) |
| DML | Data Manipulation Language | Insert, update, delete, and query data |
| DCL | Data Control Language | Grant and revoke user permissions |

### Real-World Scenario: Building an E-Commerce Platform

---

### DDL — Data Definition Language

DDL deals with the **schema** — the blueprint of your database. These changes are structural.

```sql
-- Create the products table
CREATE TABLE products (
    product_id       SERIAL PRIMARY KEY,
    name             VARCHAR(255) NOT NULL,
    price            DECIMAL(10,2) NOT NULL,
    category         VARCHAR(100),
    created_at       TIMESTAMP DEFAULT NOW()
);

-- Add a new column (e.g., introducing a discount feature later)
ALTER TABLE products ADD COLUMN discount_percent NUMERIC(5,2) DEFAULT 0;

-- Rename a column
ALTER TABLE products RENAME COLUMN name TO product_name;

-- Delete the entire table — this is irreversible
DROP TABLE products;

-- Remove all rows but keep the table structure
TRUNCATE TABLE products;
```

> **Interview Note:** `TRUNCATE` is faster than `DELETE` for clearing all rows because it does not scan rows one by one. In PostgreSQL, `TRUNCATE` inside a transaction **can** be rolled back, unlike in some other databases.

---

### DML — Data Manipulation Language

DML is what you use in day-to-day application work — reading and writing data.

```sql
-- Add a single product
INSERT INTO products (product_name, price, category)
VALUES ('iPhone 15', 79999.00, 'Electronics');

-- Insert multiple rows in one statement (more efficient than individual inserts)
INSERT INTO products (product_name, price, category) VALUES
    ('Samsung S24',  69999.00, 'Electronics'),
    ('Nike Shoes',    4999.00, 'Footwear'),
    ('Coffee Maker',  2999.00, 'Appliances');

-- Apply a sale discount to all electronics
UPDATE products
SET discount_percent = 10
WHERE category = 'Electronics';

-- Remove a discontinued product
DELETE FROM products
WHERE product_id = 5;

-- Retrieve all electronics sorted by price
SELECT product_name, price, discount_percent
FROM products
WHERE category = 'Electronics'
ORDER BY price DESC;
```

---

### DCL — Data Control Language

DCL controls who can do what in the database. This is critical in multi-team environments where data access must be restricted.

```sql
-- Create a read-only role for analysts
CREATE ROLE analyst_role;

-- Grant only SELECT access
GRANT SELECT ON products TO analyst_role;

-- Grant full write access to a developer
GRANT SELECT, INSERT, UPDATE, DELETE ON products TO dev_user;

-- Remove write permissions from analysts
REVOKE INSERT, UPDATE, DELETE ON products FROM analyst_role;

-- Assign the role to a specific user
GRANT analyst_role TO neha;
```

> **Real World:** At companies like Zomato or Swiggy, data analysts get `SELECT`-only access to prevent accidental data modification. Backend engineers get `INSERT`/`UPDATE` rights. Only DBAs have `DROP` privileges.

---

## 2. Data Types in PostgreSQL

Choosing the correct data type affects storage size, query performance, and data integrity. PostgreSQL has one of the richest type systems of any relational database.

---

### Numeric Types

| Type | Storage | Range | Use Case |
|------|---------|-------|----------|
| `SMALLINT` | 2 bytes | -32768 to 32767 | Age, star rating (1-5), small counters |
| `INTEGER` | 4 bytes | ~-2.1B to 2.1B | User IDs, order counts |
| `BIGINT` | 8 bytes | ~-9.2 Quintillion | Transaction IDs, tweet IDs, snowflake IDs |
| `SERIAL` | 4 bytes | Auto-increment | Auto-generated primary keys |
| `DECIMAL(p,s)` | Variable | User-defined | Money: `DECIMAL(10,2)` = up to 99999999.99 |
| `FLOAT` / `REAL` | 4-8 bytes | Approximate | Scientific values — never for money |

> **Warning:** Never use `FLOAT` or `REAL` for currency. Floating-point arithmetic is approximate. `0.1 + 0.2` in float equals `0.30000000000000004`. Always use `DECIMAL` or `NUMERIC` for financial values.

---

### Text Types

| Type | Behavior | Use Case |
|------|----------|----------|
| `CHAR(n)` | Fixed length, padded with spaces | Country codes: `CHAR(2)` = `'IN'` |
| `VARCHAR(n)` | Variable length, max n characters | Names, emails: `VARCHAR(255)` |
| `TEXT` | Unlimited length | Product descriptions, blog post content |

---

### Date and Time Types

```sql
-- DATE: stores only the date, no time component
birth_date DATE  -- e.g. '1995-08-15'

-- TIME: stores only the time, no date
shift_start TIME  -- e.g. '09:00:00'

-- TIMESTAMP: date + time, no timezone
created_at TIMESTAMP DEFAULT NOW()

-- TIMESTAMPTZ: date + time WITH timezone (recommended for production)
-- stores in UTC, displays in the session's local timezone
event_time TIMESTAMPTZ

-- INTERVAL: duration arithmetic
SELECT NOW() - created_at AS account_age FROM users;
SELECT created_at + INTERVAL '7 days' AS expires_at FROM subscriptions;
```

> **Production Tip:** Always use `TIMESTAMPTZ` for apps that serve users across timezones. It stores in UTC internally and converts to the session timezone automatically.

---

### Other Important Types

```sql
-- BOOLEAN
is_active BOOLEAN DEFAULT TRUE

-- UUID — universally unique identifier (great for distributed systems)
user_id UUID DEFAULT gen_random_uuid()

-- ARRAY — store a list of values in a single column
tags TEXT[] DEFAULT '{}'

INSERT INTO articles (tags) VALUES (ARRAY['sql', 'postgres', 'backend']);
SELECT * FROM articles WHERE 'sql' = ANY(tags);

-- JSONB — binary JSON (faster queries than plain JSON)
metadata JSONB

SELECT metadata->>'city' FROM users;
SELECT * FROM users WHERE metadata @> '{"plan": "premium"}';

-- ENUM — restrict a column to a fixed set of values
CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'delivered', 'cancelled');
status order_status DEFAULT 'pending';
```

---

## 3. Filtering and Sorting — WHERE, ORDER BY, LIMIT

### Sample Schema — Food Delivery App

```sql
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    restaurant    VARCHAR(100),
    city          VARCHAR(50),
    amount        DECIMAL(8,2),
    rating        SMALLINT,        -- 1 to 5
    status        VARCHAR(20),     -- 'delivered', 'cancelled', 'pending'
    ordered_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

### WHERE — Filtering Rows

`WHERE` filters rows **before** any aggregation. It acts as the first gate in the query execution pipeline.

```sql
-- Single condition
SELECT * FROM orders WHERE city = 'Mumbai';

-- Multiple conditions with AND / OR
SELECT * FROM orders
WHERE city = 'Delhi' AND amount > 500 AND status = 'delivered';

-- NOT operator
SELECT * FROM orders WHERE NOT status = 'cancelled';

-- BETWEEN — inclusive on both ends
SELECT * FROM orders WHERE amount BETWEEN 200 AND 1000;

-- IN — match against a list (cleaner than multiple OR conditions)
SELECT * FROM orders WHERE city IN ('Mumbai', 'Delhi', 'Bengaluru');

-- LIKE — pattern matching
-- % matches any number of characters
-- _ matches exactly one character
SELECT * FROM orders WHERE customer_name LIKE 'Raj%';       -- starts with Raj
SELECT * FROM orders WHERE restaurant LIKE '%Biryani%';      -- contains Biryani
SELECT * FROM orders WHERE customer_name LIKE 'A__l';        -- e.g. 'Amul', 'Atul'

-- IS NULL / IS NOT NULL
SELECT * FROM orders WHERE rating IS NULL;    -- orders without a rating

-- Combining multiple filters
SELECT order_id, customer_name, amount
FROM orders
WHERE city = 'Pune'
  AND amount > 300
  AND status IN ('delivered', 'pending')
  AND ordered_at >= '2024-01-01';
```

---

### ORDER BY — Sorting Results

```sql
-- Highest amount first
SELECT * FROM orders ORDER BY amount DESC;

-- Sort by city alphabetically, then by amount within each city
SELECT * FROM orders ORDER BY city ASC, amount DESC;

-- Sort by a computed column
SELECT *, (amount * 0.18) AS gst FROM orders ORDER BY gst DESC;

-- NULLS FIRST / NULLS LAST — PostgreSQL-specific control
SELECT * FROM orders ORDER BY rating ASC NULLS LAST;
```

---

### LIMIT and OFFSET — Pagination

```sql
-- Top 10 most expensive orders
SELECT * FROM orders ORDER BY amount DESC LIMIT 10;

-- Page 3, 10 items per page (offset = (page - 1) * page_size)
SELECT * FROM orders ORDER BY ordered_at DESC LIMIT 10 OFFSET 20;
```

> **Interview Note:** `OFFSET`-based pagination becomes slow on large datasets. `OFFSET 1000000` still forces the database to scan and discard 1 million rows. The preferred approach for large tables is **cursor-based pagination**:
> ```sql
> -- Cursor-based: O(log n) with an index, vs O(n) for OFFSET
> SELECT * FROM orders WHERE order_id > :last_seen_id ORDER BY order_id LIMIT 10;
> ```

---

### DISTINCT — Remove Duplicates

```sql
-- Unique cities that have delivered orders
SELECT DISTINCT city FROM orders WHERE status = 'delivered';

-- DISTINCT ON (PostgreSQL-specific): keep one row per group
-- Get the most recent order per customer
SELECT DISTINCT ON (customer_name)
    customer_name, order_id, amount, ordered_at
FROM orders
ORDER BY customer_name, ordered_at DESC;
```

---

## 4. Joins — INNER, LEFT, RIGHT, FULL, SELF, CROSS

### Sample Schema — HR Management System

```sql
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    dept_id    INT,      -- foreign key to departments
    manager_id INT,      -- self-referencing foreign key
    salary     DECIMAL(10,2)
);

CREATE TABLE departments (
    dept_id   SERIAL PRIMARY KEY,
    dept_name VARCHAR(100),
    location  VARCHAR(50)
);
```

---

### INNER JOIN — Only Matching Rows

Returns rows where the join condition is satisfied in **both** tables. Rows with no match on either side are excluded.

```sql
-- Get employee name with their department name
-- Employees with NULL dept_id are excluded
SELECT e.name, d.dept_name, d.location
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- Join across three tables
SELECT e.name, d.dept_name, p.proj_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN projects p    ON d.dept_id = p.dept_id;
```

---

### LEFT JOIN — All Rows from the Left Table

Returns **all** rows from the left table and matching rows from the right. Where there is no match, the right side columns are `NULL`.

```sql
-- All employees, even those not assigned to any department
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;

-- Find employees with NO department (common interview question)
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

---

### RIGHT JOIN — All Rows from the Right Table

```sql
-- All departments, even those with no employees assigned
SELECT e.name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

> **Note:** `RIGHT JOIN` is rarely used in practice. Most developers rewrite it as a `LEFT JOIN` by swapping the table order. It exists primarily for symmetry.

---

### FULL OUTER JOIN — All Rows from Both Tables

```sql
-- All employees and all departments — NULL where there is no match on either side
SELECT e.name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

---

### SELF JOIN — A Table Joining Itself

Used when a table has a column that references another row in the same table. The classic case is an employee-manager hierarchy.

```sql
-- Get each employee alongside their manager's name
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
-- Top-level executives have NULL for manager
```

---

### CROSS JOIN — Cartesian Product

Returns every possible combination of rows from both tables. Use with caution — 1000 rows x 1000 rows = 1,000,000 result rows.

```sql
-- Generate all size-color combinations for a product variant system
SELECT sizes.size, colors.color
FROM
    (VALUES ('S'), ('M'), ('L'), ('XL')) AS sizes(size)
CROSS JOIN
    (VALUES ('Red'), ('Blue'), ('Black')) AS colors(color);
-- Result: 4 x 3 = 12 rows
```

---

### Join Types Summary

| Join Type | Returns | Common Use Case |
|-----------|---------|-----------------|
| INNER JOIN | Matching rows only | Most common — fetch related data |
| LEFT JOIN | All left + matching right | Find unmatched or orphan records |
| RIGHT JOIN | All right + matching left | Same as LEFT JOIN with tables swapped |
| FULL OUTER JOIN | All rows from both tables | Find unmatched records on either side |
| SELF JOIN | Table joined to itself | Hierarchies, same-table comparisons |
| CROSS JOIN | Cartesian product | Generate combinations |

---

## 5. Aggregations — GROUP BY and HAVING

### Aggregate Functions

Aggregate functions reduce multiple rows into a single summary value.

| Function | Behavior | Example |
|----------|----------|---------|
| `COUNT(*)` | Total number of rows | Total orders |
| `COUNT(col)` | Count of non-NULL values | Count only rated orders |
| `SUM(col)` | Total of all values | Total revenue |
| `AVG(col)` | Average of all values | Average order value |
| `MAX(col)` | Highest value | Most expensive order |
| `MIN(col)` | Lowest value | Cheapest order |

---

### GROUP BY — Aggregate Per Group

`GROUP BY` divides rows into groups and applies aggregate functions independently to each group.

```sql
-- Total revenue per city
SELECT city, SUM(amount) AS total_revenue, COUNT(*) AS order_count
FROM orders
GROUP BY city
ORDER BY total_revenue DESC;

-- Average order value per restaurant per city
SELECT restaurant, city,
       AVG(amount)  AS avg_order,
       COUNT(*)     AS order_count
FROM orders
WHERE status = 'delivered'
GROUP BY restaurant, city
ORDER BY avg_order DESC;

-- Daily revenue report
SELECT DATE_TRUNC('day', ordered_at) AS order_date,
       COUNT(*)       AS daily_orders,
       SUM(amount)    AS daily_revenue
FROM orders
GROUP BY DATE_TRUNC('day', ordered_at)
ORDER BY order_date;
```

> **Rule:** Every column in `SELECT` must either appear in `GROUP BY` or be wrapped inside an aggregate function. Violating this is a syntax error in PostgreSQL.

---

### HAVING — Filter Groups

`HAVING` filters **after** grouping. `WHERE` filters rows before grouping. They are not interchangeable.

```sql
-- Cities with more than 1000 delivered orders
SELECT city, COUNT(*) AS order_count
FROM orders
WHERE status = 'delivered'        -- filters rows before grouping
GROUP BY city
HAVING COUNT(*) > 1000            -- filters groups after aggregation
ORDER BY order_count DESC;

-- Low-performing restaurants (min 50 orders to be statistically fair)
SELECT restaurant, AVG(rating) AS avg_rating, COUNT(*) AS total_orders
FROM orders
GROUP BY restaurant
HAVING AVG(rating) < 3.5 AND COUNT(*) > 50
ORDER BY avg_rating;
```

---

### Query Execution Order

Understanding this sequence is critical for debugging incorrect queries and alias-related errors.

| Step | Clause | What Happens |
|------|--------|--------------|
| 1 | FROM / JOIN | Determine source tables and build the working dataset |
| 2 | WHERE | Filter individual rows |
| 3 | GROUP BY | Group remaining rows |
| 4 | HAVING | Filter groups |
| 5 | SELECT | Compute expressions and pick columns |
| 6 | DISTINCT | Remove duplicate rows |
| 7 | ORDER BY | Sort the result set |
| 8 | LIMIT / OFFSET | Paginate |

> **Interview Classic:** Why can you not use a `SELECT` alias in `WHERE`? Because `WHERE` executes at step 2, before `SELECT` at step 5. The alias does not exist yet. PostgreSQL allows aliases in `ORDER BY` as a convenience, but it is not standard SQL.

---

## 6. Subqueries and Nested Queries

A subquery is a query nested inside another query. They make complex problems decomposable into smaller, readable steps.

### Types of Subqueries

| Type | Returns | Where It Can Appear |
|------|---------|---------------------|
| Scalar Subquery | One row, one column | SELECT, WHERE, HAVING |
| Row Subquery | One row, multiple columns | WHERE with row comparison |
| Table Subquery (derived table) | Multiple rows and columns | FROM clause |
| Correlated Subquery | Depends on the outer row | WHERE, HAVING — executes once per outer row |

---

### Scalar Subquery

```sql
-- Orders above the overall average amount
SELECT order_id, customer_name, amount
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);

-- Show each order's deviation from the overall average
SELECT
    order_id,
    amount,
    (SELECT AVG(amount) FROM orders) AS overall_avg,
    amount - (SELECT AVG(amount) FROM orders) AS deviation
FROM orders;
```

---

### IN and NOT IN Subquery

```sql
-- Employees in departments located in Mumbai or Delhi
SELECT name FROM employees
WHERE dept_id IN (
    SELECT dept_id FROM departments
    WHERE location IN ('Mumbai', 'Delhi')
);

-- Employees not assigned to any active project
SELECT name FROM employees
WHERE emp_id NOT IN (
    SELECT DISTINCT emp_id FROM project_assignments
    WHERE status = 'active'
);
```

---

### EXISTS and NOT EXISTS

`EXISTS` stops scanning as soon as one matching row is found, making it faster than `IN` for large datasets.

```sql
-- Customers who have placed at least one order above Rs 2000
SELECT DISTINCT customer_name
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM orders
    WHERE customer_name = o.customer_name AND amount > 2000
);

-- Departments with no employees (common interview question)
SELECT dept_name FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);
```

---

### FROM Subquery — Derived Table

```sql
-- Identify top-revenue cities where average order value exceeds Rs 500
SELECT city, total_revenue, avg_amount
FROM (
    SELECT
        city,
        SUM(amount) AS total_revenue,
        AVG(amount) AS avg_amount
    FROM orders
    WHERE status = 'delivered'
    GROUP BY city
) AS city_stats
WHERE avg_amount > 500
ORDER BY total_revenue DESC
LIMIT 3;
```

> **Subquery vs CTE:** CTEs (the `WITH` clause) are the modern, more readable alternative to derived table subqueries. For most cases, prefer CTEs. Derived tables can sometimes be marginally faster because PostgreSQL can inline-optimize them directly.

---

## 7. Window Functions — ROW_NUMBER, RANK, LAG, LEAD

Window functions are among the most powerful SQL features and appear frequently in SDE interviews. Unlike `GROUP BY`, they do not collapse rows — every row retains its identity while also having access to aggregated context.

### Syntax

```sql
FUNCTION_NAME() OVER (
    PARTITION BY column1   -- reset the window per group (optional)
    ORDER BY column2 DESC  -- define row order within the partition (optional)
    ROWS BETWEEN ...       -- frame specification (optional)
)
```

`PARTITION BY` is like `GROUP BY` for the window, but rows are not collapsed. `ORDER BY` defines the sequence within each partition.

---

### ROW_NUMBER — Unique Sequential Number

```sql
-- Rank each customer's orders by amount (1 = most expensive)
SELECT
    customer_name,
    order_id,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_name
        ORDER BY amount DESC
    ) AS rank_per_customer
FROM orders;

-- Interview Classic: Get the single most expensive order per customer
SELECT customer_name, order_id, amount
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY customer_name ORDER BY amount DESC
           ) AS rn
    FROM orders
) ranked
WHERE rn = 1;
```

---

### RANK vs DENSE_RANK — Handling Ties

```sql
SELECT
    student_name,
    score,
    RANK()        OVER (ORDER BY score DESC) AS rank_with_gaps,
    DENSE_RANK()  OVER (ORDER BY score DESC) AS rank_no_gaps,
    ROW_NUMBER()  OVER (ORDER BY score DESC) AS unique_row_num
FROM scores;

-- For scores: 95, 95, 90, 85
-- ROW_NUMBER  : 1, 2, 3, 4  -- always unique, no concept of ties
-- RANK        : 1, 1, 3, 4  -- gap after tie (no rank 2 is assigned)
-- DENSE_RANK  : 1, 1, 2, 3  -- no gap, consecutive ranks
```

---

### LAG and LEAD — Compare with Adjacent Rows

`LAG` looks at a previous row. `LEAD` looks at a following row. Both are essential for trend analysis, growth calculations, and session detection.

```sql
-- Month-over-month revenue growth
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month)           AS prev_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS absolute_growth,
    ROUND(
        100.0 * (revenue - LAG(revenue, 1) OVER (ORDER BY month))
              / LAG(revenue, 1) OVER (ORDER BY month),
        2
    ) AS growth_pct
FROM monthly_revenue;

-- Time until a user's next login (session analysis)
SELECT
    user_id,
    login_date,
    LEAD(login_date, 1) OVER (PARTITION BY user_id ORDER BY login_date) AS next_login,
    LEAD(login_date, 1) OVER (PARTITION BY user_id ORDER BY login_date)
        - login_date AS days_between_logins
FROM user_logins;
```

---

### Running Totals and Moving Averages

```sql
-- Running total of daily revenue
SELECT
    DATE_TRUNC('day', ordered_at) AS day,
    SUM(amount)                   AS daily_rev,
    SUM(SUM(amount)) OVER (
        ORDER BY DATE_TRUNC('day', ordered_at)
    )                             AS running_total
FROM orders
GROUP BY DATE_TRUNC('day', ordered_at);

-- 7-day moving average of daily orders
SELECT
    order_date,
    daily_orders,
    ROUND(AVG(daily_orders) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7d
FROM daily_stats;
```

> **When to use PARTITION BY:** Use it when you want the window function to reset per group. `PARTITION BY customer_name` means `ROW_NUMBER` restarts at 1 for each customer. Without `PARTITION BY`, the window spans all rows in the result set.

---

## 8. Constraints, Keys and Normalization

### Constraints — Enforcing Data Integrity

```sql
CREATE TABLE users (
    user_id  SERIAL PRIMARY KEY,              -- unique + not null + auto-index
    email    VARCHAR(255) UNIQUE NOT NULL,     -- no duplicate emails, required
    age      INT CHECK (age >= 13),            -- minimum age enforcement
    plan     VARCHAR(20) DEFAULT 'free',       -- default plan for new users
    ref_id   INT REFERENCES users(user_id)     -- self-referencing FK for referrals
);

CREATE TABLE orders (
    order_id  SERIAL PRIMARY KEY,
    user_id   INT NOT NULL,
    amount    DECIMAL(10,2) CHECK (amount > 0),
    CONSTRAINT fk_user
        FOREIGN KEY (user_id)
        REFERENCES users(user_id)
        ON DELETE CASCADE    -- automatically delete orders if user is deleted
        ON UPDATE CASCADE    -- propagate user_id changes automatically
);
```

---

### Primary Key vs Foreign Key vs Unique

| Property | PRIMARY KEY | FOREIGN KEY | UNIQUE |
|----------|-------------|-------------|--------|
| Uniqueness | Required | Not required | Required |
| NULL allowed | No | Yes | Yes (one NULL per column) |
| Count per table | One only | Multiple allowed | Multiple allowed |
| Auto-creates index | Yes | No — create manually | Yes |
| Purpose | Uniquely identify each row | Link rows between tables | Prevent duplicate values |

---

### Normalization

Normalization organizes data to minimize redundancy and prevent anomalies on insert, update, and delete.

#### First Normal Form (1NF)

Each cell must contain exactly one atomic value. No repeating groups or lists stored in a single column.

```sql
-- Violation: multiple values in one cell
-- order_id | products
-- 1        | 'Shoes, T-Shirt, Jeans'   -- not atomic

-- Fix: create a separate order_items table
CREATE TABLE order_items (
    item_id   SERIAL PRIMARY KEY,
    order_id  INT REFERENCES orders(order_id),
    product   VARCHAR(100)
);
```

#### Second Normal Form (2NF)

Must satisfy 1NF. Every non-key column must depend on the **entire** primary key, not just part of it (relevant when the primary key is composite).

```sql
-- Violation: composite PK is (order_id, product_id),
-- but product_name depends only on product_id — partial dependency
-- order_id | product_id | product_name | quantity

-- Fix: extract product_name to a products table
CREATE TABLE products    (product_id SERIAL PRIMARY KEY, product_name TEXT);
CREATE TABLE order_items (order_id INT, product_id INT, quantity INT,
                          PRIMARY KEY (order_id, product_id));
```

#### Third Normal Form (3NF)

Must satisfy 2NF. No transitive dependencies — a non-key column must not depend on another non-key column.

```sql
-- Violation: zip_code determines city, but city is stored directly in users
-- user_id | zip_code | city   -- city depends on zip_code, not on user_id

-- Fix: separate zip code reference table
CREATE TABLE zip_codes (zip_code CHAR(6) PRIMARY KEY, city VARCHAR(100));
CREATE TABLE users     (user_id SERIAL PRIMARY KEY, zip_code CHAR(6) REFERENCES zip_codes);
```

> **Production Reality:** Perfect normalization can hurt performance due to the overhead of many JOINs. Experienced engineers deliberately denormalize hot-path tables and maintain normalized tables for write operations. Materialized views and caching handle read performance over normalized data.

---

## 9. Stored Procedures and Functions

### Functions vs Stored Procedures

| Feature | FUNCTION | PROCEDURE |
|---------|----------|-----------|
| Return value | Must return a value | Optional — uses OUT parameters |
| Called via | `SELECT func()` | `CALL proc()` |
| Transaction control | Cannot `COMMIT` or `ROLLBACK` | Can `COMMIT` and `ROLLBACK` |
| Usable inline in queries | Yes | No |
| Typical purpose | Computation, transformation | Multi-step operations with side effects |

---

### Creating Functions

```sql
-- Calculate order total including GST
CREATE OR REPLACE FUNCTION calc_total_with_gst(base_amount DECIMAL)
RETURNS DECIMAL AS $$
BEGIN
    RETURN base_amount * 1.18;
END;
$$ LANGUAGE plpgsql;

-- Use it inline in a query
SELECT order_id, amount, calc_total_with_gst(amount) AS amount_with_gst
FROM orders;

-- Function that returns a result set
CREATE OR REPLACE FUNCTION get_top_orders(n INT)
RETURNS TABLE(order_id INT, customer_name TEXT, amount DECIMAL) AS $$
BEGIN
    RETURN QUERY
        SELECT o.order_id, o.customer_name::TEXT, o.amount
        FROM orders o
        ORDER BY amount DESC
        LIMIT n;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_top_orders(10);
```

---

### Creating Stored Procedures

```sql
-- Bank fund transfer with full transaction control
CREATE OR REPLACE PROCEDURE transfer_funds(
    sender_id   INT,
    receiver_id INT,
    amount      DECIMAL
)
LANGUAGE plpgsql AS $$
BEGIN
    -- Validate balance before proceeding
    IF (SELECT balance FROM accounts WHERE id = sender_id) < amount THEN
        RAISE EXCEPTION 'Insufficient balance for account %', sender_id;
    END IF;

    UPDATE accounts SET balance = balance - amount WHERE id = sender_id;
    UPDATE accounts SET balance = balance + amount WHERE id = receiver_id;

    INSERT INTO transactions (sender_id, receiver_id, amount, txn_time)
    VALUES (sender_id, receiver_id, amount, NOW());

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;

CALL transfer_funds(101, 202, 5000.00);
```

---

### Triggers — Execute Automatically on Events

```sql
-- Audit log: record every price change on the products table
CREATE TABLE product_audit (
    audit_id   SERIAL PRIMARY KEY,
    product_id INT,
    old_price  DECIMAL,
    new_price  DECIMAL,
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    changed_by TEXT DEFAULT CURRENT_USER
);

CREATE OR REPLACE FUNCTION log_price_change()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.price <> OLD.price THEN
        INSERT INTO product_audit(product_id, old_price, new_price)
        VALUES (OLD.product_id, OLD.price, NEW.price);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER price_change_trigger
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION log_price_change();
```

---

## 10. Transactions and ACID Properties

### What is a Transaction?

A transaction is a group of SQL statements that execute as a single atomic unit. Either all statements succeed together, or none of them take effect. This is the foundation of data reliability in systems like banking, booking, and e-commerce.

```sql
BEGIN;

    UPDATE accounts SET balance = balance - 5000 WHERE id = 101;
    UPDATE accounts SET balance = balance + 5000 WHERE id = 202;

COMMIT;     -- make both changes permanent

-- On any error:
ROLLBACK;   -- undo everything since BEGIN
```

---

### ACID Properties

#### A — Atomicity (All or Nothing)

If any statement in a transaction fails, the entire transaction is rolled back. No partial state is committed.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    -- If the application crashes here, PostgreSQL rolls back the debit automatically
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

#### C — Consistency (Valid State to Valid State)

A transaction must leave the database in a valid state. All constraints, triggers, and rules are enforced. You cannot commit a transaction that violates a `NOT NULL`, `CHECK`, or `FOREIGN KEY` constraint.

#### I — Isolation (Concurrent Transactions Do Not Interfere)

Isolation controls what one transaction can see from other in-progress transactions.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------------|------------|---------------------|--------------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED (default) | No | Possible | Possible |
| REPEATABLE READ | No | No | Possible |
| SERIALIZABLE | No | No | No |

```sql
-- Set isolation level for a specific transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT balance FROM accounts WHERE id = 1;
    -- ... processing ...
    SELECT balance FROM accounts WHERE id = 1;
    -- Guaranteed to return the same value both times
COMMIT;
```

> **PostgreSQL Default:** `READ COMMITTED`. PostgreSQL's MVCC implementation prevents dirty reads even without raising the isolation level. For financial operations, use `REPEATABLE READ` or `SERIALIZABLE`.

#### D — Durability (Committed Changes Survive Crashes)

Once `COMMIT` executes, changes are permanent even if the server loses power immediately after. PostgreSQL achieves this through **WAL (Write-Ahead Logging)** — changes are written to disk before the commit is confirmed to the client.

---

### SAVEPOINT — Partial Rollbacks

```sql
BEGIN;
    INSERT INTO orders ...;            -- step 1
    SAVEPOINT after_order;             -- mark a restore point

    INSERT INTO payments ...;          -- step 2 — attempt primary payment

    -- If step 2 fails, roll back only to the savepoint (step 1 is preserved)
    ROLLBACK TO SAVEPOINT after_order;

    -- Retry with an alternative payment method
    INSERT INTO payments (method = 'COD') ...;

COMMIT;
```

---

## 11. Indexes and Query Optimization

### What is an Index?

An index is a separate data structure (B-Tree by default in PostgreSQL) that allows the database engine to locate rows without scanning the entire table.

- **Without index:** Full Table Scan — O(n), reads every row in the table
- **With B-Tree index:** O(log n), navigates the tree to matching rows directly

For a table with 10 million rows, this is the difference between 10,000,000 page reads and roughly 23.

---

### Types of Indexes

```sql
-- B-Tree (default) — suitable for =, <, >, BETWEEN, ORDER BY, LIKE 'prefix%'
CREATE INDEX idx_orders_city ON orders(city);

-- Composite index — column order matters
-- (city, status) supports queries filtering on city alone
-- but NOT on status alone (leftmost prefix rule applies)
CREATE INDEX idx_orders_city_status ON orders(city, status);

-- Unique index — enforces uniqueness and speeds up lookups simultaneously
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Partial index — indexes only rows matching a condition
-- Much smaller than a full index, faster for targeted queries
CREATE INDEX idx_pending_orders ON orders(city)
WHERE status = 'pending';

-- Expression index — index on a computed value
CREATE INDEX idx_lower_email ON users(LOWER(email));
-- This query will now use the index:
SELECT * FROM users WHERE LOWER(email) = 'raj@example.com';
```

---

### EXPLAIN ANALYZE — Reading Query Plans

`EXPLAIN ANALYZE` is the primary tool for diagnosing slow queries.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE city = 'Mumbai' AND amount > 500;

-- Key terms in the output:
-- Seq Scan        = full table scan — acceptable only for small tables
-- Index Scan      = uses an index — desired for large tables
-- Index Only Scan = reads from the index without touching the main table (fastest)
-- cost=X..Y       = estimated cost units (startup..total)
-- actual time     = real execution time in milliseconds
-- rows            = estimated vs actual row count

-- Sample output:
-- Index Scan using idx_orders_city on orders
--   (cost=0.43..45.23 rows=142 width=68)
--   (actual time=0.081..1.243 rows=138 loops=1)
-- Filter: (amount > 500)
-- Planning Time: 0.3 ms
-- Execution Time: 1.4 ms
```

---

### Query Optimization Techniques

```sql
-- 1. Select only the columns you need
-- Avoid:
SELECT * FROM orders WHERE city = 'Mumbai';
-- Prefer:
SELECT order_id, customer_name, amount FROM orders WHERE city = 'Mumbai';

-- 2. Do not wrap indexed columns in functions inside WHERE
-- This disables the index on ordered_at:
SELECT * FROM orders WHERE EXTRACT(YEAR FROM ordered_at) = 2024;
-- This allows the index to be used:
SELECT * FROM orders WHERE ordered_at BETWEEN '2024-01-01' AND '2024-12-31';

-- 3. Use EXISTS instead of COUNT for existence checks
-- Avoid (scans all matching rows):
SELECT COUNT(*) > 0 FROM orders WHERE user_id = 101;
-- Prefer (stops at the first match):
SELECT EXISTS(SELECT 1 FROM orders WHERE user_id = 101);

-- 4. Covering index — include all columns the query needs
-- The query below can be answered entirely from the index, no heap access needed
CREATE INDEX idx_orders_covering ON orders(city) INCLUDE (order_id, amount);
SELECT order_id, amount FROM orders WHERE city = 'Mumbai';
-- Result: Index Only Scan
```

---

### VACUUM and ANALYZE — Table Maintenance

PostgreSQL uses **MVCC (Multi-Version Concurrency Control)** for transaction isolation. When rows are updated or deleted, old versions accumulate as dead tuples. `VACUUM` reclaims that space.

```sql
-- Reclaim space from dead rows
VACUUM orders;

-- Reclaim space AND refresh query planner statistics
VACUUM ANALYZE orders;

-- Check how much dead tuple buildup exists
SELECT
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

> `AUTOVACUUM` runs automatically in the background for routine maintenance. Run `VACUUM ANALYZE` manually after large bulk inserts or deletes to ensure the query planner has accurate statistics immediately.

---

### Indexing Best Practices

- Index columns that appear frequently in `WHERE`, `JOIN ON`, and `ORDER BY` clauses.
- For composite indexes, place the most selective (highest cardinality) column first.
- Do not index every column — each index adds overhead to every `INSERT`, `UPDATE`, and `DELETE`.
- Use partial indexes when queries always filter on a fixed condition (e.g., `WHERE status = 'active'`).
- Run `EXPLAIN ANALYZE` before and after adding an index to measure actual impact.
- Drop indexes that are never used — they consume write performance with no read benefit.
- For full-text search, use `GIN` indexes with `tsvector` instead of `LIKE '%keyword%'`.

---

## Quick Reference

### Query Execution Order

```
FROM / JOIN  ->  WHERE  ->  GROUP BY  ->  HAVING  ->  SELECT  ->  DISTINCT  ->  ORDER BY  ->  LIMIT
```

### Window vs Aggregate

| Aspect | Aggregate (GROUP BY) | Window Function (OVER) |
|--------|----------------------|------------------------|
| Row count | Collapses rows into groups | Keeps all original rows |
| Output | One row per group | One row per input row |
| Use case | Totals, counts, summaries | Rankings, running totals, comparisons |
| Access to individual rows | Lost after grouping | Preserved |

### Join Decision Guide

```
Need only matching rows?           -> INNER JOIN
Need all rows from one side?       -> LEFT JOIN
Need all rows from both sides?     -> FULL OUTER JOIN
Table references itself?           -> SELF JOIN
Need every possible combination?   -> CROSS JOIN
```

---

*This guide covers 11 core PostgreSQL topics for SDE interview preparation. Each concept is paired with real-world examples drawn from common backend systems.*

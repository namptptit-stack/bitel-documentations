# ORACLE DATABASE - QUICK START GUIDE
## Beginner's Guide & Troubleshooting

---

## 📚 TABLE OF CONTENTS

1. [Basic Concepts](#1-basic-concepts)
2. [Database Connection](#2-database-connection)
3. [Basic Queries](#3-basic-queries)
4. [Insert, Update, Delete](#4-insert-update-delete)
5. [Basic Stored Procedures](#5-basic-stored-procedures)
6. [Troubleshooting Scenarios](#6-troubleshooting-scenarios)

---

## 1. BASIC CONCEPTS

### A. What is Oracle Database?

```
Database (DB) = Data storage system
Schema = Collection of objects (tables, views, procedures)
Table = Data table (like Excel spreadsheet)
Column = Field in a table
Row = Record in a table
```

### B. Common Data Types

```sql
-- Numbers
NUMBER(10)          -- Integer (max 10 digits)
NUMBER(10,2)        -- Decimal (8 integers, 2 decimals)

-- Strings
VARCHAR2(100)       -- Variable string up to 100 chars
CHAR(10)            -- Fixed string of 10 chars

-- Dates
DATE                -- Date and time
TIMESTAMP           -- High precision date and time

-- Others
BLOB                -- Binary data (images, files)
CLOB                -- Large text
```

### C. Tools

```
SQL*Plus        → Command line tool
SQL Developer   → GUI tool (recommended for beginners)
PL/SQL Developer → Commercial tool
```

---

## 2. DATABASE CONNECTION

### A. Connect with SQL Developer

```
1. Open SQL Developer
2. Click "New Connection" (+ icon)
3. Enter information:
   - Connection Name: Your choice (e.g., MyDB)
   - Username: User name (e.g., app_user)
   - Password: Password
   - Hostname: Server address (e.g., 192.168.1.100)
   - Port: 1521 (default)
   - SID or Service Name: Database name
4. Click "Test" → "Connect"
```

### B. Test Connection

```sql
-- Simple test query
SELECT SYSDATE FROM dual;

-- Check current user
SELECT USER FROM dual;

-- View accessible tables
SELECT table_name FROM user_tables;
```

---

## 3. BASIC QUERIES

### A. SELECT - Retrieve Data

```sql
-- Select all
SELECT * FROM employees;

-- Select specific columns
SELECT employee_name, salary FROM employees;

-- WHERE - Conditions
SELECT * FROM employees WHERE salary > 5000;

-- ORDER BY - Sorting
SELECT * FROM employees ORDER BY salary DESC;  -- Descending
SELECT * FROM employees ORDER BY salary ASC;   -- Ascending

-- DISTINCT - Remove duplicates
SELECT DISTINCT department_id FROM employees;
```

### B. WHERE - Common Conditions

```sql
-- Comparison
WHERE salary = 5000          -- Equal
WHERE salary != 5000         -- Not equal
WHERE salary > 5000          -- Greater than
WHERE salary >= 5000         -- Greater than or equal
WHERE salary < 5000          -- Less than
WHERE salary <= 5000         -- Less than or equal

-- BETWEEN - Range
WHERE salary BETWEEN 3000 AND 7000

-- IN - List
WHERE department_id IN (10, 20, 30)

-- LIKE - Pattern matching
WHERE employee_name LIKE 'John%'     -- Starts with John
WHERE employee_name LIKE '%Smith'    -- Ends with Smith
WHERE employee_name LIKE '%son%'     -- Contains son

-- IS NULL / IS NOT NULL
WHERE manager_id IS NULL
WHERE manager_id IS NOT NULL

-- AND, OR
WHERE salary > 5000 AND department_id = 10
WHERE department_id = 10 OR department_id = 20
```

### C. JOIN - Combine Tables

```sql
-- INNER JOIN - Only matching records
SELECT e.employee_name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- LEFT JOIN - All records from left table
SELECT e.employee_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- RIGHT JOIN - All records from right table
SELECT e.employee_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### D. GROUP BY - Aggregations

```sql
-- COUNT - Count records
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

-- SUM - Total
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

-- AVG - Average
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;

-- MAX, MIN
SELECT department_id, 
       MAX(salary) AS max_salary,
       MIN(salary) AS min_salary
FROM employees
GROUP BY department_id;

-- HAVING - Filter groups
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 5000;
```

---

## 4. INSERT, UPDATE, DELETE

### A. INSERT - Add Data

```sql
-- Insert single record
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John Doe', 5000, SYSDATE);

-- Insert multiple records
INSERT INTO employees (employee_id, employee_name, salary)
VALUES (102, 'Jane Smith', 6000);

INSERT INTO employees (employee_id, employee_name, salary)
VALUES (103, 'Bob Wilson', 5500);

-- Insert from SELECT
INSERT INTO archive_employees
SELECT * FROM employees WHERE hire_date < DATE '2020-01-01';

-- ⚠️ IMPORTANT: Must COMMIT to save
COMMIT;
```

### B. UPDATE - Modify Data

```sql
-- Update single column
UPDATE employees
SET salary = 6000
WHERE employee_id = 101;

-- Update multiple columns
UPDATE employees
SET salary = 6500,
    department_id = 20
WHERE employee_id = 101;

-- Update with complex condition
UPDATE employees
SET salary = salary * 1.1  -- Increase 10%
WHERE department_id = 10 AND salary < 5000;

-- ⚠️ IMPORTANT
COMMIT;  -- Save changes
-- or
ROLLBACK;  -- Undo changes
```

### C. DELETE - Remove Data

```sql
-- Delete with condition
DELETE FROM employees
WHERE employee_id = 101;

-- Delete multiple records
DELETE FROM employees
WHERE department_id = 10;

-- ⚠️ DANGER: Delete all
DELETE FROM employees;  -- Deletes EVERYTHING!

-- ⚠️ IMPORTANT
COMMIT;    -- Confirm deletion
-- or
ROLLBACK;  -- Restore data
```

### ⚠️ CRITICAL NOTES

```sql
-- ALWAYS preview before UPDATE/DELETE
-- Step 1: SELECT to see affected records
SELECT * FROM employees WHERE employee_id = 101;

-- Step 2: If correct, run UPDATE/DELETE
UPDATE employees SET salary = 6000 WHERE employee_id = 101;

-- Step 3: Verify again
SELECT * FROM employees WHERE employee_id = 101;

-- Step 4: COMMIT or ROLLBACK
COMMIT;
```

---

## 5. BASIC STORED PROCEDURES

### A. Simple Procedure

```sql
-- Create procedure
CREATE OR REPLACE PROCEDURE say_hello IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello, World!');
END;
/

-- Execute procedure
SET SERVEROUTPUT ON;  -- Enable output
EXEC say_hello;
```

### B. Procedure with Parameters

```sql
-- Create procedure with parameters
CREATE OR REPLACE PROCEDURE update_salary (
    p_employee_id NUMBER,
    p_new_salary NUMBER
) IS
BEGIN
    UPDATE employees
    SET salary = p_new_salary
    WHERE employee_id = p_employee_id;
    
    COMMIT;
    
    DBMS_OUTPUT.PUT_LINE('Salary updated successfully');
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/

-- Call procedure
EXEC update_salary(101, 7000);
```

### C. Simple Function

```sql
-- Create function
CREATE OR REPLACE FUNCTION get_employee_name (
    p_employee_id NUMBER
) RETURN VARCHAR2 IS
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = p_employee_id;
    
    RETURN v_name;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN NULL;
END;
/

-- Use function
SELECT get_employee_name(101) FROM dual;
```

---

## 6. TROUBLESHOOTING SCENARIOS

### 🔴 Scenario 1: "Table or view does not exist"

**Error:**
```
ORA-00942: table or view does not exist
```

**Causes:**
- Table doesn't exist
- Wrong table name (typo, missing schema)
- No access privileges

**Solution:**
```sql
-- Check if table exists
SELECT table_name FROM user_tables WHERE table_name = 'EMPLOYEES';

-- Check privileges
SELECT * FROM user_tab_privs WHERE table_name = 'EMPLOYEES';

-- If schema prefix needed
SELECT * FROM schema_name.employees;
```

---

### 🔴 Scenario 2: "Invalid identifier"

**Error:**
```
ORA-00904: "EMPLOYE_NAME": invalid identifier
```

**Cause:**
- Wrong column name (typo: EMPLOYE_NAME instead of EMPLOYEE_NAME)

**Solution:**
```sql
-- View all columns
DESC employees;
-- or
SELECT column_name FROM user_tab_columns WHERE table_name = 'EMPLOYEES';

-- Fix query with correct name
SELECT employee_name FROM employees;
```

---

### 🔴 Scenario 3: "Too many values"

**Error:**
```
ORA-00913: too many values
```

**Cause:**
- INSERT has more values than columns
- Subquery returns more columns than expected

**Solution:**
```sql
-- ❌ WRONG: 3 columns but 4 values
INSERT INTO employees (employee_id, employee_name, salary)
VALUES (101, 'John', 5000, SYSDATE);

-- ✅ CORRECT
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000, SYSDATE);
```

---

### 🔴 Scenario 4: "Not enough values"

**Error:**
```
ORA-00947: not enough values
```

**Cause:**
- INSERT has fewer values than columns

**Solution:**
```sql
-- ❌ WRONG: Missing hire_date
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000);

-- ✅ CORRECT
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000, SYSDATE);
```

---

### 🔴 Scenario 5: Slow Query Performance

**Symptoms:**
- Query takes >10 seconds
- Application timeout

**Solution:**

```sql
-- Step 1: Check execution plan
EXPLAIN PLAN FOR
SELECT * FROM employees WHERE department_id = 10;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Step 2: Check for indexes
SELECT index_name, column_name 
FROM user_ind_columns 
WHERE table_name = 'EMPLOYEES';

-- Step 3: Create index if needed
CREATE INDEX idx_emp_dept ON employees(department_id);

-- Step 4: Update statistics
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'EMPLOYEES');
```

---

### 🔴 Scenario 6: "Cannot insert NULL"

**Error:**
```
ORA-01400: cannot insert NULL into ("SCHEMA"."EMPLOYEES"."EMPLOYEE_ID")
```

**Cause:**
- Column is NOT NULL but no value provided

**Solution:**
```sql
-- View table constraints
SELECT constraint_name, constraint_type, search_condition
FROM user_constraints
WHERE table_name = 'EMPLOYEES';

-- View NOT NULL columns
DESC employees;

-- Provide value for NOT NULL columns
INSERT INTO employees (employee_id, employee_name)
VALUES (101, 'John');  -- employee_id must have value
```

---

### 🔴 Scenario 7: "Unique constraint violated"

**Error:**
```
ORA-00001: unique constraint (SCHEMA.UK_EMPLOYEE_ID) violated
```

**Cause:**
- Inserting duplicate value in UNIQUE/PRIMARY KEY column

**Solution:**
```sql
-- Check if value exists
SELECT * FROM employees WHERE employee_id = 101;

-- If exists, use UPDATE instead of INSERT
UPDATE employees SET employee_name = 'John' WHERE employee_id = 101;

-- Or use MERGE
MERGE INTO employees e
USING (SELECT 101 AS employee_id, 'John' AS employee_name FROM dual) s
ON (e.employee_id = s.employee_id)
WHEN MATCHED THEN
    UPDATE SET e.employee_name = s.employee_name
WHEN NOT MATCHED THEN
    INSERT (employee_id, employee_name)
    VALUES (s.employee_id, s.employee_name);
```

---

### 🔴 Scenario 8: "No data found"

**Error:**
```
ORA-01403: no data found
```

**Cause:**
- SELECT INTO finds no record
- Query returns 0 rows

**Solution:**
```sql
-- ❌ WRONG: Error if no data
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = 999;  -- Doesn't exist
END;

-- ✅ CORRECT: Handle exception
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = 999;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Employee not found');
        v_name := NULL;
END;
```

---

### 🔴 Scenario 9: "Too many rows"

**Error:**
```
ORA-01422: exact fetch returns more than requested number of rows
```

**Cause:**
- SELECT INTO returns >1 record

**Solution:**
```sql
-- ❌ WRONG: Query returns multiple rows
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE department_id = 10;  -- Multiple employees in dept 10
END;

-- ✅ CORRECT: Add condition or use cursor
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = 101;  -- Unique key
END;

-- Or use cursor for multiple rows
DECLARE
    CURSOR c_emp IS
        SELECT employee_name FROM employees WHERE department_id = 10;
BEGIN
    FOR r IN c_emp LOOP
        DBMS_OUTPUT.PUT_LINE(r.employee_name);
    END LOOP;
END;
```

---

### 🔴 Scenario 10: Locked Session

**Symptoms:**
- Query/Update hangs
- Application not responding

**Solution:**
```sql
-- Step 1: Find locking session
SELECT 
    s.sid,
    s.serial#,
    s.username,
    s.program,
    s.machine,
    o.object_name,
    'ALTER SYSTEM KILL SESSION ''' || s.sid || ',' || s.serial# || ''';' AS kill_cmd
FROM v$session s
JOIN v$locked_object l ON s.sid = l.session_id
JOIN dba_objects o ON l.object_id = o.object_id
WHERE o.object_name = 'EMPLOYEES';

-- Step 2: Kill session (be careful!)
ALTER SYSTEM KILL SESSION '123,456' IMMEDIATE;

-- Step 3: Or wait for commit/rollback from locking session
```

---

## 📋 DAILY CHECKLIST

### Before Writing Code:
- [ ] Read requirements carefully
- [ ] Check table structure (DESC table_name)
- [ ] Check for indexes
- [ ] Test with small sample data

### When Writing Query:
- [ ] SELECT before UPDATE/DELETE
- [ ] Include WHERE clause (avoid updating/deleting all)
- [ ] Test with EXPLAIN PLAN
- [ ] Check rows affected (SQL%ROWCOUNT)

### After Running Query:
- [ ] Verify results
- [ ] COMMIT or ROLLBACK
- [ ] Test again
- [ ] Document changes

### Best Practices:
- [ ] Always backup before major changes
- [ ] Test on DEV before PROD
- [ ] Code review with senior
- [ ] Log all important changes

---

## 🎯 USEFUL TIPS

### 1. Avoid Common Mistakes

```sql
-- ❌ NEVER do this on PRODUCTION
DELETE FROM employees;  -- Deletes EVERYTHING!
UPDATE employees SET salary = 0;  -- Updates EVERYTHING!

-- ✅ ALWAYS use WHERE
DELETE FROM employees WHERE employee_id = 101;
UPDATE employees SET salary = 6000 WHERE employee_id = 101;
```

### 2. Format Code for Readability

```sql
-- ❌ Hard to read
SELECT e.employee_name,d.department_name FROM employees e,departments d WHERE e.department_id=d.department_id AND e.salary>5000;

-- ✅ Easy to read
SELECT 
    e.employee_name,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 5000;
```

### 3. Comment Your Code

```sql
-- Calculate total salary by department
SELECT 
    department_id,
    SUM(salary) AS total_salary  -- Sum of salaries
FROM employees
WHERE hire_date > DATE '2020-01-01'  -- Only new employees
GROUP BY department_id;
```

### 4. Use Clear Aliases

```sql
-- ❌ Unclear
SELECT e.*, d.*
FROM employees e, departments d
WHERE e.d = d.d;

-- ✅ Clear
SELECT 
    e.employee_name,
    e.salary,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```

---

## 🆘 WHEN FACING ISSUES

### Step 1: Read Error Message
- Check error code (ORA-xxxxx)
- Read error description

### Step 2: Google It
- Search: "ORA-xxxxx Oracle"
- Check Oracle documentation

### Step 3: Check Basics
- Is syntax correct?
- Are table/column names correct?
- Do you have access privileges?

### Step 4: Ask Senior
- Prepare: Error message, code, what you've tried
- Explain problem clearly
- Listen and learn

---

## 📚 REFERENCES

- Oracle Documentation: https://docs.oracle.com
- Oracle Live SQL: https://livesql.oracle.com (Practice online!)
- Oracle Base: https://oracle-base.com
- Ask Tom: https://asktom.oracle.com

---

**Happy Learning! 🚀**

**Note:** Practice makes perfect! Always practice in DEV environment.

---

**Version:** 1.0  
**Created:** 2024  
**For:** Beginners & Junior Developers
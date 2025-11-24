# ORACLE DATABASE - QUICK START GUIDE
## Hướng dẫn nhanh cho người mới bắt đầu

---

## 📚 MỤC LỤC

1. [Kiến thức cơ bản](#1-kiến-thức-cơ-bản)
2. [Kết nối Database](#2-kết-nối-database)
3. [Truy vấn cơ bản](#3-truy-vấn-cơ-bản)
4. [Insert, Update, Delete](#4-insert-update-delete)
5. [Stored Procedures cơ bản](#5-stored-procedures-cơ-bản)
6. [Troubleshooting Scenarios](#6-troubleshooting-scenarios)

---

## 1. KIẾN THỨC CƠ BẢN

### A. Oracle Database là gì?

```
Database (DB) = Nơi lưu trữ dữ liệu
Schema = Tập hợp các objects (tables, views, procedures)
Table = Bảng chứa dữ liệu (như Excel)
Column = Cột (trường dữ liệu)
Row = Hàng (bản ghi)
```

### B. Các kiểu dữ liệu thường dùng

```sql
-- Số
NUMBER(10)          -- Số nguyên (max 10 chữ số)
NUMBER(10,2)        -- Số thập phân (8 số nguyên, 2 số thập phân)

-- Chuỗi
VARCHAR2(100)       -- Chuỗi tối đa 100 ký tự
CHAR(10)            -- Chuỗi cố định 10 ký tự

-- Ngày tháng
DATE                -- Ngày và giờ
TIMESTAMP           -- Ngày giờ với độ chính xác cao

-- Khác
BLOB                -- Dữ liệu nhị phân (hình ảnh, file)
CLOB                -- Text lớn
```

### C. Công cụ làm việc

```
SQL*Plus        → Command line tool
SQL Developer   → GUI tool (khuyên dùng cho beginners)
PL/SQL Developer → Commercial tool
```

---

## 2. KẾT NỐI DATABASE

### A. Kết nối bằng SQL Developer

```
1. Mở SQL Developer
2. Click "New Connection" (dấu +)
3. Nhập thông tin:
   - Connection Name: Tên tùy ý (VD: MyDB)
   - Username: Tên user (VD: app_user)
   - Password: Mật khẩu
   - Hostname: Địa chỉ server (VD: 192.168.1.100)
   - Port: 1521 (default)
   - SID hoặc Service Name: Tên database
4. Click "Test" → "Connect"
```

### B. Kiểm tra kết nối

```sql
-- Query đơn giản để test
SELECT SYSDATE FROM dual;

-- Xem user hiện tại
SELECT USER FROM dual;

-- Xem các bảng có quyền truy cập
SELECT table_name FROM user_tables;
```

---

## 3. TRUY VẤN CƠ BẢN

### A. SELECT - Lấy dữ liệu

```sql
-- Lấy tất cả
SELECT * FROM employees;

-- Lấy cột cụ thể
SELECT employee_name, salary FROM employees;

-- WHERE - Điều kiện
SELECT * FROM employees WHERE salary > 5000;

-- ORDER BY - Sắp xếp
SELECT * FROM employees ORDER BY salary DESC;  -- Giảm dần
SELECT * FROM employees ORDER BY salary ASC;   -- Tăng dần

-- DISTINCT - Loại bỏ trùng
SELECT DISTINCT department_id FROM employees;
```

### B. WHERE - Các điều kiện thường dùng

```sql
-- So sánh
WHERE salary = 5000          -- Bằng
WHERE salary != 5000         -- Khác
WHERE salary > 5000          -- Lớn hơn
WHERE salary >= 5000         -- Lớn hơn hoặc bằng
WHERE salary < 5000          -- Nhỏ hơn
WHERE salary <= 5000         -- Nhỏ hơn hoặc bằng

-- BETWEEN - Khoảng giá trị
WHERE salary BETWEEN 3000 AND 7000

-- IN - Trong danh sách
WHERE department_id IN (10, 20, 30)

-- LIKE - Tìm kiếm pattern
WHERE employee_name LIKE 'John%'     -- Bắt đầu bằng John
WHERE employee_name LIKE '%Smith'    -- Kết thúc bằng Smith
WHERE employee_name LIKE '%son%'     -- Chứa son

-- IS NULL / IS NOT NULL
WHERE manager_id IS NULL
WHERE manager_id IS NOT NULL

-- AND, OR
WHERE salary > 5000 AND department_id = 10
WHERE department_id = 10 OR department_id = 20
```

### C. JOIN - Kết hợp bảng

```sql
-- INNER JOIN - Chỉ lấy records có match
SELECT e.employee_name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- LEFT JOIN - Lấy tất cả từ bảng trái
SELECT e.employee_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- RIGHT JOIN - Lấy tất cả từ bảng phải
SELECT e.employee_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### D. GROUP BY - Nhóm và tính toán

```sql
-- COUNT - Đếm
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

-- SUM - Tổng
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

-- AVG - Trung bình
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;

-- MAX, MIN
SELECT department_id, 
       MAX(salary) AS max_salary,
       MIN(salary) AS min_salary
FROM employees
GROUP BY department_id;

-- HAVING - Điều kiện cho GROUP BY
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 5000;
```

---

## 4. INSERT, UPDATE, DELETE

### A. INSERT - Thêm dữ liệu

```sql
-- Insert 1 record
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John Doe', 5000, SYSDATE);

-- Insert nhiều records
INSERT INTO employees (employee_id, employee_name, salary)
VALUES (102, 'Jane Smith', 6000);

INSERT INTO employees (employee_id, employee_name, salary)
VALUES (103, 'Bob Wilson', 5500);

-- Insert từ SELECT
INSERT INTO archive_employees
SELECT * FROM employees WHERE hire_date < DATE '2020-01-01';

-- ⚠️ QUAN TRỌNG: Phải COMMIT để lưu
COMMIT;
```

### B. UPDATE - Cập nhật dữ liệu

```sql
-- Update 1 cột
UPDATE employees
SET salary = 6000
WHERE employee_id = 101;

-- Update nhiều cột
UPDATE employees
SET salary = 6500,
    department_id = 20
WHERE employee_id = 101;

-- Update với điều kiện phức tạp
UPDATE employees
SET salary = salary * 1.1  -- Tăng 10%
WHERE department_id = 10 AND salary < 5000;

-- ⚠️ QUAN TRỌNG
COMMIT;  -- Lưu thay đổi
-- hoặc
ROLLBACK;  -- Hủy bỏ thay đổi
```

### C. DELETE - Xóa dữ liệu

```sql
-- Delete với điều kiện
DELETE FROM employees
WHERE employee_id = 101;

-- Delete nhiều records
DELETE FROM employees
WHERE department_id = 10;

-- ⚠️ NGUY HIỂM: Delete tất cả
DELETE FROM employees;  -- Xóa HẾT dữ liệu!

-- ⚠️ QUAN TRỌNG
COMMIT;    -- Xác nhận xóa
-- hoặc
ROLLBACK;  -- Khôi phục lại
```

### ⚠️ CHÚ Ý QUAN TRỌNG

```sql
-- LUÔN kiểm tra trước khi UPDATE/DELETE
-- Bước 1: SELECT để xem records sẽ bị ảnh hưởng
SELECT * FROM employees WHERE employee_id = 101;

-- Bước 2: Nếu đúng, chạy UPDATE/DELETE
UPDATE employees SET salary = 6000 WHERE employee_id = 101;

-- Bước 3: Kiểm tra lại
SELECT * FROM employees WHERE employee_id = 101;

-- Bước 4: COMMIT hoặc ROLLBACK
COMMIT;
```

---

## 5. STORED PROCEDURES CƠ BẢN

### A. Procedure đơn giản

```sql
-- Tạo procedure
CREATE OR REPLACE PROCEDURE say_hello IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello, World!');
END;
/

-- Chạy procedure
SET SERVEROUTPUT ON;  -- Enable output
EXEC say_hello;
```

### B. Procedure với parameters

```sql
-- Tạo procedure có tham số
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

-- Gọi procedure
EXEC update_salary(101, 7000);
```

### C. Function đơn giản

```sql
-- Tạo function
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

-- Sử dụng function
SELECT get_employee_name(101) FROM dual;
```

---

## 6. TROUBLESHOOTING SCENARIOS

### 🔴 Scenario 1: "Table or view does not exist"

**Lỗi:**
```
ORA-00942: table or view does not exist
```

**Nguyên nhân:**
- Bảng không tồn tại
- Sai tên bảng (viết sai, thiếu schema)
- Không có quyền truy cập

**Giải pháp:**
```sql
-- Kiểm tra bảng tồn tại không
SELECT table_name FROM user_tables WHERE table_name = 'EMPLOYEES';

-- Kiểm tra quyền
SELECT * FROM user_tab_privs WHERE table_name = 'EMPLOYEES';

-- Nếu cần schema prefix
SELECT * FROM schema_name.employees;
```

---

### 🔴 Scenario 2: "Invalid identifier"

**Lỗi:**
```
ORA-00904: "EMPLOYE_NAME": invalid identifier
```

**Nguyên nhân:**
- Sai tên cột (typo: EMPLOYE_NAME thay vì EMPLOYEE_NAME)

**Giải pháp:**
```sql
-- Xem tất cả cột của bảng
DESC employees;
-- hoặc
SELECT column_name FROM user_tab_columns WHERE table_name = 'EMPLOYEES';

-- Sửa lại query với tên đúng
SELECT employee_name FROM employees;
```

---

### 🔴 Scenario 3: "Too many values"

**Lỗi:**
```
ORA-00913: too many values
```

**Nguyên nhân:**
- INSERT có nhiều values hơn số cột
- Subquery trả về nhiều cột hơn expected

**Giải pháp:**
```sql
-- ❌ SAI: 3 cột nhưng 4 values
INSERT INTO employees (employee_id, employee_name, salary)
VALUES (101, 'John', 5000, SYSDATE);

-- ✅ ĐÚNG
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000, SYSDATE);
```

---

### 🔴 Scenario 4: "Not enough values"

**Lỗi:**
```
ORA-00947: not enough values
```

**Nguyên nhân:**
- INSERT có ít values hơn số cột

**Giải pháp:**
```sql
-- ❌ SAI: Thiếu hire_date
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000);

-- ✅ ĐÚNG
INSERT INTO employees (employee_id, employee_name, salary, hire_date)
VALUES (101, 'John', 5000, SYSDATE);
```

---

### 🔴 Scenario 5: Query chạy chậm

**Triệu chứng:**
- Query mất >10 giây
- Application bị timeout

**Giải pháp:**

```sql
-- Bước 1: Check execution plan
EXPLAIN PLAN FOR
SELECT * FROM employees WHERE department_id = 10;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Bước 2: Kiểm tra có index không
SELECT index_name, column_name 
FROM user_ind_columns 
WHERE table_name = 'EMPLOYEES';

-- Bước 3: Tạo index nếu cần
CREATE INDEX idx_emp_dept ON employees(department_id);

-- Bước 4: Update statistics
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'EMPLOYEES');
```

---

### 🔴 Scenario 6: "Cannot insert NULL"

**Lỗi:**
```
ORA-01400: cannot insert NULL into ("SCHEMA"."EMPLOYEES"."EMPLOYEE_ID")
```

**Nguyên nhân:**
- Cột NOT NULL nhưng không có giá trị

**Giải pháp:**
```sql
-- Xem constraint của bảng
SELECT constraint_name, constraint_type, search_condition
FROM user_constraints
WHERE table_name = 'EMPLOYEES';

-- Xem cột nào NOT NULL
DESC employees;

-- Đảm bảo provide value cho cột NOT NULL
INSERT INTO employees (employee_id, employee_name)
VALUES (101, 'John');  -- employee_id phải có giá trị
```

---

### 🔴 Scenario 7: "Unique constraint violated"

**Lỗi:**
```
ORA-00001: unique constraint (SCHEMA.UK_EMPLOYEE_ID) violated
```

**Nguyên nhân:**
- Insert giá trị trùng với UNIQUE/PRIMARY KEY

**Giải pháp:**
```sql
-- Kiểm tra giá trị đã tồn tại chưa
SELECT * FROM employees WHERE employee_id = 101;

-- Nếu tồn tại, dùng UPDATE thay vì INSERT
UPDATE employees SET employee_name = 'John' WHERE employee_id = 101;

-- Hoặc dùng MERGE
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

**Lỗi:**
```
ORA-01403: no data found
```

**Nguyên nhân:**
- SELECT INTO không tìm thấy record
- Query trả về 0 rows

**Giải pháp:**
```sql
-- ❌ SAI: Lỗi nếu không có data
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = 999;  -- Không tồn tại
END;

-- ✅ ĐÚNG: Handle exception
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

**Lỗi:**
```
ORA-01422: exact fetch returns more than requested number of rows
```

**Nguyên nhân:**
- SELECT INTO trả về >1 record

**Giải pháp:**
```sql
-- ❌ SAI: Query trả về nhiều rows
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE department_id = 10;  -- Nhiều employees trong dept 10
END;

-- ✅ ĐÚNG: Thêm điều kiện hoặc dùng cursor
DECLARE
    v_name VARCHAR2(100);
BEGIN
    SELECT employee_name INTO v_name
    FROM employees
    WHERE employee_id = 101;  -- Unique key
END;

-- Hoặc dùng cursor cho nhiều rows
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

### 🔴 Scenario 10: Session bị lock

**Triệu chứng:**
- Query/Update bị "treo"
- Application không response

**Giải pháp:**
```sql
-- Bước 1: Tìm session đang lock
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

-- Bước 2: Kill session (cẩn thận!)
ALTER SYSTEM KILL SESSION '123,456' IMMEDIATE;

-- Bước 3: Hoặc đợi và commit/rollback từ session gây lock
```

---

## 📋 CHECKLIST HÀNG NGÀY

### Trước khi viết code:
- [ ] Đọc requirement kỹ
- [ ] Kiểm tra table structure (DESC table_name)
- [ ] Xem có index không
- [ ] Test với dữ liệu mẫu nhỏ

### Khi viết query:
- [ ] SELECT trước khi UPDATE/DELETE
- [ ] Có WHERE clause (tránh update/delete hết)
- [ ] Test với EXPLAIN PLAN
- [ ] Check số rows affected (SQL%ROWCOUNT)

### Sau khi chạy query:
- [ ] Verify kết quả
- [ ] COMMIT hoặc ROLLBACK
- [ ] Test lại
- [ ] Document thay đổi

### Best Practices:
- [ ] Luôn backup trước khi thay đổi lớn
- [ ] Test trên DEV trước khi PROD
- [ ] Code review với senior
- [ ] Log mọi thay đổi quan trọng

---

## 🎯 TIPS HỮU ÍCH

### 1. Tránh những sai lầm phổ biến

```sql
-- ❌ KHÔNG BAO GIỜ làm thế này trên PRODUCTION
DELETE FROM employees;  -- Xóa HẾT!
UPDATE employees SET salary = 0;  -- Update HẾT!

-- ✅ LUÔN có WHERE
DELETE FROM employees WHERE employee_id = 101;
UPDATE employees SET salary = 6000 WHERE employee_id = 101;
```

### 2. Format code cho dễ đọc

```sql
-- ❌ Khó đọc
SELECT e.employee_name,d.department_name FROM employees e,departments d WHERE e.department_id=d.department_id AND e.salary>5000;

-- ✅ Dễ đọc
SELECT 
    e.employee_name,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 5000;
```

### 3. Comment code

```sql
-- Tính tổng lương theo department
SELECT 
    department_id,
    SUM(salary) AS total_salary  -- Tổng lương
FROM employees
WHERE hire_date > DATE '2020-01-01'  -- Chỉ nhân viên mới
GROUP BY department_id;
```

### 4. Sử dụng alias rõ ràng

```sql
-- ❌ Khó hiểu
SELECT e.*, d.*
FROM employees e, departments d
WHERE e.d = d.d;

-- ✅ Rõ ràng
SELECT 
    e.employee_name,
    e.salary,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```

---

## 🆘 KHI GẶP VẤN ĐỀ

### Bước 1: Đọc error message
- Xem mã lỗi (ORA-xxxxx)
- Đọc mô tả lỗi

### Bước 2: Google
- Search: "ORA-xxxxx Oracle"
- Tìm trên Oracle documentation

### Bước 3: Check basics
- Syntax có đúng không?
- Tên bảng/cột có đúng không?
- Có quyền truy cập không?

### Bước 4: Hỏi senior
- Chuẩn bị: Error message, code, đã thử gì
- Giải thích vấn đề rõ ràng
- Lắng nghe và học hỏi

---

## 📚 TÀI LIỆU THAM KHẢO

- Oracle Documentation: https://docs.oracle.com
- Oracle Live SQL: https://livesql.oracle.com (Practice online!)
- Oracle Base: https://oracle-base.com
- Ask Tom: https://asktom.oracle.com

---

**Chúc bạn học tốt! 🚀**

**Lưu ý:** Practice makes perfect! Hãy thực hành nhiều trên môi trường DEV.

---

**Version:** 1.0  
**Created:** 2024  
**For:** Beginners & Junior Developers
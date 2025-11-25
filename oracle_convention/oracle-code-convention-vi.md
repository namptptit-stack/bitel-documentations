# Quy ước lập trình Oracle Database cho Developers

> **Mục đích**: Hướng dẫn các nhà phát triển (Fresher/Junior) viết code SQL hiệu năng cao, clean và maintainable cho hệ thống xử lý 1 triệu requests/ngày.

---

## 📚 Mục lục

1. [Tối ưu Query với Index & Partition](#1-query-optimization-với-index--partition)
2. [Sử dụng IN vs EXISTS](#2-sử-dụng-in-vs-exists)
3. [Phân trang và Top N Records](#3-phân-trang-và-top-n-records)
4. [Xử lý dữ liệu Partition Table](#4-xử-lý-dữ-liệu-partition-table)
5. [Chiến lược Index](#5-index-strategy)
6. [Câu lệnh MERGE](#6-merge-statement)
7. [Query dữ liệu phân cấp](#7-query-hierarchical-data)
8. [Thực hành Clean Code](#8-clean-code-practices)
9. [Anti-Patterns cần tránh](#9-anti-patterns-cần-tránh)
10. [Các mẹo về Performance](#10-performance-tips)

---

## 1. Tối ưu Query với Index & Partition

### 1.1. Không sử dụng Function trên Indexed Column

#### ❌ **KHÔNG NÊN - Index sẽ bị vô hiệu hóa**

```sql
-- Index trên USER_NAME bị ignore
SELECT * 
FROM users 
WHERE UPPER(user_name) = UPPER('john_doe');

-- Index trên CREATED_DATE bị ignore
SELECT * 
FROM orders 
WHERE TO_CHAR(created_date, 'YYYY') = '2024';

-- Index trên MSISDN bị ignore
SELECT * 
FROM subscribers 
WHERE SUBSTR(msisdn, 1, 3) = '084';
```

#### ✅ **NÊN - Index được sử dụng hiệu quả**

```sql
-- Chuẩn hóa dữ liệu ở application layer hoặc dùng Function-based Index
SELECT * 
FROM users 
WHERE user_name = 'john_doe';

-- Sử dụng BETWEEN cho date range
SELECT * 
FROM orders 
WHERE created_date >= DATE '2024-01-01' 
  AND created_date < DATE '2025-01-01';

-- Sử dụng LIKE với leading character
SELECT * 
FROM subscribers 
WHERE msisdn LIKE '084%';
```

**💡 Giải thích:**
- Khi dùng function trên indexed column, Oracle **không thể sử dụng B-Tree Index** → phải Full Table Scan
- **Function-based Index** có thể dùng nhưng tốn storage và maintenance overhead
- Best practice: Chuẩn hóa data trước khi insert/update

---

### 1.2. Tránh các Operator làm mất Index

#### ❌ **KHÔNG NÊN**

```sql
-- NOT LIKE không dùng được index
SELECT * 
FROM users 
WHERE user_name NOT LIKE 'admin%';

-- NOT IN scan toàn bộ table
SELECT * 
FROM orders 
WHERE status NOT IN ('CANCELLED', 'DELETED');

-- <> operator không optimal
SELECT * 
FROM subscribers 
WHERE status <> 'INACTIVE';

-- Leading wildcard không dùng index
SELECT * 
FROM products 
WHERE product_name LIKE '%phone';
```

#### ✅ **NÊN**

```sql
-- Dùng điều kiện positive
SELECT * 
FROM users 
WHERE user_name LIKE 'user_%' 
   OR user_name LIKE 'customer_%';

-- Dùng IN thay vì NOT IN
SELECT * 
FROM orders 
WHERE status IN ('PENDING', 'PROCESSING', 'COMPLETED');

-- Dùng equal operator
SELECT * 
FROM subscribers 
WHERE status = 'ACTIVE';

-- Leading character để sử dụng index
SELECT * 
FROM products 
WHERE product_name LIKE 'iPhone%';
```

**💡 Giải thích:**
- `NOT LIKE`, `NOT IN`, `<>` thường dẫn đến **Full Table Scan**
- `LIKE '%pattern'` (trailing wildcard) không thể dùng index vì không biết starting point
- Với business requirement thực sự cần NOT, cân nhắc:
  - Đảo logic query
  - Sử dụng Bitmap Index cho low cardinality columns
  - Function-based Index nếu pattern lặp lại thường xuyên

---

## 2. Sử dụng IN vs EXISTS

### 2.1. Khi nào dùng IN?

#### ✅ **Dùng IN khi: Subquery trả về ÍT bản ghi**

```sql
-- T2 có ít records (< 1000)
SELECT s.* 
FROM subscribers s
WHERE s.province_id IN (
    SELECT province_id 
    FROM provinces 
    WHERE region = 'NORTH'
);
```

**Execution Plan:**
```
1. Full scan T2 (provinces) - nhỏ, nhanh
2. DISTINCT on province_id
3. Lookup T1 (subscribers) by index
```

---

### 2.2. Khi nào dùng EXISTS?

#### ✅ **Dùng EXISTS khi: Main table NHỎ hơn, subquery table LỚN + có INDEX**

```sql
-- T1 nhỏ hơn, T2 lớn NHƯNG có index trên SUBSCRIBER_ID
SELECT o.*
FROM orders o
WHERE EXISTS (
    SELECT 1 
    FROM subscribers s
    WHERE s.subscriber_id = o.subscriber_id
      AND s.status = 'ACTIVE'
);
```

**Execution Plan:**
```
1. Loop qua orders (O) - table nhỏ
2. Mỗi row: Index seek trên subscribers(SUBSCRIBER_ID) - nhanh
3. Short-circuit khi tìm thấy match đầu tiên
```

---

### 2.3. So sánh Performance

| Tình huống | Performance của IN | Performance của EXISTS | Khuyến nghị |
|----------|---------------|-------------------|-------------|
| T2 nhỏ (< 1K dòng), T1 lớn | ⭐⭐⭐⭐⭐ Nhanh | ⭐⭐⭐ Trung bình | **Dùng IN** |
| T1 nhỏ, T2 lớn + indexed | ⭐⭐ Chậm | ⭐⭐⭐⭐⭐ Nhanh | **Dùng EXISTS** |
| Cả 2 đều lớn | ⭐⭐⭐ Trung bình | ⭐⭐⭐ Trung bình | Cả 2 tương đương |
| T2 có duplicate | ⭐⭐⭐⭐ (auto DISTINCT) | ⭐⭐⭐⭐⭐ (stop at first) | **Dùng EXISTS** |

---

### 2.4. Ví dụ thực tế

```sql
-- BAD: T2 có 10M records, không có index
SELECT * 
FROM orders o
WHERE o.subscriber_id IN (
    SELECT subscriber_id 
    FROM subscriber_history  -- 10M dòng
    WHERE action_date >= TRUNC(SYSDATE) - 30
);
-- → Full scan 10M dòng → DISTINCT → Very slow!

-- GOOD: Đảo logic, dùng EXISTS với indexed join
SELECT * 
FROM orders o
WHERE EXISTS (
    SELECT 1 
    FROM subscriber_history sh
    WHERE sh.subscriber_id = o.subscriber_id  -- Index seek
      AND sh.action_date >= TRUNC(SYSDATE) - 30
);
-- → Loop orders → Index lookup subscriber_history → Nhanh!
```

---

## 3. Phân trang và Top N Records

### 3.1. Lấy Top N với ROWNUM

#### ✅ **Đúng cách - Subquery để sort trước**

```sql
-- Lấy 10 đơn hàng có giá trị cao nhất
SELECT * 
FROM (
    SELECT order_id, customer_id, total_amount
    FROM orders 
    ORDER BY total_amount DESC
)
WHERE ROWNUM <= 10;
```

#### ❌ **Sai cách - ROWNUM gán trước khi sort**

```sql
-- Chỉ lấy 10 dòng đầu tiên, SAU ĐÓ mới sort → SAI!
SELECT * 
FROM orders 
WHERE ROWNUM <= 10
ORDER BY total_amount DESC;
```

**💡 Giải thích:**
- `ROWNUM` được gán **TRƯỚC KHI** `ORDER BY` thực thi
- Phải wrap query trong subquery để sort trước

---

### 3.2. Lấy Top N với RANK() - Xử lý TIE

```sql
-- Lấy TOP 10 điểm cao nhất, bao gồm các điểm bằng nhau
SELECT * 
FROM (
    SELECT 
        student_id,
        student_name,
        score,
        RANK() OVER (ORDER BY score DESC) as rank_score
    FROM students
)
WHERE rank_score <= 10
ORDER BY rank_score;
```

**Kết quả:**
```
STUDENT_ID | SCORE | RANK_SCORE
-----------|-------|------------
1001       | 95    | 1
1002       | 95    | 1  ← Cùng rank
1003       | 94    | 3  ← Bỏ qua rank 2
1004       | 94    | 3
1005       | 92    | 5
...
```

---

### 3.3. Phân trang với OFFSET-FETCH (Oracle 12c+)

```sql
-- Lấy page 3, mỗi page 20 records
SELECT order_id, customer_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;

-- Page 1: OFFSET 0
-- Page 2: OFFSET 20
-- Page 3: OFFSET 40
```

#### Với Oracle 11g trở xuống:

```sql
-- Pagination với ROWNUM
SELECT * 
FROM (
    SELECT a.*, ROWNUM rnum 
    FROM (
        SELECT order_id, customer_id, order_date
        FROM orders
        ORDER BY order_date DESC
    ) a
    WHERE ROWNUM <= 60  -- page * page_size
)
WHERE rnum > 40;  -- (page - 1) * page_size
```

---

## 4. Xử lý dữ liệu Partition Table

### 4.1. Partition Pruning - Luôn filter theo Partition Key

#### ✅ **NÊN - Partition Pruning**

```sql
-- Table: ORDERS partitioned by ORDER_DATE (monthly)
-- Chỉ scan partition tháng 11/2024
SELECT order_id, customer_id, total_amount
FROM orders
WHERE order_date >= DATE '2024-11-01'
  AND order_date < DATE '2024-12-01'
  AND customer_id = 12345;

-- Execution Plan:
-- PARTITION RANGE ITERATOR (PARTITION: 2024-11)
-- → Chỉ scan 1 partition thay vì toàn bộ table
```

#### ❌ **KHÔNG NÊN - Full Partition Scan**

```sql
-- Không có partition key trong WHERE
SELECT order_id, customer_id, total_amount
FROM orders
WHERE customer_id = 12345;

-- Execution Plan:
-- PARTITION RANGE ALL
-- → Phải scan TẤT CẢ partitions → Chậm!
```

**💡 Giải thích:**
- **Partition Pruning** = Oracle tự động bỏ qua các partition không liên quan
- **Luôn include partition key** trong WHERE clause để trigger pruning
- Áp dụng cho: `SELECT`, `UPDATE`, `DELETE`

---

### 4.2. UPDATE trên Partition Table

```sql
-- ĐÚNG: Include partition key
UPDATE orders
SET status = 'COMPLETED'
WHERE order_date >= DATE '2024-11-01'
  AND order_date < DATE '2024-12-01'
  AND order_id = 999888;

-- SAI: Thiếu partition key
UPDATE orders
SET status = 'COMPLETED'
WHERE order_id = 999888;
-- → Phải scan all partitions để tìm 1 row
```

---

### 4.3. DELETE trên Partition Table

```sql
-- CÁCH 1: DELETE với partition key (Recommended)
DELETE FROM orders
WHERE order_date >= DATE '2024-01-01'
  AND order_date < DATE '2024-02-01'
  AND status = 'CANCELLED';

-- CÁCH 2: DROP partition (nhanh hơn nhiều với large data)
-- Chỉ dùng khi cần xóa toàn bộ partition
ALTER TABLE orders DROP PARTITION orders_202401;
-- → Instant, không generate UNDO/REDO
```

**💡 Mẹo Performance:**
- `DROP PARTITION` nhanh gấp 100x so với `DELETE` với large datasets
- Sử dụng cho archiving/purging old data

---

### 4.4. Local Index trên Partition Table

```sql
-- Table: TRANSACTIONS partitioned by TXN_DATE
-- Local Index: Mỗi partition có index riêng

-- Query tận dụng partition pruning + local index
SELECT * 
FROM transactions
WHERE txn_date >= DATE '2024-11-01'
  AND txn_date < DATE '2024-12-01'
  AND subscriber_id = 'VN0123456789';

-- Execution Plan:
-- 1. Partition Pruning → Chỉ scan partition NOV_2024
-- 2. Local Index Seek on subscriber_id trong partition đó
-- → Cực kỳ nhanh!
```

**Chiến lược Index cho Partition Table:**
```sql
-- Tạo Local Index (preferred)
CREATE INDEX idx_txn_subscriber 
ON transactions(subscriber_id) LOCAL;
-- → Mỗi partition có index segment riêng
-- → Dễ maintain, balance load

-- Global Index (ít dùng)
CREATE INDEX idx_txn_subscriber_global
ON transactions(subscriber_id) GLOBAL;
-- → 1 index cho toàn bộ table
-- → Phức tạp khi add/drop partition
```

---

## 5. Chiến lược Index

### 5.1. Khi nào NÊN đánh Index?

#### ✅ **Primary Key & Unique Nhược điểmtraint**

```sql
-- Tự động tạo unique index
CREATE TABLE users (
    user_id NUMBER PRIMARY KEY,
    username VARCHAR2(50) UNIQUE,
    email VARCHAR2(100) UNIQUE
);
-- → Oracle auto-create index trên USER_ID, USERNAME, EMAIL
```

---

#### ✅ **Foreign Key Columns**

```sql
-- Orders JOIN với Customers rất thường xuyên
CREATE INDEX idx_orders_customer 
ON orders(customer_id);

-- Tránh Full Table Scan khi JOIN
SELECT o.*, c.customer_name
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= TRUNC(SYSDATE) - 7;
```

---

#### ✅ **Các cột Frequently Filtered**

```sql
-- Các cột WHERE, ORDER BY, GROUP BY
CREATE INDEX idx_orders_date_status 
ON orders(order_date, status);

-- Tối ưu query pattern:
SELECT * FROM orders
WHERE order_date >= TRUNC(SYSDATE) - 30
  AND status = 'PENDING'
ORDER BY order_date DESC;
```

---

### 5.2. Khi nào KHÔNG NÊN đánh Index?

#### ❌ **Bảng nhỏ (< 10,000 dòng)**

```sql
-- Table: PROVINCES (63 dòng)
-- KHÔNG cần index ngoài PK
-- Full Table Scan nhanh hơn Index Scan với table nhỏ
```

---

#### ❌ **Cột Low Cardinality**

```sql
-- Column: STATUS có 3 giá trị (ACTIVE, INACTIVE, DELETED)
-- B-Tree Index không hiệu quả
-- Nếu cần: Dùng BITMAP INDEX

-- Thay vì:
CREATE INDEX idx_users_status ON users(status);

-- Dùng:
CREATE BITMAP INDEX idx_users_status_bmp ON users(status);
-- Chỉ dùng cho low cardinality + ít UPDATE
```

---

#### ❌ **Cột Frequently Updated**

```sql
-- Column: LAST_UPDATED timestamp update mỗi transaction
-- Index overhead > Performance gain
-- Mỗi UPDATE phải update cả Index → Chậm
```

---

### 5.3. Composite Index - Thứ tự Column

#### 📌 **Quy tắc: Column có Selectivity cao nhất đứng TRƯỚC**

```sql
-- Query pattern:
-- WHERE status = 'ACTIVE' AND created_date >= DATE '2024-01-01'

-- Selectivity:
-- - status: 3 giá trị → Selectivity = 33%
-- - created_date: 365 days/year → Selectivity = 0.27%

-- ✅ ĐÚNG: created_date trước
CREATE INDEX idx_users_date_status 
ON users(created_date, status);

-- ❌ SAI: status trước
CREATE INDEX idx_users_status_date 
ON users(status, created_date);
```

**💡 Giải thích:**
- **Selectivity** = Khả năng lọc bớt dòng
- Column có selectivity cao → Giảm dòng cần scan sớm → Nhanh hơn
- Nguyên tắc: `Unique > Date > Status`

---

### 5.4. Covering Index

```sql
-- Query thường xuyên:
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = :customer_id
  AND order_date >= TRUNC(SYSDATE) - 90;

-- ✅ Covering Index: Include tất cả columns trong SELECT
CREATE INDEX idx_orders_covering 
ON orders(customer_id, order_date, order_id, total_amount);

-- → Index-Only Scan, không cần access table → Rất nhanh!
```

**Lợi ích:**
- Không cần access table data blocks
- Giảm I/O operations
- Tăng cache hit rate

---

### 5.5. Function-Based Index

```sql
-- Query pattern với function:
SELECT * FROM users
WHERE UPPER(email) = UPPER(:email);

-- ✅ Function-based Index
CREATE INDEX idx_users_email_upper 
ON users(UPPER(email));

-- Query phải MATCH function trong index
SELECT * FROM users
WHERE UPPER(email) = 'JOHN.DOE@EXAMPLE.COM';
```

**⚠️ Trade-off:**
- **Ưu điểm**: Tối ưu function queries
- **Nhược điểm**: Tăng storage, phải maintain, rebuild khi modify function

---

## 6. Câu lệnh MERGE

### 6.1. Upsert (INSERT hoặc UPDATE)

```sql
-- Đồng bộ dữ liệu từ staging table → production table
MERGE INTO customers c
USING staging_customers s
ON (c.customer_id = s.customer_id)
WHEN MATCHED THEN
    UPDATE SET 
        c.customer_name = s.customer_name,
        c.email = s.email,
        c.updated_date = SYSDATE
WHEN NOT MATCHED THEN
    INSERT (customer_id, customer_name, email, created_date)
    VALUES (s.customer_id, s.customer_name, s.email, SYSDATE);
```

**💡 Ưu điểm:**
- 1 câu lệnh thay vì 2 (`IF EXISTS → UPDATE ELSE INSERT`)
- Atomic operation
- Giảm network roundtrips
- Tối ưu với bulk operations

---

### 6.2. Conditional Update/Insert

```sql
-- Chỉ update nếu giá trị thay đổi
MERGE INTO inventory i
USING (
    SELECT product_id, quantity, unit_price
    FROM temp_inventory
) t
ON (i.product_id = t.product_id)
WHEN MATCHED THEN
    UPDATE SET 
        i.quantity = t.quantity,
        i.unit_price = t.unit_price
    WHERE i.quantity != t.quantity 
       OR i.unit_price != t.unit_price  -- Chỉ update nếu khác
WHEN NOT MATCHED THEN
    INSERT VALUES (t.product_id, t.quantity, t.unit_price);
```

---

### 6.3. MERGE với DELETE

```sql
-- Soft delete: Update status = DELETED nếu không tồn tại trong source
MERGE INTO products p
USING temp_products t
ON (p.product_id = t.product_id)
WHEN MATCHED THEN
    UPDATE SET 
        p.product_name = t.product_name,
        p.price = t.price,
        p.status = 'ACTIVE'
WHEN NOT MATCHED BY SOURCE THEN
    UPDATE SET p.status = 'DELETED'
    WHERE p.status = 'ACTIVE';
```

---

## 7. Query dữ liệu phân cấp

### 7.1. Tree Structure với START WITH ... CONNECT BY

```sql
-- Table: CATEGORIES (category_id, parent_id, category_name)
-- Root: parent_id IS NULL

SELECT 
    LEVEL as tree_level,
    category_id,
    LPAD(' ', (LEVEL - 1) * 4, ' ') || category_name as category_display,
    SYS_CONNECT_BY_PATH(category_name, ' > ') as full_path,
    CONNECT_BY_ROOT category_id as root_category_id,
    CONNECT_BY_ISLEAF as is_leaf_node
FROM categories
START WITH parent_id IS NULL  -- Các node gốc
CONNECT BY PRIOR category_id = parent_id  -- Quan hệ Cha → Con
ORDER SIBLINGS BY category_name;
```

**Kết quả:**
```
LEVEL | CATEGORY_DISPLAY        | FULL_PATH
------|------------------------|---------------------------
1     | Electronics            |  > Electronics
2     |     Computers          |  > Electronics > Computers
3     |         Laptops        |  > Electronics > Computers > Laptops
3     |         Desktops       |  > Electronics > Computers > Desktops
2     |     Mobile Phones      |  > Electronics > Mobile Phones
```

---

### 7.2. Recursive CTE (Oracle 11gR2+)

```sql
-- Lấy tất cả subordinates của 1 manager
WITH employee_hierarchy (emp_id, emp_name, manager_id, lvl, path) AS (
    -- Anchor: Root employee
    SELECT employee_id, employee_name, manager_id, 1, employee_name
    FROM employees
    WHERE employee_id = :manager_id
    
    UNION ALL
    
    -- Recursive: Subordinates
    SELECT e.employee_id, e.employee_name, e.manager_id, h.lvl + 1,
           h.path || ' > ' || e.employee_name
    FROM employees e
      JOIN employee_hierarchy h ON e.manager_id = h.emp_id
    WHERE h.lvl < 10  -- Ngăn vòng lặp vô hạn
)
SELECT * FROM employee_hierarchy
ORDER BY lvl, emp_name;
```

---

## 8. Thực hành Clean Code

### 8.1. Quy ước Đặt tên

```sql
-- ✅ Tables: Danh từ số nhiều, lowercase
CREATE TABLE customers (...);
CREATE TABLE order_items (...);

-- ✅ Columns: Số ít, mô tả rõ ràng
customer_id NUMBER NOT NULL
customer_name VARCHAR2(100)
created_date DATE DEFAULT SYSDATE

-- ✅ Indexes: idx_{table}_{columns}
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- ✅ Primary Keys: pk_{table}
ALTER TABLE customers ADD CONSTRAINT pk_customers PRIMARY KEY (customer_id);

-- ✅ Foreign Keys: fk_{child_table}_{parent_table}
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_customers 
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

---

### 8.2. Định dạng Query

```sql
-- ❌ TỆ: Không đọc được
SELECT c.customer_id,c.customer_name,o.order_id,o.order_date,SUM(oi.quantity*oi.unit_price) FROM customers c JOIN orders o ON c.customer_id=o.customer_id JOIN order_items oi ON o.order_id=oi.order_id WHERE o.order_date>=TRUNC(SYSDATE)-30 GROUP BY c.customer_id,c.customer_name,o.order_id,o.order_date;

-- ✅ TỐT: Sạch, dễ đọc
SELECT 
    c.customer_id,
    c.customer_name,
    o.order_id,
    o.order_date,
    SUM(oi.quantity * oi.unit_price) as total_amount
FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= TRUNC(SYSDATE) - 30
GROUP BY 
    c.customer_id, 
    c.customer_name, 
    o.order_id, 
    o.order_date
ORDER BY o.order_date DESC;
```

**Quy tắc Định dạng:**
- 1 clause mỗi dòng (`SELECT`, `FROM`, `WHERE`, `GROUP BY`, `ORDER BY`)
- Indent JOIN conditions
- Căn chỉnh tên cột theo chiều dọc
- Chia lists dài thành nhiều dòng

---

### 8.3. Dùng Meaningful Aliases

```sql
-- ❌ BAD: Aliases 1 chữ cái
SELECT 
    a.id, a.name, b.id, b.date
FROM t1 a
  JOIN t2 b ON a.id = b.fk;

-- ✅ GOOD: Aliases mô tả rõ ràng
SELECT 
    cust.customer_id,
    cust.customer_name,
    ord.order_id,
    ord.order_date
FROM customers cust
  JOIN orders ord ON cust.customer_id = ord.customer_id;
```

---

### 8.4. Tránh SELECT *

```sql
-- ❌ BAD: SELECT *
SELECT * 
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id;
-- Vấn đề:
-- - Network overhead với unused columns
-- - Tên cột mơ hồ
-- - Breaks khi schema changes

-- ✅ GOOD: Columns rõ ràng
SELECT 
    o.order_id,
    o.order_date,
    o.total_amount,
    c.customer_name,
    c.email
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id;
```

---

### 8.5. Dùng Bind Variables

```sql
-- ❌ BAD: Hard-coded giá trị (SQL Injection risk + Parse overhead)
SELECT * FROM users WHERE username = 'john' AND password = 'pass123';

-- ✅ GOOD: Bind variables (Execution plan tái sử dụng + Bảo mật)
SELECT * FROM users WHERE username = :username AND password = :password;
```

**Lợi ích:**
- **Execution plan reuse** → Giảm parse time
- **Ngăn chặn SQL Injection**
- Giảm library cache fragmentation

---

## 9. Anti-Patterns cần tránh

### 9.1. SELECT trong Loop

```sql
-- ❌ ANTI-PATTERN: N+1 Query Problem
BEGIN
    FOR rec IN (SELECT order_id FROM orders WHERE order_date = TRUNC(SYSDATE)) LOOP
        -- SELECT trong loop → 1000 orders = 1000 SELECTs!
        SELECT customer_name INTO v_customer_name
        FROM customers
        WHERE customer_id = rec.customer_id;
        
        -- Xử lý...
    END LOOP;
END;

-- ✅ SOLUTION: JOIN hoặc BULK COLLECT
SELECT 
    o.order_id,
    c.customer_name
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date = TRUNC(SYSDATE);
```

---

### 9.2. Row-by-Row Xử lýing

```sql
-- ❌ BAD: Xử lý từng row
FOR rec IN (SELECT * FROM orders WHERE status = 'PENDING') LOOP
    UPDATE orders 
    SET status = 'PROCESSING'
    WHERE order_id = rec.order_id;
END LOOP;
COMMIT;

-- ✅ GOOD: Bulk update
UPDATE orders
SET status = 'PROCESSING'
WHERE status = 'PENDING';
COMMIT;
```

---

### 9.3. DISTINCT thay vì đánh đúng JOIN

```sql
-- ❌ BAD: DISTINCT để fix duplicate do bad join
SELECT DISTINCT
    o.order_id,
    c.customer_name
FROM orders o
  JOIN order_items oi ON o.order_id = oi.order_id
  JOIN customers c ON o.customer_id = c.customer_id;
-- → DISTINCT rất tốn resources

-- ✅ GOOD: Đánh đúng JOIN
SELECT 
    o.order_id,
    c.customer_name
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id;
-- → Không cần ORDER_ITEMS nếu chỉ cần customer info
```

---

### 9.4. Nested Subqueries (Subquery Hell)

```sql
-- ❌ BAD: Nested subqueries khó đọc
SELECT customer_name
FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
    WHERE order_date IN (
        SELECT order_date
        FROM order_dates
        WHERE EXTRACT(YEAR FROM order_date) = 2024
    )
);

-- ✅ GOOD: JOIN hoặc WITH clause
WITH recent_dates AS (
    SELECT DISTINCT order_date
    FROM order_dates
    WHERE EXTRACT(YEAR FROM order_date) = 2024
)
SELECT DISTINCT c.customer_name
FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN recent_dates rd ON o.order_date = rd.order_date;
```

---

## 10. Các mẹo về Performance

### 10.1. Dùng EXPLAIN PLAN

```sql
-- Analyze execution plan trước khi deploy
EXPLAIN PLAN FOR
SELECT o.order_id, c.customer_name
FROM orders o
  JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= TRUNC(SYSDATE) - 30;

-- Xem execution plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Red Flags trong Execution Plan:**
- `FULL TABLE SCAN` trên large tables
- `CARTESIAN JOIN`
- `SORT AGGREGATE` với large dataset
- `INDEX FULL SCAN` thay vì `INDEX RANGE SCAN`

---

### 10.2. Collect Statistics định kỳ

```sql
-- Sau khi bulk insert/update/delete
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'SCHEMA_NAME',
        tabname => 'ORDERS',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO',
        cascade => TRUE  -- Bao gồm indexes
    );
END;
/
```

**Khi nào cần gather stats:**
- Sau bulk data load
- Sau tạo/rebuild index
- Định kỳ hàng tuần cho active tables

---

### 10.3. Dùng Bulk Operations

```sql
-- ✅ BULK COLLECT & FORALL
DECLARE
    TYPE t_order_ids IS TABLE OF orders.order_id%TYPE;
    v_order_ids t_order_ids;
BEGIN
    -- Fetch multiple dòng at once
    SELECT order_id
    BULK COLLECT INTO v_order_ids
    FROM orders
    WHERE status = 'PENDING'
      AND ROWNUM <= 1000;
    
    -- Update multiple dòng at once
    FORALL i IN 1..v_order_ids.COUNT
        UPDATE orders
        SET status = 'PROCESSING',
            updated_date = SYSDATE
        WHERE order_id = v_order_ids(i);
    
    COMMIT;
END;
/
```

**Performance:**
- `BULK COLLECT`: Giảm context switches giữa PL/SQL ↔ SQL engine
- `FORALL`: Batch DML operations → Nhanh hơn 10-100x vs row-by-row

---

### 10.4. Tối ưu Date Operations

```sql
-- ❌ BAD: Tính toán lại mỗi row
SELECT * FROM orders
WHERE TO_CHAR(order_date, 'YYYY-MM-DD') = TO_CHAR(SYSDATE, 'YYYY-MM-DD');

-- ✅ GOOD: Tính 1 lần, dùng BETWEEN
SELECT * FROM orders
WHERE order_date >= TRUNC(SYSDATE)
  AND order_date < TRUNC(SYSDATE) + 1;
```

---

### 10.5. Limit Kết quả Sets

```sql
-- ❌ BAD: Fetch millions of dòng không cần thiết
SELECT * FROM transactions
WHERE txn_date >= DATE '2020-01-01';
-- → Có thể return 10M+ dòng!

-- ✅ GOOD: Always limit cho interactive queries
SELECT * FROM transactions
WHERE txn_date >= TRUNC(SYSDATE) - 30
  AND ROWNUM <= 1000;

-- Hoặc pagination
SELECT * FROM transactions
WHERE txn_date >= TRUNC(SYSDATE) - 30
ORDER BY txn_date DESC
FETCH FIRST 100 ROWS ONLY;
```

---

### 10.6. Tránh Implicit Conversions

```sql
-- ❌ BAD: Implicit conversion VARCHAR → NUMBER
CREATE TABLE users (
    user_id NUMBER PRIMARY KEY,
    msisdn VARCHAR2(15)
);

SELECT * FROM users WHERE msisdn = 123456789;
-- → Oracle converts 123456789 to '123456789'
-- → Index trên MSISDN không được dùng!

-- ✅ GOOD: Explicit datatype
SELECT * FROM users WHERE msisdn = '123456789';
```

---

### 10.7. Monitor Long-Running Queries

```sql
-- Tìm queries đang chạy > 1 phút
SELECT 
    s.sid,
    s.serial#,
    s.username,
    s.program,
    s.sql_id,
    ROUND(s.last_call_et / 60, 2) as running_minutes,
    sq.sql_text
FROM v$session s
  JOIN v$sql sq ON s.sql_id = sq.sql_id
WHERE s.status = 'ACTIVE'
  AND s.username IS NOT NULL
  AND s.last_call_et > 60  -- > 1 minute
ORDER BY s.last_call_et DESC;
```

---

## 11. Checklist trước khi Deploy

### Pre-Deploy Checklist

- [ ] **Explain Plan analyzed**
  - Không có Full Table Scan trên large tables
  - Indexes được sử dụng đúng cách
  - Join order hợp lý

- [ ] **Indexes reviewed**
  - Partition key included trong WHERE clause
  - Composite index column order đúng
  - Covering index cho high-frequency queries

- [ ] **Bind variables sử dụng**
  - Không hard-code giá trị
  - Parameterized queries

- [ ] **Kết quả set limited**
  - ROWNUM hoặc FETCH FIRST cho interactive queries
  - Pagination cho large datasets

- [ ] **Code reviewed**
  - Formatting clean, readable
  - Meaningful aliases
  - Comments cho complex logic

- [ ] **Testing completed**
  - Unit test với sample data
  - Performance test với production-like volume
  - Edge cases covered (NULL, empty, large datasets)

---

## 12. Resources & Tools

### Recommended Tools

1. **SQL Developer** - IDE với EXPLAIN PLAN visual
2. **Toad for Oracle** - Advanced performance tuning
3. **PL/SQL Developer** - Debugging & profiling
4. **Oracle SQL Tuning Advisor** - Automated recommendations

### Learning Resources

- Oracle Documentation: https://docs.oracle.com/en/database/
- Ask TOM: https://asktom.oracle.com/
- Oracle Performance Tuning Guide
- Oracle SQL Language Reference

---

## 📝 Tổng kết

### Key Takeaways

1. **Always think about Index & Partition** khi viết WHERE clause
2. **Avoid function trên indexed columns** → Mất index
3. **Dùng IN cho small subquery**, **EXISTS cho large table** với index
4. **Partition pruning** = Must-have cho partition tables
5. **Clean code** = Maintainable code = Happy team
6. **EXPLAIN PLAN** = Best friend cho performance tuning
7. **Bulk operations** > Row-by-row processing
8. **Test với production-like data volume** trước khi deploy

---

## 🤝 Đóng góp

Nếu bạn có suggestions hoặc tìm thấy issues, vui lòng:
1. Tạo merge request với ví dụ cụ thể
2. Discuss với team lead
3. Update document sau khi approved

---

**Document Version**: 1.0  
**Last Updated**: 2024-11  
**Maintained by**: Database Team

---

*"Tối ưu hóa quá sớm là gốc rễ của mọi tội lỗi, nhưng biết cách viết SQL hiệu quả ngay từ đầu là kỹ thuật thông minh."* 💪

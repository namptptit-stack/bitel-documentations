# CODING CONVENTION CHECKLIST - ORACLE DATABASE
## Danh sách kiểm tra chuẩn cho Developer

---

## 📋 MỤC LỤC
1. [Tối ưu hóa câu lệnh SQL](#1-tối-ưu-hóa-câu-lệnh-sql)
2. [Làm việc với Partition và Index](#2-làm-việc-với-partition-và-index)
3. [Sử dụng IN vs EXISTS](#3-sử-dụng-in-vs-exists)
4. [Cấu trúc dữ liệu hình cây](#4-cấu-trúc-dữ-liệu-hình-cây)
5. [Lấy N giá trị theo sắp xếp](#5-lấy-n-giá-trị-theo-sắp-xếp)
6. [Database Link](#6-database-link)
7. [Lệnh MERGE](#7-lệnh-merge)
8. [Xử lý dữ liệu trùng](#8-xử-lý-dữ-liệu-trùng)
9. [Temporary Tables](#9-temporary-tables)
10. [Transaction và Session](#10-transaction-và-session)
11. [Bảo mật và Best Practices](#11-bảo-mật-và-best-practices)

---

## 1. TỐI ƯU HÓA CÂU LỆNH SQL

### ✅ PHẢI LÀM

```sql
-- ✓ Sử dụng điều kiện trực tiếp trên cột có index
SELECT * FROM users WHERE user_name = 'USER_NAME';

-- ✓ Sử dụng LIKE với % ở cuối
SELECT * FROM users WHERE user_name LIKE 'USER_NAME%';

-- ✓ Sử dụng IN với giá trị cụ thể
SELECT * FROM users WHERE user_name IN ('USER1', 'USER2', 'USER3');
```

### ❌ KHÔNG NÊN LÀM

```sql
-- ✗ KHÔNG dùng hàm trên cột có index/partition
SELECT * FROM users WHERE UPPER(user_name) = UPPER('USER_NAME');

-- ✗ KHÔNG dùng NOT LIKE với cột có index
SELECT * FROM users WHERE user_name NOT LIKE 'USER_NAME%';

-- ✗ KHÔNG dùng NOT IN với cột có index
SELECT * FROM users WHERE user_name NOT IN ('USER1', 'USER2');

-- ✗ KHÔNG dùng <> với cột có index
SELECT * FROM users WHERE user_name <> 'USER_NAME';

-- ✗ KHÔNG dùng LIKE với % ở đầu
SELECT * FROM users WHERE user_name LIKE '%USER_NAME';
```

**Lý do:** Các toán tử và hàm trên làm mất tác dụng của index, khiến database phải scan toàn bộ bảng.

---

## 2. LÀM VIỆC VỚI PARTITION VÀ INDEX

### ⚠️ Nguyên tắc vàng:
- [ ] **KHÔNG** sử dụng hàm trên cột partition
- [ ] **KHÔNG** sử dụng `NOT LIKE`, `NOT IN`, `<>` trên cột partition
- [ ] **KHÔNG** sử dụng `LIKE '%text'` (% ở đầu) trên cột có index
- [ ] Luôn kiểm tra execution plan trước khi deploy

---

## 3. SỬ DỤNG IN VS EXISTS

### 📊 Quy tắc lựa chọn:

#### Dùng **IN** khi:
- [ ] Bảng con (subquery) **NHỎ HƠN** bảng chính
- [ ] Cần trả về DISTINCT values

```sql
-- Dùng IN khi T2 nhỏ hơn T1
SELECT * FROM t1 
WHERE x IN (SELECT y FROM t2);

-- Tương đương với:
SELECT * FROM t1, (SELECT DISTINCT y FROM t2) t2 
WHERE t1.x = t2.y;
```

#### Dùng **EXISTS** khi:
- [ ] Bảng con (subquery) **LỚN HƠN** bảng chính
- [ ] Cột điều kiện có index
- [ ] Cần performance tốt hơn với dữ liệu lớn

```sql
-- Dùng EXISTS khi T2 lớn hơn T1 và y có index
SELECT * FROM t1 
WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.y = t1.x);
```

**Lưu ý:** Nếu cả 2 bảng đều lớn hoặc tương đương → hiệu suất tương tự nhau

---

## 4. CẤU TRÚC DỮ LIỆU HÌNH CÂY

### 📌 Khi làm việc với dữ liệu hierarchical (cha-con):

```sql
-- Template chuẩn cho query hierarchical
SELECT 
    object_code,
    LPAD(' ', LEVEL * 5, ' ') || object_name AS display_name,
    LEVEL AS node_level,
    CONNECT_BY_ROOT(object_code) AS root_code
FROM objects
WHERE app_id = 2609
START WITH parent_id IS NULL  -- Điểm bắt đầu (node gốc)
CONNECT BY PRIOR object_id = parent_id  -- Quan hệ cha-con
ORDER SIBLINGS BY object_name;  -- Sắp xếp trong cùng cấp
```

### ✅ Checklist:
- [ ] Xác định đúng node gốc trong `START WITH`
- [ ] Định nghĩa chính xác quan hệ cha-con trong `CONNECT BY PRIOR`
- [ ] Sử dụng `LEVEL` để lấy cấp độ
- [ ] Sử dụng `CONNECT_BY_ROOT` để lấy thông tin node gốc
- [ ] Dùng `ORDER SIBLINGS BY` thay vì `ORDER BY` để giữ cấu trúc cây

---

## 5. LẤY N GIÁ TRỊ THEO SẮP XẾP

### Cách 1: Sử dụng ROWNUM (Lấy đúng N records)

```sql
-- Lấy top 10 sinh viên điểm cao nhất
SELECT * FROM (
    SELECT * FROM sinhvien
    ORDER BY diem DESC
) WHERE ROWNUM <= 10;
```

**Ưu điểm:** Đơn giản, nhanh  
**Nhược điểm:** Chỉ lấy đúng 10 records, bỏ qua các sinh viên cùng điểm

### Cách 2: Sử dụng RANK() (Lấy top N rankings)

```sql
-- Lấy sinh viên thuộc top 10 điểm (có thể > 10 người)
SELECT * FROM (
    SELECT 
        ma_sv,
        diem,
        RANK() OVER (ORDER BY diem DESC) AS rank_diem
    FROM sinhvien
)
WHERE rank_diem <= 10
ORDER BY rank_diem;
```

**Ưu điểm:** Lấy đủ tất cả sinh viên cùng ranking  
**Nhược điểm:** Có thể trả về > N records nếu có nhiều giá trị bằng nhau

### ✅ Checklist:
- [ ] Xác định rõ yêu cầu: lấy **đúng N records** hay **top N rankings**?
- [ ] **KHÔNG** dùng `WHERE ROWNUM <= N` trước khi `ORDER BY`
- [ ] Xem xét dùng `DENSE_RANK()` nếu cần ranking liên tục
- [ ] Xem xét dùng `ROW_NUMBER()` nếu cần số thứ tự duy nhất

---

## 6. DATABASE LINK

### ✅ Tạo Database Link:

```sql
CREATE SHARED DATABASE LINK dbl_cm_pos
CONNECT TO bccs_anypay_app IDENTIFIED BY password123
AUTHENTICATED BY bccs_anypay_app IDENTIFIED BY password123
USING '(DESCRIPTION =
    (ADDRESS_LIST =
        (ADDRESS = (PROTOCOL = TCP)(HOST = 10.58.3.65)(PORT = 1521))
    )
    (CONNECT_DATA =
        (SID = poscus)
    )
)';
```

### ✅ Sử dụng Database Link:

```sql
-- Query từ database khác
SELECT * FROM schema_name.table_name@dbl_cm_pos;

-- Join với bảng local
SELECT a.*, b.column1
FROM local_table a
JOIN remote_schema.remote_table@dbl_cm_pos b
ON a.id = b.id;
```

### ✅ Checklist:
- [ ] Tên database link có ý nghĩa, dễ nhớ
- [ ] Lưu credentials an toàn
- [ ] Test connection trước khi sử dụng
- [ ] Xem xét performance khi query qua network
- [ ] Cân nhắc dùng materialized view nếu query thường xuyên

---

## 7. LỆNH MERGE

### 📌 Khi nào dùng MERGE?
- [ ] Cần insert hoặc update dựa vào điều kiện
- [ ] Không muốn kiểm tra tồn tại trước
- [ ] Sync dữ liệu giữa 2 bảng

```sql
-- Template chuẩn
MERGE INTO target_table a
USING source_table b
ON (a.object_id = b.object_id)
WHEN MATCHED THEN
    UPDATE SET 
        a.status = b.status,
        a.updated_date = SYSDATE
WHEN NOT MATCHED THEN
    INSERT (object_id, status, created_date)
    VALUES (b.object_id, b.status, SYSDATE);
```

### ✅ Best Practices:
- [ ] Điều kiện ON phải có index
- [ ] Cân nhắc dùng hints nếu cần
- [ ] Test với dữ liệu mẫu trước
- [ ] Log số lượng rows affected

---

## 8. XỬ LÝ DỮ LIỆU TRÙNG

### ⚠️ Vấn đề: Bảng không có khóa chính, có nhiều bản ghi trùng

```sql
-- Xóa bản ghi trùng, giữ lại 1 bản ghi
DELETE FROM users
WHERE ROWID IN (
    SELECT MAX(ROWID) AS row_id
    FROM users
    GROUP BY user_id
    HAVING COUNT(*) > 1
);
```

### ✅ Checklist:
- [ ] Backup dữ liệu trước khi xóa
- [ ] Xác định đúng cột để group (business key)
- [ ] Chạy SELECT trước để preview
- [ ] Nếu có > 2 bản ghi trùng, chạy nhiều lần
- [ ] Cân nhắc thêm constraint sau khi xóa xong

---

## 9. TEMPORARY TABLES

### 📌 Khi nào dùng Temporary Tables?
- [ ] Cần lưu trữ dữ liệu trung gian
- [ ] Xử lý dữ liệu qua nhiều bước
- [ ] Tránh conflict giữa các session
- [ ] Không muốn tạo bảng vĩnh viễn

### Cách 1: Dữ liệu tồn tại trong Transaction

```sql
CREATE GLOBAL TEMPORARY TABLE my_temp_table (
    column1 NUMBER,
    column2 VARCHAR2(100)
) ON COMMIT DELETE ROWS;  -- Xóa khi commit
```

### Cách 2: Dữ liệu tồn tại trong Session

```sql
CREATE GLOBAL TEMPORARY TABLE my_temp_table (
    column1 NUMBER,
    column2 VARCHAR2(100)
) ON COMMIT PRESERVE ROWS;  -- Giữ lại khi commit, xóa khi end session
```

### ✅ Đặc điểm cần nhớ:
- [ ] Mỗi session thấy dữ liệu riêng của mình
- [ ] Không xung đột giữa các session
- [ ] Có thể tạo index và trigger
- [ ] Tự động dọn dẹp khi kết thúc session/transaction
- [ ] Sử dụng temporary tablespace
- [ ] Export chỉ lấy cấu trúc, không lấy dữ liệu

---

## 10. TRANSACTION VÀ SESSION

### A. Sử dụng PRAGMA AUTONOMOUS_TRANSACTION

#### 📌 Khi nào dùng?
- [ ] Cần commit/rollback độc lập trong procedure/function
- [ ] Ghi log mà không ảnh hưởng transaction chính
- [ ] Audit trail

```sql
PROCEDURE insert_log (
    p_message VARCHAR2
) IS
    PRAGMA AUTONOMOUS_TRANSACTION;  -- Quan trọng!
BEGIN
    INSERT INTO app_logs (message, log_date)
    VALUES (p_message, SYSDATE);
    
    COMMIT;  -- Chỉ commit log, không ảnh hưởng transaction chính
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
```

### B. Schema Trigger (Set Current Schema)

```sql
-- Tự động chuyển schema sau khi login
CREATE OR REPLACE TRIGGER db_anypay_app_logon
AFTER LOGON ON DATABASE
WHEN (USER = 'BCCS_ANYPAY')
BEGIN
    EXECUTE IMMEDIATE 'ALTER SESSION SET CURRENT_SCHEMA = ANYPAY_OWNER';
END;
```

**Use case:** User login khác với owner của bảng, tránh phải prefix schema mọi lúc

### C. Tracking Row Changes (ORA_ROWSCN)

```sql
-- Tạo bảng với ROWDEPENDENCIES
CREATE TABLE tbl_temp (
    id NUMBER(10),
    user_name VARCHAR2(20),
    passwd VARCHAR2(100)
) ROWDEPENDENCIES;  -- Cho phép track thời gian thay đổi từng row

-- Query thời gian thay đổi
SELECT 
    id,
    user_name,
    SCN_TO_TIMESTAMP(ORA_ROWSCN) AS last_modified
FROM tbl_temp;
```

### D. Time Travel Queries (Flashback)

```sql
-- Xem dữ liệu đã xóa trong vòng 1 ngày
SELECT * FROM staff
AS OF TIMESTAMP SYSDATE - 1
WHERE staff_id = 172143;

-- Khôi phục dữ liệu đã xóa
INSERT INTO staff
SELECT * FROM staff
AS OF TIMESTAMP SYSDATE - 1
WHERE staff_id = 172143;

COMMIT;
```

### ✅ Checklist Time Travel:
- [ ] Chỉ hoạt động trong khoảng UNDO_RETENTION
- [ ] Không thể undo DDL commands
- [ ] Cần quyền FLASHBACK
- [ ] Backup vẫn là phương án chính

---

## 11. BẢO MẬT VÀ BEST PRACTICES

### A. Tìm và Kill Session Lock

```sql
-- 1. Tìm session đang lock object
SELECT 
    c.owner,
    c.object_name,
    c.object_type,
    b.sid,
    b.serial#,
    b.status,
    b.osuser,
    b.machine,
    'ALTER SYSTEM KILL SESSION ''' || b.sid || ',' || b.serial# || ''' IMMEDIATE;' AS kill_statement
FROM v$locked_object a
JOIN v$session b ON b.sid = a.session_id
JOIN dba_objects c ON a.object_id = c.object_id;

-- 2. Kill session
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

### B. Mã hóa Stored Procedures

```sql
-- Wrap procedure để bảo mật code
BEGIN
    SYS.DBMS_DDL.create_wrapped('
        CREATE OR REPLACE PROCEDURE my_secure_proc IS
        BEGIN
            -- Your secret code here
            NULL;
        END;
    ');
END;
```

### C. Sử dụng LIKE với Escape Character

```sql
-- Tìm kiếm ký tự đặc biệt % hoặc _
SELECT * FROM test1 
WHERE col1 LIKE '\%\_' ESCAPE '\';

-- Có thể dùng ký tự escape khác (không phải % hoặc _)
SELECT * FROM test1 
WHERE col1 LIKE '!%!_' ESCAPE '!';
```

### D. Auto Create Partition

```sql
-- Procedure tự động tạo partition cho các bảng theo ngày
PROCEDURE proc_create_partition IS
    CURSOR c_partition IS
        SELECT 
            object_name,
            MAX(SUBSTR(subobject_name, LENGTH(subobject_name) - 5)) AS sub_partition
        FROM user_objects
        WHERE object_type = 'TABLE PARTITION'
            AND object_name NOT LIKE 'BIN$%'
        GROUP BY object_name;
        
    v_date DATE;
BEGIN
    FOR v_partition IN c_partition LOOP
        v_date := TO_DATE(v_partition.sub_partition, 'yyMMdd');
        
        WHILE v_date <= ADD_MONTHS(TO_DATE(v_partition.sub_partition, 'yyMMdd'), 1) LOOP
            BEGIN
                v_date := v_date + 1;
                
                EXECUTE IMMEDIATE 
                    'ALTER TABLE ' || v_partition.object_name || 
                    ' ADD PARTITION DATA20' || TO_CHAR(v_date, 'yyMMdd') ||
                    ' VALUES LESS THAN (TO_DATE(''20' || TO_CHAR(v_date, 'yyMMdd') || ''',''yyyyMMdd''))';
                    
            EXCEPTION
                WHEN OTHERS THEN
                    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
            END;
        END LOOP;
    END LOOP;
END;
```

### ✅ Partition Best Practices:
- [ ] Đặt tên partition theo quy ước có chứa thông tin ngày tháng
- [ ] Tiền tố phải giống nhau
- [ ] Tạo scheduler tự động chạy định kỳ
- [ ] Monitor partition size
- [ ] Archive/drop partition cũ

---

## 📊 SYSTEM TABLES THƯỜNG DÙNG

### ✅ Checklist giám sát hệ thống:

```sql
-- 1. Xem các câu SQL đang chạy
SELECT * FROM v$sql WHERE sql_text LIKE '%your_table%';

-- 2. Xem thông tin session
SELECT * FROM v$session WHERE username = 'YOUR_USER';

-- 3. Xem các object đang bị lock
SELECT * FROM v$locked_object;

-- 4. Xem system parameters
SELECT * FROM v$spparameter WHERE name LIKE '%memory%';

-- 5. Xem session đang hoạt động
SELECT 
    sid,
    serial#,
    username,
    status,
    osuser,
    machine,
    program,
    sql_id
FROM v$session
WHERE type = 'USER'
    AND status = 'ACTIVE';
```

---

## 🔍 TOÁN TỬ ALL, ANY, SOME

### ALL - Phải thỏa mãn TẤT CẢ

```sql
-- Tìm sinh viên lớp A có điểm lớn hơn TẤT CẢ sinh viên lớp B
SELECT * FROM sinh_vien
WHERE lop = 'A'
    AND diem > ALL (SELECT diem FROM sinh_vien WHERE lop = 'B');
```

### ANY/SOME - Chỉ cần thỏa mãn MỘT

```sql
-- Tìm sinh viên lớp A có điểm lớn hơn BẤT KỲ sinh viên nào lớp B
SELECT * FROM sinh_vien
WHERE lop = 'A'
    AND diem > ANY (SELECT diem FROM sinh_vien WHERE lop = 'B');

-- SOME tương đương ANY
SELECT * FROM sinh_vien
WHERE lop = 'A'
    AND diem > SOME (SELECT diem FROM sinh_vien WHERE lop = 'B');
```

---

## 🎯 EXPORT/IMPORT DATABASE

### Export Checklist:
```bash
# 1. Mở command prompt, gõ: exp
# 2. Nhập: username/password@tnsname
# 3. Buffer size (Enter = mặc định 4096)
# 4. Đường dẫn file .dmp
# 5. Export toàn bộ hay một số bảng?
# 6. Export cả dữ liệu hay chỉ cấu trúc?
# 7. Compress? (Y/N)
# 8. Nhập tên bảng cần export
# 9. Enter để bắt đầu
```

### Import Checklist:
```bash
# 1. Mở command prompt, gõ: imp
# 2. Nhập: username/password@tnsname
# 3. Đường dẫn file .dmp
# 4. Buffer size (Enter = mặc định)
# 5. Liệt kê nội dung? (Y/N)
# 6. Bỏ qua lỗi object đã tồn tại? (Y/N)
# 7. Import quyền? (Y/N)
# 8. Import dữ liệu? (Y/N)
# 9. Import toàn bộ hay theo schema?
# 10. Nhập schema name
# 11. Import một số bảng hay tất cả?
```

---

## ✅ CHECKLIST TỔNG QUAN CHO MỖI LỆNH SQL

### Trước khi viết code:
- [ ] Xác định xem bảng có partition/index không?
- [ ] Cột điều kiện có được index không?
- [ ] Ước lượng số lượng dữ liệu (nhỏ/vừa/lớn)?
- [ ] Query này chạy thường xuyên không?

### Khi viết code:
- [ ] Tránh dùng hàm trên cột có index
- [ ] Tránh SELECT * (chỉ lấy cột cần thiết)
- [ ] Cân nhắc IN vs EXISTS
- [ ] Dùng MERGE thay vì IF-THEN-INSERT/UPDATE
- [ ] Comment rõ ràng logic phức tạp

### Sau khi viết code:
- [ ] Chạy EXPLAIN PLAN
- [ ] Test với dữ liệu lớn
- [ ] Check số lượng rows affected
- [ ] Log execution time
- [ ] Code review

### Trước khi deploy:
- [ ] Unit test đầy đủ
- [ ] Test trên môi trường giống production
- [ ] Chuẩn bị rollback plan
- [ ] Document thay đổi
- [ ] Thông báo stakeholders

---

## 🚀 PERFORMANCE TUNING CHECKLIST

### Phát hiện bottleneck:
- [ ] Identify slow queries từ v$sql
- [ ] Analyze execution plans
- [ ] Check table/index statistics
- [ ] Monitor wait events
- [ ] Review AWR reports

### Tối ưu:
- [ ] Tạo/rebuild index phù hợp
- [ ] Partition bảng lớn
- [ ] Update statistics
- [ ] Rewrite query
- [ ] Add hints nếu cần
- [ ] Consider materialized views

### Monitor:
- [ ] Set up alerts
- [ ] Log slow queries
- [ ] Track trends
- [ ] Regular health checks

---

## 📝 GHI CHÚ QUAN TRỌNG

### ⚠️ Các lỗi thường gặp:

1. **ROWNUM trước ORDER BY**
   - ❌ SAI: `SELECT * FROM t WHERE ROWNUM <= 10 ORDER BY col`
   - ✅ ĐÚNG: `SELECT * FROM (SELECT * FROM t ORDER BY col) WHERE ROWNUM <= 10`

2. **Dùng hàm trên cột index**
   - ❌ SAI: `WHERE UPPER(name) = 'JOHN'`
   - ✅ ĐÚNG: `WHERE name = 'JOHN'` hoặc tạo function-based index

3. **LIKE với % ở đầu**
   - ❌ SAI: `WHERE name LIKE '%John'`
   - ✅ ĐÚNG: `WHERE name LIKE 'John%'`

4. **Quên commit trong autonomous transaction**
   - ❌ SAI: Log procedure không commit
   - ✅ ĐÚNG: Luôn có COMMIT trong autonomous transaction

---

## 🎓 TÀI LIỆU THAM KHẢO

- Oracle Documentation
- Oracle Performance Tuning Guide
- Tom Kyte's AskTom
- Oracle-base.com
- Oracle Optimizer Blog

---

---

## 12. EXCEPTION HANDLING VÀ ERROR MANAGEMENT

### ✅ Best Practices cho Exception Handling

```sql
CREATE OR REPLACE PROCEDURE safe_update_user (
    p_user_id NUMBER,
    p_status VARCHAR2
) IS
    e_invalid_status EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_invalid_status, -20001);
    
    v_error_code NUMBER;
    v_error_message VARCHAR2(4000);
BEGIN
    -- Validate input
    IF p_status NOT IN ('ACTIVE', 'INACTIVE', 'SUSPENDED') THEN
        RAISE_APPLICATION_ERROR(-20001, 'Invalid status: ' || p_status);
    END IF;
    
    -- Main logic
    UPDATE users 
    SET status = p_status, 
        updated_date = SYSDATE
    WHERE user_id = p_user_id;
    
    IF SQL%ROWCOUNT = 0 THEN
        RAISE NO_DATA_FOUND;
    END IF;
    
    COMMIT;
    
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        ROLLBACK;
        log_error('USER_NOT_FOUND', 'User ID: ' || p_user_id);
        RAISE_APPLICATION_ERROR(-20002, 'User not found: ' || p_user_id);
        
    WHEN e_invalid_status THEN
        ROLLBACK;
        log_error('INVALID_STATUS', p_status);
        RAISE;
        
    WHEN OTHERS THEN
        ROLLBACK;
        v_error_code := SQLCODE;
        v_error_message := SQLERRM;
        log_error('UNEXPECTED_ERROR', v_error_message);
        RAISE_APPLICATION_ERROR(-20999, 'System error: ' || v_error_message);
END;
```

### ✅ Checklist Exception Handling:
- [ ] **Luôn log errors** trước khi raise
- [ ] **Rollback** khi gặp lỗi
- [ ] **Custom error codes** (-20000 đến -20999)
- [ ] **Meaningful error messages** cho user
- [ ] **Không bỏ qua WHEN OTHERS** nếu không có lý do rõ ràng
- [ ] **Cleanup resources** trong exception block
- [ ] **Re-raise exceptions** nếu không xử lý được

---

## 13. BULK OPERATIONS VÀ PERFORMANCE

### A. BULK COLLECT - Đọc hàng loạt

```sql
DECLARE
    TYPE t_user_list IS TABLE OF users%ROWTYPE;
    v_users t_user_list;
    
    CURSOR c_active_users IS
        SELECT * FROM users WHERE status = 'ACTIVE';
BEGIN
    -- Fetch tất cả rows một lần
    OPEN c_active_users;
    FETCH c_active_users BULK COLLECT INTO v_users;
    CLOSE c_active_users;
    
    -- Process
    FOR i IN 1..v_users.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_users(i).user_name);
    END LOOP;
END;
```

### B. FORALL - Ghi hàng loạt

```sql
DECLARE
    TYPE t_user_ids IS TABLE OF NUMBER;
    v_user_ids t_user_ids;
BEGIN
    -- Collect IDs
    SELECT user_id 
    BULK COLLECT INTO v_user_ids
    FROM users 
    WHERE status = 'INACTIVE'
        AND last_login < SYSDATE - 365;
    
    -- Bulk delete (nhanh hơn nhiều so với loop)
    FORALL i IN 1..v_user_ids.COUNT
        DELETE FROM user_sessions 
        WHERE user_id = v_user_ids(i);
    
    DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' records');
    COMMIT;
END;
```

### C. BULK COLLECT với LIMIT

```sql
DECLARE
    TYPE t_user_list IS TABLE OF users%ROWTYPE;
    v_users t_user_list;
    
    CURSOR c_users IS SELECT * FROM users;
    v_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
    OPEN c_users;
    LOOP
        -- Fetch theo batch để tránh out of memory
        FETCH c_users BULK COLLECT INTO v_users LIMIT v_batch_size;
        EXIT WHEN v_users.COUNT = 0;
        
        -- Process batch
        FORALL i IN 1..v_users.COUNT
            UPDATE user_summary
            SET last_updated = SYSDATE
            WHERE user_id = v_users(i).user_id;
        
        COMMIT; -- Commit từng batch
    END LOOP;
    CLOSE c_users;
END;
```

### ✅ Bulk Operations Checklist:
- [ ] Dùng BULK COLLECT thay vì cursor loop thông thường
- [ ] Dùng FORALL thay vì loop với DML
- [ ] Set LIMIT phù hợp (1000-5000) để tránh memory issues
- [ ] Commit theo batch với dữ liệu lớn
- [ ] Sử dụng SAVE EXCEPTIONS với FORALL
- [ ] Monitor performance gain (thường 10-50x nhanh hơn)

---

## 14. COLLECTIONS VÀ ADVANCED DATATYPES

### A. Associative Arrays (Index-by Tables)

```sql
DECLARE
    TYPE t_salary_map IS TABLE OF NUMBER INDEX BY VARCHAR2(50);
    v_salaries t_salary_map;
    v_emp_name VARCHAR2(50);
BEGIN
    -- Populate
    v_salaries('John') := 5000;
    v_salaries('Mary') := 6000;
    v_salaries('Peter') := 5500;
    
    -- Access
    IF v_salaries.EXISTS('John') THEN
        DBMS_OUTPUT.PUT_LINE('John salary: ' || v_salaries('John'));
    END IF;
    
    -- Iterate
    v_emp_name := v_salaries.FIRST;
    WHILE v_emp_name IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(v_emp_name || ': ' || v_salaries(v_emp_name));
        v_emp_name := v_salaries.NEXT(v_emp_name);
    END LOOP;
END;
```

### B. Nested Tables

```sql
CREATE TYPE t_phone_numbers IS TABLE OF VARCHAR2(20);

CREATE TABLE customers (
    customer_id NUMBER PRIMARY KEY,
    customer_name VARCHAR2(100),
    phone_numbers t_phone_numbers
) NESTED TABLE phone_numbers STORE AS customer_phones;

-- Insert
INSERT INTO customers VALUES (
    1, 
    'John Doe', 
    t_phone_numbers('123-456-7890', '098-765-4321')
);

-- Query
SELECT c.customer_name, p.COLUMN_VALUE AS phone
FROM customers c, TABLE(c.phone_numbers) p;
```

### C. VARRAYs (Variable-size Arrays)

```sql
CREATE TYPE t_week_days IS VARRAY(7) OF VARCHAR2(10);

DECLARE
    v_days t_week_days := t_week_days('Mon', 'Tue', 'Wed', 'Thu', 'Fri');
BEGIN
    FOR i IN 1..v_days.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Day ' || i || ': ' || v_days(i));
    END LOOP;
END;
```

---

## 15. DYNAMIC SQL VÀ EXECUTE IMMEDIATE

### ✅ Best Practices cho Dynamic SQL

```sql
CREATE OR REPLACE PROCEDURE dynamic_query_example (
    p_table_name VARCHAR2,
    p_column_name VARCHAR2,
    p_where_clause VARCHAR2 DEFAULT NULL
) IS
    v_sql VARCHAR2(4000);
    v_count NUMBER;
    v_result SYS_REFCURSOR;
    v_value VARCHAR2(200);
BEGIN
    -- ❌ DANGER: SQL Injection vulnerable
    -- v_sql := 'SELECT COUNT(*) FROM ' || p_table_name;
    
    -- ✅ BETTER: Validate inputs
    IF p_table_name NOT IN ('USERS', 'ORDERS', 'PRODUCTS') THEN
        RAISE_APPLICATION_ERROR(-20001, 'Invalid table name');
    END IF;
    
    -- Build dynamic SQL safely
    v_sql := 'SELECT COUNT(*) FROM ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_table_name);
    
    IF p_where_clause IS NOT NULL THEN
        -- Use bind variables for user input
        v_sql := v_sql || ' WHERE ' || p_where_clause;
    END IF;
    
    EXECUTE IMMEDIATE v_sql INTO v_count;
    
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
    
    -- Dynamic cursor
    v_sql := 'SELECT ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_column_name) || 
             ' FROM ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_table_name);
    
    OPEN v_result FOR v_sql;
    LOOP
        FETCH v_result INTO v_value;
        EXIT WHEN v_result%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_value);
    END LOOP;
    CLOSE v_result;
END;
```

### ✅ Dynamic SQL Checklist:
- [ ] **LUÔN validate inputs** từ user
- [ ] **Dùng DBMS_ASSERT** để sanitize identifiers
- [ ] **Sử dụng bind variables** cho data values
- [ ] **Không concatenate** user input trực tiếp vào SQL
- [ ] **Log dynamic SQL** cho debugging
- [ ] **Handle exceptions** properly
- [ ] **Consider static SQL** nếu có thể

---

## 16. ANALYTIC FUNCTIONS (WINDOW FUNCTIONS)

### A. Ranking Functions

```sql
-- ROW_NUMBER: Số thứ tự unique
SELECT 
    employee_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    -- RANK: Cùng rank nếu cùng giá trị, skip ranks
    RANK() OVER (ORDER BY salary DESC) AS rank,
    -- DENSE_RANK: Cùng rank nếu cùng giá trị, không skip
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- Partition by department
SELECT 
    department_id,
    employee_name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
FROM employees;
```

### B. Aggregate Functions với Window

```sql
SELECT 
    employee_name,
    salary,
    department_id,
    -- Running total
    SUM(salary) OVER (ORDER BY employee_id) AS running_total,
    -- Department average
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg,
    -- Moving average (last 3 rows)
    AVG(salary) OVER (ORDER BY employee_id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
    -- Difference from department average
    salary - AVG(salary) OVER (PARTITION BY department_id) AS diff_from_avg
FROM employees;
```

### C. LAG và LEAD

```sql
SELECT 
    order_date,
    order_amount,
    -- Previous order
    LAG(order_amount, 1) OVER (ORDER BY order_date) AS prev_order,
    -- Next order
    LEAD(order_amount, 1) OVER (ORDER BY order_date) AS next_order,
    -- Change from previous
    order_amount - LAG(order_amount, 1, 0) OVER (ORDER BY order_date) AS change,
    -- % change
    ROUND(
        (order_amount - LAG(order_amount, 1) OVER (ORDER BY order_date)) / 
        LAG(order_amount, 1) OVER (ORDER BY order_date) * 100, 2
    ) AS pct_change
FROM orders;
```

### D. FIRST_VALUE và LAST_VALUE

```sql
SELECT 
    employee_name,
    salary,
    department_id,
    -- Highest salary in department
    FIRST_VALUE(salary) OVER (
        PARTITION BY department_id 
        ORDER BY salary DESC
    ) AS highest_salary,
    -- Lowest salary in department
    LAST_VALUE(salary) OVER (
        PARTITION BY department_id 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_salary
FROM employees;
```

---

## 17. PIVOT VÀ UNPIVOT

### A. PIVOT - Chuyển rows thành columns

```sql
-- Dữ liệu gốc: year, quarter, amount
-- Kết quả: year, Q1, Q2, Q3, Q4

SELECT * FROM (
    SELECT year, quarter, amount
    FROM sales
)
PIVOT (
    SUM(amount)
    FOR quarter IN ('Q1' AS Q1, 'Q2' AS Q2, 'Q3' AS Q3, 'Q4' AS Q4)
)
ORDER BY year;
```

### B. UNPIVOT - Chuyển columns thành rows

```sql
-- Dữ liệu gốc: year, Q1, Q2, Q3, Q4
-- Kết quả: year, quarter, amount

SELECT * FROM quarterly_sales
UNPIVOT (
    amount FOR quarter IN (Q1, Q2, Q3, Q4)
)
ORDER BY year, quarter;
```

---

## 18. REGULAR EXPRESSIONS

### ✅ Các hàm REGEXP trong Oracle

```sql
-- REGEXP_LIKE: Tìm pattern
SELECT * FROM users
WHERE REGEXP_LIKE(email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');

-- REGEXP_SUBSTR: Extract substring
SELECT 
    phone,
    REGEXP_SUBSTR(phone, '\d{3}') AS area_code,
    REGEXP_SUBSTR(phone, '\d{3}', 1, 2) AS exchange
FROM contacts;

-- REGEXP_REPLACE: Replace pattern
UPDATE users
SET phone = REGEXP_REPLACE(phone, '[^0-9]', '')  -- Remove non-digits
WHERE phone IS NOT NULL;

-- REGEXP_INSTR: Find position
SELECT 
    email,
    REGEXP_INSTR(email, '@') AS at_position
FROM users;

-- REGEXP_COUNT: Count occurrences
SELECT 
    description,
    REGEXP_COUNT(description, '\d') AS digit_count
FROM products;
```

### Common Patterns:

```sql
-- Email validation
WHERE REGEXP_LIKE(email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')

-- Phone numbers (Vietnam)
WHERE REGEXP_LIKE(phone, '^(0|\+84)[0-9]{9,10}$')

-- Only letters
WHERE REGEXP_LIKE(name, '^[A-Za-z ]+$')

-- Only numbers
WHERE REGEXP_LIKE(code, '^[0-9]+$')

-- Extract domain from email
REGEXP_SUBSTR(email, '@(.+)', 1, 1, NULL, 1) AS domain

-- Extract first word
REGEXP_SUBSTR(full_name, '\w+') AS first_name
```

---

## 19. MATERIALIZED VIEWS

### A. Tạo Materialized View

```sql
-- Basic materialized view
CREATE MATERIALIZED VIEW mv_sales_summary
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS
SELECT 
    product_id,
    TO_CHAR(sale_date, 'YYYY-MM') AS month,
    SUM(amount) AS total_amount,
    COUNT(*) AS transaction_count
FROM sales
GROUP BY product_id, TO_CHAR(sale_date, 'YYYY-MM');

-- With refresh schedule
CREATE MATERIALIZED VIEW mv_daily_summary
BUILD IMMEDIATE
REFRESH COMPLETE
START WITH SYSDATE
NEXT SYSDATE + 1  -- Refresh daily
AS
SELECT 
    department_id,
    COUNT(*) AS emp_count,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

-- Fast refresh (require MV log)
CREATE MATERIALIZED VIEW LOG ON sales
WITH ROWID, SEQUENCE (product_id, sale_date, amount)
INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW mv_sales_fast
BUILD IMMEDIATE
REFRESH FAST ON COMMIT
AS
SELECT 
    product_id,
    SUM(amount) AS total_amount
FROM sales
GROUP BY product_id;
```

### B. Quản lý Materialized View

```sql
-- Manual refresh
EXEC DBMS_MVIEW.REFRESH('MV_SALES_SUMMARY', 'C');  -- Complete
EXEC DBMS_MVIEW.REFRESH('MV_SALES_SUMMARY', 'F');  -- Fast

-- Refresh multiple MVs
BEGIN
    DBMS_MVIEW.REFRESH_ALL_MVIEWS(
        number_of_failures => 0,
        method => 'C',
        rollback_seg => NULL,
        refresh_after_errors => FALSE
    );
END;

-- Check last refresh
SELECT 
    mview_name,
    last_refresh_date,
    staleness,
    compile_state
FROM user_mviews;

-- Drop MV
DROP MATERIALIZED VIEW mv_sales_summary;
```

### ✅ MV Best Practices:
- [ ] Dùng cho queries phức tạp, chạy thường xuyên
- [ ] COMPLETE refresh cho aggregations
- [ ] FAST refresh cho incremental updates (cần MV log)
- [ ] Monitor staleness
- [ ] Consider storage space
- [ ] Schedule refresh off-peak hours

---

## 20. HINTS VÀ QUERY OPTIMIZATION

### A. Common Hints

```sql
-- Force index usage
SELECT /*+ INDEX(e emp_dept_idx) */ 
    employee_name, salary
FROM employees e
WHERE department_id = 10;

-- Force full table scan
SELECT /*+ FULL(e) */ 
    * 
FROM employees e;

-- Parallel execution
SELECT /*+ PARALLEL(employees, 4) */ 
    COUNT(*)
FROM employees;

-- First rows optimization
SELECT /*+ FIRST_ROWS(10) */ 
    employee_name, salary
FROM employees
ORDER BY salary DESC;

-- All rows optimization
SELECT /*+ ALL_ROWS */ 
    *
FROM large_table;

-- Use hash join
SELECT /*+ USE_HASH(e d) */ 
    e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Use nested loop
SELECT /*+ USE_NL(e d) */ 
    e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Append hint for direct path insert
INSERT /*+ APPEND */ INTO archive_table
SELECT * FROM main_table WHERE year < 2020;
```

### B. Analyze Execution Plans

```sql
-- Explain plan
EXPLAIN PLAN FOR
SELECT * FROM employees WHERE department_id = 10;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Get actual execution stats
SELECT /*+ GATHER_PLAN_STATISTICS */ 
    * 
FROM employees 
WHERE salary > 5000;

SELECT * FROM TABLE(
    DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST')
);

-- Autotrace
SET AUTOTRACE ON EXPLAIN
SELECT * FROM employees WHERE department_id = 10;
```

---

## 21. PARTITIONING STRATEGIES

### A. Range Partitioning

```sql
CREATE TABLE sales (
    sale_id NUMBER,
    sale_date DATE,
    amount NUMBER
)
PARTITION BY RANGE (sale_date) (
    PARTITION p_2023_q1 VALUES LESS THAN (TO_DATE('2023-04-01', 'YYYY-MM-DD')),
    PARTITION p_2023_q2 VALUES LESS THAN (TO_DATE('2023-07-01', 'YYYY-MM-DD')),
    PARTITION p_2023_q3 VALUES LESS THAN (TO_DATE('2023-10-01', 'YYYY-MM-DD')),
    PARTITION p_2023_q4 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD'))
);
```

### B. List Partitioning

```sql
CREATE TABLE employees (
    employee_id NUMBER,
    employee_name VARCHAR2(100),
    department_id NUMBER
)
PARTITION BY LIST (department_id) (
    PARTITION p_sales VALUES (10, 11, 12),
    PARTITION p_it VALUES (20, 21, 22),
    PARTITION p_hr VALUES (30, 31, 32),
    PARTITION p_others VALUES (DEFAULT)
);
```

### C. Hash Partitioning

```sql
CREATE TABLE customers (
    customer_id NUMBER,
    customer_name VARCHAR2(100)
)
PARTITION BY HASH (customer_id)
PARTITIONS 8;
```

### D. Composite Partitioning

```sql
CREATE TABLE orders (
    order_id NUMBER,
    order_date DATE,
    customer_id NUMBER,
    amount NUMBER
)
PARTITION BY RANGE (order_date)
SUBPARTITION BY HASH (customer_id)
SUBPARTITIONS 4 (
    PARTITION p_2023 VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD')),
    PARTITION p_2024 VALUES LESS THAN (TO_DATE('2025-01-01', 'YYYY-MM-DD'))
);
```

### E. Partition Maintenance

```sql
-- Add partition
ALTER TABLE sales ADD PARTITION p_2024_q1 
VALUES LESS THAN (TO_DATE('2024-04-01', 'YYYY-MM-DD'));

-- Drop partition
ALTER TABLE sales DROP PARTITION p_2023_q1;

-- Truncate partition
ALTER TABLE sales TRUNCATE PARTITION p_2023_q1;

-- Split partition
ALTER TABLE sales SPLIT PARTITION p_2024 AT (TO_DATE('2024-07-01', 'YYYY-MM-DD'))
INTO (PARTITION p_2024_h1, PARTITION p_2024_h2);

-- Merge partitions
ALTER TABLE sales MERGE PARTITIONS p_2024_h1, p_2024_h2 INTO PARTITION p_2024;

-- Exchange partition
ALTER TABLE sales EXCHANGE PARTITION p_2023_q1 
WITH TABLE sales_2023_q1_archive;
```

---

## 22. SEQUENCES VÀ IDENTITY COLUMNS

### A. Sequences

```sql
-- Create sequence
CREATE SEQUENCE seq_user_id
START WITH 1000
INCREMENT BY 1
MAXVALUE 999999999
NOCACHE  -- Hoặc CACHE 20 cho performance
NOCYCLE;

-- Use in INSERT
INSERT INTO users (user_id, user_name)
VALUES (seq_user_id.NEXTVAL, 'John Doe');

-- Get current value (without incrementing)
SELECT seq_user_id.CURRVAL FROM dual;

-- Reset sequence
ALTER SEQUENCE seq_user_id RESTART START WITH 1000;

-- Drop sequence
DROP SEQUENCE seq_user_id;
```

### B. Identity Columns (12c+)

```sql
-- Generated always
CREATE TABLE employees (
    employee_id NUMBER GENERATED ALWAYS AS IDENTITY,
    employee_name VARCHAR2(100),
    hire_date DATE
);

-- Generated by default
CREATE TABLE orders (
    order_id NUMBER GENERATED BY DEFAULT AS IDENTITY,
    order_date DATE,
    amount NUMBER
);

-- With options
CREATE TABLE products (
    product_id NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY (
        START WITH 1000
        INCREMENT BY 1
        MAXVALUE 999999
        CACHE 20
    ),
    product_name VARCHAR2(200)
);
```

---

## 23. INDEXES BEST PRACTICES

### A. Index Types

```sql
-- B-tree index (default)
CREATE INDEX idx_users_email ON users(email);

-- Unique index
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Composite index
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);

-- Function-based index
CREATE INDEX idx_users_upper_email ON users(UPPER(email));

-- Bitmap index (for low cardinality)
CREATE BITMAP INDEX idx_users_status ON users(status);

-- Partial index (11g+)
CREATE INDEX idx_active_users ON users(user_id)
WHERE status = 'ACTIVE';
```

### B. Index Monitoring

```sql
-- Monitor index usage
ALTER INDEX idx_users_email MONITORING USAGE;

-- Check usage
SELECT * FROM v$object_usage WHERE index_name = 'IDX_USERS_EMAIL';

-- Stop monitoring
ALTER INDEX idx_users_email NOMONITORING USAGE;

-- Find unused indexes
SELECT 
    index_name,
    table_name,
    monitoring,
    used,
    start_monitoring,
    end_monitoring
FROM v$object_usage
WHERE used = 'NO';
```

### C. Index Maintenance

```sql
-- Rebuild index
ALTER INDEX idx_users_email REBUILD ONLINE;

-- Rebuild tablespace
ALTER INDEX idx_users_email REBUILD TABLESPACE users_idx_ts;

-- Coalesce (instead of rebuild)
ALTER INDEX idx_users_email COALESCE;

-- Make invisible (test impact)
ALTER INDEX idx_users_email INVISIBLE;

-- Make visible again
ALTER INDEX idx_users_email VISIBLE;
```

### ✅ Index Checklist:
- [ ] Index foreign keys
- [ ] Index columns in WHERE clauses
- [ ] Index columns in JOIN conditions
- [ ] Consider composite indexes for multiple columns
- [ ] Don't over-index (slows down DML)
- [ ] Monitor and drop unused indexes
- [ ] Rebuild fragmented indexes
- [ ] Use function-based indexes when needed

---

## 24. DBMS_SCHEDULER - JOB SCHEDULING

### A. Tạo Jobs

```sql
-- Simple repeating job
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'DAILY_CLEANUP_JOB',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN cleanup_old_data; END;',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
        enabled         => TRUE,
        comments        => 'Daily cleanup at 2 AM'
    );
END;

-- Job calling stored procedure
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'MONTHLY_REPORT_JOB',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'generate_monthly_report',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=MONTHLY; BYMONTHDAY=1; BYHOUR=6',
        enabled         => TRUE
    );
END;

-- Job with program and schedule
BEGIN
    -- Create program
    DBMS_SCHEDULER.CREATE_PROGRAM (
        program_name   => 'BACKUP_PROGRAM',
        program_type   => 'STORED_PROCEDURE',
        program_action => 'backup_database',
        enabled        => TRUE
    );
    
    -- Create schedule
    DBMS_SCHEDULER.CREATE_SCHEDULE (
        schedule_name   => 'NIGHTLY_SCHEDULE',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=23; BYMINUTE=0',
        comments        => 'Every night at 11 PM'
    );
    
    -- Create job using program and schedule
    DBMS_SCHEDULER.CREATE_JOB (
        job_name      => 'NIGHTLY_BACKUP_JOB',
        program_name  => 'BACKUP_PROGRAM',
        schedule_name => 'NIGHTLY_SCHEDULE',
        enabled       => TRUE
    );
END;
```

### B. Quản lý Jobs

```sql
-- Run immediately
BEGIN
    DBMS_SCHEDULER.RUN_JOB('DAILY_CLEANUP_JOB');
END;

-- Stop job
BEGIN
    DBMS_SCHEDULER.STOP_JOB('DAILY_CLEANUP_JOB');
END;

-- Enable/Disable
BEGIN
    DBMS_SCHEDULER.DISABLE('DAILY_CLEANUP_JOB');
    DBMS_SCHEDULER.ENABLE('DAILY_CLEANUP_JOB');
END;

-- Drop job
BEGIN
    DBMS_SCHEDULER.DROP_JOB('DAILY_CLEANUP_JOB', FORCE => TRUE);
END;

-- Modify job
BEGIN
    DBMS_SCHEDULER.SET_ATTRIBUTE(
        name      => 'DAILY_CLEANUP_JOB',
        attribute => 'repeat_interval',
        value     => 'FREQ=DAILY; BYHOUR=3; BYMINUTE=0'
    );
END;
```

### C. Monitor Jobs

```sql
-- View all jobs
SELECT 
    job_name,
    enabled,
    state,
    next_run_date,
    last_start_date,
    last_run_duration,
    failure_count
FROM dba_scheduler_jobs
WHERE owner = USER;

-- View job run history
SELECT 
    log_date,
    job_name,
    status,
    error#,
    additional_info
FROM dba_scheduler_job_run_details
WHERE job_name = 'DAILY_CLEANUP_JOB'
ORDER BY log_date DESC;
```

---

## 25. DBMS_OUTPUT VÀ DEBUGGING

### A. DBMS_OUTPUT Usage

```sql
-- Enable output (SQL*Plus/SQL Developer)
SET SERVEROUTPUT ON SIZE 1000000

-- Basic usage
BEGIN
    DBMS_OUTPUT.PUT_LINE('Debug message');
    DBMS_OUTPUT.PUT_LINE('Variable value: ' || v_variable);
END;

-- Format output
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count FROM users;
    DBMS_OUTPUT.PUT_LINE('Total users: ' || TO_CHAR(v_count));
    DBMS_OUTPUT.PUT_LINE(RPAD('=', 50, '='));
END;
```

### B. Better Debugging với DBMS_APPLICATION_INFO

```sql
-- Set module and action
BEGIN
    DBMS_APPLICATION_INFO.SET_MODULE(
        module_name => 'USER_MAINTENANCE',
        action_name => 'CLEANUP_INACTIVE'
    );
    
    -- Your code here
    cleanup_inactive_users;
    
    -- Clear
    DBMS_APPLICATION_INFO.SET_MODULE(NULL, NULL);
END;

-- Monitor từ session khác
SELECT 
    sid,
    serial#,
    username,
    module,
    action,
    client_info
FROM v$session
WHERE module = 'USER_MAINTENANCE';
```

### C. Logging Best Practices

```sql
-- Create log table
CREATE TABLE application_logs (
    log_id NUMBER GENERATED ALWAYS AS IDENTITY,
    log_date TIMESTAMP DEFAULT SYSTIMESTAMP,
    log_level VARCHAR2(10),
    module_name VARCHAR2(100),
    procedure_name VARCHAR2(100),
    message VARCHAR2(4000),
    error_code NUMBER,
    error_message VARCHAR2(4000),
    username VARCHAR2(100) DEFAULT USER,
    session_id NUMBER
);

-- Logging procedure
CREATE OR REPLACE PROCEDURE log_message (
    p_level VARCHAR2,
    p_module VARCHAR2,
    p_procedure VARCHAR2,
    p_message VARCHAR2,
    p_error_code NUMBER DEFAULT NULL
) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO application_logs (
        log_level, module_name, procedure_name, 
        message, error_code, error_message, session_id
    ) VALUES (
        p_level, p_module, p_procedure,
        p_message, p_error_code, SQLERRM, SYS_CONTEXT('USERENV', 'SESSIONID')
    );
    COMMIT;
END;

-- Usage
BEGIN
    log_message('INFO', 'USER_MODULE', 'update_user', 'Starting update');
    
    -- Your code
    
    log_message('INFO', 'USER_MODULE', 'update_user', 'Update completed');
EXCEPTION
    WHEN OTHERS THEN
        log_message('ERROR', 'USER_MODULE', 'update_user', 'Update failed', SQLCODE);
        RAISE;
END;
```

---

## 26. SECURITY BEST PRACTICES

### A. Tránh SQL Injection

```sql
-- ❌ VULNERABLE
CREATE OR REPLACE PROCEDURE bad_login (p_username VARCHAR2, p_password VARCHAR2) IS
    v_sql VARCHAR2(1000);
    v_count NUMBER;
BEGIN
    v_sql := 'SELECT COUNT(*) FROM users WHERE username = ''' || 
             p_username || ''' AND password = ''' || p_password || '''';
    EXECUTE IMMEDIATE v_sql INTO v_count;
END;

-- ✅ SAFE: Use bind variables
CREATE OR REPLACE PROCEDURE safe_login (p_username VARCHAR2, p_password VARCHAR2) IS
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM users
    WHERE username = p_username
        AND password = p_password;
END;

-- ✅ SAFE: With dynamic SQL
CREATE OR REPLACE PROCEDURE safe_dynamic (p_username VARCHAR2) IS
    v_sql VARCHAR2(1000);
    v_count NUMBER;
BEGIN
    v_sql := 'SELECT COUNT(*) FROM users WHERE username = :1';
    EXECUTE IMMEDIATE v_sql INTO v_count USING p_username;
END;
```

### B. Encryption

```sql
-- Encrypt sensitive data
DECLARE
    v_key RAW(128) := UTL_RAW.CAST_TO_RAW('MySecretKey12345');
    v_encrypted RAW(2000);
    v_decrypted VARCHAR2(2000);
    v_original VARCHAR2(100) := 'Sensitive Data';
BEGIN
    -- Encrypt
    v_encrypted := DBMS_CRYPTO.ENCRYPT(
        src => UTL_RAW.CAST_TO_RAW(v_original),
        typ => DBMS_CRYPTO.ENCRYPT_AES128 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
        key => v_key
    );
    
    -- Decrypt
    v_decrypted := UTL_RAW.CAST_TO_VARCHAR2(
        DBMS_CRYPTO.DECRYPT(
            src => v_encrypted,
            typ => DBMS_CRYPTO.ENCRYPT_AES128 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
            key => v_key
        )
    );
END;
```

### C. Row Level Security

```sql
-- Create policy function
CREATE OR REPLACE FUNCTION user_data_policy (
    p_schema VARCHAR2,
    p_object VARCHAR2
) RETURN VARCHAR2 IS
BEGIN
    RETURN 'user_id = SYS_CONTEXT(''USERENV'', ''SESSION_USER'')';
END;

-- Apply policy
BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema   => 'APP_SCHEMA',
        object_name     => 'SENSITIVE_DATA',
        policy_name     => 'USER_ACCESS_POLICY',
        function_schema => 'APP_SCHEMA',
        policy_function => 'user_data_policy',
        statement_types => 'SELECT, UPDATE, DELETE'
    );
END;
```

---

## 27. BACKUP VÀ RECOVERY TIPS

### A. Export/Import Best Practices

```bash
# Modern Data Pump Export
expdp username/password@database \
  directory=DATA_PUMP_DIR \
  dumpfile=backup_%U.dmp \
  logfile=export.log \
  parallel=4 \
  compression=all \
  schemas=APP_SCHEMA

# Modern Data Pump Import
impdp username/password@database \
  directory=DATA_PUMP_DIR \
  dumpfile=backup_01.dmp \
  logfile=import.log \
  parallel=4 \
  table_exists_action=replace \
  schemas=APP_SCHEMA
```

### B. Flashback Table

```sql
-- Enable row movement (required for flashback)
ALTER TABLE users ENABLE ROW MOVEMENT;

-- Flashback to timestamp
FLASHBACK TABLE users TO TIMESTAMP 
    TIMESTAMP '2024-01-01 10:00:00';

-- Flashback to SCN
FLASHBACK TABLE users TO SCN 1234567;

-- Flashback before DROP
FLASHBACK TABLE users TO BEFORE DROP;
```

---

## 📞 HỖ TRỢ

Nếu có thắc mắc về coding convention:
1. Review document này
2. Check với Tech Lead
3. Tham khảo Oracle documentation
4. Discuss trong team meeting
5. Log issue trên Jira/Confluence

### Resources hữu ích:
- Oracle Documentation: https://docs.oracle.com
- AskTom: https://asktom.oracle.com
- Oracle Base: https://oracle-base.com
- Oracle LiveSQL: https://livesql.oracle.com

---

## 📋 QUICK REFERENCE - COMMON QUERIES

```sql
-- Kill session
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;

-- Check table size
SELECT 
    segment_name,
    ROUND(bytes/1024/1024, 2) AS size_mb
FROM user_segments
WHERE segment_type = 'TABLE'
ORDER BY bytes DESC;

-- Check index size
SELECT 
    index_name,
    ROUND(bytes/1024/1024, 2) AS size_mb
FROM user_segments
WHERE segment_type = 'INDEX'
ORDER BY bytes DESC;

-- Find duplicate records
SELECT column_name, COUNT(*)
FROM table_name
GROUP BY column_name
HAVING COUNT(*) > 1;

-- Get execution time
SET TIMING ON
-- Your query here
SET TIMING OFF

-- Check tablespace usage
SELECT 
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024/1024, 2) AS size_gb,
    ROUND(SUM(maxbytes)/1024/1024/1024, 2) AS max_size_gb
FROM dba_data_files
GROUP BY tablespace_name;
```

---

**Version:** 2.0  
**Last Updated:** 2024  
**Maintainer:** Development Team  
**Contributors:** DevOps, DBA Team

---

**LƯU Ý CUỐI:** 

1. Checklist này là **guideline**, không phải quy tắc cứng nhắc
2. Trong trường hợp đặc biệt, có thể cần điều chỉnh
3. Mọi **exception phải được document** rõ ràng và review kỹ
4. **Performance testing** là bắt buộc cho mọi thay đổi quan trọng
5. **Security** luôn được ưu tiên hàng đầu
6. **Backup** trước khi thực hiện thay đổi lớn
7. **Code review** và **peer review** là bắt buộc

### Quy trình Deploy:
1. ✅ Code review (2 reviewers minimum)
2. ✅ Unit testing (coverage >= 80%)
3. ✅ Integration testing
4. ✅ Performance testing
5. ✅ Security scan
6. ✅ UAT approval
7. ✅ Backup production
8. ✅ Deploy to production
9. ✅ Smoke testing
10. ✅ Monitor for 24h

---

**HAPPY CODING! 🚀**

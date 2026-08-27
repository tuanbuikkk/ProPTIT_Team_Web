# 1. Thứ tự thực thi logic của truy vấn

Khi viết SQL, **thứ tự viết câu lệnh** không hoàn toàn giống với **thứ tự SQL thực thi logic**.

### Ví dụ

```sql
SELECT department, AVG(salary)
FROM Employees
WHERE salary > 1000
GROUP BY department
HAVING AVG(salary) > 2000
ORDER BY department;
```

Mặc dù ta viết theo thứ tự:

```text
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
```

Nhưng SQL sẽ **xử lý logic** theo thứ tự:

```text
FROM
→ JOIN / ON
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT
→ DISTINCT
→ ORDER BY
→ LIMIT
```

---

## 1.1. FROM

**Tôi lấy dữ liệu ở đâu?**

SQL xác định bảng dữ liệu cần sử dụng.

```sql
FROM Employees
```

→ Lấy dữ liệu từ bảng `Employees`.

---

## 1.2. JOIN / ON

**Tôi cần kết hợp thêm bảng nào?**

Nếu có `JOIN`, SQL kết hợp các bảng dựa trên điều kiện `ON`.

```sql
FROM Employees
JOIN Departments
ON Employees.department_id = Departments.department_id
```

→ Kết hợp dữ liệu từ `Employees` và `Departments`.

---

## 1.3. WHERE

**Tôi loại dòng nào?**

`WHERE` lọc **từng dòng dữ liệu** trước khi `GROUP BY`.

```sql
WHERE salary > 1000
```

→ Chỉ giữ lại những nhân viên có `salary > 1000`.

---

## 1.4. GROUP BY

**Tôi gom theo cái gì?**

Sau khi lọc bằng `WHERE`, SQL gom các dòng thành từng nhóm.

```sql
GROUP BY department
```

→ Gom nhân viên theo từng `department`.

Ví dụ:

```text
IT       → các nhân viên thuộc IT
HR       → các nhân viên thuộc HR
Sales    → các nhân viên thuộc Sales
```

---

## 1.5. COUNT / SUM / AVG

**Tôi tính gì trên mỗi nhóm?**

Các hàm tổng hợp được tính trên từng nhóm.

Ví dụ:

```sql
AVG(salary)
```

→ Tính mức lương trung bình của từng phòng ban.

```sql
COUNT(*)
```

→ Đếm số nhân viên trong từng phòng ban.

```sql
SUM(salary)
```

→ Tính tổng lương của từng phòng ban.

---

## 1.6. HAVING

**Tôi loại nhóm nào?**

`HAVING` lọc **các nhóm** sau khi đã `GROUP BY`.

```sql
HAVING AVG(salary) > 2000
```

→ Chỉ giữ lại những phòng ban có mức lương trung bình lớn hơn `2000`.

### Phân biệt WHERE và HAVING

Đây là điểm **rất quan trọng**:

* `WHERE` → lọc **dòng**
* `HAVING` → lọc **nhóm**

Ví dụ:

```sql
WHERE salary > 1000
```

→ Loại những **nhân viên** có lương ≤ 1000.

```sql
HAVING AVG(salary) > 2000
```

→ Loại những **phòng ban** có lương trung bình ≤ 2000.

---

## 1.7. SELECT

**Tôi muốn output gì?**

Sau khi dữ liệu đã được lọc và gom nhóm, SQL mới xác định những cột nào cần xuất ra.

```sql
SELECT department, AVG(salary)
```

→ Output gồm:

* `department`
* `AVG(salary)`

---

## 1.8. DISTINCT

**Tôi có cần loại bỏ dữ liệu trùng không?**

Nếu có `DISTINCT`, SQL loại bỏ các dòng trùng nhau trong kết quả.

```sql
SELECT DISTINCT department
FROM Employees;
```

→ Mỗi `department` chỉ xuất hiện một lần.

---

## 1.9. ORDER BY

**Tôi muốn sắp xếp thế nào?**

Sau khi có kết quả, SQL sắp xếp dữ liệu.

```sql
ORDER BY department;
```

→ Sắp xếp theo `department`.

Có thể dùng:

```sql
ORDER BY department ASC;
```

→ Tăng dần.

Hoặc:

```sql
ORDER BY department DESC;
```

→ Giảm dần.

---

## 1.10. LIMIT

**Tôi chỉ muốn lấy bao nhiêu dòng?**

Cuối cùng, `LIMIT` giới hạn số dòng kết quả trả về.

```sql
LIMIT 5;
```

→ Chỉ lấy 5 dòng đầu tiên sau khi đã sắp xếp.

---

# 2. Áp dụng vào ví dụ

```sql
SELECT department, AVG(salary)
FROM Employees
WHERE salary > 1000
GROUP BY department
HAVING AVG(salary) > 2000
ORDER BY department;
```

SQL có thể hiểu theo từng bước:

### Bước 1: FROM

```sql
FROM Employees
```

→ Lấy toàn bộ dữ liệu từ `Employees`.

### Bước 2: WHERE

```sql
WHERE salary > 1000
```

→ Chỉ giữ nhân viên có lương > 1000.

### Bước 3: GROUP BY

```sql
GROUP BY department
```

→ Gom những nhân viên còn lại theo từng phòng ban.

### Bước 4: AVG

```sql
AVG(salary)
```

→ Tính lương trung bình của từng phòng ban.

### Bước 5: HAVING

```sql
HAVING AVG(salary) > 2000
```

→ Chỉ giữ những phòng ban có lương trung bình > 2000.

### Bước 6: SELECT

```sql
SELECT department, AVG(salary)
```

→ Lấy tên phòng ban và lương trung bình.

### Bước 7: ORDER BY

```sql
ORDER BY department
```

→ Sắp xếp kết quả theo tên phòng ban.

---

# 3. Mẹo nhớ

Có thể nhớ câu hỏi mà SQL lần lượt trả lời:

```text
FROM:
Tôi lấy dữ liệu ở đâu?

JOIN / ON:
Tôi cần kết hợp dữ liệu nào?

WHERE:
Tôi loại dòng nào?

GROUP BY:
Tôi gom theo cái gì?

COUNT / SUM / AVG:
Tôi tính gì?

HAVING:
Tôi loại nhóm nào?

SELECT:
Tôi muốn output gì?

DISTINCT:
Tôi có cần loại dữ liệu trùng không?

ORDER BY:
Tôi muốn sắp xếp thế nào?

LIMIT:
Tôi chỉ muốn lấy bao nhiêu dòng?
```

### Công thức tư duy quan trọng nhất

```text
FROM
→ Lấy dữ liệu

WHERE
→ Lọc dòng

GROUP BY
→ Gom nhóm

COUNT / SUM / AVG
→ Tính toán trên nhóm

HAVING
→ Lọc nhóm

SELECT
→ Chọn thứ muốn xuất ra

ORDER BY
→ Sắp xếp

LIMIT
→ Giới hạn kết quả
```

> **Điểm cần nhớ nhất:** `WHERE` xảy ra **trước** `GROUP BY`, còn `HAVING` xảy ra **sau** `GROUP BY`. Vì vậy `WHERE` dùng để lọc **dòng**, còn `HAVING` dùng để lọc **nhóm**.

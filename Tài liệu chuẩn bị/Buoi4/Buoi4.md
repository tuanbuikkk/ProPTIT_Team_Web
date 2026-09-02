# 1. Tối ưu truy vấn

Tối ưu truy vấn là tìm cách để database trả về cùng một kết quả nhưng **tốn ít thời gian, CPU, RAM và I/O hơn**.

Các cách thường thấy:

- Không dùng `SELECT *` mà chọn 1 cột cụ thể.
- Sự dụng index.
- Tối ưu `JOIN` đi hoặc dùng `EXISTS`, `IN`.
- Dùng `EXPLAIN`.

---

# 2. Index

- Index một công cụ mạnh mẽ giúp cải thiện hiệu suất truy vấn dữ liệu.
- Có 2 loại index:

## 2.1. Clustered Index

Đây là dạng chỉ mục sắp xếp lại dữ liệu thực tế trong bảng theo thứ tự của chỉ mục.

Mỗi bảng chỉ có thể có một Clustered Index vì dữ liệu chỉ có thể được sắp xếp theo một thứ tự duy nhất.

```sql
CREATE CLUSTERED INDEX idx_name
ON table_name (column_name);
```

## 2.2. Non-Clustered Index

Không sắp xếp lại dữ liệu bảng, mà chỉ lưu trữ thông tin chỉ mục ở một khu vực riêng biệt.

Một bảng có nhiều non-clustered index.

```sql
CREATE NONCLUSTERED INDEX idx_emp_id
ON Employees (EmployeeID);
```

## 2.3. Unique Index

Đảm bảo rằng không có hai hàng trong bảng có cùng giá trị trên các cột được lập chỉ mục.

```sql
CREATE UNIQUE INDEX idx_name
ON table_name (column_name);
```

## 2.4. Full-Text Index

Được sử dụng cho các cột văn bản dài, cho phép tìm kiếm từ hoặc cụm từ một cách hiệu quả.

```sql
CREATE FULLTEXT INDEX ON table_name (column_name)
KEY INDEX idx_name;
```

---

# 3. Sử dụng Index để tối ưu truy vấn

## Sử dụng index khi

- Khi bảng có dữ liệu lớn và bạn thường xuyên thực hiện các truy vấn phức tạp.
- Khi cần tối ưu hóa các câu lệnh `SELECT` với điều kiện `WHERE`, `JOIN`, và `ORDER BY`.

## Sử dụng EXPLAIN để kiểm tra hiệu suất truy vấn

Lệnh `EXPLAIN` cho phép bạn kiểm tra xem SQL có sử dụng đúng Index khi truy vấn hay không.

## Thêm Index vào bảng

Thêm Index vào các cột thường xuyên được truy vấn giúp tăng tốc độ truy vấn.

```sql
ALTER TABLE table_name
ADD INDEX (column_name);
```

## Hạn chế index

- Index làm tăng kích thước của cơ sở dữ liệu, đặc biệt khi bảng có nhiều Index.
- Việc cập nhật, chèn thêm, hoặc xóa dữ liệu trong bảng có Index sẽ mất nhiều thời gian hơn vì SQL cần cập nhật cả chỉ mục.

---

# 4. Transaction

- Là một nhóm các thao tác database được xem như một công việc duy nhất.
- Ví dụ A chuyển tiền cho B -> A -5tr, B + 5tr. Hành động chuyển tiền và nhận tiền này là 1 transaction.
- Tức là cùng thành công hoặc cùng thất bại.

Dễ hiểu hơn thì giả dụ nếu quá trình chuyển tiền lỗi khiến A chuyển tiền -5tr nhưng B không nhận được tiền -> 5tr biến mất khỏi hệ thống. Đó là lí do cần transaction.

## Quá trình thường sẽ là

```sql
START TRANSACTION;

UPDATE ...
INSERT ...
DELETE ...

COMMIT;
```

Nếu lỗi:

```sql
ROLLBACK;
```

---

# 5. ACID

## A - Atomicity (Tính nguyên tử)

Mọi thay đổi về dữ liệu phải đảm bảo trọn vẹn, nếu các tiến trình thực hiện cùng thành công/thất bại hoặc là sẽ không có bất kỳ sự thay đổi nào về dữ liệu nếu có sự cố tiến trình xảy ra.

### Ví dụ

Khi bạn chuyển tiền từ tài khoản A sang tài khoản B, hệ thống thực hiện hai thao tác:

1. Trừ tiền ở A.
2. Cộng tiền vào B.

Nếu việc trừ tiền thành công nhưng cộng tiền thất bại (ví dụ do lỗi kết nối), hệ thống phải rollback toàn bộ giao dịch, không để tài khoản A bị mất tiền.

---

## C - Consistency (Tính nhất quán)

Khi có một Transaction được hoàn thành, tất cả dữ liệu phải được bảo toàn các mối liên kết dù cho tiến trình thao tác thành công hay thất bại.

### Ví dụ

Trong một hệ thống ngân hàng, tổng số tiền trong toàn hệ thống phải giữ nguyên sau mỗi giao dịch.

Nếu bạn chuyển 1 triệu từ A sang B:

```text
A: -1 triệu
B: +1 triệu
```

Tổng tiền của cả hệ thống không thay đổi.

Nếu sau giao dịch mà tổng tiền bị thay đổi (do lỗi logic hoặc vi phạm ràng buộc), hệ thống phải từ chối giao dịch đó.

---

## I - Isolation (Tính độc lập)

Nếu cùng một lúc, có nhiều tiến trình cùng diễn ra thì cần một cơ chế đưa ra để bảo đảm rằng các tiến trình này có thể hoạt động song song mà không ảnh hưởng đến nhau.

### Ví dụ

Trong trường hợp một khách hàng chuyển tiền vào ngân hàng và một nhân viên kế toán rút tiền ra từ ngân hàng cùng diễn ra một lúc.

Lúc này, cơ sở dữ liệu sẽ thực hiện đồng thời hai hành động là cộng thêm vào số dư của khách hàng, đồng thời trừ đi số tiền kế toán rút ra.

Sẽ có cơ chế để cả hai hành động này được diễn ra thành công song song mà không ảnh hưởng đến việc xử lý database.

---

## D - Durability (Tính bền vững)

Đảm bảo rằng khi các Transaction diễn ra thành công thì tác dụng nó tạo ra với cơ sở dữ liệu phải bền vững.

Dù hệ thống có xảy ra bất kỳ lỗi gì thì dữ liệu luôn được khôi phục lại nguyên trạng.

### Ví dụ

Đối với các tiến trình giao dịch tiền qua ngân hàng. Khi các Transaction hoàn tất, dữ liệu sẽ được ghi lại dưới dạng đĩa cứng, các giao dịch cũng được ghi chép lại.

Nếu có bất kỳ sự cố nào xảy ra đều có thể dễ dàng backup lại data.

---

# 6. Dirty Read

- Là đọc dữ liệu chưa được chuyển giao.
- Ví dụ A có 10tr update số dư 5tr **NHƯNG chưa commit**.
- Dirty Read là B đọc dữ liệu đã update của A nhưng chưa commit.

### Hậu quả

B không biết 5tr có trở thành dữ liệu chính thức không, hay sẽ bị `ROLLBACK` hoặc dùng dữ liệu đấy để đưa ra quyết định sai.

---

# 7. Dirty Write

- Là ghi đè lên thay đổi chưa được chuyển giao.
- Ví dụ A có 10tr update số dư 5tr nhưng chưa commit.
- Dirty Write là B thay đổi dữ liệu đã update của A nhưng chưa commit.

### Hậu quả

Trạng thái cuối cùng khó kiểm soát.
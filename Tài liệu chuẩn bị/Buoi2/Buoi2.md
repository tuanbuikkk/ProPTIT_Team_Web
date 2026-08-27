# 1. Thiết kế cơ sở dữ liệu

Thiết kế cơ sở dữ liệu là quá trình xác định cấu trúc lưu trữ, tổ chức và quản lý dữ liệu sao cho giảm thiểu sự dư thừa, đảm bảo tính toàn vẹn và tối ưu hóa hiệu năng truy vấn. 

Thường sẽ đi theo các giai đoạn:
* Xác định yêu cầu : “Database cần gì”
* Xác định Entity 
* Xác định Attribute
* Xác định Relationship : “MQH là 1:1, 1:N hay N:N”
* Xác định Primary Key, Foreign Key 
* Chuẩn hóa : “1NF? 2NF? 3NF”
* viết SQL 

---

# 2. Lược đồ quan hệ E-R

**Entity - Relationship** : là cách mô hình hóa database trước khi biến nó thành các bảng. 

Trong đó Entity là đối tượng mà database cần lưu thông tin và Entity thì có các attribute. Relationship thì mô tả các Entity có mối quan hệ gì với nhau.
	
Ví dụ: 		
* 1 trường đại học thì thường có Entity : SinhVien, MonHoc, Khoa, Lop
* Trong đó Entity SinhVien có attribute : MaSV, HoTen, NgaySinh,...
* Về quan hệ thì chia làm 3 loại là : 
  * **1:1** : 1 SinhVien có 1 MaSV và 1 MaSV thuộc về 1 SinhVien
  * **1:N** : 1 SinhVien có 1 HoTen nhưng rất nhiều HoTen có thể là của nhiều SinhVien
  * **N:N** : Nhiều SinhVien học 1 Mon và nhiều Mon có nhiều SinhVien học

---

# 3. Mô hình dữ liệu quan hệ

Mô hình quan hệ biểu diễn dữ liệu bằng bảng. 

Trong bảng có:
* **Bảng** : Tập hợp các dữ liệu có cấu trúc về một Entity
* **Row** : Bao gồm tập hợp giá trị của tất cả các attribute trên một hàng ngang. 
* **Column** : Một attribute của 1 Entity
* **Primary Key** : dùng để xác định duy nhất một dòng. 
* **Foreign Key** : Là thứ các nối các bảng
	
---

# 4. Chuẩn hóa dữ liệu

Normalization là quá trình tổ chức các bảng để:
* giảm dữ liệu dư thừa
* tránh mâu thuẫn dữ liệu
* tránh anomaly
* giúp database dễ bảo trì

## 4.1 1NF — First Normal Form

Một bảng đạt 1NF khi mỗi ô chỉ chứa một giá trị.

Ví dụ đạt 1NF:

| MaSV | HoTen | MonHoc |
|---|---|---|
| SV01 | Tuấn | SQL, Java, |
| SV02 | Nam | SQL, Python |

-> Cột MonHoc chứa nhiều giá trị: SQL, Java  
Tách ra để thành 1NF:

| MaSV | HoTen | MonHoc |
|---|---|---|
| SV01 | Tuấn | SQL |
| SV01 | Tuấn | Java |
| SV02 | Nam | SQL |
| SV02 | Nam | Python |


## 43.2 2NF — Second Normal Form 

Một bảng đạt 2NF khi: Đã đạt 1NF và không có phụ thuộc bộ phận vào khóa chính.

Để hiểu rõ thì cần nắm được khái niệm:
* `MaSV -> Tên` thì MaSV là khóa chính
* `MaMon -> TenMon` thì MaMon là khóa chính
* nhưng cần `(MaSV, MaMon) -> Diem` thì (MaSV, MaMon) là khóa chính ghép

Ta xem xét 1 vấn đề, có 1 bảng:

**DANGKY**

| MaSV | MaMon | HoTen | TenMon | Diem |
|---|---|---|---|---|
| SV01 | SQL | Tuấn | CSDL | 9 |
| SV01 | JAVA | Tuấn | Java | 8 |
| SV02 | SQL | Nam | CSDL | 7 |

Nhìn thì thấy (MaSV, MaMon) là khóa chính ghép. Tuy nhiên:
* `MaMon -> TenMon` không cần MaSV
* Ngược lại: `MaSV -> HoTen` không cần MaMon

Việc TenMon hoặc HoTen chỉ phụ thuộc vào 1 bộ phận của khóa chính nên đây không là bảng 2NF.  
Để khắc phục ta có thể tách ra làm 3 bảng như sau:

**SINHVIEN**

| MaSV | HoTen |
|---|---|

**MONHOC**

| MaMon | TenMon |
|---|---|

**DANGKY**

| MaSV | MaMon | Diem |
|---|---|---|


## 4.3 3NF

Một bảng đạt 3NF khi đạt 2NF và không có phụ thuộc bắc cầu. 

Phụ thuộc bắc cầu là gì ?  
Ta có bảng:

**SINHVIEN**

| MaSV | HoTen | MaMon | TenMon |
|---|---|---|---|

* `MaSV -> MaMon`
* `MaMon -> TenMon`
* Tính chất bắc cầu : `MaSV -> MaMon -> TenMon`

Ta thấy TenMon không thực sự phụ thuộc trực tiếp MaSV -> sự phụ thuộc bắc cầu.  
Để chuyển thành bảng 3NF, ta tách như sau:

**SINH VIEN**

| MaSV | HoTen | MaKhoa |
|---|---|---|

**MON**

| MaMon | TenMon |
|---|---|
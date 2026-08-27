## 1. Cơ sở dữ liệu (CSDL) là gì?
Cơ sở dữ liệu (CSDL) là một tập hợp các dữ liệu được tổ chức có cấu trúc, có liên quan logic với nhau và được lưu trữ trên các thiết bị ghi nhớ.

### Đặc điểm chính của CSDL:
*   **Có cấu trúc:** Dữ liệu không nằm lộn xộn mà được sắp xếp theo quy tắc nhất định.
*   **Tính chia sẻ:** Nhiều người dùng hoặc nhiều ứng dụng có thể cùng truy cập vào một CSDL tại cùng một thời điểm.
*   **Giảm thiểu dư thừa:** Dữ liệu được thiết kế để giúp tiết kiệm không gian lưu trữ và đảm bảo tính nhất quán.

---

## 2. Hệ quản trị CSDL (DBMS)
Hệ quản trị CSDL là các phần mềm cung cấp môi trường để người dùng có thể tạo lập, lưu trữ, thao tác và kiểm soát CSDL. 

> **Ví dụ:** Nếu CSDL là "kho hàng" chứa đồ vật, thì Hệ quản trị CSDL chính là "người thủ kho".

### Các chức năng chính:
1.  **Cung cấp môi trường tạo lập dữ liệu:** Cho phép khai báo cấu trúc dữ liệu, các kiểu dữ liệu và các ràng buộc.
2.  **Cung cấp ngôn ngữ thao tác dữ liệu:** Cho phép người dùng thêm, sửa, xóa và truy vấn dữ liệu.
3.  **Đảm bảo truy cập đồng thời:** Cho phép nhiều người dùng hoặc ứng dụng truy cập dữ liệu cùng lúc một cách chính xác.
4.  **Bảo mật và an toàn dữ liệu:** Phân quyền người dùng và cung cấp cơ chế sao lưu (backup) & phục hồi khi có sự cố.

---

## 3. Cú pháp SQL cơ bản (MS SQL Server)

### Tạo Database:
```sql
CREATE DATABASE TenDatabase;
```

### Tạo Table:
```sql
CREATE TABLE TenBang ( 
  TenCot1 KieuDuLieu [RangBuoc], 
  TenCot2 KieuDuLieu [RangBuoc],
  ... 
); 

```
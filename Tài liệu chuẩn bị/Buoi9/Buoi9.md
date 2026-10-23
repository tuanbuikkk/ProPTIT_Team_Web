# Buổi 9: Thao tác với Database trong Spring Boot -- JPA

> Tài liệu học tập dành cho người mới học Spring Boot. Nội dung tập
> trung vào **mối quan hệ giữa các Entity trong JPA** và **Derived Query
> Methods của Spring Data JPA**. Ví dụ dùng Java, Spring Data JPA và cơ
> sở dữ liệu quan hệ như MySQL.

## Mục lục

-   [1. Tổng quan](#1-tổng-quan)
-   [2. Các mối quan hệ trong JPA](#2-các-mối-quan-hệ-trong-jpa)
    -   [2.1. Entity và quan hệ trong
        database](#21-entity-và-quan-hệ-trong-database)
    -   [2.2. OneToMany và ManyToOne](#22-onetomany-và-manytoone)
    -   [2.3. OneToOne](#23-onetoone)
    -   [2.4. ManyToMany](#24-manytomany)
    -   [2.5. Bảng tổng hợp các quan hệ](#25-bảng-tổng-hợp-các-quan-hệ)
    -   [2.6. Owning side, mappedBy và
        JoinColumn](#26-owning-side-mappedby-và-joincolumn)
    -   [2.7. CascadeType và
        orphanRemoval](#27-cascadetype-và-orphanremoval)
    -   [2.8. LAZY, EAGER và lỗi thường
        gặp](#28-lazy-eager-và-lỗi-thường-gặp)
-   [3. Derived Query Methods](#3-derived-query-methods)
    -   [3.1. Khái niệm và cơ chế](#31-khái-niệm-và-cơ-chế)
    -   [3.2. Cấu trúc tên hàm](#32-cấu-trúc-tên-hàm)
    -   [3.3. Các từ khóa phổ biến](#33-các-từ-khóa-phổ-biến)
    -   [3.4. Pageable và Sort](#34-pageable-và-sort)
    -   [3.5. Ví dụ thực tế](#35-ví-dụ-thực-tế)
    -   [3.6. Ưu điểm, nhược điểm và khi nào
        dùng](#36-ưu-điểm-nhược-điểm-và-khi-nào-dùng)
-   [4. Bài tập tự luyện](#4-bài-tập-tự-luyện)
-   [5. Checklist ghi nhớ](#5-checklist-ghi-nhớ)
-   [6. Tài liệu tham khảo](#6-tài-liệu-tham-khảo)

------------------------------------------------------------------------

## 1. Tổng quan

Trong ứng dụng Spring Boot, dữ liệu thường được lưu trong hệ quản trị cơ
sở dữ liệu như MySQL hoặc PostgreSQL. JPA (Java Persistence API) là đặc
tả giúp ánh xạ đối tượng Java với dữ liệu quan hệ; Hibernate là một
triển khai JPA phổ biến. Spring Data JPA xây dựng thêm lớp Repository để
giảm lượng code truy cập dữ liệu.

Ba khái niệm cần phân biệt:

-   **Entity**: lớp Java được ánh xạ với bảng dữ liệu, thường dùng
    `@Entity`.
-   **Repository**: giao diện cung cấp các thao tác đọc/ghi dữ liệu,
    thường kế thừa `JpaRepository`.
-   **Relationship mapping**: khai báo quan hệ giữa các Entity bằng các
    annotation như `@ManyToOne`, `@OneToMany`.

Ví dụ luồng xử lý đơn giản:

1.  Controller nhận yêu cầu HTTP.
2.  Service xử lý nghiệp vụ.
3.  Repository thực hiện truy vấn.
4.  JPA provider (ví dụ Hibernate) chuyển thao tác Entity thành SQL và
    trao đổi với database.

> Các ví dụ dưới đây dùng Jakarta Persistence (`jakarta.persistence.*`)
> như các dự án Spring Boot hiện đại. Nếu dự án cũ dùng phiên bản Spring
> Boot trước đây, import có thể là `javax.persistence.*`.

## 2. Các mối quan hệ trong JPA

### 2.1. Entity và quan hệ trong database

Giả sử có hệ thống quản lý đơn hàng:

-   Một `Customer` có thể có nhiều `Order`.
-   Mỗi `Order` thuộc về một `Customer`.
-   Mỗi `Order` có nhiều `OrderItem`.
-   Mỗi `OrderItem` tham chiếu đến một `Product`.

Ở database, quan hệ thường được thể hiện bằng **khóa ngoại (Foreign Key
-- FK)**. JPA cho phép biểu diễn mối quan hệ ấy bằng thuộc tính Java
thay vì tự thao tác khóa ngoại trong từng câu SQL.

Ví dụ cấu trúc bảng:

``` sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(12, 2),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

`orders.customer_id` là khóa ngoại trỏ đến `customers.id`. Một customer
có thể xuất hiện trong nhiều dòng `orders`, nhưng mỗi order chỉ có một
`customer_id`.

### 2.2. OneToMany và ManyToOne

#### Khái niệm

`One-to-Many (1:N)` nghĩa là một đối tượng liên kết với nhiều đối tượng
khác. `Many-to-One (N:1)` nhìn từ chiều ngược lại: nhiều đối tượng cùng
liên kết đến một đối tượng.

Ví dụ: một khách hàng có nhiều đơn hàng, nhưng một đơn hàng chỉ thuộc về
một khách hàng.

-   Phía `Order`: dùng `@ManyToOne`.
-   Phía `Customer`: có thể dùng `@OneToMany` để lấy danh sách đơn hàng.

Trong database, khóa ngoại thường nằm ở bảng phía **nhiều**
(`orders.customer_id`). Vì vậy `@ManyToOne` thường là owning side (phía
sở hữu quan hệ).

#### Entity `Customer`

``` java
import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    protected Customer() {
    }

    public Customer(String name) {
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public List<Order> getOrders() {
        return orders;
    }

    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}
```

#### Entity `Order`

``` java
import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Column(name = "total_amount")
    private BigDecimal totalAmount;

    protected Order() {
    }

    public Order(BigDecimal totalAmount) {
        this.totalAmount = totalAmount;
    }

    public Long getId() {
        return id;
    }

    public Customer getCustomer() {
        return customer;
    }

    public void setCustomer(Customer customer) {
        this.customer = customer;
    }
}
```

#### Giải thích từng annotation

-   `@ManyToOne`: nhiều đơn hàng có thể tham chiếu đến cùng một
    customer.
-   `@JoinColumn(name = "customer_id")`: chỉ định cột khóa ngoại trong
    bảng `orders`.
-   `optional = false`: quan hệ bắt buộc ở mức mapping; nên phối hợp với
    `nullable = false` và ràng buộc database để bảo đảm dữ liệu.
-   `mappedBy = "customer"`: cho biết collection `Customer.orders` là
    phía đảo ngược; `customer` là **tên thuộc tính Java trong Entity
    `Order`**, không phải tên cột `customer_id`.
-   `FetchType.LAZY`: trì hoãn tải collection `orders` cho đến khi cần
    truy cập, tùy ngữ cảnh persistence.
-   Khởi tạo `new ArrayList<>()` giúp collection không bị `null` khi
    thêm phần tử.

**Vì sao nên cập nhật cả hai phía trong Java?**

Với quan hệ hai chiều, database được cập nhật dựa trên owning side
(`Order.customer`). Nếu chỉ gọi `customer.getOrders().add(order)` mà
không đặt `order.setCustomer(customer)`, quan hệ trong bộ nhớ có thể
không khớp với dữ liệu được lưu. Hàm `addOrder()` phía trên cập nhật cả
hai phía.

Ví dụ sử dụng trong service:

``` java
Customer customer = customerRepository.findById(1L)
        .orElseThrow();

Order order = new Order(new BigDecimal("250000"));
customer.addOrder(order);
orderRepository.save(order);
```

Ví dụ minh họa này giả định `Customer` đã tồn tại. Nếu cấu hình cascade
phù hợp hoặc dùng transaction, cách lưu có thể thay đổi theo nghiệp vụ.

> **Lưu ý:** Không phải lúc nào cũng cần khai báo quan hệ hai chiều. Nếu
> ứng dụng chỉ cần biết đơn hàng thuộc khách hàng nào, có thể chỉ khai
> báo `@ManyToOne` để mô hình đơn giản hơn.

### 2.3. OneToOne

#### Khái niệm

`One-to-One (1:1)` nghĩa là một đối tượng liên kết với tối đa một đối
tượng khác và ngược lại. Ví dụ, mỗi user có một hồ sơ cá nhân riêng.

Database thường đặt FK ở một bảng và thêm ràng buộc `UNIQUE` để một bản
ghi không thể được tham chiếu bởi nhiều bản ghi khác.

Ví dụ:

``` java
@Entity
@Table(name = "user_profiles")
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 500)
    private String bio;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;
}
```

Nếu `User` cần điều hướng ngược sang hồ sơ:

``` java
@OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
private UserProfile profile;
```

Trong ví dụ này, `UserProfile.user` là owning side vì nó khai báo
`@JoinColumn`. `User.profile` dùng `mappedBy = "user"` để trỏ tới thuộc
tính đó.

**Khi nào dùng?** Khi dữ liệu thực sự có quan hệ 1:1 và cần quản lý như
một thực thể riêng. Không nên tách bảng chỉ vì muốn dùng `@OneToOne`;
hãy cân nhắc nhu cầu truy cập, bảo mật, vòng đời và độ phức tạp của mô
hình.

### 2.4. ManyToMany

#### Khái niệm

`Many-to-Many (N:N)` nghĩa là mỗi đối tượng ở cả hai phía có thể liên
kết với nhiều đối tượng phía còn lại.

Ví dụ: một sinh viên học nhiều khóa học và mỗi khóa học có nhiều sinh
viên.

Quan hệ này thường cần **bảng trung gian**:

``` sql
CREATE TABLE student_course (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

Mapping hai chiều:

``` java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses = new ArrayList<>();
}
```

``` java
@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private List<Student> students = new ArrayList<>();
}
```

Giải thích:

-   `@JoinTable`: khai báo bảng trung gian.
-   `joinColumns`: khóa ngoại từ bảng trung gian trỏ đến Entity ở owning
    side (`Student`).
-   `inverseJoinColumns`: khóa ngoại trỏ đến Entity còn lại (`Course`).
-   `mappedBy = "courses"`: `Course.students` là phía đảo ngược; thuộc
    tính `courses` trong `Student` quản lý mapping.

#### Khi nào không nên dùng `@ManyToMany` trực tiếp?

Nếu quan hệ có dữ liệu riêng, hãy tạo Entity trung gian. Ví dụ, đăng ký
khóa học có ngày đăng ký, điểm số, trạng thái hoặc học phí:

-   `Student`
-   `Course`
-   `Enrollment` (có `student`, `course`, `enrolledAt`, `score`,
    `status`)

Khi ấy `Enrollment` không chỉ là bảng nối mà còn chứa nghiệp vụ, nên cần
Entity riêng. Tương tự, trong đơn hàng, chi tiết đơn hàng cần lưu số
lượng và giá tại thời điểm mua; nên dùng `OrderItem` thay vì quan hệ
`Order`--`Product` dạng `@ManyToMany`.

### 2.5. Bảng tổng hợp các quan hệ

  -----------------------------------------------------------------------
  Quan hệ           Ý nghĩa           Cấu trúc database Ví dụ
                                      thường gặp        
  ----------------- ----------------- ----------------- -----------------
  `@OneToOne`       Một -- một        FK ở một bảng,    User --
                                      thường có         UserProfile
                                      `UNIQUE`          

  `@OneToMany`      Một -- nhiều      FK nằm ở bảng     Customer -- Order
                                      phía nhiều        

  `@ManyToOne`      Nhiều -- một      Entity hiện tại   Order -- Customer
                                      giữ FK            

  `@ManyToMany`     Nhiều -- nhiều    Bảng trung gian   Student -- Course
                                      chứa hai FK       
  -----------------------------------------------------------------------

### 2.6. Owning side, mappedBy và JoinColumn

Đây là phần rất quan trọng khi mapping quan hệ.

**Owning side (phía sở hữu)** là phía quyết định mapping quan hệ xuống
database. Với quan hệ hai chiều, phía này thường khai báo `@JoinColumn`
hoặc `@JoinTable`.

**Inverse side (phía đảo ngược)** dùng `mappedBy` để tham chiếu thuộc
tính quan hệ ở owning side.

Ví dụ:

``` java
// Owning side
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

``` java
// Inverse side
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

Cần phân biệt:

-   `customer_id`: tên cột trong database.
-   `customer`: tên thuộc tính Java trong Entity `Order`.
-   `mappedBy = "customer"` phải tham chiếu tên thuộc tính Java.

Nếu khai báo quan hệ hai chiều nhưng quên `mappedBy`, JPA có thể hiểu
rằng hai phía là hai quan hệ riêng và tạo thêm bảng/cột liên kết ngoài
dự kiến.

### 2.7. CascadeType và orphanRemoval

#### Cascade là gì?

Cascade cho phép một thao tác trên Entity này được truyền sang Entity
liên quan. Cascade **không phải** là loại quan hệ và cũng không đồng
nghĩa với khóa ngoại.

  -----------------------------------------------------------------------
  Giá trị                             Ý nghĩa
  ----------------------------------- -----------------------------------
  `CascadeType.PERSIST`               Persist Entity cha có thể persist
                                      Entity liên quan

  `CascadeType.MERGE`                 Merge Entity cha có thể merge
                                      Entity liên quan

  `CascadeType.REMOVE`                Xóa Entity cha có thể xóa Entity
                                      liên quan

  `CascadeType.REFRESH`               Refresh cha có thể refresh Entity
                                      liên quan

  `CascadeType.DETACH`                Detach cha có thể detach Entity
                                      liên quan

  `CascadeType.ALL`                   Bao gồm tất cả các thao tác trên
  -----------------------------------------------------------------------

Ví dụ:

``` java
@OneToMany(
    mappedBy = "customer",
    cascade = {CascadeType.PERSIST, CascadeType.MERGE}
)
private List<Order> orders = new ArrayList<>();
```

Với cấu hình này, thao tác persist hoặc merge trên `Customer` có thể lan
sang các `Order` liên quan. Nó không tự động bao gồm xóa.

#### `orphanRemoval = true` là gì?

`orphanRemoval = true` cho phép xóa Entity con khỏi database khi Entity
con bị loại khỏi quan hệ cha--con được quản lý. Tính năng này thường phù
hợp khi Entity con thuộc quyền sở hữu vòng đời của cha, như `Order` và
`OrderItem`.

``` java
@OneToMany(
    mappedBy = "order",
    cascade = CascadeType.ALL,
    orphanRemoval = true
)
private List<OrderItem> items = new ArrayList<>();
```

Khi một `OrderItem` bị loại khỏi `items` của một `Order` đang được quản
lý, nó có thể bị xóa khỏi database khi persistence context đồng bộ thay
đổi.

#### Phân biệt Cascade REMOVE và orphanRemoval

-   `CascadeType.REMOVE`: xóa cha thì thao tác xóa được lan sang các con
    liên quan.
-   `orphanRemoval = true`: con bị loại khỏi quan hệ cha--con cũng có
    thể bị xóa, kể cả khi cha vẫn tồn tại.

**Cảnh báo:** Cẩn thận với `CascadeType.REMOVE` hoặc `CascadeType.ALL` ở
`@ManyToMany`. Ví dụ xóa một sinh viên không nên làm xóa các khóa học mà
nhiều sinh viên khác đang học. Thông thường chỉ cần xóa các liên kết
trong bảng trung gian, không xóa Entity `Course`.

Không bật cascade theo thói quen. Hãy xác định Entity nào thực sự sở hữu
vòng đời của Entity nào.

### 2.8. LAZY, EAGER và lỗi thường gặp

-   `FetchType.LAZY`: trì hoãn tải dữ liệu liên quan cho tới khi được
    truy cập; collection `@OneToMany` thường mặc định LAZY.
-   `FetchType.EAGER`: yêu cầu dữ liệu liên quan phải được tải sớm;
    không đảm bảo provider luôn dùng đúng một câu SQL.

Đối với quan hệ `@ManyToOne` và `@OneToOne`, mặc định của JPA là EAGER;
vì vậy nhiều dự án chủ động khai báo LAZY để tránh tải thừa. Việc tải
thực tế còn phụ thuộc provider và cách truy vấn.

**Lỗi thường gặp:**

1.  **`LazyInitializationException`**: truy cập collection LAZY sau khi
    persistence context/session đã đóng. Có thể xử lý bằng cách truy vấn
    dữ liệu cần thiết trong transaction, dùng DTO hoặc fetch join phù
    hợp.
2.  **N+1 query**: lấy N bản ghi cha, sau đó mỗi bản ghi lại phát sinh
    truy vấn riêng để lấy dữ liệu con. Cân nhắc `JOIN FETCH`,
    `@EntityGraph`, projection hoặc batch fetching dựa trên nhu cầu.
3.  **Vòng lặp JSON**: `Customer` chứa `orders`, mỗi `Order` lại chứa
    `customer`; trả Entity trực tiếp qua API có thể tạo vòng lặp
    serialize. Thực tế nên dùng DTO hoặc cấu hình serialize phù hợp.
4.  **Chỉ cập nhật một phía của quan hệ hai chiều**: object trong bộ nhớ
    không đồng nhất. Hãy viết helper method để cập nhật cả hai phía.
5.  **Cascade xóa quá rộng**: vô tình xóa dữ liệu còn được Entity khác
    sử dụng.

------------------------------------------------------------------------

## 3. Derived Query Methods

### 3.1. Khái niệm và cơ chế

**Derived Query Method** là cách khai báo truy vấn bằng tên phương thức
trong Repository. Spring Data JPA phân tích tên hàm, kiểm tra thuộc tính
của Entity và tạo truy vấn tương ứng. Ta không cần tự viết SQL/JPQL cho
các trường hợp phổ biến.

Ví dụ:

``` java
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByName(String name);
}
```

Nếu `Product` có thuộc tính `name`, Spring Data JPA có thể tạo truy vấn
tương đương:

``` sql
SELECT * FROM products WHERE name = ?;
```

SQL thực tế còn phụ thuộc database, mapping và JPA provider; ví dụ trên
chỉ minh họa ý tưởng.

#### Cơ chế hoạt động khái quát

1.  Ứng dụng khởi động, Spring phát hiện Repository.
2.  Spring Data phân tích tên phương thức, ví dụ `findByName`.
3.  Nó xác định Entity và thuộc tính tương ứng (`Product.name`).
4.  Khi phương thức được gọi, truy vấn được thực thi thông qua JPA
    provider.
5.  Kết quả được ánh xạ thành Entity hoặc kiểu trả về đã khai báo.

Mặc định, Spring Data JPA ưu tiên truy vấn đã khai báo nếu có; nếu không
tìm thấy, chiến lược mặc định `CREATE_IF_NOT_FOUND` sẽ thử tạo truy vấn
từ tên hàm.

### 3.2. Cấu trúc tên hàm

Cấu trúc cơ bản:

``` text
[Tiền tố][Distinct/Top/First...]By[Điều kiện][OrderBy...]
```

Ví dụ:

``` java
List<Product> findDistinctByCategoryOrderByPriceDesc(String category);
```

Có thể đọc thành:

-   `find`: truy vấn dữ liệu.
-   `Distinct`: loại bỏ kết quả trùng theo ngữ nghĩa truy vấn.
-   `ByCategory`: lọc theo thuộc tính `category`.
-   `OrderByPriceDesc`: sắp xếp giá giảm dần.

Các tiền tố thường dùng:

  --------------------------------------------------------------------------------
  Tiền tố                 Công dụng               Ví dụ
  ----------------------- ----------------------- --------------------------------
  `findBy`, `readBy`,     Tìm dữ liệu             `findByName(name)`
  `getBy`                                         

  `existsBy`              Kiểm tra tồn tại        `existsByEmail(email)`

  `countBy`               Đếm bản ghi             `countByCategory(category)`

  `deleteBy`, `removeBy`  Xóa theo điều kiện      `deleteById(id)`

  `findTopNBy`,           Giới hạn số kết quả     `findTop5ByOrderByPriceDesc()`
  `findFirstNBy`                                  
  --------------------------------------------------------------------------------

Tên thuộc tính phải khớp với **property của Entity Java**, không phải
tên cột database. Ví dụ Entity có `createdAt` và database dùng
`created_at`, phương thức nên dùng `findByCreatedAtAfter(...)`.

### 3.3. Các từ khóa phổ biến

  --------------------------------------------------------------------------------------------
  Từ khóa                 Ý nghĩa                 Ví dụ
  ----------------------- ----------------------- --------------------------------------------
  `And`                   Kết hợp điều kiện AND   `findByNameAndCategory(name, category)`

  `Or`                    Kết hợp điều kiện OR    `findByNameOrCategory(name, category)`

  `Is`, `Equals`          Bằng                    `findByCategory(category)`

  `Not`                   Không bằng              `findByStatusNot(status)`

  `LessThan`              Nhỏ hơn                 `findByPriceLessThan(price)`

  `LessThanEqual`         Nhỏ hơn hoặc bằng       `findByPriceLessThanEqual(price)`

  `GreaterThan`           Lớn hơn                 `findByPriceGreaterThan(price)`

  `GreaterThanEqual`      Lớn hơn hoặc bằng       `findByPriceGreaterThanEqual(price)`

  `Between`               Nằm trong khoảng        `findByPriceBetween(min, max)`

  `Before`, `After`       Trước/sau mốc thời gian `findByCreatedAtAfter(time)`
                          hoặc giá trị có thứ tự  

  `IsNull`                Giá trị là NULL         `findByDescriptionIsNull()`

  `IsNotNull`             Giá trị không NULL      `findByDescriptionIsNotNull()`

  `Like`                  Khớp mẫu LIKE do tham   `findByNameLike(pattern)`
                          số cung cấp             

  `Containing`            Chứa chuỗi con          `findByNameContaining(keyword)`

  `StartingWith`          Bắt đầu bằng chuỗi      `findByNameStartingWith(prefix)`

  `EndingWith`            Kết thúc bằng chuỗi     `findByNameEndingWith(suffix)`

  `IgnoreCase`            Không phân biệt         `findByNameIgnoreCase(name)`
                          hoa/thường cho thuộc    
                          tính phù hợp            

  `In`                    Thuộc tập giá trị       `findByCategoryIn(categories)`

  `NotIn`                 Không thuộc tập giá trị `findByCategoryNotIn(categories)`

  `True`, `False`         Kiểm tra boolean        `findByActiveTrue()`

  `OrderBy...Asc`         Sắp xếp tăng dần        `findByCategoryOrderByPriceAsc(category)`

  `OrderBy...Desc`        Sắp xếp giảm dần        `findByCategoryOrderByPriceDesc(category)`

  `Distinct`              Yêu cầu kết quả         `findDistinctByCategory(category)`
                          distinct                
  --------------------------------------------------------------------------------------------

Ví dụ cụ thể:

``` java
List<Product> findByPriceBetween(
    BigDecimal minPrice,
    BigDecimal maxPrice
);

List<Product> findByNameContainingIgnoreCase(String keyword);

List<Product> findByActiveTrueAndPriceLessThanEqual(
    BigDecimal maxPrice
);

List<Product> findByCategoryIn(List<String> categories);
```

Lưu ý:

-   Với `Containing`, `StartingWith`, `EndingWith`, Spring Data JPA tự
    thêm ký tự `%` cần thiết cho LIKE.
-   Với `Like`, thông thường bạn truyền mẫu như `"%coffee%"`.
-   Không nên tự giả định thứ tự ưu tiên giữa nhiều điều kiện `And` và
    `Or` phức tạp. Nếu logic rắc rối, hãy viết truy vấn rõ ràng bằng
    `@Query`.
-   Derived query có thể đi qua thuộc tính lồng nhau. Ví dụ
    `findByCustomerName(...)` có thể truy vấn theo `order.customer.name`
    nếu mô hình Entity phù hợp. Khi tên thuộc tính gây nhập nhằng, có
    thể dùng dấu gạch dưới để chỉ ranh giới, chẳng hạn
    `findByCustomer_Name(...)`.
-   Nếu tên hàm tham chiếu thuộc tính không tồn tại, ứng dụng thường báo
    lỗi lúc khởi tạo Repository hoặc tạo truy vấn; đây là cách phát hiện
    sai sót sớm.

#### Kiểu trả về nên chọn

``` java
Optional<Product> findById(Long id);
List<Product> findByCategory(String category);
boolean existsByName(String name);
long countByCategory(String category);
```

-   `Optional<T>` phù hợp khi kỳ vọng không có hoặc có một kết quả.
-   `List<T>` phù hợp khi cần nhiều kết quả.
-   `boolean` phù hợp cho kiểm tra tồn tại.
-   `long` phù hợp cho số lượng bản ghi.

Nếu điều kiện có thể khớp nhiều bản ghi, không nên khai báo kiểu trả về
đơn như `Product` mà không đảm bảo tính duy nhất; kết quả có nhiều bản
ghi có thể gây lỗi không mong muốn.

### 3.4. Pageable và Sort

#### Sort: sắp xếp động

`Sort` cho phép quyết định thứ tự kết quả tại thời điểm gọi hàm, thay vì
cố định thứ tự trong tên phương thức.

Repository:

``` java
List<Product> findByCategory(String category, Sort sort);
```

Cách gọi:

``` java
List<Product> products = productRepository.findByCategory(
    "DRINK",
    Sort.by(Sort.Direction.DESC, "price")
);
```

Có thể sắp xếp theo nhiều thuộc tính:

``` java
Sort sort = Sort.by(Sort.Direction.ASC, "category")
    .and(Sort.by(Sort.Direction.DESC, "price"));
```

Tên thuộc tính truyền vào `Sort` là property của Entity (ví dụ `price`),
không phải tên cột database (ví dụ `product_price`), trừ trường hợp
query/projection có alias hợp lệ tương ứng.

#### Pageable: phân trang

Khi bảng có hàng chục nghìn bản ghi, không nên gửi tất cả kết quả về
giao diện trong một lần. `Pageable` cho phép lấy một phần dữ liệu.

Repository:

``` java
Page<Product> findByCategory(String category, Pageable pageable);
```

Service hoặc Controller có thể tạo yêu cầu trang:

``` java
Pageable pageable = PageRequest.of(
    0, // page index bắt đầu từ 0
    10, // 10 sản phẩm mỗi trang
    Sort.by(Sort.Direction.ASC, "name")
);

Page<Product> page = productRepository.findByCategory("DRINK", pageable);
```

Một số thông tin quan trọng từ `Page<Product>`:

``` java
page.getContent();       // Danh sách trong trang hiện tại
page.getNumber();        // Chỉ số trang, bắt đầu từ 0
page.getSize();          // Kích thước trang yêu cầu
page.getTotalElements(); // Tổng số bản ghi thỏa điều kiện
page.getTotalPages();    // Tổng số trang
page.hasNext();          // Có trang tiếp theo không?
```

#### Phân biệt Page, Slice và List

  -----------------------------------------------------------------------
  Kiểu trả về             Thông tin               Đặc điểm
  ----------------------- ----------------------- -----------------------
  `List<T>`               Chỉ có danh sách kết    Khi dùng cùng
                          quả                     `Pageable`, có thể giới
                                                  hạn dữ liệu nhưng không
                                                  có metadata tổng trang

  `Page<T>`               Dữ liệu + tổng bản      Có thể cần thêm truy
                          ghi + tổng trang        vấn `COUNT`, hữu ích
                                                  khi giao diện cần biết
                                                  tổng số trang

  `Slice<T>`              Dữ liệu + thông tin còn Thường tránh truy vấn
                          phần tiếp theo hay      đếm tổng, phù hợp tải
                          không                   thêm/infinite scroll
  -----------------------------------------------------------------------

`PageRequest.of(page, size)` dùng chỉ số trang bắt đầu từ **0**. Ví dụ
`PageRequest.of(0, 10)` là trang đầu, lấy tối đa 10 bản ghi.

**Lưu ý hiệu năng:** phân trang theo offset có thể chậm khi offset rất
lớn. Với dữ liệu cực lớn, có thể cần cân nhắc keyset
pagination/scrolling hoặc thiết kế truy vấn phù hợp; đây là chủ đề nâng
cao hơn phần nhập môn.

#### Nhận tham số phân trang từ API

Ví dụ endpoint:

``` java
@GetMapping("/products")
public Page<Product> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    Pageable pageable = PageRequest.of(page, size);
    return productRepository.findAll(pageable);
}
```

Trong ứng dụng thực tế, cần kiểm tra giới hạn `page` và `size` để tránh
người dùng yêu cầu kích thước quá lớn. Nên dùng DTO thay vì trả Entity
trực tiếp nếu API cần kiểm soát cấu trúc dữ liệu trả về.

### 3.5. Ví dụ thực tế

Giả sử có Entity `Product`:

``` java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String category;
    private BigDecimal price;
    private boolean active;

    protected Product() {
    }

    // Constructor, getter và setter được lược bớt
}
```

Repository:

``` java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.JpaRepository;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // Tìm sản phẩm theo tên chính xác
    Optional<Product> findByName(String name);

    // Tìm tên có chứa từ khóa, không phân biệt hoa/thường
    List<Product> findByNameContainingIgnoreCase(String keyword);

    // Sản phẩm đang bán và giá nằm trong khoảng
    List<Product> findByActiveTrueAndPriceBetween(
        BigDecimal minPrice,
        BigDecimal maxPrice
    );

    // Sản phẩm thuộc danh mục, giá giảm dần
    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Kiểm tra có sản phẩm mang tên này đang bán không
    boolean existsByNameAndActiveTrue(String name);

    // Đếm số sản phẩm theo danh mục
    long countByCategory(String category);

    // Lấy sản phẩm theo danh mục với phân trang
    Page<Product> findByCategory(String category, Pageable pageable);

    // Lấy sản phẩm theo danh mục và sắp xếp động
    List<Product> findByCategory(String category, Sort sort);

    // Lấy 5 sản phẩm có giá cao nhất
    List<Product> findTop5ByOrderByPriceDesc();
}
```

Cách dùng trong Service:

``` java
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public List<Product> searchByName(String keyword) {
        return productRepository.findByNameContainingIgnoreCase(keyword);
    }

    public Page<Product> getProducts(String category, int page, int size) {
        Pageable pageable = PageRequest.of(
            page,
            size,
            Sort.by(Sort.Direction.ASC, "name")
        );

        return productRepository.findByCategory(category, pageable);
    }
}
```

Cần import `Page`, `Pageable`, `PageRequest`, `Sort` từ
`org.springframework.data.domain.*` và `Service` từ
`org.springframework.stereotype.Service`.

#### Đọc một tên hàm từ trái sang phải

`findByActiveTrueAndPriceBetween(minPrice, maxPrice)`:

1.  `find`: lấy dữ liệu.
2.  `By`: bắt đầu phần điều kiện.
3.  `ActiveTrue`: chỉ lấy sản phẩm có `active = true`.
4.  `And`: kết hợp thêm điều kiện AND.
5.  `PriceBetween`: giá nằm trong khoảng `minPrice` và `maxPrice`.
6.  Hai tham số truyền vào theo thứ tự xuất hiện của các điều kiện cần
    giá trị.

### 3.6. Ưu điểm, nhược điểm và khi nào dùng

#### Ưu điểm

-   Viết nhanh, ít boilerplate, không cần tự viết câu SQL/JPQL cho truy
    vấn đơn giản.
-   Tên hàm thể hiện tương đối rõ mục đích truy vấn.
-   Spring Data JPA kiểm tra property khi tạo truy vấn, giúp phát hiện
    lỗi tên thuộc tính sớm.
-   Có thể kết hợp với `Pageable`, `Sort`, `Top` và `First`.

#### Nhược điểm

-   Tên hàm rất dài khi có nhiều điều kiện.
-   Không phải lựa chọn dễ đọc nhất cho logic phức tạp, nhiều phép JOIN,
    tổng hợp hoặc `GROUP BY`.
-   Cần hiểu rõ tên property của Entity và cách parser phân tích tên
    phương thức.
-   Truy vấn được sinh tự động không có nghĩa là chắc chắn tối ưu; vẫn
    cần để ý index, số lượng dữ liệu và SQL thực tế.

#### Khi nào nên dùng?

  -----------------------------------------------------------------------
  Tình huống                          Cách nên cân nhắc
  ----------------------------------- -----------------------------------
  Tìm theo một vài thuộc tính đơn     Derived Query
  giản                                

  Kiểm tra tồn tại hoặc đếm           `existsBy...`, `countBy...`

  Danh sách có phân trang/sắp xếp     Derived Query + `Pageable`/`Sort`

  Nhiều JOIN, `GROUP BY`, truy vấn    `@Query` với JPQL
  cần đọc rõ từng phần                

  Cần cú pháp riêng của database hoặc Native query khi thực sự cần
  SQL đặc thù                         
  -----------------------------------------------------------------------

Không có quy tắc cứng rằng cứ từ ba điều kiện trở lên phải dùng
`@Query`. Hãy chọn cách thể hiện dễ hiểu, dễ kiểm thử và đáp ứng hiệu
năng.

------------------------------------------------------------------------

## 4. Bài tập tự luyện

### Bài 1 -- Quan hệ Entity

Một hệ thống có `Author` và `Book`. Mỗi tác giả viết nhiều sách, mỗi
sách thuộc một tác giả.

1.  Nên dùng annotation nào ở `Author`?
2.  Nên dùng annotation nào ở `Book`?
3.  Khóa ngoại nên nằm ở bảng nào?
4.  `mappedBy` cần trỏ tới thuộc tính nào?

**Gợi ý:** khóa ngoại thường nằm ở bảng `books`, phía `Book` dùng
`@ManyToOne`, còn `Author.books` dùng `@OneToMany(mappedBy = "...")`.

### Bài 2 -- Chọn quan hệ

Một `Student` có thể đăng ký nhiều `Course`; mỗi `Course` có nhiều
`Student`. Hãy tạo mapping `@ManyToMany` hai chiều. Sau đó, nếu cần lưu
ngày đăng ký và điểm số, hãy giải thích vì sao nên tạo Entity trung gian
`Enrollment`.

### Bài 3 -- Cascade

Trong quan hệ `Order`--`OrderItem`, khi xóa một `Order`, các `OrderItem`
của nó cũng phải được xóa. Khi xóa một item khỏi danh sách của order
đang được quản lý, item đó cũng phải bị xóa khỏi database.

-   Nên cân nhắc cấu hình nào?
-   Vì sao không nên áp dụng cấu hình xóa tương tự giữa `Student` và
    `Course`?

### Bài 4 -- Viết Derived Query

Với `Product` có `name`, `category`, `price`, `active`, hãy khai báo các
phương thức:

1.  Tìm sản phẩm đang bán.
2.  Tìm sản phẩm có giá nhỏ hơn hoặc bằng mức cho trước.
3.  Tìm tên có chứa từ khóa, không phân biệt hoa/thường.
4.  Đếm sản phẩm thuộc một danh mục.
5.  Kiểm tra sản phẩm có tên cụ thể đang bán hay không.
6.  Lấy 10 sản phẩm mới nhất theo `createdAt`.
7.  Lấy sản phẩm theo danh mục, phân trang 20 bản ghi mỗi trang và sắp
    xếp giá tăng dần.

### Bài 5 -- Phân trang

Giải thích sự khác nhau giữa `Page<Product>`, `Slice<Product>` và
`List<Product>` khi dùng `Pageable`. Trong trường hợp nào bạn không cần
biết tổng số bản ghi?

------------------------------------------------------------------------

## 5. Checklist ghi nhớ

-   [ ] Phân biệt `@OneToMany` và `@ManyToOne`.
-   [ ] Biết khóa ngoại thường nằm ở phía nhiều.
-   [ ] Phân biệt `@JoinColumn` với `mappedBy`.
-   [ ] Hiểu owning side và inverse side.
-   [ ] Biết khi nào dùng `@OneToOne`, `@ManyToMany`, Entity trung gian.
-   [ ] Phân biệt `CascadeType.REMOVE` với `orphanRemoval`.
-   [ ] Hiểu rủi ro khi dùng `EAGER`, `CascadeType.ALL` và quan hệ hai
    chiều.
-   [ ] Đọc được tên hàm Derived Query và chọn kiểu trả về phù hợp.
-   [ ] Biết dùng `Pageable`, `Page`, `Slice`, `Sort`.
-   [ ] Biết lúc nào Derived Query không còn dễ đọc và nên cân nhắc
    `@Query`.

## 6. Tài liệu tham khảo

1.  Tài liệu Buổi 9 của TeamWeb-Preparations:\
    https://github.com/tngocjm/TeamWeb-Preparations/blob/Buoi9/Preparation-9.md
2.  Spring Data JPA -- JPA Query Methods:\
    https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
3.  Spring Data -- Defining Query Methods:\
    https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html
4.  Jakarta Persistence -- `@OneToMany`:\
    https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/onetomany
5.  Jakarta Persistence -- `@ManyToMany`:\
    https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/manytomany

> Tài liệu được biên soạn lại theo mục tiêu học tập, kèm ví dụ minh họa.
> Một số chi tiết về SQL được sinh ra, cách fetch và thời điểm đồng bộ
> dữ liệu có thể khác tùy JPA provider, cấu hình và hệ quản trị cơ sở dữ
> liệu.

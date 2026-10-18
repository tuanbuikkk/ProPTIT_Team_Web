\# TÀI LIỆU: CÁC PHƯƠNG PHÁP TỐI ƯU JPA TRONG SPRING BOOT



\## 2. Lazy / Eager Initialization



\### 2.1. Khái niệm



Lazy và Eager là hai chiến lược tải dữ liệu (fetching strategies) của

JPA, dùng để quyết định \*\*dữ liệu liên quan đến một entity sẽ được tải

ngay hay chỉ khi cần sử dụng\*\*.



Giả sử hệ thống bán hàng có hai bảng:



\*\*Bảng `orders`\*\*



&#x20;    id    total   customer\_id

&#x20;   101   500000             1

&#x20;   102   300000             2



\*\*Bảng `customers`\*\*



&#x20;   id name

&#x20;    1 Nguyễn An

&#x20;    2 Trần Bình



Một `Order` có thể liên kết với một `Customer`. Khi lấy đơn hàng, ta có

thể chỉ cần mã đơn và tổng tiền, chưa cần tên khách hàng. Lazy và Eager

cho phép kiểm soát việc tải phần dữ liệu liên quan này.



\### 2.2. Lazy initialization --- tải khi cần



\*\*Khái niệm:\*\* `FetchType.LAZY` cho phép trì hoãn việc tải dữ liệu liên

quan cho đến khi ứng dụng thực sự truy cập dữ liệu đó. 



Ví dụ:



``` java

@Entity

public class Order {



&#x20;   @Id

&#x20;   @GeneratedValue(strategy = GenerationType.IDENTITY)

&#x20;   private Long id;



&#x20;   private BigDecimal total;



&#x20;   @ManyToOne(fetch = FetchType.LAZY)

&#x20;   @JoinColumn(name = "customer\_id")

&#x20;   private Customer customer;

}

```



Khi gọi:



``` java

Order order = orderRepository.findById(101L)

&#x20;       .orElseThrow();



System.out.println(order.getTotal());

```



Hibernate có thể chỉ cần truy vấn bảng `orders`:



``` sql

SELECT id, total, customer\_id

FROM orders

WHERE id = 101;

```



Khi tiếp tục gọi:



``` java

System.out.println(order.getCustomer().getName());

```



Hibernate mới có thể truy vấn bảng `customers`:



``` sql

SELECT id, name

FROM customers

WHERE id = 1;

```



\#### Lazy giải quyết bài toán nào?



\-   Không phải tải các đối tượng liên quan mà API không sử dụng.

\-   Giảm lượng dữ liệu đọc từ cơ sở dữ liệu.

\-   Tránh tạo những truy vấn lớn không cần thiết.

\-   Cho phép chọn dữ liệu cần tải ở từng trường hợp sử dụng.



\#### Lazy hoạt động cốt lõi như thế nào?



Hibernate thường dùng proxy hoặc cơ chế tải trì hoãn tương ứng để đại

diện cho dữ liệu quan hệ chưa được tải. Khi truy cập dữ liệu cần thiết,

Hibernate thực hiện truy vấn bổ sung.



Điểm quan trọng: Lazy \*\*không có nghĩa là không truy vấn dữ liệu liên

quan\*\*. Nó chỉ trì hoãn thời điểm truy vấn, nên nếu truy cập quan hệ

trong một vòng lặp, Lazy có thể dẫn đến N+1 query.



\#### Nhược điểm của Lazy



Một lỗi phổ biến là `LazyInitializationException`.



Ví dụ, khi entity đã rời khỏi persistence context mà ứng dụng vẫn truy

cập quan hệ chưa được tải:



``` java

Order order = orderRepository.findById(101L)

&#x20;       .orElseThrow();



// Nếu persistence context đã đóng:

System.out.println(order.getCustomer().getName());

```



Nếu `customer` chưa được tải và không còn persistence context để thực

hiện truy vấn, Hibernate có thể ném ngoại lệ.



Cách xử lý tốt là lấy dữ liệu cần thiết trong phạm vi giao dịch hoặc

truy vấn chủ động quan hệ cần dùng, thay vì mặc định chuyển mọi quan hệ

sang Eager.



\### 2.3. Eager initialization --- tải sớm



\*\*Khái niệm:\*\* `FetchType.EAGER` yêu cầu JPA provider phải bảo đảm dữ

liệu được tải sớm khi entity được truy xuất. 



Ví dụ:



``` java

@ManyToOne(fetch = FetchType.EAGER)

@JoinColumn(name = "customer\_id")

private Customer customer;

```



Khi lấy một đơn hàng, Hibernate phải bảo đảm thông tin `customer` đã

được tải trước khi trả kết quả về cho ứng dụng. Một cách thực hiện có

thể là:



``` sql

SELECT o.id, o.total, o.customer\_id,

&#x20;      c.id, c.name

FROM orders o

LEFT JOIN customers c

&#x20;   ON o.customer\_id = c.id

WHERE o.id = 101;

```





\#### Eager giải quyết bài toán nào?



Eager phù hợp khi dữ liệu liên quan gần như luôn cần ngay trong trường

hợp sử dụng cụ thể.



Tuy nhiên, nếu cấu hình Eager rộng rãi trên entity, ứng dụng có thể phải

tải những dữ liệu không cần thiết, tăng chi phí truy vấn và phát sinh

N+1 trong một số tình huống. Hibernate khuyến nghị ưu tiên quan hệ Lazy

và chủ động xác định dữ liệu cần tải theo từng truy vấn.



\### 2.4. So sánh Lazy và Eager



| Tiêu chí | Lazy | Eager |

| :--- | :--- | :--- |

| \*\*Thời điểm tải\*\* | Khi cần truy cập, tùy provider | Phải bảo đảm được tải sớm |

| \*\*Dữ liệu không sử dụng\*\* | Thường tránh tải | Vẫn có thể phải tải |

| \*\*Số câu SQL\*\* | Có thể tăng nếu truy cập nhiều quan hệ | Có thể dùng JOIN hoặc truy vấn bổ sung |

| \*\*Rủi ro điển hình\*\* | `LazyInitializationException`, N+1 | Tải thừa dữ liệu, N+1 do truy vấn bổ sung |

| \*\*Kiểm soát theo API\*\* | Linh hoạt khi kết hợp query/fetch plan | Kém linh hoạt nếu cấu hình Eager cố định |



\### 2.5. Cú pháp và mặc định quan trọng



Theo Jakarta Persistence, mặc định thường gặp là:



\-   `@OneToMany`: LAZY.

\-   `@ManyToMany`: LAZY.

\-   `@ManyToOne`: EAGER.

\-   `@OneToOne`: EAGER.



\------------------------------------------------------------------------



\## 3. Pagination 



\### 3.1. Khái niệm



\*\*Pagination là kỹ thuật chia tập dữ liệu thành nhiều phần nhỏ và chỉ

truy xuất một phần dữ liệu trong mỗi lần yêu cầu.\*\*



Giả sử bảng `products` có 100.000 sản phẩm. Nếu API gọi:



``` java

List<Product> products = productRepository.findAll();

```



ứng dụng có thể phải đọc, ánh xạ và giữ một lượng lớn đối tượng trong bộ

nhớ, trong khi giao diện chỉ hiển thị 20 sản phẩm.



Thay vì lấy toàn bộ dữ liệu, ta phân trang để mỗi request chỉ lấy một

nhóm sản phẩm.



Spring Data JPA cung cấp `Pageable`, `Page`, `Slice` và các cơ chế giới

hạn kết quả để hỗ trợ việc này.



\### 3.2. Pagination hoạt động như thế nào?



Ví dụ có 100 sản phẩm, mỗi trang chứa 10 sản phẩm. Với phân trang offset

thông thường, cơ sở dữ liệu sử dụng giới hạn số bản ghi và vị trí bắt

đầu. Cú pháp SQL cụ thể có thể khác giữa các hệ quản trị cơ sở dữ liệu.



SQL minh họa cho trang thứ hai:



``` sql

SELECT \* FROM products

ORDER BY id

LIMIT 10 OFFSET 10;

```



Lệnh này lấy 10 bản ghi, bỏ qua 10 bản ghi đầu tiên.



\### 3.3. Cú pháp Pagination trong Spring Data JPA



\#### Bước 1: Repository



``` java

public interface ProductRepository

&#x20;       extends JpaRepository<Product, Long> {

}

```



`JpaRepository` đã cung cấp các phương thức hỗ trợ phân trang, nên không

cần tự viết SQL cho trường hợp cơ bản.



\#### Bước 2: Service



``` java

@Service

public class ProductService {



&#x20;   private final ProductRepository productRepository;



&#x20;   public ProductService(ProductRepository productRepository) {

&#x20;       this.productRepository = productRepository;

&#x20;   }



&#x20;   public Page<Product> getProducts(int page, int size) {

&#x20;       Pageable pageable = PageRequest.of(

&#x20;           page,

&#x20;           size,

&#x20;           Sort.by("id").ascending()

&#x20;       );



&#x20;       return productRepository.findAll(pageable);

&#x20;   }

}

```



Giải thích:



\-   `PageRequest.of(page, size)`: tạo yêu cầu phân trang.

\-   `page`: chỉ số trang, bắt đầu từ `0`.

\-   `size`: số bản ghi tối đa trên một trang.

\-   `Sort`: quy định thứ tự sắp xếp.

\-   `findAll(pageable)`: truy vấn dữ liệu theo yêu cầu phân trang.



\#### Bước 3: Controller



``` java

@RestController

@RequestMapping("/api/products")

public class ProductController {



&#x20;   private final ProductService productService;



&#x20;   public ProductController(ProductService productService) {

&#x20;       this.productService = productService;

&#x20;   }



&#x20;   @GetMapping

&#x20;   public Page<Product> getProducts(

&#x20;           @RequestParam(defaultValue = "0") int page,

&#x20;           @RequestParam(defaultValue = "10") int size) {



&#x20;       return productService.getProducts(page, size);

&#x20;   }

}

```



Ví dụ request:



``` http

GET /api/products?page=0\&size=10

GET /api/products?page=1\&size=10

```



Phản hồi kiểu `Page` cung cấp danh sách dữ liệu và thông tin phân trang

như tổng số bản ghi, tổng số trang, số trang hiện tại và trạng thái

trang đầu/cuối.



\### 3.4. Phân biệt `Page`, `Slice` và `List`



| Kiểu trả về | Đặc điểm | Khi nào nên dùng |

| :--- | :--- | :--- |

| `Page<T>` | Có metadata tổng số bản ghi và số trang; có thể cần truy vấn `COUNT` | Giao diện cần hiển thị tổng số trang |

| `Slice<T>` | Biết có phần dữ liệu tiếp theo hay không; không cần tổng số bản ghi | Nút "Xem thêm", cuộn danh sách |

| `List<T>` kết hợp `Pageable` | Trả về danh sách giới hạn; không cung cấp metadata của `Page` | Chỉ cần lấy một phần dữ liệu |



Spring Data JPA lưu ý rằng truy vấn đếm tổng số bản ghi có thể tốn kém.

Ngoài ra, offset-based pagination có thể chậm khi offset quá lớn.



\-   Trang quản lý sản phẩm cần biết "trang 1/200": dùng `Page`.

\-   Trang tin tức có nút "Xem thêm": dùng `Slice`.

\-   Tác vụ chỉ cần lấy 20 bản ghi mới nhất: có thể dùng `List` với

&#x20;   `Pageable` hoặc `findTop20...`.



\### 3.5. Pagination giải quyết vấn đề gì trong web Spring Boot?



1\.  Giảm lượng dữ liệu trả về trong một HTTP response.

2\.  Giảm số đối tượng cần ánh xạ và xử lý ở mỗi request.

3\.  Hạn chế áp lực bộ nhớ khi xử lý danh sách lớn.

4\.  Hỗ trợ giao diện phân trang, tìm kiếm và sắp xếp.

5\.  Giúp giới hạn khối lượng dữ liệu truyền qua mạng.



\*\*Giới hạn quan trọng:\*\* Pagination không tự làm cho câu truy vấn nhanh

trong mọi trường hợp. Nếu dùng offset rất lớn hoặc bộ lọc không có index

phù hợp, cơ sở dữ liệu vẫn có thể phải làm nhiều việc.



Với dữ liệu rất lớn và cần duyệt tuần tự, có thể cân nhắc keyset

pagination (phân trang theo khóa) thay vì offset. Keyset dùng giá trị

của bản ghi cuối cùng làm mốc cho lần truy vấn tiếp theo, nhưng đòi hỏi

điều kiện sắp xếp phù hợp và index tốt.



Ví dụ SQL minh họa:



``` sql

SELECT \*

FROM products

WHERE id > 1000

ORDER BY id

LIMIT 20;

```



Cách này lấy 20 sản phẩm có `id` lớn hơn 1000, thay vì bỏ qua 1000 bản

ghi đầu tiên.



\------------------------------------------------------------------------



\## 4. Caching 



\### 4.1. Khái niệm



\*\*Caching là kỹ thuật lưu tạm dữ liệu đã được truy xuất hoặc tính toán

để những lần truy cập tiếp theo có thể sử dụng lại, thay vì luôn phải

thực hiện lại công việc từ đầu.\*\*



Ví dụ, trang sản phẩm thường xuyên được truy cập. Nếu thông tin sản phẩm

ít thay đổi, việc đọc lại cùng một dữ liệu từ database trong mỗi request

có thể gây lãng phí tài nguyên.



Cache cho phép tận dụng kết quả đã có để giảm số lần truy vấn database

và cải thiện thời gian phản hồi.



\### 4.2. Các tầng cache cần phân biệt



\#### 1. First-level cache



\-   \*\*Phạm vi:\*\* persistence context / EntityManager.

\-   Mặc định được bật.

\-   Quản lý entity trong cùng persistence context, giúp tránh nạp lại

&#x20;   cùng một entity theo cách thông thường trong phạm vi đó.



\#### 2. Second-level cache



\-   \*\*Phạm vi:\*\* giữa các persistence context, tùy cấu hình.

\-   Có thể tái sử dụng dữ liệu entity hoặc collection qua nhiều giao

&#x20;   dịch và yêu cầu khác nhau trong ứng dụng.



\#### 3. Query cache của Hibernate



\-   \*\*Phạm vi:\*\* kết quả truy vấn đã được cache.

\-   Dùng cho những truy vấn phù hợp và cần được cấu hình rõ ràng.

\-   Không đồng nghĩa với việc toàn bộ entity luôn được cache.



\#### 4. Spring Cache



\-   \*\*Phạm vi:\*\* kết quả của phương thức ứng dụng.

\-   Cache dữ liệu trả về từ service hoặc phương thức khác, với các triển

&#x20;   khai như bộ nhớ cục bộ hoặc Redis.



Hibernate xác nhận first-level cache nằm trong persistence context và

được bật mặc định. Second-level cache có thể chia sẻ dữ liệu ngoài phạm

vi một persistence context; query cache là một cơ chế riêng.



\### 4.3. First-level cache



Ví dụ:



``` java

@Transactional

public void testCache() {

&#x20;   Product p1 = entityManager.find(Product.class, 1L);

&#x20;   Product p2 = entityManager.find(Product.class, 1L);

}

```



Trong cùng persistence context, nếu entity có ID `1` đã được quản lý,

lần gọi thứ hai thông thường trả về entity đã có mà không cần thực hiện

thêm truy vấn SELECT để tải lại entity đó.



Cốt lõi của cơ chế này là persistence context duy trì danh tính và trạng

thái của entity được quản lý.



\*\*Không nên nhầm:\*\* First-level cache không phải bộ nhớ cache dùng chung

giữa tất cả request. Khi persistence context kết thúc, phạm vi này cũng

kết thúc.



\### 4.4. Second-level cache của Hibernate



Second-level cache hữu ích khi nhiều persistence context cần đọc lại

cùng dữ liệu, chẳng hạn danh mục sản phẩm, loại sản phẩm hoặc dữ liệu

tham chiếu ít thay đổi.



Một cấu hình minh họa với Hibernate và một cache provider tương thích:



``` java

@Entity

@Cacheable

@org.hibernate.annotations.Cache(

&#x20;   usage = org.hibernate.annotations.CacheConcurrencyStrategy.READ\_WRITE

)

public class Category {



&#x20;   @Id

&#x20;   private Long id;



&#x20;   private String name;

}

```



Để ví dụ hoạt động, cần cấu hình second-level cache và cài đặt cache

provider tương thích với phiên bản Hibernate đang sử dụng. Chỉ thêm

annotation không có nghĩa là đã hoàn tất toàn bộ cấu hình.



`READ\_WRITE` là chiến lược đồng bộ cache dành cho những trường hợp cần

kiểm soát việc đọc/ghi; không phải lựa chọn tối ưu cho mọi entity.



\#### Cơ chế hoạt động



1\.  Ứng dụng tìm entity.

2\.  Hibernate kiểm tra persistence context.

3\.  Nếu không có, Hibernate có thể kiểm tra second-level cache nếu

&#x20;   entity được cấu hình cache.

4\.  Nếu cache miss, Hibernate truy vấn database.

5\.  Dữ liệu phù hợp có thể được đưa vào cache để sử dụng lại.



\#### Khi nào nên sử dụng?



\-   Dữ liệu được đọc thường xuyên.

\-   Tần suất cập nhật thấp hoặc có chiến lược đồng bộ phù hợp.

\-   Nhiều request cần dùng chung một tập dữ liệu.

\-   Chi phí truy vấn database đáng kể so với chi phí cache.



Không nên mặc định cache mọi entity, nhất là dữ liệu thay đổi liên tục.



\### 4.5. Spring Cache với `@Cacheable`



Spring Cache thường dễ áp dụng ở tầng Service hơn khi muốn cache kết quả

của một phương thức.



Ví dụ:



``` java

@Configuration

@EnableCaching

public class CacheConfig {

}

```



Service:



``` java

@Service

public class ProductService {



&#x20;   private final ProductRepository productRepository;



&#x20;   public ProductService(ProductRepository productRepository) {

&#x20;       this.productRepository = productRepository;

&#x20;   }



&#x20;   @Cacheable(value = "products", key = "#id")

&#x20;   public Product getProduct(Long id) {

&#x20;       return productRepository.findById(id)

&#x20;               .orElseThrow();

&#x20;   }



&#x20;   @CacheEvict(value = "products", key = "#id")

&#x20;   public void deleteProduct(Long id) {

&#x20;       productRepository.deleteById(id);

&#x20;   }

}

```



Để chạy thực tế, cần thêm dependency và cấu hình cache provider thích

hợp. Spring Cache hỗ trợ nhiều cách lưu cache, trong đó có cache cục bộ

và Redis.



\#### `@Cacheable` hoạt động như thế nào?



Khi gọi `getProduct(1L)`:



\-   Cache có khóa `1`: Spring có thể trả về giá trị cache mà không gọi

&#x20;   phần thân phương thức.

\-   Cache không có khóa `1`: phương thức chạy, lấy dữ liệu từ repository

&#x20;   và kết quả được đưa vào cache theo cấu hình.



`@CacheEvict` loại bỏ dữ liệu cache tương ứng sau thao tác xóa. Với cập

nhật, cần sử dụng cơ chế cập nhật hoặc xóa cache phù hợp để tránh dữ

liệu cũ.



\*\*Lưu ý:\*\* Spring Cache là cache ở tầng ứng dụng, không đồng nhất với

first-level hoặc second-level cache của Hibernate. Cache một entity ở

tầng service cũng không tự động bảo đảm mọi dữ liệu liên quan đã được

cache.



\### 4.6. So sánh các cơ chế cache



| Tiêu chí | First-level | Second-level | Spring Cache |

| :--- | :--- | :--- | :--- |

| \*\*Phạm vi\*\* | Một persistence context | Nhiều persistence context | Theo cấu hình và cache provider |

| \*\*Cấu hình\*\* | Mặc định có | Cần cấu hình | Cần bật và chọn provider |

| \*\*Đối tượng lưu\*\* | Entity được quản lý | Entity/collection được chọn | Kết quả phương thức |

| \*\*Dùng chung qua request\*\* | Không mặc định | Có thể | Có thể |

| \*\*Kiểm soát nghiệp vụ\*\* | Thấp | Theo cấu hình Hibernate | Linh hoạt theo service |



\------------------------------------------------------------------------



\## 5. Manual Query / Native Query



\### 5.1. Khái niệm



Spring Data JPA cung cấp nhiều cách tạo truy vấn:



1\.  Derived query: Spring Data suy ra truy vấn từ tên phương thức.

2\.  Manual query: lập trình viên tự định nghĩa JPQL hoặc SQL.

3\.  Native query: viết SQL trực tiếp theo cú pháp của hệ quản trị cơ sở

&#x20;   dữ liệu.



Tài liệu Spring Data JPA xác nhận có thể định nghĩa truy vấn qua tên

phương thức, `@Query` hoặc truy vấn native.



\#### Ví dụ derived query



``` java

List<Product> findByPriceGreaterThan(

&#x20;   BigDecimal price

);

```



Spring Data JPA suy ra truy vấn dựa trên tên phương thức. Cách này ngắn

gọn, phù hợp với điều kiện đơn giản.



\### 5.2. Manual query với JPQL



JPQL (Jakarta Persistence Query Language) truy vấn dựa trên \*\*entity và

thuộc tính Java\*\*, thay vì trực tiếp dựa trên tên bảng và cột vật lý.



Ví dụ:



``` java

public interface ProductRepository

&#x20;       extends JpaRepository<Product, Long> {



&#x20;   @Query("""

&#x20;       SELECT p

&#x20;       FROM Product p

&#x20;       WHERE p.price >= :minPrice

&#x20;       ORDER BY p.price ASC

&#x20;       """)

&#x20;   List<Product> findProductsByMinPrice(

&#x20;       @Param("minPrice") BigDecimal minPrice

&#x20;   );

}

```



Trong JPQL, `Product` là tên entity, còn `p.price` là thuộc tính Java.

Hibernate sẽ chuyển JPQL thành SQL phù hợp với database đang dùng.



\#### Khi nào nên dùng JPQL thủ công?



\-   Điều kiện truy vấn phức tạp.

\-   Có `JOIN`, `GROUP BY`, `HAVING` hoặc truy vấn tổng hợp.

\-   Muốn kiểm soát câu truy vấn nhưng vẫn làm việc với entity.

\-   Tên phương thức repository trở nên quá dài hoặc khó đọc.



\### 5.3. Native query --- SQL thuần



Native query cho phép sử dụng SQL theo hệ quản trị cơ sở dữ liệu, hữu

ích khi cần tính năng đặc thù hoặc muốn kiểm soát truy vấn sát với cấu

trúc database.



``` java

public interface ProductRepository

&#x20;       extends JpaRepository<Product, Long> {



&#x20;   @Query(value = """

&#x20;       SELECT \*

&#x20;       FROM products

&#x20;       WHERE price >= :minPrice

&#x20;       ORDER BY price ASC

&#x20;       """, nativeQuery = true)

&#x20;   List<Product> findProductsNative(

&#x20;       @Param("minPrice") BigDecimal minPrice

&#x20;   );

}

```



Đoạn SQL trên sử dụng tên bảng `products` và cột `price`, không sử dụng

tên entity như JPQL.



Với Spring Data JPA phiên bản hỗ trợ `@NativeQuery`, có thể dùng

annotation đó thay cho `@Query(nativeQuery = true)`. Cần kiểm tra phiên

bản dự án trước khi chọn cú pháp.



\### 5.4. So sánh JPQL và Native SQL



| Tiêu chí | JPQL | Native SQL |

| :--- | :--- | :--- |

| \*\*Truy vấn dựa trên\*\* | Entity và thuộc tính | Bảng và cột database |

| \*\*Khả năng độc lập database\*\* | Cao hơn | Thấp hơn |

| \*\*Tận dụng tính năng SQL riêng\*\* | Hạn chế hơn | Linh hoạt hơn |

| \*\*Ánh xạ kết quả\*\* | Thường thuận tiện với entity | Có thể cần mapping hoặc projection |

| \*\*Bảo trì khi đổi schema\*\* | Tương đối thuận tiện nếu mapping được quản lý tốt | Có thể phải sửa SQL |

| \*\*Kiểm soát SQL đặc thù\*\* | Gián tiếp qua provider | Trực tiếp hơn |



\### 5.5. Native query có luôn nhanh hơn JPQL không?



\*\*Không.\*\* JPQL được Hibernate chuyển thành SQL, và hiệu suất cuối cùng

phụ thuộc vào SQL được tạo ra, kế hoạch thực thi, index, dữ liệu và

database.



Native query giúp kiểm soát câu SQL, nhưng bản thân việc viết SQL trực

tiếp không bảo đảm truy vấn nhanh hơn.



Ví dụ, nếu cả JPQL và native query đều tạo ra cùng một câu SQL với cùng

kế hoạch thực thi, hiệu năng có thể tương đương.



Native query hữu ích khi cần:



\-   Dùng hàm hoặc cú pháp đặc thù của database.

\-   Viết truy vấn tổng hợp phức tạp.

\-   Tận dụng CTE, window function hoặc tính năng SQL mà JPQL không biểu

&#x20;   đạt thuận tiện.

\-   Phân tích và kiểm soát SQL cụ thể.



\### 5.6. Những lưu ý quan trọng



\#### Luôn dùng tham số cho dữ liệu đầu vào



Nên viết:



``` java

@Query("""

&#x20;   SELECT p FROM Product p

&#x20;   WHERE p.name = :name

&#x20;   """)

List<Product> findByName(@Param("name") String name);

```



Không nên nối chuỗi đầu vào trực tiếp vào câu SQL hoặc JPQL. Việc nối

chuỗi thiếu kiểm soát có thể gây SQL injection.



\#### Pagination với native query



Native query vẫn có thể phân trang. Tuy nhiên, với truy vấn phức tạp,

cần cung cấp `countQuery` phù hợp nếu Spring Data JPA không thể suy ra

truy vấn đếm chính xác.



``` java

@Query(

&#x20;   value = """

&#x20;       SELECT \*

&#x20;       FROM products

&#x20;       WHERE price >= :minPrice

&#x20;       """,

&#x20;   countQuery = """

&#x20;       SELECT COUNT(\*)

&#x20;       FROM products

&#x20;       WHERE price >= :minPrice

&#x20;       """,

&#x20;   nativeQuery = true

)

Page<Product> findByMinPrice(

&#x20;   @Param("minPrice") BigDecimal minPrice,

&#x20;   Pageable pageable

);

```



Ở ví dụ này, truy vấn dữ liệu lấy một phần sản phẩm, còn `countQuery`

đếm tổng số sản phẩm thỏa mãn điều kiện.



\------------------------------------------------------------------------



\## 6. Dynamic Query

\### 6.1. Khái niệm



\*\*Dynamic Query là cách xây dựng truy vấn dựa trên các điều kiện được

cung cấp tại thời điểm chạy chương trình.\*\*



Phương pháp này đặc biệt hữu ích với các API tìm kiếm có nhiều bộ lọc

tùy chọn.



Ví dụ, website bán hàng cho phép người dùng tìm sản phẩm theo:



\-   Tên sản phẩm.

\-   Giá thấp nhất và giá cao nhất.

\-   Danh mục.

\-   Trạng thái còn hàng.

\-   Thứ tự sắp xếp.



Người dùng không bắt buộc phải nhập tất cả các trường. Vì vậy, câu truy

vấn cần thay đổi theo những điều kiện thực sự được cung cấp.



Spring Data JPA hỗ trợ nhiều cách xây dựng truy vấn động, trong đó có

`Specification` dựa trên JPA Criteria API.



\### 6.2. Bài toán nếu không dùng Dynamic Query



Giả sử repository có các phương thức:



``` java

findByName(String name);



findByPriceBetween(

&#x20;   BigDecimal min,

&#x20;   BigDecimal max

);



findByNameAndPriceBetween(

&#x20;   String name,

&#x20;   BigDecimal min,

&#x20;   BigDecimal max

);



findByNameAndCategoryId(

&#x20;   String name,

&#x20;   Long categoryId

);

```



Khi bổ sung thêm nhiều điều kiện, số tổ hợp có thể tăng rất nhanh. Việc

tạo một phương thức riêng cho mỗi tổ hợp khiến repository khó bảo trì.



Dynamic Query cho phép tạo điều kiện dựa trên dữ liệu đầu vào, thay vì

phải định nghĩa tất cả các tổ hợp từ trước.



\### 6.5. Dynamic Query khác gì Manual Query?



| Tiêu chí | Manual Query cố định | Dynamic Query |

| :--- | :--- | :--- |

| \*\*Điều kiện\*\* | Thường được xác định trước | Thay đổi theo đầu vào |

| \*\*Cách triển khai\*\* | JPQL/SQL hoặc tên phương thức | Specification, Criteria, Querydsl hoặc cách khác |

| \*\*Phù hợp\*\* | Truy vấn có yêu cầu ổn định | Tìm kiếm nhiều bộ lọc tùy chọn |

| \*\*Khả năng tái sử dụng\*\* | Phụ thuộc cách tổ chức | Có thể tái sử dụng từng điều kiện |

| \*\*Độ phức tạp\*\* | Đơn giản khi ít điều kiện | Có thể phức tạp khi nhiều điều kiện và quan hệ |



\*\*Lưu ý:\*\* Dynamic Query không mặc nhiên nhanh hơn truy vấn cố định. Lợi

ích chính là tính linh hoạt và khả năng bảo trì. Hiệu năng vẫn phụ thuộc

vào SQL sinh ra, index, dữ liệu và kế hoạch thực thi.



\------------------------------------------------------------------------



\## 7. Phòng tránh N+1 Query



Đây là một trong những vấn đề quan trọng nhất khi tối ưu JPA/Hibernate.



\### 7.1. N+1 Query là gì?



\*\*N+1 Query là tình trạng ứng dụng thực hiện một truy vấn để lấy danh

sách đối tượng chính, sau đó phát sinh thêm N truy vấn để lấy dữ liệu

liên quan cho từng đối tượng.\*\*



Ví dụ hệ thống có:



\-   100 đơn hàng.

\-   Mỗi đơn hàng liên kết với một khách hàng.

\-   API cần hiển thị mã đơn và tên khách hàng.



Code:



``` java

List<Order> orders = orderRepository.findAll();



for (Order order : orders) {

&#x20;   System.out.println(

&#x20;       order.getCustomer().getName()

&#x20;   );

}

```



Giả sử `customer` chưa được tải, Hibernate có thể thực hiện:



``` sql

\-- Truy vấn 1: lấy danh sách đơn hàng

SELECT \*

FROM orders;

```



Sau đó:



``` sql

\-- Truy vấn 2: lấy khách hàng của đơn 1

SELECT \*

FROM customers

WHERE id = 1;



\-- Truy vấn 3: lấy khách hàng của đơn 2

SELECT \*

FROM customers

WHERE id = 2;



\-- ...

```



Nếu 100 đơn hàng cần tải 100 khách hàng khác nhau, tổng số truy vấn có

thể là 101.



\### 7.2. Tại sao N+1 làm ứng dụng chậm?



Mỗi truy vấn thường có chi phí cố định, chẳng hạn:



\-   Gửi yêu cầu từ ứng dụng đến database.

\-   Xử lý truy vấn và lấy kết quả.

\-   Truyền dữ liệu qua kết nối.

\-   Ánh xạ dữ liệu thành đối tượng Java.



Khi số truy vấn tăng từ 1 lên 101, chi phí của các lần trao đổi với

database có thể chiếm phần đáng kể trong tổng thời gian phản hồi.



N+1 không nhất thiết có nghĩa là database xử lý tổng lượng dữ liệu lớn

hơn nhiều; vấn đề thường nằm ở việc chia một công việc thành quá nhiều

truy vấn riêng lẻ.



\### 7.3. Cách 1 --- dùng `JOIN FETCH`



Đây là giải pháp phổ biến khi API biết trước cần dữ liệu quan hệ nào.



\#### Repository



``` java

public interface OrderRepository

&#x20;       extends JpaRepository<Order, Long> {



&#x20;   @Query("""

&#x20;       SELECT o

&#x20;       FROM Order o

&#x20;       JOIN FETCH o.customer

&#x20;       """)

&#x20;   List<Order> findAllWithCustomer();

}

```



\#### Service



``` java

List<Order> orders =

&#x20;       orderRepository.findAllWithCustomer();



for (Order order : orders) {

&#x20;   System.out.println(

&#x20;       order.getCustomer().getName()

&#x20;   );

}

```



`JOIN FETCH` yêu cầu truy vấn lấy đơn hàng đồng thời nạp quan hệ

`customer`.



SQL minh họa:



``` sql

SELECT o.\*, c.\*

FROM orders o

JOIN customers c

&#x20;   ON o.customer\_id = c.id;

```



Khi quan hệ đã được tải, việc gọi `getCustomer().getName()` không cần

truy vấn riêng để tải khách hàng đó.



\#### `JOIN FETCH` khác `JOIN` thông thường như thế nào?



\-   `JOIN` thông thường dùng để kết hợp bảng và có thể lọc theo dữ liệu

&#x20;   quan hệ, nhưng không mặc nhiên yêu cầu quan hệ được tải vào entity.

\-   `JOIN FETCH` vừa kết hợp bảng vừa chỉ định rằng quan hệ cần được nạp

&#x20;   cùng với entity gốc.



Đây là khác biệt quan trọng khi làm việc với entity và quan hệ Lazy.



\### 7.4. Cách 2 --- dùng `@EntityGraph`



`EntityGraph` cho phép chỉ định các thuộc tính hoặc quan hệ cần tải cho

một truy vấn cụ thể mà không phải luôn viết `JOIN FETCH` trong JPQL.



Ví dụ:



``` java

public interface OrderRepository

&#x20;       extends JpaRepository<Order, Long> {



&#x20;   @EntityGraph(attributePaths = {"customer"})

&#x20;   List<Order> findAll();

}

```



Trong ví dụ này, truy vấn được yêu cầu nạp thêm quan hệ `customer`.



Lợi ích:



\-   Có thể giữ nguyên cấu trúc phương thức repository.

\-   Xác định quan hệ cần tải theo từng trường hợp sử dụng.

\-   Tránh phải cấu hình tất cả quan hệ thành Eager.



Tuy nhiên, `EntityGraph` không bảo đảm rằng provider sẽ luôn tạo ra đúng

một câu SQL. Cần kiểm tra SQL thực tế nếu đang tối ưu hiệu năng.



\### 7.5. Cách 3 --- dùng Batch Fetching



Batch fetching là cách Hibernate tải nhiều đối tượng hoặc quan hệ trong

một truy vấn theo nhóm, thường sử dụng điều kiện `IN`.



Ví dụ, thay vì:



``` sql

SELECT \* FROM customers WHERE id = 1;

SELECT \* FROM customers WHERE id = 2;

SELECT \* FROM customers WHERE id = 3;

```



Hibernate có thể tải một nhóm:



``` sql

SELECT \*

FROM customers

WHERE id IN (1, 2, 3);

```



Có thể cấu hình bằng annotation Hibernate:



``` java

@OneToMany(mappedBy = "customer")

@org.hibernate.annotations.BatchSize(size = 20)

private List<Order> orders;

```



Hoặc cấu hình batch fetch toàn cục trong `application.properties`:



``` properties

spring.jpa.properties.hibernate.default\_batch\_fetch\_size=20

```



Batch fetching giúp giảm số truy vấn khi tải các quan hệ theo từng nhóm.

Tuy nhiên, nó thường không loại bỏ hoàn toàn nguyên nhân N+1 như một

truy vấn fetch join phù hợp. Batch fetching là phương án giảm thiểu số

lần truy vấn, không phải giải pháp triệt để trong mọi tình huống.



\### 7.6. So sánh các cách phòng tránh N+1



| Giải pháp | Cốt lõi | Phù hợp khi |

| :--- | :--- | :--- |

| `JOIN FETCH` | Kết hợp truy vấn chính và tải quan hệ | Biết rõ quan hệ cần dùng |

| `@EntityGraph` | Khai báo kế hoạch tải theo truy vấn | Muốn tách kế hoạch tải khỏi JPQL |

| Batch fetching | Gom nhiều lần tải riêng thành các nhóm | Không muốn JOIN tạo kết quả quá lớn |

| Second-level cache | Tái sử dụng dữ liệu liên quan đã cache | Dữ liệu được đọc lặp lại và phù hợp để cache |





\## 8. Validation



\### 8.1. Khái niệm



\*\*Validation (kiểm tra tính hợp lệ của dữ liệu)\*\* là bước xác minh dữ liệu có thỏa mãn các quy tắc nghiệp vụ/định dạng trước khi xử lý hoặc lưu vào cơ sở dữ liệu. Ví dụ: email phải đúng định dạng, tuổi phải ≥ 18, mật khẩu tối thiểu 8 ký tự, số lượng sản phẩm phải dương.



Trong thế giới Java, chuẩn để làm việc này là \*\*Bean Validation\*\*:



\- \*\*Bean Validation\*\* là một \*đặc tả\* (specification) .Nó định nghĩa các annotation như `@NotNull`, `@Size`, `@Email`... và API để kiểm tra.

\- \*\*Hibernate Validator\*\* là \*bản cài đặt tham chiếu\* (reference implementation) phổ biến nhất của đặc tả này. (Tên "Hibernate" dễ gây nhầm: nó \*\*không\*\* phụ thuộc vào Hibernate ORM, chỉ cùng một nhóm phát triển.)

\- \*\*Spring Boot\*\* tích hợp sẵn thông qua `spring-boot-starter-validation`, tự cấu hình một `Validator` (`LocalValidatorFactoryBean`) để Spring MVC và Spring Data dùng.



Ý tưởng cốt lõi: \*\*khai báo ràng buộc (constraint) bằng annotation ngay trên field của class\*\*, thay vì viết `if/else` rải rác khắp nơi.



\### 8.2. Giải quyết bài toán nào trong web Spring Boot?



| Bài toán | Cách Validation giúp |

|---|---|

| Client gửi JSON thiếu field / sai kiểu / sai định dạng | Chặn ngay ở Controller, trả `400 Bad Request` |

| Code đầy `if (name == null \\|\\| name.isEmpty()) ...` | Thay bằng annotation khai báo, code sạch, dễ đọc |

| Quy tắc lặp lại ở nhiều API (cùng một DTO) | Khai báo một lần trên DTO, dùng lại mọi nơi |

| Cần thông báo lỗi chi tiết theo từng field | Mỗi `ConstraintViolation` cho biết field nào, giá trị gì, lỗi gì |

| Dữ liệu xấu lọt vào DB | Có thể kiểm tra thêm ở Service, hoặc Hibernate tự validate trước khi persist |

| Bảo mật cơ bản (độ phức tạp mật khẩu, giới hạn độ dài chống payload quá lớn) | Ràng buộc `@Size`, `@Pattern`... |



\### 8.3. Cách hoạt động (cốt lõi)



\*\*Bước 1 – Khai báo ràng buộc\*\* trên field của DTO/Entity.



\*\*Bước 2 – Kích hoạt kiểm tra\*\* bằng `@Valid` hoặc `@Validated` tại điểm cần kiểm tra.



\*\*Bước 3 – Spring gọi `Validator`\*\*, nếu có vi phạm thì ném exception.



Có hai cơ chế khác nhau tùy vị trí đặt annotation:



\- Validate `@RequestBody` 





\- Validate tham số đơn giản (`@RequestParam`, `@PathVariable`) và tham số của method trong Service\*\*







\### 8.4. Cài đặt



Từ Spring Boot 2.3, `spring-boot-starter-web` \*\*không còn\*\* kéo theo starter validation, nên cần thêm thủ công:



\*\*Maven\*\*

```xml

<dependency>

&#x20;   <groupId>org.springframework.boot</groupId>

&#x20;   <artifactId>spring-boot-starter-validation</artifactId>

</dependency>

```



\*\*Gradle\*\*

```groovy

implementation 'org.springframework.boot:spring-boot-starter-validation'

```



Starter này kéo theo `hibernate-validator` và Jakarta Validation API.



\### 8.5. Các annotation ràng buộc thường dùng



| Annotation | Ý nghĩa | Áp dụng cho |

|---|---|---|

| `@NotNull` | Không được `null` | Mọi kiểu |

| `@NotEmpty` | Không `null` và không rỗng (size > 0) | `String`, `Collection`, `Map`, mảng |

| `@NotBlank` | Không `null`, và sau khi `trim` phải có ký tự | `String` |

| `@Size(min, max)` | Độ dài chuỗi / kích thước collection | `String`, `Collection`... |

| `@Min`, `@Max` | Giá trị tối thiểu / tối đa | Số nguyên, `BigDecimal` |

| `@Positive`, `@PositiveOrZero`, `@Negative` | Dương / không âm / âm | Số |

| `@DecimalMin`, `@DecimalMax` | Giới hạn số thập phân (dạng chuỗi) | Số |

| `@Digits(integer, fraction)` | Số chữ số phần nguyên / phần thập phân | Số |

| `@Email` | Đúng định dạng email | `String` |

| `@Pattern(regexp)` | Khớp regex | `String` |

| `@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent` | Ràng buộc về thời gian | `LocalDate`, `LocalDateTime`... |

| `@AssertTrue`, `@AssertFalse` | Biểu thức boolean phải đúng/sai | `boolean` |



> \*\*Lưu ý hay nhầm\*\*: `@NotNull` chỉ kiểm tra `null`; `""` vẫn hợp lệ. Với chuỗi nhập từ người dùng, thường dùng `@NotBlank`.



\### 8.6. Ví dụ cơ bản: DTO + Controller



\*\*DTO (đặt ràng buộc trên field)\*\*

```java

public class CreateUserRequest {



&#x20;   @NotBlank(message = "Tên không được để trống")

&#x20;   @Size(max = 50, message = "Tên tối đa 50 ký tự")

&#x20;   private String name;



&#x20;   @NotBlank(message = "Email không được để trống")

&#x20;   @Email(message = "Email không đúng định dạng")

&#x20;   private String email;



&#x20;   @NotNull(message = "Tuổi không được để trống")

&#x20;   @Min(value = 18, message = "Phải từ 18 tuổi trở lên")

&#x20;   private Integer age;



&#x20;   @NotBlank

&#x20;   @Pattern(regexp = "^(?=.\*\[A-Za-z])(?=.\*\\\\d).{8,}$",

&#x20;            message = "Mật khẩu tối thiểu 8 ký tự, gồm chữ và số")

&#x20;   private String password;



&#x20;   // getter / setter (hoặc dùng record / Lombok @Data)

}

```



Có thể dùng `record` :

```java

public record CreateUserRequest(

&#x20;       @NotBlank String name,

&#x20;       @Email @NotBlank String email,

&#x20;       @NotNull @Min(18) Integer age) {}

```



\*\*Controller (kích hoạt kiểm tra bằng `@Valid`)\*\*

```java

@RestController

@RequestMapping("/api/users")

public class UserController {



&#x20;   @PostMapping

&#x20;   public ResponseEntity<String> create(@Valid @RequestBody CreateUserRequest request) {

&#x20;       // Đến được đây nghĩa là request đã hợp lệ

&#x20;       return ResponseEntity.ok("created");

&#x20;   }

}

```



\*\*Validate `@RequestParam` / `@PathVariable`\*\*

```java

@RestController

@Validated                       // bật validate tham số đơn giản

@RequestMapping("/api/products")

public class ProductController {



&#x20;   @GetMapping("/{id}")

&#x20;   public Product get(@PathVariable @Min(1) Long id) { ... }



&#x20;   @GetMapping

&#x20;   public List<Product> search(@RequestParam @Size(min = 2) String keyword) { ... }

}

```



\### 8.7. So sánh với các cách làm tương tự



| Cách | Ưu điểm | Nhược điểm | Khi nào dùng |

|---|---|---|---|

| \*\*Bean Validation (annotation)\*\* | Khai báo, tái sử dụng, chuẩn, tích hợp Spring/JPA | Khó biểu diễn luật nghiệp vụ phức tạp nhiều bước | Ràng buộc định dạng, phạm vi, bắt buộc |

| \*\*`if/else` thủ công\*\* | Linh hoạt tuyệt đối | Lặp code, khó bảo trì, dễ quên | Luật nghiệp vụ rất đặc thù, phụ thuộc nhiều bảng |

| \*\*Spring `Validator` interface\*\* (`supports`/`validate`) | Kiểm soát lập trình hoàn toàn, tách khỏi model | Dài dòng hơn annotation | Luật phức tạp, không muốn "bẩn" DTO |

| \*\*Ràng buộc DB\*\* (`NOT NULL`, `UNIQUE`, `CHECK`, FK) | Đảm bảo toàn vẹn dữ liệu tuyệt đối, chống race condition | Lỗi đến muộn, thông báo khó thân thiện | \*\*Luôn nên có\*\* như tuyến cuối |

| \*\*Validate phía client (JS)\*\* | Phản hồi tức thì cho người dùng | Có thể bị bỏ qua, \*\*không\*\* đáng tin | Chỉ để cải thiện UX, \*\*không\*\* thay thế server |



\---



\## 9. Exception Handling



\### 9.1. Khái niệm



\*\*Exception handling trong Spring Boot web\*\* là cách \*\*bắt các ngoại lệ phát sinh trong quá trình xử lý request và chuyển chúng thành HTTP response có ý nghĩa\*\* (mã trạng thái + nội dung lỗi) thay vì để lộ stack trace hay trả trang lỗi mặc định.



Nếu không xử lý, client sẽ nhận:

\- Trang "Whitelabel Error Page" , hoặc

\- JSON lỗi mặc định của Spring Boot (`timestamp`, `status`, `error`, `path`) với `500 Internal Server Error` cho mọi lỗi chưa biết – không rõ ràng và không nhất quán giữa các API.



\### 9.2. Giải quyết bài toán nào?



| Bài toán | Cách Exception Handling giúp |

|---|---|

| Mỗi controller `try/catch` một kiểu, code trùng lặp | Gom xử lý lỗi \*\*tập trung\*\* một nơi |

| Response lỗi mỗi API một khuôn | Thống nhất cấu trúc lỗi (ví dụ chuẩn `ProblemDetail`) |

| Lộ chi tiết nội bộ (stack trace, câu SQL) | Che giấu, chỉ trả thông điệp an toàn |

| Trả sai mã HTTP (mọi lỗi đều 500) | Ánh xạ đúng: 400, 401, 403, 404, 409, 422, 500... |

| Cần log lỗi tập trung | Một chỗ để log/monitor |



\### 9.3. Cách hoạt động (cốt lõi)



Khi một controller method ném exception, `DispatcherServlet` chuyển exception cho chuỗi \*\*`HandlerExceptionResolver`\*\*, thử lần lượt:



1\. \*\*`ExceptionHandlerExceptionResolver`\*\* – tìm method có `@ExceptionHandler` phù hợp (trong controller đó, rồi tới các `@ControllerAdvice`).

2\. \*\*`ResponseStatusExceptionResolver`\*\* – xử lý exception gắn `@ResponseStatus` hoặc kiểu `ResponseStatusException`.

3\. \*\*`DefaultHandlerExceptionResolver`\*\* – ánh xạ các exception chuẩn của Spring MVC (ví dụ `HttpRequestMethodNotSupportedException` → 405, `HttpMessageNotReadableException` → 400).



Với `@ExceptionHandler`, Spring chọn handler \*\*gần nhất với kiểu exception\*\* 



\### 9.4. Các cách xử lý exception



\#### Cách 1 – `try/catch` thủ công (không khuyến khích cho lỗi chung)

```java

@GetMapping("/{id}")

public ResponseEntity<?> get(@PathVariable Long id) {

&#x20;   try {

&#x20;       return ResponseEntity.ok(service.find(id));

&#x20;   } catch (UserNotFoundException e) {

&#x20;       return ResponseEntity.status(404).body(e.getMessage());

&#x20;   }

}

```

Nhược điểm: lặp lại ở mọi method.



\#### Cách 2 – `@ExceptionHandler` cục bộ (trong một controller)

```java

@RestController

public class UserController {



&#x20;   @GetMapping("/users/{id}")

&#x20;   public User get(@PathVariable Long id) {

&#x20;       return service.find(id);   // có thể ném UserNotFoundException

&#x20;   }



&#x20;   @ExceptionHandler(UserNotFoundException.class)

&#x20;   public ResponseEntity<String> handleNotFound(UserNotFoundException ex) {

&#x20;       return ResponseEntity.status(HttpStatus.NOT\_FOUND).body(ex.getMessage());

&#x20;   }

}

```

Chỉ có tác dụng với \*\*controller chứa nó\*\*.



\#### Cách 3 – `@ResponseStatus` trên class exception (đơn giản, tĩnh)

```java

@ResponseStatus(value = HttpStatus.NOT\_FOUND, reason = "Không tìm thấy người dùng")

public class UserNotFoundException extends RuntimeException { }

```

Gọn nhưng \*\*tĩnh\*\*: không tùy biến được nội dung theo từng tình huống.



\#### Cách 4 – `ResponseStatusException` (ném trực tiếp, nhanh gọn)

```java

throw new ResponseStatusException(HttpStatus.NOT\_FOUND, "Không tìm thấy user " + id);

```

Ít kiểm soát cấu trúc response và dễ trộn logic HTTP vào service.





\### 9.5. Một số exception thường gặp cần map



| Exception | HTTP gợi ý | Ghi chú |

|---|---|---|

| `MethodArgumentNotValidException` | 400 | Validation `@RequestBody` |

| `ConstraintViolationException` | 400 | Validation tham số/service |

| `HttpMessageNotReadableException` | 400 | JSON sai cú pháp |

| `MethodArgumentTypeMismatchException` | 400 | Sai kiểu path/query param |

| `AuthenticationException` (Spring Security) | 401 | Thường xử lý ở tầng Security filter |

| `AccessDeniedException` | 403 | Tương tự |

| `EntityNotFoundException` / tự định nghĩa | 404 | |

| `DataIntegrityViolationException` | 409 | Vi phạm unique/FK |

| `ObjectOptimisticLockingFailureException` | 409 | Xung đột phiên bản |

| `Exception` (fallback) | 500 | Log đầy đủ, trả thông điệp chung |





\### 9.6. So sánh các cách xử lý



| Cách | Phạm vi | Linh hoạt | Dùng khi |

|---|---|---|---|

| `try/catch` | Một đoạn code | Cao nhưng lặp | Cần phục hồi cục bộ (retry, fallback) |

| `@ExceptionHandler` trong controller | Một controller | Trung bình | Lỗi chỉ riêng controller đó |

| `@ResponseStatus` | Một class exception | Thấp (tĩnh) | Mapping đơn giản status ↔ exception |

| `ResponseStatusException` | Tại chỗ ném | Trung bình | Prototype, lỗi đơn giản |

| `@(Rest)ControllerAdvice` | \*\*Toàn ứng dụng\*\* | \*\*Cao\*\* | \*\*Chuẩn mực cho dự án thực tế\*\* |

| `ResponseEntityExceptionHandler` + `ProblemDetail` | Toàn ứng dụng | Cao, chuẩn hóa | Spring 6+/Boot 3+, muốn response theo RFC |



\---



\## 10. Override default fetch plan với JOIN FETCH / Entity Graph



\### 10.1. Khái niệm nền tảng



\*\*Fetch plan (kế hoạch nạp dữ liệu)\*\* là quyết định: \*khi nạp một entity từ DB, các entity/collection liên quan sẽ được nạp \*\*lúc nào\*\* và \*\*bằng cách nào\*\*\* (cùng một câu SQL hay những câu riêng).



\*\*Hai chiến lược nạp (`FetchType`) trong JPA\*\*



| Chiến lược | Ý nghĩa |

|---|---|

| `LAZY` | Nạp \*\*khi truy cập\*\* lần đầu (qua proxy). Entity liên quan chưa có cho tới khi bạn gọi getter. |

| `EAGER` | Nạp \*\*ngay\*\* cùng entity cha. |



\*\*Mặc định theo đặc tả JPA\*\*



| Quan hệ | Mặc định |

|---|---|

| `@ManyToOne`, `@OneToOne` | `EAGER` |

| `@OneToMany`, `@ManyToMany` | `LAZY` |



Đây chính là \*\*default fetch plan\*\* – nó được "gắn cứng" trong mapping của entity, áp dụng cho \*\*mọi\*\* truy vấn. Vấn đề: một mapping tĩnh không thể phù hợp với mọi use case. Ví dụ màn hình "danh sách bài viết" cần kèm tác giả, còn màn hình khác thì không cần.



\*\*"Override default fetch plan"\*\* nghĩa là: \*\*giữ mapping ở mức `LAZY`\*\* (an toàn, tiết kiệm), rồi \*\*ở từng truy vấn cụ thể\*\* chỉ rõ "lần này hãy nạp kèm quan hệ X" bằng \*\*`JOIN FETCH`\*\* (JPQL) hoặc \*\*Entity Graph\*\* (`@EntityGraph`). Nhiều bài viết cũng khuyên: đặt mặc định là `LAZY` và chỉ nạp thêm khi thật sự cần.



> Tránh `EAGER` cố định trên mapping: nó áp dụng cả khi bạn không cần dữ liệu đó, và không thể "tắt" theo từng truy vấn.



\### 10.2. Bài toán nó giải quyết: N+1 query



\*\*N+1 SELECT\*\* xảy ra khi JPA/Hibernate chạy \*\*1 query\*\* lấy danh sách entity cha, rồi \*\*N query phụ\*\* (mỗi cha một query) để nạp dữ liệu liên quan do lazy loading.



\*\*Ví dụ\*\*

```java

@Entity

public class Author {

&#x20;   @Id @GeneratedValue private Long id;

&#x20;   private String name;



&#x20;   @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)

&#x20;   private List<Post> posts = new ArrayList<>();

}



@Entity

public class Post {

&#x20;   @Id @GeneratedValue private Long id;

&#x20;   private String title;



&#x20;   @ManyToOne(fetch = FetchType.LAZY)

&#x20;   private Author author;

}

```



```java

List<Author> authors = authorRepository.findAll();   // 1 query: SELECT \* FROM author

for (Author a : authors) {

&#x20;   System.out.println(a.getPosts().size());          // N query: SELECT \* FROM post WHERE author\_id = ?

}

```

Với 100 tác giả → \*\*101 câu SQL\*\*. Dữ liệu càng lớn, hiệu năng càng tệ.



\*\*Cách phát hiện\*\*

```properties

spring.jpa.show-sql=true

spring.jpa.properties.hibernate.format\_sql=true

spring.jpa.properties.hibernate.generate\_statistics=true

\# hoặc: logging.level.org.hibernate.SQL=DEBUG

```

Quan sát log: thấy nhiều câu `select ... from post where author\_id=?` lặp lại là dấu hiệu N+1. Có thể dùng test đếm số query (Hibernate Statistics, datasource-proxy...).



\### 10.3. JOIN FETCH



\#### Khái niệm

`JOIN FETCH` là cú pháp \*\*JPQL/HQL\*\* yêu cầu Hibernate nạp \*\*cả entity cha lẫn quan hệ\*\* trong \*\*cùng một câu SQL JOIN\*\*, và khởi tạo luôn quan hệ đó (không còn proxy lười). Từ khóa `FETCH` là đặc thù của JPA; dùng được với cả `JOIN` và `LEFT JOIN`.



\#### Cú pháp

```java

public interface AuthorRepository extends JpaRepository<Author, Long> {



&#x20;   // INNER JOIN FETCH: chỉ trả tác giả CÓ ít nhất 1 bài

&#x20;   @Query("SELECT a FROM Author a JOIN FETCH a.posts")

&#x20;   List<Author> findAllWithPosts();



&#x20;   // LEFT JOIN FETCH: trả cả tác giả chưa có bài

&#x20;   @Query("SELECT a FROM Author a LEFT JOIN FETCH a.posts")

&#x20;   List<Author> findAllWithPostsOrEmpty();



&#x20;   // Kèm điều kiện, và nạp nhiều quan hệ ManyToOne

&#x20;   @Query("""

&#x20;          SELECT p FROM Post p

&#x20;          JOIN FETCH p.author

&#x20;          WHERE p.title LIKE %:kw%

&#x20;          """)

&#x20;   List<Post> searchWithAuthor(@Param("kw") String keyword);

}

```



SQL sinh ra (rút gọn): `select a.\*, p.\* from author a join post p on a.id = p.author\_id` → chỉ \*\*1 query\*\*.



\### 10.4. Entity Graph



\#### Khái niệm

\*\*Entity Graph\*\*cho phép \*\*khai báo, tách biệt khỏi câu truy vấn\*\*, đồ thị các thuộc tính/quan hệ cần nạp cùng entity gốc. Spring Data JPA hỗ trợ qua annotation \*\*`@EntityGraph`\*\* trên method repository. Ưu điểm: không phải viết JPQL dài chỉ để thêm fetch, và áp dụng được cả cho \*\*derived query\*\* (`findByName...`) và các method có sẵn như `findAll`, `findById`.



\#### Cách 1 – `attributePaths` (đơn giản, phổ biến nhất)

```java

public interface PostRepository extends JpaRepository<Post, Long> {



&#x20;   @EntityGraph(attributePaths = {"author"})

&#x20;   List<Post> findByTitleContaining(String keyword);



&#x20;   // Override method có sẵn của JpaRepository

&#x20;   @Override

&#x20;   @EntityGraph(attributePaths = {"author", "comments"})

&#x20;   List<Post> findAll();



&#x20;   @EntityGraph(attributePaths = {"author"})

&#x20;   Optional<Post> findById(Long id);

}

```

Chỉ \*\*một query\*\* lấy cả `Post` và `Author` (SQL sinh ra dạng `left join author ...`).



\#### Cách 2 – `@NamedEntityGraph` (đồ thị đặt tên, tái sử dụng)

Khai báo trên entity:

```java

@Entity

@NamedEntityGraph(

&#x20;   name = "Post.withAuthorAndComments",

&#x20;   attributeNodes = {

&#x20;       @NamedAttributeNode("author"),

&#x20;       @NamedAttributeNode("comments")

&#x20;   }

)

public class Post { ... }

```

Dùng ở repository:

```java

@EntityGraph(value = "Post.withAuthorAndComments")

List<Post> findAll();

```



\#### Cách 3 – Subgraph (nạp quan hệ lồng sâu nhiều cấp)

Ví dụ `Post → comments → user`:

```java

@NamedEntityGraph(

&#x20;   name = "Post.deep",

&#x20;   attributeNodes = @NamedAttributeNode(value = "comments", subgraph = "comments-sub"),

&#x20;   subgraphs = @NamedSubgraph(

&#x20;       name = "comments-sub",

&#x20;       attributeNodes = @NamedAttributeNode("user"))

)

```

Hoặc ngắn gọn bằng dấu chấm trong `attributePaths`:

```java

@EntityGraph(attributePaths = {"comments", "comments.user"})

```



\- \*\*Không\*\* có entity graph → 1 query lấy post + \*\*N query\*\* lấy từng author.

\- \*\*Có\*\* `@EntityGraph(attributePaths = "author")` → \*\*1 query\*\* (join) lấy cả hai.



\### 10.5. JOIN FETCH và Entity Graph: khác nhau thế nào?



| Tiêu chí | `JOIN FETCH` | `@EntityGraph` |

|---|---|---|

| Cách viết | Trong câu JPQL/HQL (`@Query`) | Annotation tách khỏi query |

| Áp dụng cho derived query / `findAll` có sẵn | Không (phải viết `@Query`) | \*\*Có\*\* |

| Loại join sinh ra | Theo bạn viết: `JOIN FETCH` → inner, `LEFT JOIN FETCH` → left | Thường là \*\*left outer join\*\* |

| Tái sử dụng | Phải lặp lại trong từng query | \*\*Có\*\* (named graph) |

| Nhiều cấp lồng | Viết nhiều `JOIN FETCH` | `subgraph` / `attributePaths` có dấu chấm |

| Loại trùng khi fetch collection | Có thể cần `DISTINCT` (Hibernate 5) | Kết quả thường không lặp (xem ghi chú) |

| Kiểm soát chi tiết (điều kiện `WHERE` phức tạp, `GROUP BY`) | \*\*Tốt\*\* – bạn nắm toàn bộ câu truy vấn | Hạn chế – chỉ khai báo "nạp gì" |

| Chuẩn | JPQL/HQL | JPA 2.1+ |



\*\*Khi nào chọn cái nào?\*\*

\- Cần \*\*nhanh, gọn\*\*, nạp thêm quan hệ cho method có sẵn → `@EntityGraph`.

\- Query \*\*phức tạp\*\* (điều kiện, sắp xếp, tính toán) và cần fetch kèm → `JOIN FETCH` trong `@Query`.

\- Cần \*\*nhiều graph khác nhau\*\* cho nhiều màn hình → named entity graph, hoặc nhiều method repository với `attributePaths` khác nhau.



\### 10.6. So với các giải pháp N+1 khác



| Giải pháp | Cơ chế | Ưu | Nhược |

|---|---|---|---|

| `JOIN FETCH` / `@EntityGraph` | 1 query JOIN | Ít round-trip nhất | Kết quả phình (cartesian product), rắc rối khi phân trang với collection |

| \*\*Batch fetching\*\* `hibernate.default\_batch\_fetch\_size=N` hoặc `@BatchSize` | Gom lazy load thành `WHERE id IN (...)` theo lô | Cấu hình nhẹ, vẫn lazy, phân trang an toàn | Vẫn nhiều hơn 1 query (1 + N/batch) |

| `@Fetch(FetchMode.JOIN)` | Ý định đặt JOIN trên mapping | — | \*\*Không có tác dụng\*\* với truy vấn Spring Data sinh bằng Criteria API |




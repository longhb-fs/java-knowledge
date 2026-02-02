# Ngày 13: DTO Pattern & MapStruct

## Mục tiêu hôm nay
- Hiểu sâu DTO Pattern và tại sao cần nó
- Cài đặt và cấu hình MapStruct
- Viết Mapper interface với các mapping types
- Xử lý nested mapping, custom converters
- Tích hợp MapStruct với Spring

---

## 1. DTO PATTERN

### 1.1. Tại sao cần DTO?

```
┌─────────────────────────────────────────────────────────────┐
│                    VẤN ĐỀ KHI DÙNG ENTITY TRỰC TIẾP         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SECURITY RISK                                            │
│     Entity User có field "password" → trả về cho client?    │
│     Entity có audit fields (createdBy, updatedBy) → lộ?     │
│                                                              │
│  2. PERFORMANCE                                              │
│     Entity có LAZY relationships → N+1 khi serialize JSON    │
│     Trả về nhiều data không cần thiết                        │
│                                                              │
│  3. COUPLING                                                 │
│     Thay đổi Entity → ảnh hưởng API contract                 │
│     Frontend phụ thuộc vào database schema                   │
│                                                              │
│  4. CIRCULAR REFERENCE                                       │
│     Product → Category → Products → Category... → Loop!      │
│     JSON serialization fails                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘

GIẢI PHÁP: DTO (Data Transfer Object)

Entity (Database)          DTO (API)
┌─────────────────┐        ┌─────────────────┐
│ User            │        │ UserResponse    │
│ - id            │───────>│ - id            │
│ - email         │        │ - email         │
│ - password ❌   │        │ - fullName      │
│ - fullName      │        │ - avatarUrl     │
│ - avatarUrl     │        └─────────────────┘
│ - createdAt     │
│ - updatedAt     │
│ - createdBy     │
│ - orders (LAZY) │
└─────────────────┘
```

### 1.2. DTO Types trong project

```
┌─────────────────────────────────────────────────────────────┐
│                       DTO TYPES                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REQUEST DTOs (Client → Server)                              │
│  ├── ProductCreateRequest   - Tạo mới                        │
│  ├── ProductUpdateRequest   - Cập nhật                       │
│  ├── ProductSearchRequest   - Tìm kiếm/filter                │
│  ├── LoginRequest           - Đăng nhập                      │
│  └── RegisterRequest        - Đăng ký                        │
│                                                              │
│  RESPONSE DTOs (Server → Client)                             │
│  ├── ProductResponse        - Chi tiết sản phẩm              │
│  ├── ProductListResponse    - List (ít field hơn)            │
│  ├── UserResponse           - Thông tin user                 │
│  ├── ApiResponse<T>         - Wrapper chung                  │
│  └── PagedResponse<T>       - Phân trang                     │
│                                                              │
│  INTERNAL DTOs                                               │
│  ├── ProductDTO             - Giữa services                  │
│  └── OrderSummaryDTO        - Tổng hợp từ nhiều entity       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. MAPSTRUCT SETUP

### 2.1. Dependencies (pom.xml)

```xml
<properties>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
</properties>

<dependencies>
    <!-- MapStruct -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <!-- Lombok PHẢI trước MapStruct -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </path>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${mapstruct.version}</version>
                    </path>
                    <!-- Lombok + MapStruct binding -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>0.2.0</version>
                    </path>
                </annotationProcessorPaths>
                <compilerArgs>
                    <!-- Default component model = Spring -->
                    <arg>-Amapstruct.defaultComponentModel=spring</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 2.2. Verify Setup

```bash
# Compile để MapStruct generate code
mvn clean compile

# Check generated code
ls target/generated-sources/annotations/com/example/ecommerce/dto/mapper/
# ProductMapperImpl.java  (generated!)
```

---

## 3. BASIC MAPPER

### 3.1. ProductMapper

```java
package com.example.ecommerce.dto.mapper;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.entity.Product;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    // ═══ Entity → Response ═══

    @Mapping(target = "categoryId", source = "category.id")
    @Mapping(target = "categoryName", source = "category.name")
    @Mapping(target = "categorySlug", source = "category.slug")
    @Mapping(target = "effectivePrice", expression = "java(product.getEffectivePrice())")
    @Mapping(target = "inStock", expression = "java(product.isInStock())")
    ProductResponse toResponse(Product product);

    // ═══ Request → Entity ═══

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "slug", ignore = true)  // Generated in service
    @Mapping(target = "category", ignore = true)  // Set in service
    @Mapping(target = "images", ignore = true)
    @Mapping(target = "tags", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Product toEntity(ProductCreateRequest request);

    // ═══ List mapping ═══

    List<ProductResponse> toResponseList(List<Product> products);
}
```

### 3.2. Generated Code (tham khảo)

```java
// target/generated-sources/annotations/.../ProductMapperImpl.java
@Component
public class ProductMapperImpl implements ProductMapper {

    @Override
    public ProductResponse toResponse(Product product) {
        if (product == null) {
            return null;
        }

        ProductResponse.ProductResponseBuilder productResponse = ProductResponse.builder();

        productResponse.categoryId(productCategoryId(product));
        productResponse.categoryName(productCategoryName(product));
        productResponse.categorySlug(productCategorySlug(product));
        productResponse.id(product.getId());
        productResponse.name(product.getName());
        productResponse.slug(product.getSlug());
        productResponse.sku(product.getSku());
        productResponse.price(product.getPrice());
        // ... more fields

        // Expression
        productResponse.effectivePrice(product.getEffectivePrice());
        productResponse.inStock(product.isInStock());

        return productResponse.build();
    }

    private Long productCategoryId(Product product) {
        if (product == null) return null;
        Category category = product.getCategory();
        if (category == null) return null;
        return category.getId();
    }

    // ... more helper methods
}
```

---

## 4. MAPPING TYPES

### 4.1. Basic Mapping

```java
@Mapper(componentModel = "spring")
public interface CategoryMapper {

    // Tự động map fields cùng tên
    CategoryResponse toResponse(Category category);

    // Explicit mapping
    @Mapping(target = "parentName", source = "parent.name")
    @Mapping(target = "productCount", expression = "java(category.getProducts().size())")
    CategoryResponse toDetailResponse(Category category);
}
```

### 4.2. Nested Mapping

```java
@Mapper(componentModel = "spring", uses = {ProductImageMapper.class, TagMapper.class})
public interface ProductMapper {

    @Mapping(target = "images", source = "images")  // Dùng ProductImageMapper
    @Mapping(target = "tags", source = "tags")      // Dùng TagMapper
    ProductResponse toResponse(Product product);
}

@Mapper(componentModel = "spring")
public interface ProductImageMapper {
    ProductImageResponse toResponse(ProductImage image);
    List<ProductImageResponse> toResponseList(List<ProductImage> images);
}

@Mapper(componentModel = "spring")
public interface TagMapper {
    TagResponse toResponse(Tag tag);
    List<TagResponse> toResponseList(Collection<Tag> tags);
}
```

### 4.3. Custom Methods trong Mapper

```java
@Mapper(componentModel = "spring")
public interface ProductMapper {

    @Mapping(target = "effectivePrice", source = "product")
    ProductResponse toResponse(Product product);

    // Custom method - MapStruct sẽ dùng
    default BigDecimal mapEffectivePrice(Product product) {
        if (product.getSalePrice() != null && product.getSalePrice().compareTo(BigDecimal.ZERO) > 0) {
            return product.getSalePrice();
        }
        return product.getPrice();
    }

    // Hoặc dùng expression
    @Mapping(target = "discountPercent", expression = "java(calculateDiscount(product))")
    ProductResponse toDetailResponse(Product product);

    default Integer calculateDiscount(Product product) {
        if (product.getSalePrice() == null || product.getPrice() == null) {
            return 0;
        }
        return product.getPrice().subtract(product.getSalePrice())
                .divide(product.getPrice(), 2, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100))
                .intValue();
    }
}
```

### 4.4. Update Mapping (Partial Update)

```java
@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Update entity từ request (chỉ update non-null fields)
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "slug", ignore = true)
    @Mapping(target = "sku", ignore = true)  // SKU không được update
    @Mapping(target = "category", ignore = true)
    @Mapping(target = "images", ignore = true)
    @Mapping(target = "tags", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    void updateEntityFromRequest(ProductUpdateRequest request, @MappingTarget Product product);
}

// Sử dụng trong Service:
@Transactional
public ProductResponse update(Long id, ProductUpdateRequest request) {
    Product product = findById(id);

    productMapper.updateEntityFromRequest(request, product);  // Partial update

    // Handle category, images, tags separately...

    product = productRepository.save(product);
    return productMapper.toResponse(product);
}
```

---

## 5. ADVANCED MAPPING

### 5.1. Mapping với @AfterMapping

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    @Mapping(target = "statusText", ignore = true)
    @Mapping(target = "paymentStatusText", ignore = true)
    OrderResponse toResponse(Order order);

    @AfterMapping
    default void setStatusTexts(Order order, @MappingTarget OrderResponse response) {
        response.setStatusText(mapStatusToVietnamese(order.getStatus()));
        response.setPaymentStatusText(mapPaymentStatusToVietnamese(order.getPaymentStatus()));
    }

    default String mapStatusToVietnamese(OrderStatus status) {
        return switch (status) {
            case PENDING -> "Chờ xử lý";
            case CONFIRMED -> "Đã xác nhận";
            case PROCESSING -> "Đang xử lý";
            case SHIPPING -> "Đang giao hàng";
            case DELIVERED -> "Đã giao hàng";
            case CANCELLED -> "Đã hủy";
            case REFUNDED -> "Đã hoàn tiền";
        };
    }
}
```

### 5.2. Mapping với @BeforeMapping

```java
@Mapper(componentModel = "spring")
public interface ProductMapper {

    ProductResponse toResponse(Product product);

    @BeforeMapping
    default void checkProduct(Product product) {
        if (product == null) {
            throw new IllegalArgumentException("Product cannot be null");
        }
    }
}
```

### 5.3. Mapping với Qualifiers

```java
@Mapper(componentModel = "spring")
public interface PriceMapper {

    @Qualifier
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.CLASS)
    @interface FormattedPrice {}

    @Qualifier
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.CLASS)
    @interface RawPrice {}

    @FormattedPrice
    default String formatPrice(BigDecimal price) {
        if (price == null) return null;
        NumberFormat formatter = NumberFormat.getCurrencyInstance(new Locale("vi", "VN"));
        return formatter.format(price);
    }

    @RawPrice
    default BigDecimal rawPrice(BigDecimal price) {
        return price;
    }
}

@Mapper(componentModel = "spring", uses = PriceMapper.class)
public interface ProductMapper {

    @Mapping(target = "formattedPrice", source = "price", qualifiedBy = PriceMapper.FormattedPrice.class)
    @Mapping(target = "price", source = "price", qualifiedBy = PriceMapper.RawPrice.class)
    ProductResponse toResponse(Product product);
}
```

### 5.4. Mapping Enum

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    // Automatic enum mapping (same name)
    OrderResponse toResponse(Order order);

    // Custom enum mapping
    @ValueMappings({
        @ValueMapping(source = "PENDING", target = "CHO_XU_LY"),
        @ValueMapping(source = "CONFIRMED", target = "DA_XAC_NHAN"),
        @ValueMapping(source = "CANCELLED", target = "DA_HUY"),
        @ValueMapping(source = MappingConstants.ANY_REMAINING, target = "KHAC")
    })
    OrderStatusVi toVietnameseStatus(OrderStatus status);
}
```

---

## 6. MAPPER TRONG SERVICE

### 6.1. Inject Mapper

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;  // Inject mapper

    @Override
    @Transactional
    public ProductResponse create(ProductCreateRequest request) {
        // Map request → entity
        Product product = productMapper.toEntity(request);

        // Set relationships
        if (request.getCategoryId() != null) {
            Category category = categoryRepository.findById(request.getCategoryId())
                    .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));
            product.setCategory(category);
        }

        // Generate slug
        product.setSlug(generateUniqueSlug(request.getName()));

        // Save
        product = productRepository.save(product);

        // Map entity → response
        return productMapper.toResponse(product);
    }

    @Override
    @Transactional
    public ProductResponse update(Long id, ProductUpdateRequest request) {
        Product product = findById(id);

        // Partial update
        productMapper.updateEntityFromRequest(request, product);

        // Handle relationships...

        product = productRepository.save(product);
        return productMapper.toResponse(product);
    }

    @Override
    public PagedResponse<ProductResponse> search(ProductSearchRequest request) {
        Page<Product> page = productRepository.findAll(buildSpec(request), buildPageable(request));

        // Map page with mapper
        Page<ProductResponse> responsePage = page.map(productMapper::toResponse);

        return PagedResponse.from(responsePage);
    }
}
```

---

## 7. TESTING MAPPER

```java
@SpringBootTest
class ProductMapperTest {

    @Autowired
    private ProductMapper productMapper;

    @Test
    void toResponse_shouldMapAllFields() {
        // Given
        Category category = Category.builder()
                .id(1L)
                .name("Điện thoại")
                .slug("dien-thoai")
                .build();

        Product product = Product.builder()
                .id(1L)
                .name("iPhone 15")
                .slug("iphone-15")
                .sku("IP15-128")
                .price(new BigDecimal("22990000"))
                .salePrice(new BigDecimal("21990000"))
                .stockQuantity(100)
                .isActive(true)
                .isFeatured(true)
                .category(category)
                .build();

        // When
        ProductResponse response = productMapper.toResponse(product);

        // Then
        assertThat(response.getId()).isEqualTo(1L);
        assertThat(response.getName()).isEqualTo("iPhone 15");
        assertThat(response.getCategoryId()).isEqualTo(1L);
        assertThat(response.getCategoryName()).isEqualTo("Điện thoại");
        assertThat(response.getEffectivePrice()).isEqualTo(new BigDecimal("21990000"));
        assertThat(response.getInStock()).isTrue();
    }

    @Test
    void toEntity_shouldMapRequestToEntity() {
        // Given
        ProductCreateRequest request = new ProductCreateRequest();
        request.setName("MacBook Pro");
        request.setSku("MBP-14");
        request.setPrice(new BigDecimal("49990000"));

        // When
        Product product = productMapper.toEntity(request);

        // Then
        assertThat(product.getName()).isEqualTo("MacBook Pro");
        assertThat(product.getSku()).isEqualTo("MBP-14");
        assertThat(product.getPrice()).isEqualTo(new BigDecimal("49990000"));
        assertThat(product.getId()).isNull();  // ignored
        assertThat(product.getSlug()).isNull(); // ignored
    }
}
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Cấu hình MapStruct trong pom.xml
- [ ] Tạo ProductMapper với basic mapping
- [ ] Tạo nested mappers (ImageMapper, TagMapper)
- [ ] Implement updateEntityFromRequest cho partial update
- [ ] Tạo @AfterMapping cho custom logic
- [ ] Inject mapper vào Service và refactor
- [ ] Viết unit tests cho mappers
- [ ] Compile và verify generated code

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Tại sao cần DTO Pattern | ☐ |
| MapStruct vs Manual mapping | ☐ |
| @Mapper annotation options | ☐ |
| @Mapping với source/target | ☐ |
| Nested mapping với uses = {} | ☐ |
| @BeanMapping nullValuePropertyMappingStrategy | ☐ |
| @AfterMapping, @BeforeMapping | ☐ |
| Expression trong mapping | ☐ |
| Qualifiers cho custom conversion | ☐ |

---

**Trước đó:** [← Ngày 12 - CRUD API](day-12-crud-api.md)

**Tiếp theo:** [Ngày 14 - Bean Validation →](day-14-validation.md)

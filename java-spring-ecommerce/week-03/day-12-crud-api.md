# Ngày 12: CRUD API Hoàn Chỉnh

## Mục tiêu hôm nay
- Xây dựng full CRUD cho Product API
- Tạo Service layer hoàn chỉnh
- Xử lý relationships (Category, Images)
- Slug generation và validation
- Soft delete vs Hard delete

---

## 1. REQUEST DTOs

### 1.1. ProductCreateRequest

```java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
public class ProductCreateRequest {

    @NotBlank(message = "Tên sản phẩm không được trống")
    @Size(min = 2, max = 200, message = "Tên sản phẩm từ 2-200 ký tự")
    private String name;

    @NotBlank(message = "SKU không được trống")
    @Size(min = 2, max = 50, message = "SKU từ 2-50 ký tự")
    @Pattern(regexp = "^[A-Z0-9-]+$", message = "SKU chỉ chứa chữ in hoa, số và dấu -")
    private String sku;

    @NotNull(message = "Giá không được trống")
    @Positive(message = "Giá phải lớn hơn 0")
    @Digits(integer = 10, fraction = 2, message = "Giá không hợp lệ")
    private BigDecimal price;

    @PositiveOrZero(message = "Giá khuyến mãi phải >= 0")
    private BigDecimal salePrice;

    @Size(max = 5000, message = "Mô tả tối đa 5000 ký tự")
    private String description;

    @PositiveOrZero(message = "Số lượng phải >= 0")
    private Integer stockQuantity = 0;

    private Long categoryId;

    private String mainImage;

    private List<String> images;

    private List<Long> tagIds;

    private Boolean isActive = true;

    private Boolean isFeatured = false;
}
```

### 1.2. ProductUpdateRequest

```java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
public class ProductUpdateRequest {

    @Size(min = 2, max = 200, message = "Tên sản phẩm từ 2-200 ký tự")
    private String name;

    @Positive(message = "Giá phải lớn hơn 0")
    private BigDecimal price;

    @PositiveOrZero(message = "Giá khuyến mãi phải >= 0")
    private BigDecimal salePrice;

    @Size(max = 5000, message = "Mô tả tối đa 5000 ký tự")
    private String description;

    @PositiveOrZero(message = "Số lượng phải >= 0")
    private Integer stockQuantity;

    private Long categoryId;

    private String mainImage;

    private List<String> images;

    private List<Long> tagIds;

    private Boolean isActive;

    private Boolean isFeatured;
}
```

---

## 2. RESPONSE DTOs

### 2.1. ProductResponse

```java
package com.example.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductResponse {

    private Long id;
    private String name;
    private String slug;
    private String sku;
    private String description;
    private BigDecimal price;
    private BigDecimal salePrice;
    private BigDecimal effectivePrice;
    private Integer stockQuantity;
    private Boolean isActive;
    private Boolean isFeatured;
    private Boolean inStock;
    private String mainImage;

    // Category info
    private Long categoryId;
    private String categoryName;
    private String categorySlug;

    // Images
    private List<ProductImageResponse> images;

    // Tags
    private List<TagResponse> tags;

    // Timestamps
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### 2.2. Related DTOs

```java
// ProductImageResponse.java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductImageResponse {
    private Long id;
    private String url;
    private String altText;
    private Integer sortOrder;
}

// TagResponse.java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TagResponse {
    private Long id;
    private String name;
    private String slug;
}

// CategoryResponse.java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CategoryResponse {
    private Long id;
    private String name;
    private String slug;
    private String description;
    private String imageUrl;
    private Long parentId;
    private String parentName;
    private Boolean isActive;
    private Integer productCount;
}
```

---

## 3. SERVICE LAYER

### 3.1. ProductService Interface

```java
package com.example.ecommerce.service;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.request.ProductSearchRequest;
import com.example.ecommerce.dto.request.ProductUpdateRequest;
import com.example.ecommerce.dto.response.PagedResponse;
import com.example.ecommerce.dto.response.ProductResponse;

import java.util.List;

public interface ProductService {

    // CRUD
    ProductResponse create(ProductCreateRequest request);
    ProductResponse getById(Long id);
    ProductResponse getBySlug(String slug);
    ProductResponse update(Long id, ProductUpdateRequest request);
    void delete(Long id);

    // Search & Filter
    PagedResponse<ProductResponse> search(ProductSearchRequest request);
    List<ProductResponse> getFeatured();
    PagedResponse<ProductResponse> getByCategory(Long categoryId, int page, int size);

    // Stock management
    void updateStock(Long id, int quantity);
    void decreaseStock(Long id, int quantity);
}
```

### 3.2. ProductServiceImpl

```java
package com.example.ecommerce.service.impl;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.request.ProductSearchRequest;
import com.example.ecommerce.dto.request.ProductUpdateRequest;
import com.example.ecommerce.dto.response.PagedResponse;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.entity.Category;
import com.example.ecommerce.entity.Product;
import com.example.ecommerce.entity.ProductImage;
import com.example.ecommerce.entity.Tag;
import com.example.ecommerce.exception.BadRequestException;
import com.example.ecommerce.exception.DuplicateResourceException;
import com.example.ecommerce.exception.ResourceNotFoundException;
import com.example.ecommerce.repository.CategoryRepository;
import com.example.ecommerce.repository.ProductRepository;
import com.example.ecommerce.repository.TagRepository;
import com.example.ecommerce.repository.specification.ProductSpecBuilder;
import com.example.ecommerce.service.ProductService;
import com.example.ecommerce.util.SlugUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final TagRepository tagRepository;

    // ═══════════════════════════════════════════════════════════
    // CREATE
    // ═══════════════════════════════════════════════════════════

    @Override
    @Transactional
    public ProductResponse create(ProductCreateRequest request) {
        log.info("Creating product: {}", request.getName());

        // Validate unique SKU
        if (productRepository.existsBySku(request.getSku())) {
            throw new DuplicateResourceException("SKU '" + request.getSku() + "' đã tồn tại");
        }

        // Generate unique slug
        String slug = generateUniqueSlug(request.getName());

        // Build Product entity
        Product product = Product.builder()
                .name(request.getName())
                .slug(slug)
                .sku(request.getSku())
                .description(request.getDescription())
                .price(request.getPrice())
                .salePrice(request.getSalePrice())
                .stockQuantity(request.getStockQuantity() != null ? request.getStockQuantity() : 0)
                .isActive(request.getIsActive() != null ? request.getIsActive() : true)
                .isFeatured(request.getIsFeatured() != null ? request.getIsFeatured() : false)
                .mainImage(request.getMainImage())
                .build();

        // Set Category
        if (request.getCategoryId() != null) {
            Category category = categoryRepository.findById(request.getCategoryId())
                    .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));
            product.setCategory(category);
        }

        // Add Images
        if (request.getImages() != null && !request.getImages().isEmpty()) {
            int sortOrder = 0;
            for (String imageUrl : request.getImages()) {
                ProductImage image = ProductImage.builder()
                        .url(imageUrl)
                        .sortOrder(sortOrder++)
                        .build();
                product.addImage(image);
            }
        }

        // Add Tags
        if (request.getTagIds() != null && !request.getTagIds().isEmpty()) {
            Set<Tag> tags = new HashSet<>(tagRepository.findAllById(request.getTagIds()));
            product.setTags(tags);
        }

        // Save
        product = productRepository.save(product);
        log.info("Created product with ID: {}", product.getId());

        return toResponse(product);
    }

    // ═══════════════════════════════════════════════════════════
    // READ
    // ═══════════════════════════════════════════════════════════

    @Override
    public ProductResponse getById(Long id) {
        Product product = findProductById(id);
        return toResponse(product);
    }

    @Override
    public ProductResponse getBySlug(String slug) {
        Product product = productRepository.findBySlug(slug)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "slug", slug));
        return toResponse(product);
    }

    @Override
    public PagedResponse<ProductResponse> search(ProductSearchRequest request) {
        // Build Specification
        Specification<Product> spec = ProductSpecBuilder.builder()
                .active()
                .categoryId(request.getCategoryId())
                .keyword(request.getKeyword())
                .priceBetween(request.getMinPrice(), request.getMaxPrice())
                .build();

        if (Boolean.TRUE.equals(request.getInStockOnly())) {
            spec = spec.and((root, query, cb) -> cb.greaterThan(root.get("stockQuantity"), 0));
        }

        if (Boolean.TRUE.equals(request.getFeaturedOnly())) {
            spec = spec.and((root, query, cb) -> cb.isTrue(root.get("isFeatured")));
        }

        // Pagination
        Sort sort = buildSort(request.getSortBy(), request.getSortDir());
        Pageable pageable = PageRequest.of(request.getPage(), request.getSize(), sort);

        // Execute query
        Page<Product> page = productRepository.findAll(spec, pageable);

        return PagedResponse.from(page, this::toResponse);
    }

    @Override
    public List<ProductResponse> getFeatured() {
        List<Product> products = productRepository.findByIsActiveTrueAndIsFeaturedTrue();
        return products.stream().map(this::toResponse).toList();
    }

    @Override
    public PagedResponse<ProductResponse> getByCategory(Long categoryId, int page, int size) {
        // Verify category exists
        if (!categoryRepository.existsById(categoryId)) {
            throw new ResourceNotFoundException("Category", "id", categoryId);
        }

        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<Product> productPage = productRepository.findByCategoryIdAndIsActiveTrue(categoryId, pageable);

        return PagedResponse.from(productPage, this::toResponse);
    }

    // ═══════════════════════════════════════════════════════════
    // UPDATE
    // ═══════════════════════════════════════════════════════════

    @Override
    @Transactional
    public ProductResponse update(Long id, ProductUpdateRequest request) {
        log.info("Updating product ID: {}", id);

        Product product = findProductById(id);

        // Update fields if provided
        if (request.getName() != null) {
            product.setName(request.getName());
            // Update slug if name changes
            product.setSlug(generateUniqueSlug(request.getName(), product.getId()));
        }

        if (request.getPrice() != null) {
            product.setPrice(request.getPrice());
        }

        if (request.getSalePrice() != null) {
            product.setSalePrice(request.getSalePrice());
        }

        if (request.getDescription() != null) {
            product.setDescription(request.getDescription());
        }

        if (request.getStockQuantity() != null) {
            product.setStockQuantity(request.getStockQuantity());
        }

        if (request.getIsActive() != null) {
            product.setIsActive(request.getIsActive());
        }

        if (request.getIsFeatured() != null) {
            product.setIsFeatured(request.getIsFeatured());
        }

        if (request.getMainImage() != null) {
            product.setMainImage(request.getMainImage());
        }

        // Update Category
        if (request.getCategoryId() != null) {
            Category category = categoryRepository.findById(request.getCategoryId())
                    .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));
            product.setCategory(category);
        }

        // Update Images (replace all)
        if (request.getImages() != null) {
            product.getImages().clear();
            int sortOrder = 0;
            for (String imageUrl : request.getImages()) {
                ProductImage image = ProductImage.builder()
                        .url(imageUrl)
                        .sortOrder(sortOrder++)
                        .build();
                product.addImage(image);
            }
        }

        // Update Tags (replace all)
        if (request.getTagIds() != null) {
            Set<Tag> tags = new HashSet<>(tagRepository.findAllById(request.getTagIds()));
            product.setTags(tags);
        }

        product = productRepository.save(product);
        log.info("Updated product ID: {}", id);

        return toResponse(product);
    }

    // ═══════════════════════════════════════════════════════════
    // DELETE
    // ═══════════════════════════════════════════════════════════

    @Override
    @Transactional
    public void delete(Long id) {
        log.info("Deleting product ID: {}", id);

        Product product = findProductById(id);

        // Soft delete (recommended)
        product.setIsActive(false);
        productRepository.save(product);

        // Hard delete (uncomment if needed)
        // productRepository.delete(product);

        log.info("Deleted product ID: {}", id);
    }

    // ═══════════════════════════════════════════════════════════
    // STOCK MANAGEMENT
    // ═══════════════════════════════════════════════════════════

    @Override
    @Transactional
    public void updateStock(Long id, int quantity) {
        Product product = findProductById(id);
        product.setStockQuantity(quantity);
        productRepository.save(product);
    }

    @Override
    @Transactional
    public void decreaseStock(Long id, int quantity) {
        int updated = productRepository.decreaseStock(id, quantity);
        if (updated == 0) {
            throw new BadRequestException("Không đủ số lượng trong kho");
        }
    }

    // ═══════════════════════════════════════════════════════════
    // HELPER METHODS
    // ═══════════════════════════════════════════════════════════

    private Product findProductById(Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
    }

    private String generateUniqueSlug(String name) {
        return generateUniqueSlug(name, null);
    }

    private String generateUniqueSlug(String name, Long excludeId) {
        String baseSlug = SlugUtils.toSlug(name);
        String slug = baseSlug;
        int counter = 1;

        while (true) {
            boolean exists = excludeId != null
                    ? productRepository.existsBySlugAndIdNot(slug, excludeId)
                    : productRepository.existsBySlug(slug);

            if (!exists) {
                return slug;
            }
            slug = baseSlug + "-" + counter++;
        }
    }

    private Sort buildSort(String sortBy, String sortDir) {
        if (sortBy == null) sortBy = "createdAt";
        if (sortDir == null) sortDir = "desc";

        return sortDir.equalsIgnoreCase("asc")
                ? Sort.by(sortBy).ascending()
                : Sort.by(sortBy).descending();
    }

    private ProductResponse toResponse(Product product) {
        ProductResponse.ProductResponseBuilder builder = ProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .slug(product.getSlug())
                .sku(product.getSku())
                .description(product.getDescription())
                .price(product.getPrice())
                .salePrice(product.getSalePrice())
                .effectivePrice(product.getEffectivePrice())
                .stockQuantity(product.getStockQuantity())
                .isActive(product.getIsActive())
                .isFeatured(product.getIsFeatured())
                .inStock(product.isInStock())
                .mainImage(product.getMainImage())
                .createdAt(product.getCreatedAt())
                .updatedAt(product.getUpdatedAt());

        // Category
        if (product.getCategory() != null) {
            builder.categoryId(product.getCategory().getId())
                   .categoryName(product.getCategory().getName())
                   .categorySlug(product.getCategory().getSlug());
        }

        // Images
        if (product.getImages() != null && !product.getImages().isEmpty()) {
            builder.images(product.getImages().stream()
                    .map(img -> ProductImageResponse.builder()
                            .id(img.getId())
                            .url(img.getUrl())
                            .altText(img.getAltText())
                            .sortOrder(img.getSortOrder())
                            .build())
                    .toList());
        }

        // Tags
        if (product.getTags() != null && !product.getTags().isEmpty()) {
            builder.tags(product.getTags().stream()
                    .map(tag -> TagResponse.builder()
                            .id(tag.getId())
                            .name(tag.getName())
                            .slug(tag.getSlug())
                            .build())
                    .toList());
        }

        return builder.build();
    }
}
```

---

## 4. SLUG UTILITY

```java
package com.example.ecommerce.util;

import java.text.Normalizer;
import java.util.Locale;
import java.util.regex.Pattern;

public class SlugUtils {

    private static final Pattern NONLATIN = Pattern.compile("[^\\w-]");
    private static final Pattern WHITESPACE = Pattern.compile("[\\s]");
    private static final Pattern EDGESDHASHES = Pattern.compile("(^-|-$)");

    /**
     * Convert string to URL-friendly slug
     * "iPhone 15 Pro Max" -> "iphone-15-pro-max"
     * "Điện thoại Samsung" -> "dien-thoai-samsung"
     */
    public static String toSlug(String input) {
        if (input == null || input.isBlank()) {
            return "";
        }

        // Normalize Vietnamese characters
        String normalized = Normalizer.normalize(input, Normalizer.Form.NFD);

        // Remove diacritics
        normalized = normalized.replaceAll("[\\p{InCombiningDiacriticalMarks}]", "");

        // Replace đ/Đ
        normalized = normalized.replace("đ", "d").replace("Đ", "D");

        // Lowercase
        String slug = normalized.toLowerCase(Locale.ROOT);

        // Replace whitespace with hyphen
        slug = WHITESPACE.matcher(slug).replaceAll("-");

        // Remove non-latin characters
        slug = NONLATIN.matcher(slug).replaceAll("");

        // Remove leading/trailing hyphens
        slug = EDGESDHASHES.matcher(slug).replaceAll("");

        // Remove consecutive hyphens
        slug = slug.replaceAll("-+", "-");

        return slug;
    }
}
```

---

## 5. CATEGORY CRUD (Tương tự)

```java
@RestController
@RequestMapping("/api/categories")
@RequiredArgsConstructor
public class CategoryController {

    private final CategoryService categoryService;

    @GetMapping
    public ApiResponse<List<CategoryResponse>> getAllCategories() {
        return ApiResponse.success(categoryService.getAll());
    }

    @GetMapping("/tree")
    public ApiResponse<List<CategoryResponse>> getCategoryTree() {
        return ApiResponse.success(categoryService.getTree());
    }

    @GetMapping("/{id}")
    public ApiResponse<CategoryResponse> getCategory(@PathVariable Long id) {
        return ApiResponse.success(categoryService.getById(id));
    }

    @GetMapping("/slug/{slug}")
    public ApiResponse<CategoryResponse> getCategoryBySlug(@PathVariable String slug) {
        return ApiResponse.success(categoryService.getBySlug(slug));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<CategoryResponse>> createCategory(
            @Valid @RequestBody CategoryCreateRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.created(categoryService.create(request)));
    }

    @PutMapping("/{id}")
    public ApiResponse<CategoryResponse> updateCategory(
            @PathVariable Long id,
            @Valid @RequestBody CategoryUpdateRequest request) {
        return ApiResponse.success(categoryService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ApiResponse<Void> deleteCategory(@PathVariable Long id) {
        categoryService.delete(id);
        return ApiResponse.success("Xóa thành công", null);
    }
}
```

---

## 6. TEST API VỚI CURL

```bash
# ═══ CREATE ═══
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPhone 15 Pro Max",
    "sku": "IP15PM-256",
    "price": 34990000,
    "salePrice": 33990000,
    "description": "Điện thoại Apple iPhone 15 Pro Max",
    "stockQuantity": 100,
    "categoryId": 1,
    "isActive": true,
    "isFeatured": true
  }'

# ═══ GET ALL ═══
curl http://localhost:8080/api/products

# ═══ SEARCH ═══
curl "http://localhost:8080/api/products?keyword=iphone&minPrice=20000000&page=0&size=10"

# ═══ GET BY ID ═══
curl http://localhost:8080/api/products/1

# ═══ GET BY SLUG ═══
curl http://localhost:8080/api/products/slug/iphone-15-pro-max

# ═══ UPDATE ═══
curl -X PUT http://localhost:8080/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{
    "price": 32990000,
    "isFeatured": true
  }'

# ═══ DELETE ═══
curl -X DELETE http://localhost:8080/api/products/1
```

---

## 7. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo Request DTOs (Create, Update, Search)
- [ ] Tạo Response DTOs
- [ ] Implement ProductService với full CRUD
- [ ] Tạo SlugUtils helper
- [ ] Implement soft delete
- [ ] Test tất cả endpoints với Postman/curl
- [ ] Xử lý relationships (Category, Tags, Images)
- [ ] Validate unique constraints (SKU, slug)

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Request DTO vs Entity | ☐ |
| Response DTO mapping | ☐ |
| Service layer pattern | ☐ |
| @Transactional usage | ☐ |
| Slug generation | ☐ |
| Soft delete vs Hard delete | ☐ |
| Handling relationships in CRUD | ☐ |
| Unique constraint validation | ☐ |

---

**Trước đó:** [← Ngày 11 - REST Controller](day-11-rest-controller.md)

**Tiếp theo:** [Ngày 13 - DTO & MapStruct →](day-13-dto-mapstruct.md)

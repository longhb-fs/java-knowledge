# Day 25: Search & Filtering

## Mục tiêu học tập
- Implement full-text search với Elasticsearch
- Tạo dynamic filters với Specifications
- Build faceted navigation
- Optimize search performance

---

## 1. Search Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEARCH ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     USER SEARCH                            │  │
│  │  "laptop gaming 16gb ram under $1500"                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   SEARCH STRATEGIES                        │  │
│  │                                                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                │  │
│  │  │  Simple (MySQL) │  │   Advanced      │                │  │
│  │  │  - LIKE queries │  │ (Elasticsearch) │                │  │
│  │  │  - Good for     │  │ - Full-text     │                │  │
│  │  │    small data   │  │ - Fuzzy match   │                │  │
│  │  │  - Limited      │  │ - Highlighting  │                │  │
│  │  │    features     │  │ - Aggregations  │                │  │
│  │  └─────────────────┘  └─────────────────┘                │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     FILTERING                              │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐            │  │
│  │  │  Category  │ │   Price    │ │   Brand    │            │  │
│  │  │ Electronics│ │ $500-$1500 │ │   Dell     │            │  │
│  │  └────────────┘ └────────────┘ └────────────┘            │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐            │  │
│  │  │   Rating   │ │   Stock    │ │   Sort     │            │  │
│  │  │  4+ stars  │ │  In stock  │ │ Price Low  │            │  │
│  │  └────────────┘ └────────────┘ └────────────┘            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              RESULTS + FACETS (Aggregations)              │  │
│  │                                                            │  │
│  │  Products: 156 results                                    │  │
│  │                                                            │  │
│  │  Categories:          Brands:           Price Ranges:     │  │
│  │  - Laptops (89)      - Dell (34)       - $500-1000 (45)  │  │
│  │  - Desktops (45)     - HP (28)         - $1000-1500 (67) │  │
│  │  - Monitors (22)     - Lenovo (22)     - $1500+ (44)     │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. MySQL Full-Text Search (Simple Approach)

### 2.1 Add Full-Text Index

```sql
-- V13__Add_fulltext_search_index.sql

-- Add FULLTEXT index to products
ALTER TABLE products
ADD FULLTEXT INDEX idx_product_fulltext (name, description);

-- Add FULLTEXT index to categories
ALTER TABLE categories
ADD FULLTEXT INDEX idx_category_fulltext (name, description);
```

### 2.2 Product Search Repository

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;

@Repository
public interface ProductRepository extends
        JpaRepository<Product, Long>,
        JpaSpecificationExecutor<Product> {

    // Simple LIKE search
    @Query("SELECT p FROM Product p WHERE p.active = true AND " +
           "(LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(p.description) LIKE LOWER(CONCAT('%', :keyword, '%')))")
    Page<Product> searchByKeyword(@Param("keyword") String keyword, Pageable pageable);

    // MySQL FULLTEXT search
    @Query(value = "SELECT * FROM products WHERE active = true AND " +
           "MATCH(name, description) AGAINST(:keyword IN NATURAL LANGUAGE MODE)",
           countQuery = "SELECT COUNT(*) FROM products WHERE active = true AND " +
           "MATCH(name, description) AGAINST(:keyword IN NATURAL LANGUAGE MODE)",
           nativeQuery = true)
    Page<Product> fullTextSearch(@Param("keyword") String keyword, Pageable pageable);

    // FULLTEXT with relevance score
    @Query(value = "SELECT *, " +
           "MATCH(name, description) AGAINST(:keyword) AS relevance " +
           "FROM products WHERE active = true AND " +
           "MATCH(name, description) AGAINST(:keyword IN NATURAL LANGUAGE MODE) " +
           "ORDER BY relevance DESC",
           nativeQuery = true)
    List<Product> fullTextSearchWithScore(@Param("keyword") String keyword);

    // Category filter
    Page<Product> findByActiveAndCategoryId(boolean active, Long categoryId, Pageable pageable);

    // Price range
    Page<Product> findByActiveAndPriceBetween(
            boolean active, BigDecimal minPrice, BigDecimal maxPrice, Pageable pageable);

    // Rating filter
    @Query("SELECT p FROM Product p WHERE p.active = true AND " +
           "p.ratingStats.averageRating >= :minRating")
    Page<Product> findByMinimumRating(@Param("minRating") Double minRating, Pageable pageable);
}
```

---

## 3. Dynamic Filtering với Specifications

### 3.1 ProductSearchCriteria

```java
package com.ecommerce.dto.request;

import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
public class ProductSearchCriteria {

    private String keyword;
    private List<Long> categoryIds;
    private List<Long> brandIds;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private Double minRating;
    private Boolean inStock;
    private Boolean onSale;
    private List<String> attributes;  // e.g., "color:red", "size:large"

    private String sortBy = "createdAt";
    private String sortDirection = "desc";
}
```

### 3.2 ProductSpecification

```java
package com.ecommerce.specification;

import com.ecommerce.domain.entity.Product;
import com.ecommerce.domain.entity.Product_;
import com.ecommerce.dto.request.ProductSearchCriteria;
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.util.StringUtils;

import java.util.ArrayList;
import java.util.List;

public class ProductSpecification {

    public static Specification<Product> fromCriteria(ProductSearchCriteria criteria) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            // Always filter active products
            predicates.add(cb.isTrue(root.get("active")));
            predicates.add(cb.isFalse(root.get("deleted")));

            // Keyword search
            if (StringUtils.hasText(criteria.getKeyword())) {
                String keyword = "%" + criteria.getKeyword().toLowerCase() + "%";
                predicates.add(cb.or(
                        cb.like(cb.lower(root.get("name")), keyword),
                        cb.like(cb.lower(root.get("description")), keyword),
                        cb.like(cb.lower(root.get("sku")), keyword)
                ));
            }

            // Category filter
            if (criteria.getCategoryIds() != null && !criteria.getCategoryIds().isEmpty()) {
                predicates.add(root.get("category").get("id").in(criteria.getCategoryIds()));
            }

            // Brand filter
            if (criteria.getBrandIds() != null && !criteria.getBrandIds().isEmpty()) {
                predicates.add(root.get("brand").get("id").in(criteria.getBrandIds()));
            }

            // Price range
            if (criteria.getMinPrice() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("price"), criteria.getMinPrice()));
            }
            if (criteria.getMaxPrice() != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("price"), criteria.getMaxPrice()));
            }

            // Rating filter
            if (criteria.getMinRating() != null) {
                predicates.add(cb.greaterThanOrEqualTo(
                        root.get("ratingStats").get("averageRating"),
                        criteria.getMinRating()
                ));
            }

            // Stock filter
            if (Boolean.TRUE.equals(criteria.getInStock())) {
                predicates.add(cb.greaterThan(root.get("stockQuantity"), 0));
            }

            // On sale filter
            if (Boolean.TRUE.equals(criteria.getOnSale())) {
                predicates.add(cb.isNotNull(root.get("salePrice")));
                predicates.add(cb.lessThan(root.get("salePrice"), root.get("price")));
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }

    // Individual specifications for composition
    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<Product> hasKeyword(String keyword) {
        return (root, query, cb) -> {
            if (!StringUtils.hasText(keyword)) {
                return cb.conjunction();
            }
            String pattern = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                    cb.like(cb.lower(root.get("name")), pattern),
                    cb.like(cb.lower(root.get("description")), pattern)
            );
        };
    }

    public static Specification<Product> inCategory(Long categoryId) {
        return (root, query, cb) -> {
            if (categoryId == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("category").get("id"), categoryId);
        };
    }

    public static Specification<Product> inCategories(List<Long> categoryIds) {
        return (root, query, cb) -> {
            if (categoryIds == null || categoryIds.isEmpty()) {
                return cb.conjunction();
            }
            return root.get("category").get("id").in(categoryIds);
        };
    }

    public static Specification<Product> priceBetween(java.math.BigDecimal min, java.math.BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) {
                return cb.conjunction();
            }
            if (min != null && max != null) {
                return cb.between(root.get("price"), min, max);
            }
            if (min != null) {
                return cb.greaterThanOrEqualTo(root.get("price"), min);
            }
            return cb.lessThanOrEqualTo(root.get("price"), max);
        };
    }

    public static Specification<Product> hasMinRating(Double minRating) {
        return (root, query, cb) -> {
            if (minRating == null) {
                return cb.conjunction();
            }
            return cb.greaterThanOrEqualTo(
                    root.get("ratingStats").get("averageRating"),
                    minRating
            );
        };
    }

    public static Specification<Product> inStock() {
        return (root, query, cb) ->
                cb.greaterThan(root.get("stockQuantity"), 0);
    }
}
```

---

## 4. Search Service

### 4.1 ProductSearchService

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.ProductSearchCriteria;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.dto.response.SearchResultDto;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.util.List;

public interface ProductSearchService {

    SearchResultDto search(ProductSearchCriteria criteria, Pageable pageable);

    List<ProductDto> autocomplete(String query, int limit);

    List<String> getSuggestions(String query);
}
```

### 4.2 ProductSearchServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.Product;
import com.ecommerce.dto.request.ProductSearchCriteria;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.dto.response.SearchResultDto;
import com.ecommerce.mapper.ProductMapper;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.service.ProductSearchService;
import com.ecommerce.specification.ProductSpecification;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProductSearchServiceImpl implements ProductSearchService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    @Override
    @Transactional(readOnly = true)
    public SearchResultDto search(ProductSearchCriteria criteria, Pageable pageable) {
        log.debug("Searching products with criteria: {}", criteria);

        // Build specification
        Specification<Product> spec = ProductSpecification.fromCriteria(criteria);

        // Build sort
        Sort sort = buildSort(criteria.getSortBy(), criteria.getSortDirection());
        Pageable sortedPageable = PageRequest.of(
                pageable.getPageNumber(),
                pageable.getPageSize(),
                sort
        );

        // Execute search
        Page<Product> products = productRepository.findAll(spec, sortedPageable);

        // Get facets
        Map<String, List<SearchResultDto.FacetValue>> facets = calculateFacets(criteria);

        // Build result
        return SearchResultDto.builder()
                .products(products.map(productMapper::toDto))
                .totalElements(products.getTotalElements())
                .totalPages(products.getTotalPages())
                .facets(facets)
                .appliedFilters(buildAppliedFilters(criteria))
                .build();
    }

    @Override
    @Transactional(readOnly = true)
    @Cacheable(value = "autocomplete", key = "#query")
    public List<ProductDto> autocomplete(String query, int limit) {
        if (query == null || query.length() < 2) {
            return Collections.emptyList();
        }

        Page<Product> products = productRepository.searchByKeyword(
                query,
                PageRequest.of(0, limit)
        );

        return products.stream()
                .map(productMapper::toDto)
                .collect(Collectors.toList());
    }

    @Override
    @Transactional(readOnly = true)
    @Cacheable(value = "suggestions", key = "#query")
    public List<String> getSuggestions(String query) {
        if (query == null || query.length() < 2) {
            return Collections.emptyList();
        }

        // Get product names matching query
        return productRepository.searchByKeyword(query, PageRequest.of(0, 10))
                .stream()
                .map(Product::getName)
                .distinct()
                .limit(5)
                .collect(Collectors.toList());
    }

    // ========== Private Methods ==========

    private Sort buildSort(String sortBy, String direction) {
        Sort.Direction dir = "asc".equalsIgnoreCase(direction)
                ? Sort.Direction.ASC
                : Sort.Direction.DESC;

        return switch (sortBy != null ? sortBy.toLowerCase() : "") {
            case "price" -> Sort.by(dir, "price");
            case "name" -> Sort.by(dir, "name");
            case "rating" -> Sort.by(dir, "ratingStats.averageRating");
            case "popularity" -> Sort.by(dir, "salesCount");
            case "newest" -> Sort.by(Sort.Direction.DESC, "createdAt");
            default -> Sort.by(Sort.Direction.DESC, "createdAt");
        };
    }

    private Map<String, List<SearchResultDto.FacetValue>> calculateFacets(
            ProductSearchCriteria criteria) {

        Map<String, List<SearchResultDto.FacetValue>> facets = new HashMap<>();

        // Category facets
        facets.put("categories", getCategoryFacets(criteria));

        // Brand facets
        facets.put("brands", getBrandFacets(criteria));

        // Price range facets
        facets.put("priceRanges", getPriceRangeFacets(criteria));

        // Rating facets
        facets.put("ratings", getRatingFacets(criteria));

        return facets;
    }

    private List<SearchResultDto.FacetValue> getCategoryFacets(ProductSearchCriteria criteria) {
        // Clone criteria without category filter to count all categories
        Specification<Product> spec = ProductSpecification.fromCriteria(
                cloneCriteriaWithout(criteria, "category"));

        // Group by category and count
        // In a real app, this would be a custom query
        List<Object[]> categoryCounts = productRepository.findAll(spec).stream()
                .filter(p -> p.getCategory() != null)
                .collect(Collectors.groupingBy(
                        p -> p.getCategory().getId(),
                        Collectors.counting()
                ))
                .entrySet().stream()
                .map(e -> new Object[]{e.getKey(), e.getValue()})
                .toList();

        return categoryCounts.stream()
                .map(arr -> SearchResultDto.FacetValue.builder()
                        .value(arr[0].toString())
                        .count(((Long) arr[1]).intValue())
                        .build())
                .collect(Collectors.toList());
    }

    private List<SearchResultDto.FacetValue> getBrandFacets(ProductSearchCriteria criteria) {
        // Similar implementation to getCategoryFacets
        return Collections.emptyList();
    }

    private List<SearchResultDto.FacetValue> getPriceRangeFacets(ProductSearchCriteria criteria) {
        List<SearchResultDto.FacetValue> ranges = new ArrayList<>();

        // Define price ranges
        BigDecimal[][] priceRanges = {
                {BigDecimal.ZERO, new BigDecimal("50")},
                {new BigDecimal("50"), new BigDecimal("100")},
                {new BigDecimal("100"), new BigDecimal("500")},
                {new BigDecimal("500"), new BigDecimal("1000")},
                {new BigDecimal("1000"), null}
        };

        for (BigDecimal[] range : priceRanges) {
            ProductSearchCriteria rangeCriteria = cloneCriteriaWithout(criteria, "price");
            rangeCriteria.setMinPrice(range[0]);
            rangeCriteria.setMaxPrice(range[1]);

            Specification<Product> spec = ProductSpecification.fromCriteria(rangeCriteria);
            long count = productRepository.count(spec);

            String label = range[1] != null
                    ? String.format("$%s - $%s", range[0], range[1])
                    : String.format("$%s+", range[0]);

            ranges.add(SearchResultDto.FacetValue.builder()
                    .value(label)
                    .min(range[0])
                    .max(range[1])
                    .count((int) count)
                    .build());
        }

        return ranges;
    }

    private List<SearchResultDto.FacetValue> getRatingFacets(ProductSearchCriteria criteria) {
        List<SearchResultDto.FacetValue> ratings = new ArrayList<>();

        for (int stars = 4; stars >= 1; stars--) {
            ProductSearchCriteria ratingCriteria = cloneCriteriaWithout(criteria, "rating");
            ratingCriteria.setMinRating((double) stars);

            Specification<Product> spec = ProductSpecification.fromCriteria(ratingCriteria);
            long count = productRepository.count(spec);

            ratings.add(SearchResultDto.FacetValue.builder()
                    .value(stars + "+ stars")
                    .count((int) count)
                    .build());
        }

        return ratings;
    }

    private ProductSearchCriteria cloneCriteriaWithout(ProductSearchCriteria criteria, String exclude) {
        ProductSearchCriteria clone = new ProductSearchCriteria();
        clone.setKeyword(criteria.getKeyword());

        if (!"category".equals(exclude)) {
            clone.setCategoryIds(criteria.getCategoryIds());
        }
        if (!"brand".equals(exclude)) {
            clone.setBrandIds(criteria.getBrandIds());
        }
        if (!"price".equals(exclude)) {
            clone.setMinPrice(criteria.getMinPrice());
            clone.setMaxPrice(criteria.getMaxPrice());
        }
        if (!"rating".equals(exclude)) {
            clone.setMinRating(criteria.getMinRating());
        }
        clone.setInStock(criteria.getInStock());
        clone.setOnSale(criteria.getOnSale());

        return clone;
    }

    private List<SearchResultDto.AppliedFilter> buildAppliedFilters(ProductSearchCriteria criteria) {
        List<SearchResultDto.AppliedFilter> filters = new ArrayList<>();

        if (criteria.getKeyword() != null) {
            filters.add(new SearchResultDto.AppliedFilter("keyword", criteria.getKeyword()));
        }
        if (criteria.getMinPrice() != null || criteria.getMaxPrice() != null) {
            String priceLabel = String.format("$%s - $%s",
                    criteria.getMinPrice() != null ? criteria.getMinPrice() : "0",
                    criteria.getMaxPrice() != null ? criteria.getMaxPrice() : "∞");
            filters.add(new SearchResultDto.AppliedFilter("price", priceLabel));
        }
        if (criteria.getMinRating() != null) {
            filters.add(new SearchResultDto.AppliedFilter("rating", criteria.getMinRating() + "+ stars"));
        }
        if (Boolean.TRUE.equals(criteria.getInStock())) {
            filters.add(new SearchResultDto.AppliedFilter("stock", "In Stock"));
        }

        return filters;
    }
}
```

---

## 5. Search Response DTO

```java
package com.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.domain.Page;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SearchResultDto {

    private Page<ProductDto> products;
    private long totalElements;
    private int totalPages;
    private Map<String, List<FacetValue>> facets;
    private List<AppliedFilter> appliedFilters;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class FacetValue {
        private String value;
        private String label;
        private int count;
        private BigDecimal min;
        private BigDecimal max;
        private boolean selected;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AppliedFilter {
        private String type;
        private String value;
    }
}
```

---

## 6. Search Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.ProductSearchCriteria;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.dto.response.SearchResultDto;
import com.ecommerce.service.ProductSearchService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/v1/search")
@RequiredArgsConstructor
@Tag(name = "Search", description = "Product search APIs")
public class SearchController {

    private final ProductSearchService searchService;

    @GetMapping
    @Operation(summary = "Search products with filters")
    public ResponseEntity<ApiResponse<SearchResultDto>> search(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) List<Long> categories,
            @RequestParam(required = false) List<Long> brands,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(required = false) Double minRating,
            @RequestParam(required = false) Boolean inStock,
            @RequestParam(required = false) Boolean onSale,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDirection,
            @PageableDefault(size = 20) Pageable pageable) {

        ProductSearchCriteria criteria = new ProductSearchCriteria();
        criteria.setKeyword(q);
        criteria.setCategoryIds(categories);
        criteria.setBrandIds(brands);
        criteria.setMinPrice(minPrice);
        criteria.setMaxPrice(maxPrice);
        criteria.setMinRating(minRating);
        criteria.setInStock(inStock);
        criteria.setOnSale(onSale);
        criteria.setSortBy(sortBy);
        criteria.setSortDirection(sortDirection);

        SearchResultDto result = searchService.search(criteria, pageable);
        return ResponseEntity.ok(ApiResponse.success(result));
    }

    @GetMapping("/autocomplete")
    @Operation(summary = "Get autocomplete suggestions")
    public ResponseEntity<ApiResponse<List<ProductDto>>> autocomplete(
            @RequestParam String q,
            @RequestParam(defaultValue = "5") int limit) {

        List<ProductDto> suggestions = searchService.autocomplete(q, limit);
        return ResponseEntity.ok(ApiResponse.success(suggestions));
    }

    @GetMapping("/suggestions")
    @Operation(summary = "Get search query suggestions")
    public ResponseEntity<ApiResponse<List<String>>> suggestions(@RequestParam String q) {
        List<String> suggestions = searchService.getSuggestions(q);
        return ResponseEntity.ok(ApiResponse.success(suggestions));
    }
}
```

---

## 7. Elasticsearch Integration (Advanced)

### 7.1 Dependencies

```xml
<dependencies>
    <!-- Elasticsearch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>
</dependencies>
```

### 7.2 Elasticsearch Document

```java
package com.ecommerce.search.document;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Document(indexName = "products")
@Setting(settingPath = "elasticsearch/product-settings.json")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductDocument {

    @Id
    private Long id;

    @MultiField(mainField = @Field(type = FieldType.Text, analyzer = "standard"),
            otherFields = {
                    @InnerField(suffix = "keyword", type = FieldType.Keyword),
                    @InnerField(suffix = "autocomplete", type = FieldType.Text,
                            analyzer = "autocomplete", searchAnalyzer = "standard")
            })
    private String name;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String description;

    @Field(type = FieldType.Keyword)
    private String sku;

    @Field(type = FieldType.Keyword)
    private String slug;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Double)
    private BigDecimal salePrice;

    @Field(type = FieldType.Long)
    private Long categoryId;

    @Field(type = FieldType.Keyword)
    private String categoryName;

    @Field(type = FieldType.Long)
    private Long brandId;

    @Field(type = FieldType.Keyword)
    private String brandName;

    @Field(type = FieldType.Double)
    private Double averageRating;

    @Field(type = FieldType.Integer)
    private Integer totalReviews;

    @Field(type = FieldType.Integer)
    private Integer stockQuantity;

    @Field(type = FieldType.Boolean)
    private Boolean active;

    @Field(type = FieldType.Keyword)
    private String thumbnailUrl;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Nested)
    private List<AttributeValue> attributes;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdAt;

    @Field(type = FieldType.Integer)
    private Integer salesCount;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AttributeValue {
        @Field(type = FieldType.Keyword)
        private String name;

        @Field(type = FieldType.Keyword)
        private String value;
    }
}
```

### 7.3 Elasticsearch Repository

```java
package com.ecommerce.search.repository;

import com.ecommerce.search.document.ProductDocument;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, Long> {
}
```

### 7.4 Elasticsearch Service

```java
package com.ecommerce.search.service;

import co.elastic.clients.elasticsearch._types.query_dsl.*;
import com.ecommerce.dto.request.ProductSearchCriteria;
import com.ecommerce.dto.response.SearchResultDto;
import com.ecommerce.search.document.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.client.elc.NativeQueryBuilder;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.HighlightQuery;
import org.springframework.data.elasticsearch.core.query.highlight.Highlight;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightField;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;

import java.util.ArrayList;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class ElasticsearchProductService {

    private final ElasticsearchOperations elasticsearchOperations;

    public SearchResultDto search(ProductSearchCriteria criteria, Pageable pageable) {

        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Must be active
        boolQuery.filter(TermQuery.of(t -> t.field("active").value(true))._toQuery());

        // Keyword search
        if (StringUtils.hasText(criteria.getKeyword())) {
            boolQuery.must(MultiMatchQuery.of(m -> m
                    .query(criteria.getKeyword())
                    .fields("name^3", "name.autocomplete^2", "description", "tags")
                    .type(TextQueryType.BestFields)
                    .fuzziness("AUTO")
            )._toQuery());
        }

        // Category filter
        if (criteria.getCategoryIds() != null && !criteria.getCategoryIds().isEmpty()) {
            boolQuery.filter(TermsQuery.of(t -> t
                    .field("categoryId")
                    .terms(ts -> ts.value(
                            criteria.getCategoryIds().stream()
                                    .map(id -> co.elastic.clients.elasticsearch._types.FieldValue.of(id))
                                    .toList()
                    ))
            )._toQuery());
        }

        // Price range
        if (criteria.getMinPrice() != null || criteria.getMaxPrice() != null) {
            RangeQuery.Builder rangeQuery = new RangeQuery.Builder().field("price");
            if (criteria.getMinPrice() != null) {
                rangeQuery.gte(co.elastic.clients.json.JsonData.of(criteria.getMinPrice()));
            }
            if (criteria.getMaxPrice() != null) {
                rangeQuery.lte(co.elastic.clients.json.JsonData.of(criteria.getMaxPrice()));
            }
            boolQuery.filter(rangeQuery.build()._toQuery());
        }

        // Rating filter
        if (criteria.getMinRating() != null) {
            boolQuery.filter(RangeQuery.of(r -> r
                    .field("averageRating")
                    .gte(co.elastic.clients.json.JsonData.of(criteria.getMinRating()))
            )._toQuery());
        }

        // In stock filter
        if (Boolean.TRUE.equals(criteria.getInStock())) {
            boolQuery.filter(RangeQuery.of(r -> r
                    .field("stockQuantity")
                    .gt(co.elastic.clients.json.JsonData.of(0))
            )._toQuery());
        }

        // Build query
        NativeQuery searchQuery = new NativeQueryBuilder()
                .withQuery(boolQuery.build()._toQuery())
                .withPageable(pageable)
                .withHighlightQuery(new HighlightQuery(
                        new Highlight(List.of(
                                new HighlightField("name"),
                                new HighlightField("description")
                        )),
                        ProductDocument.class
                ))
                .build();

        SearchHits<ProductDocument> searchHits = elasticsearchOperations.search(
                searchQuery, ProductDocument.class);

        // Convert to DTOs
        // ... (implementation similar to previous)

        return SearchResultDto.builder()
                // ... build result
                .build();
    }

    public List<String> autocomplete(String query, int limit) {
        NativeQuery searchQuery = new NativeQueryBuilder()
                .withQuery(MultiMatchQuery.of(m -> m
                        .query(query)
                        .fields("name.autocomplete", "name")
                        .type(TextQueryType.BoolPrefix)
                )._toQuery())
                .withPageable(PageRequest.of(0, limit))
                .build();

        SearchHits<ProductDocument> hits = elasticsearchOperations.search(
                searchQuery, ProductDocument.class);

        return hits.getSearchHits().stream()
                .map(SearchHit::getContent)
                .map(ProductDocument::getName)
                .toList();
    }
}
```

---

## 8. Search Settings JSON

```json
// src/main/resources/elasticsearch/product-settings.json
{
  "analysis": {
    "analyzer": {
      "autocomplete": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "autocomplete_filter"]
      }
    },
    "filter": {
      "autocomplete_filter": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 20
      }
    }
  }
}
```

---

## 9. Sync Products to Elasticsearch

```java
package com.ecommerce.search.sync;

import com.ecommerce.domain.entity.Product;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.search.document.ProductDocument;
import com.ecommerce.search.repository.ProductSearchRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProductIndexer {

    private final ProductRepository productRepository;
    private final ProductSearchRepository searchRepository;

    /**
     * Sync single product to Elasticsearch
     */
    public void indexProduct(Product product) {
        ProductDocument document = mapToDocument(product);
        searchRepository.save(document);
        log.debug("Indexed product: {}", product.getId());
    }

    /**
     * Remove product from Elasticsearch
     */
    public void deleteProduct(Long productId) {
        searchRepository.deleteById(productId);
        log.debug("Deleted product from index: {}", productId);
    }

    /**
     * Full reindex - runs nightly
     */
    @Scheduled(cron = "0 0 2 * * ?")  // 2 AM
    @Transactional(readOnly = true)
    public void reindexAll() {
        log.info("Starting full product reindex");

        // Delete all
        searchRepository.deleteAll();

        // Reindex all active products
        Iterable<ProductDocument> documents = productRepository
                .findAllByActiveTrue()
                .stream()
                .map(this::mapToDocument)
                .collect(Collectors.toList());

        searchRepository.saveAll(documents);

        log.info("Full reindex completed");
    }

    private ProductDocument mapToDocument(Product product) {
        return ProductDocument.builder()
                .id(product.getId())
                .name(product.getName())
                .description(product.getDescription())
                .sku(product.getSku())
                .slug(product.getSlug())
                .price(product.getPrice())
                .salePrice(product.getSalePrice())
                .categoryId(product.getCategory() != null ? product.getCategory().getId() : null)
                .categoryName(product.getCategory() != null ? product.getCategory().getName() : null)
                .brandId(product.getBrand() != null ? product.getBrand().getId() : null)
                .brandName(product.getBrand() != null ? product.getBrand().getName() : null)
                .averageRating(product.getRatingStats() != null
                        ? product.getRatingStats().getAverageRating() : 0.0)
                .totalReviews(product.getRatingStats() != null
                        ? product.getRatingStats().getTotalReviews() : 0)
                .stockQuantity(product.getStockQuantity())
                .active(product.isActive())
                .thumbnailUrl(product.getThumbnailUrl())
                .createdAt(product.getCreatedAt())
                .salesCount(product.getSalesCount())
                .build();
    }
}
```

---

## 10. Bài tập thực hành

### Bài tập 1: Basic Search
- [x] Implement MySQL LIKE search
- [x] Add full-text search index
- [x] Create ProductSpecification

### Bài tập 2: Faceted Search
- [x] Calculate category facets
- [x] Calculate price range facets
- [x] Calculate rating facets

### Bài tập 3: Elasticsearch (Optional)
- [ ] Setup Elasticsearch
- [ ] Create ProductDocument
- [ ] Implement search with fuzzy matching

---

## 11. Checklist

- [ ] Add MySQL full-text index
- [ ] Create ProductSearchCriteria DTO
- [ ] Implement ProductSpecification
- [ ] Create ProductSearchService
- [ ] Implement facet calculation
- [ ] Create SearchController
- [ ] Add caching for autocomplete
- [ ] (Optional) Setup Elasticsearch
- [ ] (Optional) Implement product indexer
- [ ] Test search with various filters

---

## Điều hướng

- [← Day 24: Reviews & Ratings](./day-24-reviews-ratings.md)
- [Day 26: Unit Testing →](../week-06/day-26-unit-testing.md)
- [Về trang chính](../00-overview.md)

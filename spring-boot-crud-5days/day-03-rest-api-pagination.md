# Day 3: REST API + Pagination

## Mục tiêu
- Tạo REST Controller
- DTO pattern với MapStruct
- Validation
- **Pagination response** chuẩn cho frontend

## 1. Controller - So sánh với ASP.NET Core

### 1.1. ASP.NET Core Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class MarketInfoController : ControllerBase
{
    [HttpGet]
    public ActionResult<List<MarketInfoDto>> GetAll() { }

    [HttpGet("{id}")]
    public ActionResult<MarketInfoDto> GetById(long id) { }

    [HttpPost]
    public ActionResult<MarketInfoDto> Create([FromBody] CreateRequest request) { }
}
```

### 1.2. Spring Boot Controller

```java
@RestController
@RequestMapping("/api/market-info")
@RequiredArgsConstructor
@Validated
public class MarketInfoController {

    private final MarketInfoService service;

    @GetMapping
    public ResponseEntity<PageResponse<MarketInfoResponse>> search(
            @Valid MarketInfoSearchRequest request) {
        Page<MarketInfoResponse> result = service.search(request);
        return ResponseEntity.ok(PageResponse.of(result));
    }

    @GetMapping("/{id}")
    public ResponseEntity<MarketInfoResponse> getById(@PathVariable Long id) {
        MarketInfoResponse result = service.findById(id);
        return ResponseEntity.ok(result);
    }

    @PostMapping
    public ResponseEntity<MarketInfoResponse> create(
            @Valid @RequestBody MarketInfoCreateRequest request) {
        MarketInfoResponse result = service.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }
}
```

### 1.3. So sánh Annotation

| ASP.NET Core | Spring Boot | Mô tả |
|--------------|-------------|-------|
| `[ApiController]` | `@RestController` | REST controller |
| `[Route("api/...")]` | `@RequestMapping("/api/...")` | Base route |
| `[HttpGet]` | `@GetMapping` | GET method |
| `[HttpPost]` | `@PostMapping` | POST method |
| `[FromBody]` | `@RequestBody` | Request body |
| `[FromQuery]` | `@RequestParam` hoặc DTO | Query params |
| `ActionResult<T>` | `ResponseEntity<T>` | Response wrapper |

## 2. DTOs

### 2.1. Response DTO

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MarketInfoResponse {
    private Long id;
    private String indexCode;
    private BigDecimal value;
    private Long tradingVolume;
    private BigDecimal tradingValue;
    private Long foreignBuyVolume;
    private BigDecimal foreignBuyValue;
    private Long foreignSellVolume;
    private BigDecimal foreignSellValue;
    private BigDecimal netBuySell;
    private BigDecimal change5Sessions;
    private BigDecimal change10Sessions;
    private BigDecimal change30Sessions;
    private Integer stocksUp;
    private Integer stocksDown;
    private Integer stocksUnchanged;
    private LocalDate tradingDate;
}
```

### 2.2. Request DTO với Validation

```java
@Data
public class MarketInfoSearchRequest {

    // Quick search
    @Size(max = 100, message = "Keyword không quá 100 ký tự")
    private String keyword;

    // Filters
    @Size(max = 50, message = "Index code không quá 50 ký tự")
    private String indexCode;

    @DecimalMin(value = "0", message = "Giá trị phải >= 0")
    private BigDecimal minValue;

    @DecimalMax(value = "999999999", message = "Giá trị không hợp lệ")
    private BigDecimal maxValue;

    // Pagination
    @Min(value = 0, message = "Page phải >= 0")
    private Integer page = 0;

    @Min(value = 1, message = "Size phải >= 1")
    @Max(value = 100, message = "Size tối đa 100")
    private Integer size = 10;

    @Pattern(regexp = "^(indexCode|value|tradingDate)$", message = "Sort field không hợp lệ")
    private String sortBy = "indexCode";

    @Pattern(regexp = "^(asc|desc)$", message = "Sort direction phải là asc hoặc desc")
    private String sortDir = "asc";

    // Helper method để tạo Pageable
    public Pageable toPageable() {
        Sort sort = Sort.by(
            sortDir.equalsIgnoreCase("desc") ? Sort.Direction.DESC : Sort.Direction.ASC,
            sortBy
        );
        return PageRequest.of(page, size, sort);
    }
}
```

### 2.3. Create Request DTO

```java
@Data
public class MarketInfoCreateRequest {

    @NotBlank(message = "Index code không được trống")
    @Size(max = 50)
    private String indexCode;

    @NotNull(message = "Value không được null")
    @DecimalMin(value = "0")
    private BigDecimal value;

    private Long tradingVolume;
    private BigDecimal tradingValue;

    @NotNull(message = "Trading date không được null")
    private LocalDate tradingDate;
}
```

## 3. Pagination Response - Chuẩn cho Frontend

### 3.1. Page Response Wrapper

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PageResponse<T> {
    private List<T> items;
    private int currentPage;
    private int totalPages;
    private long totalCount;
    private int pageSize;
    private boolean hasNext;
    private boolean hasPrevious;

    /**
     * Convert từ Spring Page sang PageResponse
     */
    public static <T> PageResponse<T> of(Page<T> page) {
        return PageResponse.<T>builder()
            .items(page.getContent())
            .currentPage(page.getNumber())
            .totalPages(page.getTotalPages())
            .totalCount(page.getTotalElements())
            .pageSize(page.getSize())
            .hasNext(page.hasNext())
            .hasPrevious(page.hasPrevious())
            .build();
    }
}
```

### 3.2. JSON Response Format

```json
{
  "items": [
    {
      "id": 1,
      "indexCode": "HNX30",
      "value": 200.000,
      "tradingVolume": 1000000,
      ...
    }
  ],
  "currentPage": 0,
  "totalPages": 8,
  "totalCount": 80,
  "pageSize": 10,
  "hasNext": true,
  "hasPrevious": false
}
```

## 4. MapStruct - DTO Mapping

### 4.1. Dependency

```xml
<properties>
    <mapstruct.version>1.6.3</mapstruct.version>
</properties>

<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${mapstruct.version}</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>${mapstruct.version}</version>
    <scope>provided</scope>
</dependency>
```

### 4.2. Mapper Interface

```java
@Mapper(componentModel = "spring")
public interface MarketInfoMapper {

    // Entity -> Response
    MarketInfoResponse toResponse(MarketInfo entity);

    // List Entity -> List Response
    List<MarketInfoResponse> toResponseList(List<MarketInfo> entities);

    // Page Entity -> Page Response
    default Page<MarketInfoResponse> toResponsePage(Page<MarketInfo> page) {
        return page.map(this::toResponse);
    }

    // Request -> Entity
    MarketInfo toEntity(MarketInfoCreateRequest request);

    // Update entity from request
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntity(@MappingTarget MarketInfo entity, MarketInfoUpdateRequest request);
}
```

### 4.3. So sánh với AutoMapper (.NET)

```csharp
// .NET AutoMapper
CreateMap<MarketInfo, MarketInfoDto>();
var dto = _mapper.Map<MarketInfoDto>(entity);
```

```java
// Spring MapStruct
@Autowired
private MarketInfoMapper mapper;
var dto = mapper.toResponse(entity);
```

## 5. Validation

### 5.1. Common Validation Annotations

| Annotation | Mô tả |
|------------|-------|
| `@NotNull` | Không được null |
| `@NotBlank` | Không được null/empty/whitespace (String) |
| `@NotEmpty` | Không được null/empty (Collection, String) |
| `@Size(min, max)` | Độ dài String/Collection |
| `@Min`, `@Max` | Giá trị số min/max |
| `@Email` | Email format |
| `@Pattern` | Regex pattern |
| `@Past`, `@Future` | Date trong quá khứ/tương lai |

### 5.2. Exception Handler cho Validation

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .toList();

        ErrorResponse response = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .message("Validation failed")
            .errors(errors)
            .timestamp(LocalDateTime.now())
            .build();

        return ResponseEntity.badRequest().body(response);
    }

    // Handle resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse response = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .message(ex.getMessage())
            .timestamp(LocalDateTime.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }

    // Handle generic exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        ErrorResponse response = ErrorResponse.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .message("Internal server error")
            .timestamp(LocalDateTime.now())
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}

@Data
@Builder
public class ErrorResponse {
    private int status;
    private String message;
    private List<String> errors;
    private LocalDateTime timestamp;
}
```

## 6. Ví dụ hoàn chỉnh

### 6.1. Controller đầy đủ

```java
@RestController
@RequestMapping("/api/market-info")
@RequiredArgsConstructor
@Validated
@Tag(name = "Market Info", description = "Quản lý thông tin thị trường")
public class MarketInfoController {

    private final MarketInfoService service;

    /**
     * Tra cứu thông tin thị trường (có phân trang + filter)
     */
    @GetMapping
    @Operation(summary = "Tra cứu thông tin thị trường")
    public ResponseEntity<PageResponse<MarketInfoResponse>> search(
            @Valid MarketInfoSearchRequest request) {
        Page<MarketInfoResponse> result = service.search(request);
        return ResponseEntity.ok(PageResponse.of(result));
    }

    /**
     * Xem chi tiết thông tin thị trường
     */
    @GetMapping("/{id}")
    @Operation(summary = "Xem chi tiết thông tin thị trường")
    public ResponseEntity<MarketInfoResponse> getById(
            @PathVariable @Min(1) Long id) {
        MarketInfoResponse result = service.findById(id);
        return ResponseEntity.ok(result);
    }

    /**
     * Tạo mới thông tin thị trường
     */
    @PostMapping
    @Operation(summary = "Tạo mới thông tin thị trường")
    public ResponseEntity<MarketInfoResponse> create(
            @Valid @RequestBody MarketInfoCreateRequest request) {
        MarketInfoResponse result = service.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }

    /**
     * Cập nhật thông tin thị trường
     */
    @PutMapping("/{id}")
    @Operation(summary = "Cập nhật thông tin thị trường")
    public ResponseEntity<MarketInfoResponse> update(
            @PathVariable @Min(1) Long id,
            @Valid @RequestBody MarketInfoUpdateRequest request) {
        MarketInfoResponse result = service.update(id, request);
        return ResponseEntity.ok(result);
    }

    /**
     * Xóa thông tin thị trường
     */
    @DeleteMapping("/{id}")
    @Operation(summary = "Xóa thông tin thị trường")
    public ResponseEntity<Void> delete(@PathVariable @Min(1) Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 6.2. Service Implementation

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class MarketInfoServiceImpl implements MarketInfoService {

    private final MarketInfoRepository repository;
    private final MarketInfoMapper mapper;

    @Override
    public Page<MarketInfoResponse> search(MarketInfoSearchRequest request) {
        log.info("Searching market info: {}", request);

        Specification<MarketInfo> spec = MarketInfoSpecification.fromRequest(request);
        Page<MarketInfo> page = repository.findAll(spec, request.toPageable());

        return mapper.toResponsePage(page);
    }

    @Override
    public MarketInfoResponse findById(Long id) {
        MarketInfo entity = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("MarketInfo", "id", id));
        return mapper.toResponse(entity);
    }

    @Override
    @Transactional
    public MarketInfoResponse create(MarketInfoCreateRequest request) {
        MarketInfo entity = mapper.toEntity(request);
        entity = repository.save(entity);
        return mapper.toResponse(entity);
    }

    @Override
    @Transactional
    public MarketInfoResponse update(Long id, MarketInfoUpdateRequest request) {
        MarketInfo entity = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("MarketInfo", "id", id));

        mapper.updateEntity(entity, request);
        entity = repository.save(entity);

        return mapper.toResponse(entity);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!repository.existsById(id)) {
            throw new ResourceNotFoundException("MarketInfo", "id", id);
        }
        repository.deleteById(id);
    }
}
```

## 7. API Documentation với OpenAPI

### 7.1. Dependency

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

### 7.2. Configuration

```yaml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
```

Truy cập Swagger UI: `http://localhost:8080/swagger-ui.html`

## 8. Tổng kết Day 3

| Concept | C#/.NET | Java Spring |
|---------|---------|-------------|
| Controller | `ControllerBase` | `@RestController` |
| Route | `[Route]` | `@RequestMapping` |
| Response | `ActionResult<T>` | `ResponseEntity<T>` |
| Validation | `DataAnnotations` | `Jakarta Validation` |
| DTO Mapping | AutoMapper | MapStruct |
| API Docs | Swagger/NSwag | SpringDoc OpenAPI |

### Key Takeaways
1. **PageResponse wrapper** - Format chuẩn cho frontend
2. **Validation** - Validate ở DTO level
3. **Exception Handler** - Xử lý lỗi tập trung
4. **MapStruct** - Compile-time mapping, performance tốt hơn reflection

## Navigation

- [← Day 2: JPA Specification](./day-02-jpa-specification.md)
- [Day 4: Export Excel CSV →](./day-04-export-excel-csv.md)

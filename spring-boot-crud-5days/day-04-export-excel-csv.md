# Day 4: Export Excel/CSV

## Mục tiêu
- Export dữ liệu ra Excel (Apache POI)
- Export dữ liệu ra CSV (OpenCSV)
- Download file từ API
- Áp dụng cho chức năng "Xuất file" trong SRS

## 1. Dependencies

```xml
<!-- Apache POI for Excel -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.3.0</version>
</dependency>

<!-- OpenCSV for CSV -->
<dependency>
    <groupId>com.opencsv</groupId>
    <artifactId>opencsv</artifactId>
    <version>5.10</version>
</dependency>
```

## 2. Export Excel với Apache POI

### 2.1. Excel Export Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ExcelExportService {

    /**
     * Export danh sách MarketInfo ra Excel
     */
    public ByteArrayInputStream exportMarketInfo(List<MarketInfoResponse> data) {
        try (Workbook workbook = new XSSFWorkbook()) {
            Sheet sheet = workbook.createSheet("Thông tin thị trường");

            // Create header style
            CellStyle headerStyle = createHeaderStyle(workbook);

            // Create header row
            Row headerRow = sheet.createRow(0);
            String[] headers = {
                "STT", "Chỉ số", "Giá trị", "KL giao dịch", "GT giao dịch",
                "KL NĐTNN mua", "GT NĐTNN mua", "KL NĐTNN bán", "GT NĐTNN bán",
                "Mua/bán ròng", "Biến động 5 phiên", "Biến động 10 phiên",
                "Biến động 30 phiên", "Ngày giao dịch"
            };

            for (int i = 0; i < headers.length; i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue(headers[i]);
                cell.setCellStyle(headerStyle);
            }

            // Create data rows
            CellStyle numberStyle = createNumberStyle(workbook);
            CellStyle dateStyle = createDateStyle(workbook);

            int rowNum = 1;
            for (MarketInfoResponse item : data) {
                Row row = sheet.createRow(rowNum);

                row.createCell(0).setCellValue(rowNum);  // STT
                row.createCell(1).setCellValue(item.getIndexCode());

                // Số với format
                createNumberCell(row, 2, item.getValue(), numberStyle);
                createNumberCell(row, 3, item.getTradingVolume(), numberStyle);
                createNumberCell(row, 4, item.getTradingValue(), numberStyle);
                createNumberCell(row, 5, item.getForeignBuyVolume(), numberStyle);
                createNumberCell(row, 6, item.getForeignBuyValue(), numberStyle);
                createNumberCell(row, 7, item.getForeignSellVolume(), numberStyle);
                createNumberCell(row, 8, item.getForeignSellValue(), numberStyle);
                createNumberCell(row, 9, item.getNetBuySell(), numberStyle);
                createNumberCell(row, 10, item.getChange5Sessions(), numberStyle);
                createNumberCell(row, 11, item.getChange10Sessions(), numberStyle);
                createNumberCell(row, 12, item.getChange30Sessions(), numberStyle);

                // Date
                Cell dateCell = row.createCell(13);
                if (item.getTradingDate() != null) {
                    dateCell.setCellValue(item.getTradingDate().toString());
                }

                rowNum++;
            }

            // Auto-size columns
            for (int i = 0; i < headers.length; i++) {
                sheet.autoSizeColumn(i);
            }

            // Write to ByteArrayOutputStream
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            workbook.write(out);
            return new ByteArrayInputStream(out.toByteArray());

        } catch (IOException e) {
            log.error("Error exporting Excel", e);
            throw new ExportException("Lỗi xuất file Excel", e);
        }
    }

    private CellStyle createHeaderStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();

        // Background color
        style.setFillForegroundColor(IndexedColors.DARK_GREEN.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);

        // Font
        Font font = workbook.createFont();
        font.setColor(IndexedColors.WHITE.getIndex());
        font.setBold(true);
        style.setFont(font);

        // Border
        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);

        // Alignment
        style.setAlignment(HorizontalAlignment.CENTER);

        return style;
    }

    private CellStyle createNumberStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        DataFormat format = workbook.createDataFormat();
        style.setDataFormat(format.getFormat("#,##0.000"));
        return style;
    }

    private CellStyle createDateStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        DataFormat format = workbook.createDataFormat();
        style.setDataFormat(format.getFormat("dd-MM-yyyy"));
        return style;
    }

    private void createNumberCell(Row row, int column, Number value, CellStyle style) {
        Cell cell = row.createCell(column);
        if (value != null) {
            cell.setCellValue(value.doubleValue());
            cell.setCellStyle(style);
        }
    }
}
```

### 2.2. Export với nhiều Sheet (nếu data > 1 triệu rows)

```java
/**
 * Export với multiple sheets nếu data lớn
 * Mỗi sheet tối đa 1 triệu rows (Excel limit)
 */
public ByteArrayInputStream exportLargeData(List<MarketInfoResponse> data) {
    final int MAX_ROWS_PER_SHEET = 1_000_000;

    try (Workbook workbook = new XSSFWorkbook()) {
        int sheetNum = 1;
        int totalRows = data.size();
        int processedRows = 0;

        while (processedRows < totalRows) {
            Sheet sheet = workbook.createSheet("Data_" + sheetNum);

            // Header
            createHeaderRow(sheet, workbook);

            // Data rows
            int rowNum = 1;
            int endIndex = Math.min(processedRows + MAX_ROWS_PER_SHEET - 1, totalRows);

            for (int i = processedRows; i < endIndex; i++) {
                createDataRow(sheet, rowNum++, data.get(i));
            }

            processedRows = endIndex;
            sheetNum++;
        }

        ByteArrayOutputStream out = new ByteArrayOutputStream();
        workbook.write(out);
        return new ByteArrayInputStream(out.toByteArray());

    } catch (IOException e) {
        throw new ExportException("Lỗi xuất file Excel", e);
    }
}
```

## 3. Export CSV với OpenCSV

### 3.1. CSV Export Service

```java
@Service
@Slf4j
public class CsvExportService {

    /**
     * Export danh sách MarketInfo ra CSV
     */
    public ByteArrayInputStream exportMarketInfo(List<MarketInfoResponse> data) {
        try (ByteArrayOutputStream out = new ByteArrayOutputStream();
             OutputStreamWriter writer = new OutputStreamWriter(out, StandardCharsets.UTF_8);
             CSVWriter csvWriter = new CSVWriter(writer)) {

            // BOM for UTF-8 (để Excel đọc được tiếng Việt)
            out.write(0xEF);
            out.write(0xBB);
            out.write(0xBF);

            // Header
            String[] header = {
                "STT", "Chỉ số", "Giá trị", "KL giao dịch", "GT giao dịch",
                "KL NĐTNN mua", "GT NĐTNN mua", "KL NĐTNN bán", "GT NĐTNN bán",
                "Mua/bán ròng", "Biến động 5 phiên", "Biến động 10 phiên",
                "Biến động 30 phiên", "Ngày giao dịch"
            };
            csvWriter.writeNext(header);

            // Data
            int stt = 1;
            for (MarketInfoResponse item : data) {
                String[] row = {
                    String.valueOf(stt++),
                    item.getIndexCode(),
                    formatNumber(item.getValue()),
                    formatNumber(item.getTradingVolume()),
                    formatNumber(item.getTradingValue()),
                    formatNumber(item.getForeignBuyVolume()),
                    formatNumber(item.getForeignBuyValue()),
                    formatNumber(item.getForeignSellVolume()),
                    formatNumber(item.getForeignSellValue()),
                    formatNumber(item.getNetBuySell()),
                    formatNumber(item.getChange5Sessions()),
                    formatNumber(item.getChange10Sessions()),
                    formatNumber(item.getChange30Sessions()),
                    formatDate(item.getTradingDate())
                };
                csvWriter.writeNext(row);
            }

            csvWriter.flush();
            return new ByteArrayInputStream(out.toByteArray());

        } catch (IOException e) {
            log.error("Error exporting CSV", e);
            throw new ExportException("Lỗi xuất file CSV", e);
        }
    }

    private String formatNumber(Number value) {
        if (value == null) return "";
        if (value instanceof BigDecimal) {
            return ((BigDecimal) value).setScale(3, RoundingMode.HALF_UP).toString();
        }
        return value.toString();
    }

    private String formatDate(LocalDate date) {
        if (date == null) return "";
        return date.format(DateTimeFormatter.ofPattern("dd-MM-yyyy"));
    }
}
```

### 3.2. CSV với Bean Mapping (Alternative)

```java
// DTO với @CsvBindByName
@Data
public class MarketInfoCsvDto {
    @CsvBindByName(column = "STT")
    private int stt;

    @CsvBindByName(column = "Chỉ số")
    private String indexCode;

    @CsvBindByName(column = "Giá trị")
    private String value;

    // ... other fields
}

// Export với StatefulBeanToCsv
public ByteArrayInputStream exportWithBean(List<MarketInfoCsvDto> data) {
    try (ByteArrayOutputStream out = new ByteArrayOutputStream();
         Writer writer = new OutputStreamWriter(out, StandardCharsets.UTF_8)) {

        StatefulBeanToCsv<MarketInfoCsvDto> beanToCsv = new StatefulBeanToCsvBuilder<MarketInfoCsvDto>(writer)
            .withQuotechar(CSVWriter.NO_QUOTE_CHARACTER)
            .build();

        beanToCsv.write(data);
        writer.flush();

        return new ByteArrayInputStream(out.toByteArray());
    } catch (Exception e) {
        throw new ExportException("Lỗi xuất CSV", e);
    }
}
```

## 4. Controller cho Export

### 4.1. Export Endpoints

```java
@RestController
@RequestMapping("/api/market-info")
@RequiredArgsConstructor
public class MarketInfoController {

    private final MarketInfoService service;
    private final ExcelExportService excelExportService;
    private final CsvExportService csvExportService;

    /**
     * Export Excel
     */
    @GetMapping("/export/excel")
    public ResponseEntity<Resource> exportExcel(MarketInfoSearchRequest request) {
        // Get all data matching filter (no pagination for export)
        List<MarketInfoResponse> data = service.findAllForExport(request);

        // Generate file
        ByteArrayInputStream stream = excelExportService.exportMarketInfo(data);

        // Generate filename
        String filename = generateFilename("ThongTinThiTruong", "xlsx");

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=" + filename)
            .contentType(MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .body(new InputStreamResource(stream));
    }

    /**
     * Export CSV
     */
    @GetMapping("/export/csv")
    public ResponseEntity<Resource> exportCsv(MarketInfoSearchRequest request) {
        List<MarketInfoResponse> data = service.findAllForExport(request);

        ByteArrayInputStream stream = csvExportService.exportMarketInfo(data);

        String filename = generateFilename("ThongTinThiTruong", "csv");

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=" + filename)
            .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
            .body(new InputStreamResource(stream));
    }

    /**
     * Generate filename theo format: [TenFile]_yyyyMMdd_HHmmss.[ext]
     */
    private String generateFilename(String baseName, String extension) {
        String timestamp = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
        return String.format("%s_%s.%s", baseName, timestamp, extension);
    }
}
```

### 4.2. Service method cho Export (không phân trang)

```java
@Service
public class MarketInfoServiceImpl implements MarketInfoService {

    /**
     * Lấy tất cả data theo filter (không phân trang) - dùng cho export
     */
    @Override
    public List<MarketInfoResponse> findAllForExport(MarketInfoSearchRequest request) {
        Specification<MarketInfo> spec = MarketInfoSpecification.fromRequest(request);

        // Sort theo request
        Sort sort = Sort.by(
            request.getSortDir().equalsIgnoreCase("desc") ?
                Sort.Direction.DESC : Sort.Direction.ASC,
            request.getSortBy()
        );

        List<MarketInfo> entities = repository.findAll(spec, sort);
        return mapper.toResponseList(entities);
    }
}
```

## 5. Streaming Export (cho data rất lớn)

### 5.1. Streaming Excel với SXSSFWorkbook

```java
/**
 * Streaming export - không load tất cả vào memory
 * Dùng cho data > 100k rows
 */
public void exportLargeExcelStreaming(
        MarketInfoSearchRequest request,
        OutputStream outputStream) {

    // SXSSFWorkbook: streaming workbook, chỉ giữ 100 rows trong memory
    try (SXSSFWorkbook workbook = new SXSSFWorkbook(100)) {
        Sheet sheet = workbook.createSheet("Data");

        // Header
        Row headerRow = sheet.createRow(0);
        // ... create header

        // Stream data từ database
        int rowNum = 1;
        try (Stream<MarketInfo> stream = repository.streamAll(
                MarketInfoSpecification.fromRequest(request))) {

            Iterator<MarketInfo> iterator = stream.iterator();
            while (iterator.hasNext()) {
                MarketInfo entity = iterator.next();
                Row row = sheet.createRow(rowNum++);
                // ... populate row
            }
        }

        workbook.write(outputStream);

    } catch (IOException e) {
        throw new ExportException("Lỗi xuất file", e);
    }
}
```

### 5.2. Repository với Stream

```java
@Repository
public interface MarketInfoRepository extends JpaRepository<MarketInfo, Long> {

    @Query("SELECT m FROM MarketInfo m")
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
    Stream<MarketInfo> streamAll(Specification<MarketInfo> spec);
}
```

## 6. Export Format Configuration

### 6.1. Configurable Export

```java
@Configuration
@ConfigurationProperties(prefix = "app.export")
@Data
public class ExportConfig {
    private int maxRows = 1_000_000;
    private String dateFormat = "dd-MM-yyyy";
    private String numberFormat = "#,##0.000";
    private String filenamePrefix = "Export";
}
```

```yaml
# application.yml
app:
  export:
    max-rows: 1000000
    date-format: "dd-MM-yyyy"
    number-format: "#,##0.000"
    filename-prefix: "ThongTinThiTruong"
```

## 7. Error Handling

### 7.1. Custom Exception

```java
public class ExportException extends RuntimeException {
    public ExportException(String message) {
        super(message);
    }

    public ExportException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### 7.2. Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ExportException.class)
    public ResponseEntity<ErrorResponse> handleExportError(ExportException ex) {
        ErrorResponse response = ErrorResponse.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .message(ex.getMessage())
            .timestamp(LocalDateTime.now())
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
```

## 8. Tổng kết Day 4

| Feature | Library | Use case |
|---------|---------|----------|
| Excel export | Apache POI | Báo cáo có format phức tạp |
| CSV export | OpenCSV | Data đơn giản, import vào hệ thống khác |
| Large data | SXSSFWorkbook | Data > 100k rows |

### Key Takeaways
1. **Apache POI** - Tạo Excel với đầy đủ format (màu, font, border)
2. **OpenCSV** - Nhanh, đơn giản cho CSV
3. **SXSSFWorkbook** - Streaming mode cho data lớn
4. **UTF-8 BOM** - Cần thêm BOM để Excel đọc được tiếng Việt trong CSV
5. **Filename format** - `[TenFile]_yyyyMMdd_HHmmss.[ext]`

## Navigation

- [← Day 3: REST API Pagination](./day-03-rest-api-pagination.md)
- [Day 5: Angular Integration →](./day-05-angular-integration.md)

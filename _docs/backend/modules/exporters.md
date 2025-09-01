---
layout: documentation
title: "Exporting Data"
subtitle: "A flexible framework for exporting data to various formats like Excel (XLSX)."
permalink: /docs/backend/modules/exporters/
section: backend
---
# PreBoot-Exporters Module

## Overview

The PreBoot-Exporters module provides a flexible and extensible framework for exporting data to various formats. Currently, it includes a comprehensive Excel (XLSX) exporter with support for customization, localization, and advanced formatting options.

The module is designed with a pluggable architecture that allows easy extension to support additional export formats while maintaining consistent API across all exporters.

### Key Features

- **Multi-format Support**: Extensible architecture for supporting various export formats
- **Excel (XLSX) Export**: Full-featured Excel export with streaming support
- **Value Translation**: Built-in localization and value transformation capabilities
- **Row Decoration**: Customizable row styling and formatting
- **Memory Optimization**: Streaming export for large datasets
- **UTF-8 Support**: Full Unicode support in filenames and content
- **Date/Time Handling**: Automatic detection and formatting of temporal data
- **Integration Ready**: Seamless integration with preboot-query controllers

## Module Structure

The module consists of two main components:

1. **preboot-exporters-api**: Defines interfaces and contracts for exporters
2. **preboot-exporters-excel**: Provides Excel (XLSX) export implementation

### Dependencies

```xml
<!-- For API module -->
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-exporters-api</artifactId>
</dependency>

<!-- For Excel export functionality -->
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-exporters-excel</artifactId>
</dependency>
```

## Integration with PreBoot-Query

The exporters module integrates seamlessly with preboot-query controllers, providing automatic export capabilities:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController extends FilterableController<Order, Long> {
    
    public OrderController(OrderRepository repository, List<DataExporter> exporters) {
        super(repository, false, exporters); // Pass exporters to enable export
    }
    
    @Override
    protected Map<String, String> prepareExportLabels() {
        return Map.of(
            "orderNumber", "Order Number",
            "amount", "Amount",
            "status", "Status",
            "createdAt", "Created Date"
        );
    }
}
```

**Available Export Endpoint**:
- `POST /api/orders/export/xlsx`: Export orders to Excel format

## Core Interfaces

### DataExporter Interface

The main interface that all exporters must implement:

```java
public interface DataExporter {
    String getSupportedFormat();
    MediaType getContentType();
    <T> void exportToResponse(String fileName, Map<String, String> labels, 
                             HttpServletResponse response, Locale locale, 
                             Stream<T> data) throws IOException;
}
```

### ValueTranslator Interface

Enables custom value transformation and localization:

```java
public interface ValueTranslator {
    String translate(String column, String value, Locale locale);
}
```

### RowDecorator Interface

Allows custom row styling and formatting:

```java
public interface RowDecorator {
    void decorate(Object row);
}
```

### RowDecoratorProvider Interface

Factory for creating row decorators:

```java
public interface RowDecoratorProvider {
    RowDecorator provide(Object workbook, List<String> columns);
}
```

## Excel Export Service

### Basic Configuration

The Excel exporter is automatically configured but can be customized:

```properties
# Excel export configuration
excel.export.window-size=100
spring.application.timezone=Europe/Warsaw
```

### Basic Usage

```java
@Service
public class ReportService {
    private final DataExporter excelExporter;
    
    public ReportService(@Qualifier("excelService") DataExporter excelExporter) {
        this.excelExporter = excelExporter;
    }
    
    public void exportOrders(List<Order> orders, HttpServletResponse response) throws IOException {
        Map<String, String> labels = Map.of(
            "id", "Order ID",
            "orderNumber", "Order Number", 
            "amount", "Amount",
            "status", "Status",
            "createdAt", "Created Date"
        );
        
        excelExporter.exportToResponse(
            "orders_export", 
            labels, 
            response, 
            Locale.getDefault(), 
            orders.stream()
        );
    }
}
```

### Advanced Excel Export

```java
@Service
public class AdvancedExportService {
    private final ExcelService excelService;
    
    public void exportWithCustomization(List<Order> data, HttpServletResponse response) 
            throws IOException {
        
        Map<String, String> labels = Map.of(
            "orderNumber", "Order Number",
            "customerName", "Customer",
            "amount", "Total Amount",
            "status", "Order Status",
            "createdAt", "Order Date"
        );
        
        // Custom row decorator for styling
        RowDecoratorProvider decoratorProvider = new CustomRowDecoratorProvider();
        
        // Create workbook with custom styling
        SXSSFWorkbook workbook = excelService.createWorkbook(
            data.stream(), 
            Locale.getDefault(), 
            labels, 
            decoratorProvider
        );
        
        // Write to response
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setHeader("Content-Disposition", "attachment; filename=\"orders.xlsx\"");
        workbook.write(response.getOutputStream());
        workbook.close();
    }
}
```

## Value Translation and Localization

### Custom Value Translator

Create custom translator for localization:

```java
@Component
public class OrderValueTranslator implements ValueTranslator {
    
    @Override
    public String translate(String column, String value, Locale locale) {
        if ("status".equals(column) && value != null) {
            return translateStatus(value, locale);
        }
        
        if ("priority".equals(column) && value != null) {
            return translatePriority(value, locale);
        }
        
        return value; // Return original value if no translation needed
    }
    
    private String translateStatus(String status, Locale locale) {
        if (locale.getLanguage().equals("pl")) {
            return switch (status) {
                case "NEW" -> "Nowy";
                case "IN_PROGRESS" -> "W trakcie";
                case "COMPLETED" -> "Zakończony";
                case "CANCELLED" -> "Anulowany";
                default -> status;
            };
        } else if (locale.getLanguage().equals("de")) {
            return switch (status) {
                case "NEW" -> "Neu";
                case "IN_PROGRESS" -> "In Bearbeitung";
                case "COMPLETED" -> "Abgeschlossen";
                case "CANCELLED" -> "Storniert";
                default -> status;
            };
        }
        
        // Default English
        return switch (status) {
            case "NEW" -> "New";
            case "IN_PROGRESS" -> "In Progress";
            case "COMPLETED" -> "Completed";
            case "CANCELLED" -> "Cancelled";
            default -> status;
        };
    }
    
    private String translatePriority(String priority, Locale locale) {
        if (locale.getLanguage().equals("pl")) {
            return switch (priority) {
                case "HIGH" -> "Wysoki";
                case "MEDIUM" -> "Średni";
                case "LOW" -> "Niski";
                default -> priority;
            };
        }
        return priority;
    }
}
```

### Usage with Different Locales

```java
@RestController
@RequestMapping("/api/export")
public class LocalizedExportController {
    private final OrderRepository orderRepository;
    private final DataExporter excelExporter;
    
    @PostMapping("/orders/{locale}")
    public void exportOrdersLocalized(
            @PathVariable String locale,
            @RequestBody SearchRequest searchRequest,
            HttpServletResponse response) throws IOException {
        
        Locale requestLocale = Locale.forLanguageTag(locale);
        
        SearchParams params = SearchParams.builder()
            .filters(searchRequest.filters())
            .sortField(searchRequest.sortField())
            .sortDirection(searchRequest.sortDirection())
            .build();
        
        Stream<Order> orders = orderRepository.findAllAsStream(params);
        
        Map<String, String> labels = getLocalizedLabels(requestLocale);
        
        excelExporter.exportToResponse(
            "orders_" + locale,
            labels,
            response,
            requestLocale,
            orders
        );
    }
    
    private Map<String, String> getLocalizedLabels(Locale locale) {
        if (locale.getLanguage().equals("pl")) {
            return Map.of(
                "orderNumber", "Numer zamówienia",
                "amount", "Kwota",
                "status", "Status",
                "createdAt", "Data utworzenia"
            );
        }
        
        return Map.of(
            "orderNumber", "Order Number",
            "amount", "Amount", 
            "status", "Status",
            "createdAt", "Created Date"
        );
    }
}
```

## Row Decoration and Styling

### Custom Row Decorator

Create custom styling for Excel rows:

```java
public class CustomRowDecorator extends ExcelRowDecorator {
    private final CellStyle headerStyle;
    private final CellStyle evenRowStyle;
    private final CellStyle oddRowStyle;
    private final CellStyle amountStyle;
    
    public CustomRowDecorator(Workbook workbook) {
        this.headerStyle = createHeaderStyle(workbook);
        this.evenRowStyle = createEvenRowStyle(workbook);
        this.oddRowStyle = createOddRowStyle(workbook);
        this.amountStyle = createAmountStyle(workbook);
    }
    
    @Override
    protected void decoratePoiRow(Row row) {
        int rowNum = row.getRowNum();
        
        if (rowNum == 0) {
            // Header row
            for (Cell cell : row) {
                cell.setCellStyle(headerStyle);
            }
        } else {
            // Data rows - alternate styling
            CellStyle rowStyle = (rowNum % 2 == 0) ? evenRowStyle : oddRowStyle;
            
            for (Cell cell : row) {
                cell.setCellStyle(rowStyle);
                
                // Special styling for amount columns
                if (cell.getColumnIndex() == getAmountColumnIndex()) {
                    cell.setCellStyle(amountStyle);
                }
            }
        }
    }
    
    private CellStyle createHeaderStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        Font font = workbook.createFont();
        font.setBold(true);
        font.setColor(IndexedColors.WHITE.getIndex());
        style.setFont(font);
        style.setFillForegroundColor(IndexedColors.DARK_BLUE.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);
        return style;
    }
    
    private CellStyle createEvenRowStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        addBorders(style);
        return style;
    }
    
    private CellStyle createOddRowStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        style.setFillForegroundColor(IndexedColors.WHITE.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        addBorders(style);
        return style;
    }
    
    private CellStyle createAmountStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        style.setDataFormat(workbook.getCreationHelper()
            .createDataFormat().getFormat("#,##0.00"));
        style.setAlignment(HorizontalAlignment.RIGHT);
        addBorders(style);
        return style;
    }
    
    private void addBorders(CellStyle style) {
        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);
    }
    
    private int getAmountColumnIndex() {
        // Return the index of amount column - would be determined dynamically
        return 2; // Example
    }
}
```

### Custom Row Decorator Provider

```java
public class CustomRowDecoratorProvider implements RowDecoratorProvider {
    
    @Override
    public RowDecorator provide(Object workbook, List<String> columns) {
        if (workbook instanceof Workbook wb) {
            return new CustomRowDecorator(wb);
        }
        throw new IllegalArgumentException("Workbook must be instance of Apache POI Workbook");
    }
}
```

## Data Type Handling

The Excel exporter automatically handles various data types:

### Automatic Data Type Detection

```java
public class ExportDataExample {
    private Long id;                    // → Numeric cell
    private String name;                // → Text cell
    private BigDecimal amount;          // → Numeric cell with decimal formatting
    private LocalDateTime createdAt;    // → Formatted date string
    private Instant updatedAt;          // → Formatted date string (ISO format detection)
    private Boolean active;             // → Boolean cell
    private List<String> tags;          // → JSON string representation
    
    // Getters and setters...
}
```

### Date and Time Formatting

The exporter automatically detects and formats temporal data:

- **ISO Instant format**: `2023-01-01T10:00:00Z` → `01.01.2023 / 11:00` (with timezone conversion)
- **LocalDateTime format**: `2023-01-01T10:00:00` → `01.01.2023 / 10:00`
- **Custom format**: Can be configured via server timezone setting

```properties
# Configure timezone for date formatting
spring.application.timezone=Europe/Warsaw
```

## Performance Optimization

### Memory Management

The Excel exporter uses Apache POI's `SXSSFWorkbook` for memory-efficient streaming:

```properties
# Configure window size for memory optimization
excel.export.window-size=100  # Keep only 100 rows in memory
```

### Large Dataset Export

```java
@Service
public class LargeDatasetExportService {
    private final OrderRepository orderRepository;
    private final ExcelService excelService;
    
    public void exportLargeDataset(SearchParams searchParams, HttpServletResponse response) 
            throws IOException {
        
        // Use streaming to avoid loading all data into memory
        Stream<Order> orderStream = orderRepository.findAllAsStream(searchParams);
        
        Map<String, String> labels = Map.of(
            "id", "ID",
            "orderNumber", "Order Number",
            "amount", "Amount"
        );
        
        // Stream is automatically closed by ExcelService
        excelService.exportToResponse(
            "large_export",
            labels,
            response,
            Locale.getDefault(),
            orderStream
        );
    }
}
```

### Configuration for Performance

```properties
# Excel export performance tuning
excel.export.window-size=500           # Larger window for better performance
spring.application.timezone=UTC        # Use UTC to avoid timezone calculations
```

## Custom Exporter Implementation

### Creating a CSV Exporter

```java
@Component
public class CsvExporter implements DataExporter {
    
    @Override
    public String getSupportedFormat() {
        return "csv";
    }
    
    @Override
    public MediaType getContentType() {
        return MediaType.parseMediaType("text/csv");
    }
    
    @Override
    public <T> void exportToResponse(String fileName, Map<String, String> labels, 
                                   HttpServletResponse response, Locale locale, 
                                   Stream<T> data) throws IOException {
        
        String csvFileName = fileName.endsWith(".csv") ? fileName : fileName + ".csv";
        response.setHeader("Content-Disposition", 
                          "attachment; filename=\"" + csvFileName + "\"");
        response.setContentType("text/csv; charset=UTF-8");
        
        try (PrintWriter writer = response.getWriter()) {
            // Write header
            writer.println(String.join(",", labels.values()));
            
            ObjectMapper objectMapper = new ObjectMapper();
            
            // Write data rows
            data.forEach(item -> {
                ObjectNode node = objectMapper.valueToTree(item);
                List<String> values = new ArrayList<>();
                
                for (String key : labels.keySet()) {
                    JsonNode value = node.get(key);
                    String csvValue = (value != null && !value.isNull()) 
                        ? escapeCsvValue(value.asText()) 
                        : "";
                    values.add(csvValue);
                }
                
                writer.println(String.join(",", values));
            });
        }
    }
    
    private String escapeCsvValue(String value) {
        if (value.contains(",") || value.contains("\"") || value.contains("\n")) {
            return "\"" + value.replace("\"", "\"\"") + "\"";
        }
        return value;
    }
}
```

### Registering Custom Exporters

```java
@Configuration
public class ExportConfiguration {
    
    @Bean
    public List<DataExporter> dataExporters(ExcelService excelService, CsvExporter csvExporter) {
        return List.of(excelService, csvExporter);
    }
}
```

## Integration Examples

### Complete Export Controller

```java
@RestController
@RequestMapping("/api/reports")
public class ReportController {
    private final OrderRepository orderRepository;
    private final List<DataExporter> exporters;
    
    public ReportController(OrderRepository orderRepository, List<DataExporter> exporters) {
        this.orderRepository = orderRepository;
        this.exporters = exporters;
    }
    
    @PostMapping("/orders/export/{format}")
    public void exportOrders(
            @PathVariable String format,
            @RequestBody ExportRequest exportRequest,
            @RequestParam(defaultValue = "en") String locale,
            HttpServletResponse response) throws IOException {
        
        DataExporter exporter = findExporter(format);
        if (exporter == null) {
            throw new UnsupportedOperationException("Format not supported: " + format);
        }
        
        SearchParams params = SearchParams.builder()
            .filters(exportRequest.searchRequest().filters())
            .sortField(exportRequest.searchRequest().sortField())
            .sortDirection(exportRequest.searchRequest().sortDirection())
            .unpaged(true) // Export all matching records
            .build();
        
        Stream<Order> orders = orderRepository.findAllAsStream(params);
        Locale requestLocale = Locale.forLanguageTag(locale);
        
        Map<String, String> labels = getLabelsForLocale(requestLocale);
        
        exporter.exportToResponse(
            exportRequest.fileName(),
            labels,
            response,
            requestLocale,
            orders
        );
    }
    
    @PostMapping("/orders/export/{format}/projection/{projection}")
    public void exportOrdersWithProjection(
            @PathVariable String format,
            @PathVariable String projection,
            @RequestBody ExportRequest exportRequest,
            HttpServletResponse response) throws IOException {
        
        DataExporter exporter = findExporter(format);
        Class<?> projectionClass = resolveProjectionClass(projection);
        
        SearchParams params = SearchParams.builder()
            .filters(exportRequest.searchRequest().filters())
            .unpaged(true)
            .build();
        
        Stream<?> projectedData = orderRepository.findAllProjectedByAsStream(params, projectionClass);
        
        Map<String, String> labels = getProjectionLabels(projection);
        
        exporter.exportToResponse(
            exportRequest.fileName(),
            labels,
            response,
            Locale.getDefault(),
            projectedData
        );
    }
    
    private DataExporter findExporter(String format) {
        return exporters.stream()
            .filter(exporter -> exporter.getSupportedFormat().equalsIgnoreCase(format))
            .findFirst()
            .orElse(null);
    }
    
    private Map<String, String> getLabelsForLocale(Locale locale) {
        // Implementation depends on your localization strategy
        return Map.of(
            "orderNumber", "Order Number",
            "customerName", "Customer",
            "amount", "Amount",
            "status", "Status",
            "createdAt", "Created Date"
        );
    }
    
    private Class<?> resolveProjectionClass(String projection) {
        return switch (projection) {
            case "summary" -> OrderSummaryProjection.class;
            case "detailed" -> OrderDetailedProjection.class;
            default -> throw new IllegalArgumentException("Unknown projection: " + projection);
        };
    }
    
    private Map<String, String> getProjectionLabels(String projection) {
        return switch (projection) {
            case "summary" -> Map.of(
                "orderNumber", "Order Number",
                "totalAmount", "Total Amount",
                "status", "Status"
            );
            case "detailed" -> Map.of(
                "orderNumber", "Order Number", 
                "customerName", "Customer",
                "customerEmail", "Email",
                "amount", "Amount",
                "itemCount", "Items",
                "status", "Status",
                "createdAt", "Created",
                "lastModified", "Last Modified"
            );
            default -> Map.of();
        };
    }
}
```

### Service Layer Integration

```java
@Service
@Transactional
public class ExportService {
    private final OrderRepository orderRepository;
    private final List<DataExporter> exporters;
    private final ApplicationEventPublisher eventPublisher;
    
    public void exportOrdersAsync(ExportRequest request, String userEmail) {
        CompletableFuture.runAsync(() -> {
            try {
                exportOrders(request, userEmail);
            } catch (Exception e) {
                log.error("Export failed for user: " + userEmail, e);
                // Send failure notification
            }
        });
    }
    
    private void exportOrders(ExportRequest request, String userEmail) throws IOException {
        // Implementation with email notification
        DataExporter exporter = findExporter("xlsx");
        
        SearchParams params = convertToSearchParams(request);
        Stream<Order> orders = orderRepository.findAllAsStream(params);
        
        // Export to temporary file
        Path tempFile = Files.createTempFile("export_", ".xlsx");
        
        try (OutputStream outputStream = Files.newOutputStream(tempFile)) {
            MockHttpServletResponse response = new MockHttpServletResponse();
            response.setOutputStream(new ServletOutputStream() {
                @Override
                public void write(int b) throws IOException {
                    outputStream.write(b);
                }
                
                @Override
                public boolean isReady() { return true; }
                
                @Override
                public void setWriteListener(WriteListener listener) {}
            });
            
            exporter.exportToResponse(
                request.fileName(),
                getDefaultLabels(),
                response,
                Locale.getDefault(),
                orders
            );
            
            // Send email with attachment
            sendExportEmail(userEmail, tempFile, request.fileName());
            
        } finally {
            Files.deleteIfExists(tempFile);
        }
    }
}
```

## Error Handling

### Common Issues and Solutions

```java
@RestControllerAdvice
public class ExportExceptionHandler {
    
    @ExceptionHandler(UnsupportedOperationException.class)
    public ResponseEntity<ErrorResponse> handleUnsupportedFormat(UnsupportedOperationException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("UNSUPPORTED_FORMAT", ex.getMessage()));
    }
    
    @ExceptionHandler(IOException.class)
    public ResponseEntity<ErrorResponse> handleIOException(IOException ex) {
        log.error("Export IO error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("EXPORT_ERROR", "Failed to generate export file"));
    }
    
    @ExceptionHandler(OutOfMemoryError.class)
    public ResponseEntity<ErrorResponse> handleOutOfMemory(OutOfMemoryError ex) {
        log.error("Export out of memory", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("EXPORT_TOO_LARGE", 
                   "Dataset too large for export. Please add filters to reduce the data size."));
    }
}
```

### Validation and Safety

```java
@Component
public class ExportValidator {
    
    @Value("${export.max-rows:100000}")
    private int maxRows;
    
    public void validateExportRequest(ExportRequest request) {
        if (request.searchRequest().unpaged()) {
            // Check if unpaged export is safe
            long estimatedRows = estimateRowCount(request.searchRequest());
            if (estimatedRows > maxRows) {
                throw new IllegalArgumentException(
                    "Export too large. Estimated rows: " + estimatedRows + 
                    ", maximum allowed: " + maxRows);
            }
        }
    }
    
    private long estimateRowCount(SearchRequest searchRequest) {
        // Implementation to estimate row count without full execution
        return 0; // Placeholder
    }
}
```

## Testing

### Unit Testing Excel Export

```java
@ExtendWith(MockitoExtension.class)
class ExcelServiceTest {
    
    @Mock
    private ValueTranslator valueTranslator;
    
    private ExcelService excelService;
    private ObjectMapper objectMapper;
    
    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper();
        excelService = new ExcelService(objectMapper, valueTranslator);
        ReflectionTestUtils.setField(excelService, "windowSize", 100);
        ReflectionTestUtils.setField(excelService, "serverZoneId", "Europe/Warsaw");
    }
    
    @Test
    void createWorkbook_WithValidData_ShouldGenerateCorrectExcel() throws IOException {
        // Arrange
        List<TestEntity> data = List.of(
            new TestEntity(1L, "Test 1", new BigDecimal("100.50")),
            new TestEntity(2L, "Test 2", new BigDecimal("200.75"))
        );
        
        Map<String, String> labels = Map.of(
            "id", "ID",
            "name", "Name", 
            "amount", "Amount"
        );
        
        when(valueTranslator.translate(anyString(), anyString(), any(Locale.class)))
            .thenAnswer(invocation -> invocation.getArgument(1));
        
        // Act
        SXSSFWorkbook workbook = excelService.createWorkbook(
            data.stream(), 
            Locale.getDefault(), 
            labels
        );
        
        // Assert
        assertThat(workbook.getNumberOfSheets()).isEqualTo(1);
        
        Sheet sheet = workbook.getSheetAt(0);
        assertThat(sheet.getPhysicalNumberOfRows()).isEqualTo(3); // Header + 2 data rows
        
        // Verify header
        Row headerRow = sheet.getRow(0);
        assertThat(headerRow.getCell(0).getStringCellValue()).isEqualTo("ID");
        assertThat(headerRow.getCell(1).getStringCellValue()).isEqualTo("Name");
        assertThat(headerRow.getCell(2).getStringCellValue()).isEqualTo("Amount");
        
        // Verify data
        Row dataRow1 = sheet.getRow(1);
        assertThat(dataRow1.getCell(0).getNumericCellValue()).isEqualTo(1.0);
        assertThat(dataRow1.getCell(1).getStringCellValue()).isEqualTo("Test 1");
        assertThat(dataRow1.getCell(2).getNumericCellValue()).isEqualTo(100.50);
        
        workbook.close();
    }
}
```

### Integration Testing

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ExportIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void exportEndpoint_ShouldGenerateValidExcelFile() {
        // Arrange
        ExportRequest request = ExportRequest.builder()
            .fileName("test_export")
            .searchRequest(SearchRequest.empty())
            .build();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<ExportRequest> entity = new HttpEntity<>(request, headers);
        
        // Act
        ResponseEntity<byte[]> response = restTemplate.postForEntity(
            "/api/orders/export/xlsx", 
            entity, 
            byte[].class
        );
        
        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getHeaders().getContentType().toString())
            .contains("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        
        byte[] excelData = response.getBody();
        assertThat(excelData).isNotNull();
        
        // Verify Excel content
        try (InputStream inputStream = new ByteArrayInputStream(excelData);
             Workbook workbook = new XSSFWorkbook(inputStream)) {
            
            assertThat(workbook.getNumberOfSheets()).isEqualTo(1);
            Sheet sheet = workbook.getSheetAt(0);
            assertThat(sheet.getPhysicalNumberOfRows()).isGreaterThan(0);
        }
    }
}
```

## Best Practices

### Performance Best Practices

1. **Use Streaming for Large Datasets**
   ```java
   // Good - uses streaming
   Stream<Order> orders = orderRepository.findAllAsStream(params);
   
   // Avoid - loads all data into memory
   List<Order> orders = orderRepository.findAll(params).getContent();
   ```

2. **Configure Memory Settings**
   ```properties
   # Adjust window size based on available memory
   excel.export.window-size=200
   
   # Set appropriate JVM heap size
   -Xmx2g -Xms1g
   ```

3. **Add Export Limits**
   ```properties
   export.max-rows=50000
   export.timeout-seconds=300
   ```

### Security Best Practices

1. **Validate File Names**
   ```java
   private String sanitizeFileName(String fileName) {
       return fileName.replaceAll("[^a-zA-Z0-9._-]", "_");
   }
   ```

2. **Add Authentication and Authorization**
   ```java
   @PreAuthorize("hasRole('EXPORT_DATA')")
   @PostMapping("/export/{format}")
   public void export(...) {
       // Export implementation
   }
   ```

3. **Audit Export Operations**
   ```java
   @EventListener
   public void handleExportEvent(ExportCompletedEvent event) {
       auditService.logExport(event.getUserId(), event.getEntityType(), 
                             event.getRowCount(), event.getTimestamp());
   }
   ```

### Code Organization

1. **Separate Concerns**
    - Keep export logic in dedicated services
    - Use separate controllers for different export scenarios
    - Create specific DTOs for export operations

2. **Configuration Management**
   ```java
   @ConfigurationProperties(prefix = "export")
   @Data
   public class ExportProperties {
       private int maxRows = 100000;
       private int timeoutSeconds = 300;
       private String defaultLocale = "en";
       private Excel excel = new Excel();
       
       @Data
       public static class Excel {
           private int windowSize = 100;
           private String dateFormat = "dd.MM.yyyy / HH:mm";
       }
   }
   ```

## Troubleshooting

### Common Issues

1. **Out of Memory Errors**
    - **Cause**: Large datasets or small window size
    - **Solution**: Increase window size or add pagination
    - **Prevention**: Implement row count validation

2. **Slow Export Performance**
    - **Cause**: Complex queries or inefficient data processing
    - **Solution**: Optimize queries, use database streaming
    - **Prevention**: Profile export operations

3. **Character Encoding Issues**
    - **Cause**: Non-UTF8 characters in data
    - **Solution**: Ensure UTF-8 encoding throughout the pipeline
    - **Prevention**: Validate input data encoding

4. **Date Formatting Problems**
    - **Cause**: Timezone mismatches or invalid date strings
    - **Solution**: Configure consistent timezone settings
    - **Prevention**: Use proper temporal types

### Debugging

Enable debug logging:
```properties
logging.level.io.preboot.exporters=DEBUG
logging.level.org.apache.poi=INFO
```

Monitor memory usage:
```java
@Component
public class ExportMonitor {
    
    @EventListener
    public void monitorExport(ExportStartedEvent event) {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        log.info("Export started - Memory usage: {} MB", usedMemory / 1024 / 1024);
    }
}
```

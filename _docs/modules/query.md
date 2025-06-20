---
layout: documentation
title: "Query & Filtering"
subtitle: "Dynamic filtering, sorting, and pagination with Spring Data JDBC using the preboot-query module."
permalink: /docs/modules/query/
---
# Spring Data JDBC Filterable Repository

## Overview

The Filterable Repository module extends Spring Data JDBC to provide dynamic filtering, sorting, and pagination capabilities. It allows for flexible querying of entities with support for:

- Dynamic filtering with multiple operators (equals, like, greater than, less than, between, etc.)
- Complex logical operations (AND, OR) with support for nested conditions
- Nested entity filtering
- Custom projections with computed properties
- Pagination and sorting
- Type-safe criteria building
- UUID-based entity operations
- Aggregate reference support
- Web API integration with REST controllers
- Data export capabilities

### Key Features

- **Dynamic Filtering**: Create complex queries at runtime without writing custom SQL
- **Complex Logic Support**: Build nested AND/OR conditions for sophisticated queries
- **Type Safety**: Leverages Spring's type conversion system for safe parameter handling
- **Projections Support**: Create custom view models with computed properties using SpEL
- **Nested Entity Support**: Filter by properties of related entities
- **UUID Operations**: Full support for UUID-based entity operations
- **Integration with Spring Data JDBC**: Extends standard repository pattern
- **Web API Ready**: Built-in REST controllers for immediate API exposure
- **Export Support**: Built-in data export to various formats

## Module Dependencies

This module serves as a foundation for other preboot modules:
- **preboot-securedata**: Extends this module with multi-tenant security features
- **preboot-exporters-api**: Provides data export functionality

## Configuration

### Dependencies

Add the following dependencies to your project:

```xml
<dependencies>
   <dependency>
      <groupId>io.preboot</groupId>
      <artifactId>preboot-query</artifactId>
   </dependency>
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jdbc</artifactId>
   </dependency>
   <!-- for web API integration -->
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
   </dependency>
</dependencies>
```

### Spring Configuration

1. Enable JDBC Repositories:

```java
@Configuration
@EnableJdbcRepositories
public class JdbcConfig {
   @Bean
   public ConversionService conversionService() {
      return new DefaultFormattingConversionService();
   }
}
```

2. Create Required Beans:

The module requires several beans that are automatically configured when using Spring Boot:

- `JdbcTemplate`
- `NamedParameterJdbcTemplate`
- `RelationalMappingContext`
- `JdbcConverter`
- `ConversionService`

## Basic Usage

### 1. Define Your Entity

```java
@Table("orders")
public class Order {
   @Id
   private Long id;
   private String orderNumber;
   private BigDecimal amount;
   private String status;

   @MappedCollection(idColumn = "order_id")
   private Set<OrderItem> orderItems;
}
```

### 2. Create Repository Interface

```java
public interface OrderRepository extends FilterableRepository<Order, Long> {}
```

### 3. Implement Repository

```java
@Repository
class OrderRepositoryImpl extends FilterableFragmentImpl<Order, Long> {
   public OrderRepositoryImpl(FilterableFragmentContext context) {
      super(context, Order.class);
   }
}
```

### 4. Basic Filtering

```java
// Create search parameters
SearchParams params = SearchParams.criteria(
                FilterCriteria.eq("status", "COMPLETED"),
                FilterCriteria.gt("amount", new BigDecimal("100"))
        ).build();

// Find all matching orders
Page<Order> orders = orderRepository.findAll(params);
```

## UUID Support

The module provides comprehensive support for entities that use UUID as a business identifier alongside their database ID.

### UUID Entity Configuration

1. **Implement HasUuid Interface**:
```java
@Table("orders")
public class Order implements HasUuid {
    @Id
    private Long id;
    private UUID uuid;
    private String orderNumber;
    private BigDecimal amount;
    
    @Override
    public UUID getUuid() {
        return uuid;
    }
    
    @Override
    public void setUuid(UUID uuid) {
        this.uuid = uuid;
    }
}
```

2. **Create UUID Repository Interface**:
```java
public interface OrderRepository extends FilterableUuidRepository<Order, Long> {}
```

3. **Implement UUID Repository**:
```java
@Repository
class OrderRepositoryImpl extends FilterableUuidFragmentImpl<Order, Long> {
    public OrderRepositoryImpl(FilterableFragmentContext context) {
        super(context, Order.class);
    }
}
```

### UUID Repository Operations

UUID repositories provide additional methods for UUID-based operations:

```java
// Find by UUID
Optional<Order> order = orderRepository.findByUuid(uuid);

// Check existence by UUID
boolean exists = orderRepository.existsByUuid(uuid);

// Delete by UUID
orderRepository.deleteByUuid(uuid);

// All standard operations still work
Page<Order> orders = orderRepository.findAll(SearchParams.empty());
```

### UUID Best Practices

1. **UUID Generation**
   - UUIDs are automatically generated if not provided
   - Consider using UUID version 7 for better database performance
   - Always index UUID columns in your database

2. **Database Design**
   - Create unique constraints on UUID columns
   - Consider composite indexes for frequently filtered UUID combinations
   - Use appropriate UUID storage format for your database

## Advanced Filtering

### Available Filter Operators

The module supports a comprehensive set of filter operators:

- `eq`: Equals (case-sensitive)
- `neq`: Not equals
- `eqic`: Case-insensitive equals (uses LOWER() function)
- `like`: Case-insensitive pattern matching (uses ILIKE in PostgreSQL)
- `gt`: Greater than
- `lt`: Less than
- `gte`: Greater than or equal
- `lte`: Less than or equal
- `between`: Between two values (inclusive)
- `in`: Value in a set of values
- `ao`: Array overlap (for string arrays)
- `isnull`: Is null
- `isnotnull`: Is not null

### Complex Logical Operations

#### Simple OR Condition
```java
// Find orders that are either COMPLETED or have amount > 1000
List<FilterCriteria> statusOrAmount = Arrays.asList(
                FilterCriteria.eq("status", "COMPLETED"),
                FilterCriteria.gt("amount", new BigDecimal("1000"))
        );

SearchParams params = SearchParams.criteria(
        FilterCriteria.or(statusOrAmount)
).build();
```

#### Nested AND/OR Conditions
```java
// Complex condition: (status = 'COMPLETED' AND (amount > 200 OR amount < 100))
//                   OR
//                   (status = 'PENDING' AND amount >= 400)

// Create the (amount > 200 OR amount < 100) sub-condition
List<FilterCriteria> amountConditions = Arrays.asList(
        FilterCriteria.gt("amount", new BigDecimal("200")),
        FilterCriteria.lt("amount", new BigDecimal("100"))
);

// Create first main group: status = 'COMPLETED' AND (amount > 200 OR amount < 100)
List<FilterCriteria> firstGroup = Arrays.asList(
        FilterCriteria.eq("status", "COMPLETED"),
        FilterCriteria.or(amountConditions)
);

// Create second main group: status = 'PENDING' AND amount >= 400
List<FilterCriteria> secondGroup = Arrays.asList(
        FilterCriteria.eq("status", "PENDING"),
        FilterCriteria.gte("amount", new BigDecimal("400"))
);

// Combine both main groups with OR
SearchParams params = SearchParams.criteria(
        FilterCriteria.or(Arrays.asList(
                FilterCriteria.and(firstGroup),
                FilterCriteria.and(secondGroup)
        ))
).build();
```

### Case-Insensitive String Comparison

The `eqic` operator provides case-insensitive string comparison:

```java
// Will match "COMPLETED", "completed", "Completed", etc.
SearchParams params = SearchParams.criteria(
    FilterCriteria.eqic("status", "completed")
).build();

// Can also be used with nested properties
SearchParams params = SearchParams.criteria(
    FilterCriteria.eqic("orderItems.productCode", "prod-a")
).build();
```

### Array Operations

For PostgreSQL array columns:

```java
// Find orders where tags array contains any of the specified values
SearchParams params = SearchParams.criteria(
    FilterCriteria.ao("tags", "priority", "discount")
).build();
```

### Nested Property Filtering

Filter by properties of related entities:

```java
// Filter by properties of nested entities
SearchParams params = SearchParams.criteria(
                FilterCriteria.eq("orderItems.productCode", "PROD-A"),
                FilterCriteria.gt("orderItems.quantity", 3)
        ).build();

Page<Order> orders = orderRepository.findAll(params);
```

## Aggregate References

The module supports aggregate references through the `@AggregateReference` annotation, allowing you to establish relationships between aggregates while maintaining aggregate boundaries.

### Configuration

1. **Define Referenced Entity**:
```java
@Table("categories")
public class Category {
    @Id
    private Long id;
    private UUID uuid;
    private String name;
    private String description;
}
```

2. **Configure Reference in Source Entity**:
```java
@Table("products")
public class Product {
    @Id
    private Long id;
    private UUID uuid;
    private String name;
    private BigDecimal price;

    @AggregateReference(
        target = Category.class,
        targetColumn = "uuid",
        sourceColumn = "category_uuid",
        alias = "category"
    )
    private UUID categoryUuid;
}
```

### Usage Examples

1. **Simple Projection with Referenced Properties**:
```java
public interface ProductWithCategory {
    UUID getUuid();
    String getName();
    BigDecimal getPrice();
    
    @Value("#{target.category.name}")
    String getCategoryName();
    
    @Value("#{target.category.description}")
    String getCategoryDescription();
}

// Using the projection
SearchParams params = SearchParams.empty();
Page<ProductWithCategory> products = repository.findAllProjectedBy(params, ProductWithCategory.class);
```

2. **Filtering by Referenced Properties**:
```java
// Find all products in the "Electronics" category
SearchParams params = SearchParams.criteria(
    FilterCriteria.eq("category.name", "Electronics")
).build();

// Complex conditions involving reference properties
SearchParams params = SearchParams.criteria(
    FilterCriteria.gt("price", new BigDecimal("100")),
    FilterCriteria.like("category.description", "%devices%")
).build();
```

## Projections

### Simple Projections

Define projection interfaces for custom views:

```java
// Define projection interface
public interface OrderSummary {
   String getOrderNumber();
   BigDecimal getAmount();

   @Value("#{target.amount > 150 ? 'High Value' : 'Standard'}")
   String getValueCategory();
}

// Use projection in query
SearchParams params = SearchParams.empty();
Page<OrderSummary> summaries = orderRepository.findAllProjectedBy(params, OrderSummary.class);
```

### Complex Projections with Collections

The module supports projections involving collections where each collection item has an aggregate reference:

```java
// Main entity with collection
@Table("transactions")
public class Transaction implements HasUuid {
    @Id
    private Long id;
    private UUID uuid;
    private String name;
    private BigDecimal amount;
    private LocalDate transactionDate;

    @MappedCollection(idColumn = "transaction_id")
    private Set<TransactionCategory> categories;
}

// Collection item with reference
@Table("transaction_categories")
public class TransactionCategory {
    @AggregateReference(
        target = Category.class,
        targetColumn = "uuid",
        sourceColumn = "category_uuid",
        alias = "category"
    )
    private UUID categoryUuid;
}
```

**Advanced Projection Interface**:
```java
// Interface for category details in collections
public interface CategoryInfo {
    UUID getCategoryUuid();

    @Value("#{target.category.name}")
    String getName();

    @Value("#{target.category.color}")
    String getColor();
}

// Main projection interface
public interface TransactionWithCategories {
    UUID getUuid();
    String getName();
    BigDecimal getAmount();
    LocalDate getTransactionDate();

    @Value("#{target.categories}")
    List<CategoryInfo> getCategories();
    
    @Value("#{target.amount > 1000 ? 'High' : 'Normal'}")
    String getAmountCategory();
}
```

### Projection Best Practices

1. **SpEL Expression Design**
   - Keep expressions simple and readable
   - Handle null cases in expressions
   - Use parentheses for complex operations
   - When referencing aggregates, use the alias defined in @AggregateReference

2. **Performance Considerations**
   - Use specific projections to minimize data fetching
   - Consider pagination when dealing with collections
   - Be aware of N+1 query implications

## Pagination and Sorting

```java
SearchParams params = SearchParams.builder()
        .page(0)
        .size(20)
        .sortField("amount")
        .sortDirection(Sort.Direction.DESC)
        .build();

Page<Order> orders = orderRepository.findAll(params);
```

## Web API Integration

The module provides ready-to-use REST controllers for immediate API exposure.

### Controller Types

#### 1. FilterableController (Read-Only)

Base controller providing read operations and search functionality:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderSearchController extends FilterableController<Order, Long> {
    
    public OrderSearchController(OrderRepository repository) {
        // Enable projections support
        super(repository, true);
    }
    
    @Override
    protected <P> Class<P> resolveProjectionClass(String projectionName) {
        return switch (projectionName) {
            case "summary" -> (Class<P>) OrderSummary.class;
            case "detail" -> (Class<P>) OrderDetail.class;
            default -> null;
        };
    }
}
```

**Available Endpoints**:
- `GET /{id}`: Get by ID
- `POST /search`: Search with request body
- `POST /find`: Find one
- `POST /count`: Count matching entities
- `POST /search/{projection}`: Search with projection
- `POST /export/{format}`: Export data

#### 2. CrudFilterableController (Full CRUD)

Extends FilterableController with full CRUD operations:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController extends CrudFilterableController<Order, Long> {
    
    public OrderController(OrderRepository repository) {
        super(repository, true);
    }
    
    // Override hook methods for custom logic
    @Override
    protected void beforeCreate(Order entity) {
        // Custom validation or processing
    }
    
    @Override
    protected void afterCreate(Order entity) {
        // Post-creation logic
    }
}
```

**Additional Endpoints**:
- `POST /`: Create new entity
- `PUT /{id}`: Full update
- `PATCH /{id}`: Partial update
- `DELETE /{id}`: Delete entity

#### 3. UUID Controllers

For entities using UUID as business identifier:

**UuidFilterableController** (Read-Only):
```java
@RestController
@RequestMapping("/api/orders")
public class OrderSearchController extends UuidFilterableController<Order> {
    public OrderSearchController(FilterableUuidRepository<Order, Long> repository) {
        super(repository);
    }
}
```

**CrudUuidFilterableController** (Full CRUD):
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController extends CrudUuidFilterableController<Order> {
    public OrderController(FilterableUuidRepository<Order, Long> repository) {
        super(repository);
    }
}
```

UUID controllers use UUID in URLs instead of database IDs:
- `GET /{uuid}`: Get by UUID
- `PUT /{uuid}`: Update by UUID
- `PATCH /{uuid}`: Partial update by UUID
- `DELETE /{uuid}`: Delete by UUID

### Search Request Format

The `SearchRequest` record defines the structure for API requests:

```java
public record SearchRequest(
    @Min(0) Integer page,
    @Min(1) @Max(100) Integer size,
    @Pattern(regexp = "^[a-zA-Z0-9_]+$") String sortField,
    Sort.Direction sortDirection,
    List<FilterCriteria> filters,
    boolean unpaged
)
```

**Example API Request**:
```json
{
  "filters": [
    {
      "field": "status",
      "operator": "eq",
      "value": "COMPLETED"
    },
    {
      "field": "amount",
      "operator": "gt",
      "value": 100
    }
  ],
  "page": 0,
  "size": 20,
  "sortField": "amount",
  "sortDirection": "DESC"
}
```

**Complex Query Example**:
```json
{
  "filters": [
    {
      "logicalOperator": "OR",
      "children": [
        {
          "logicalOperator": "AND",
          "children": [
            {
              "field": "status",
              "operator": "eq",
              "value": "COMPLETED"
            },
            {
              "field": "amount",
              "operator": "gt",
              "value": 200
            }
          ]
        },
        {
          "field": "status",
          "operator": "eq",
          "value": "PENDING"
        }
      ]
    }
  ],
  "page": 0,
  "size": 20
}
```

### CRUD Operation Hooks

Controllers provide hooks for customizing operations:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController extends CrudFilterableController<Order, Long> {
    
    @Override
    protected void validateCreate(Order entity) {
        // Custom validation logic
    }
    
    @Override
    protected void beforeCreate(Order entity) {
        // Pre-creation processing
    }
    
    @Override
    protected void afterCreate(Order entity) {
        // Post-creation processing
    }
    
    // Similar hooks available for update, patch, and delete operations
}
```

## Data Export

The module includes built-in data export capabilities supporting various formats.

### Export Configuration

Controllers automatically support export when data exporters are available:

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController extends FilterableController<Order, Long> {
    
    public OrderController(OrderRepository repository, List<DataExporter> exporters) {
        super(repository, false, exporters);
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

### Export Request Format

```java
public record ExportRequest(
    @Pattern(regexp = "^[a-zA-Z0-9_\\-. ]+$") String fileName,
    @NotNull @Valid SearchRequest searchRequest
)
```

**Example Export Request**:
```json
{
  "fileName": "completed_orders_2024",
  "searchRequest": {
    "filters": [
      {
        "field": "status",
        "operator": "eq",
        "value": "COMPLETED"
      }
    ],
    "sortField": "createdAt",
    "sortDirection": "DESC",
    "unpaged": true
  }
}
```

### Export Endpoint

- `POST /export/{format}`: Export data in specified format (e.g., xlsx, csv, pdf)

## Performance Optimization

### Query Optimization

1. **Use Appropriate Indexes**
   - Index frequently filtered columns
   - Consider composite indexes for multiple filter conditions
   - Use functional indexes for case-insensitive searches

2. **Pagination Strategy**
   - Always use pagination for large result sets
   - Consider cursor-based pagination for very large datasets

3. **Projection Usage**
   - Use projections to fetch only required fields
   - Avoid loading entire entities when only specific fields are needed

### Database Considerations

1. **PostgreSQL Optimizations**
   - Use ILIKE indexes for case-insensitive searches
   - Consider partial indexes for commonly filtered subsets
   - Optimize UUID storage and indexing

2. **Query Analysis**
   - Enable SQL logging to analyze generated queries
   - Use database query analyzers to identify bottlenecks

## Error Handling

The module provides specific exceptions for different error conditions:

- `FilteringException`: Base exception for all filtering operations
- `InvalidFilterCriteriaException`: Invalid filter parameters or operations
- `PropertyNotFoundException`: When referenced property doesn't exist
- `TypeConversionException`: Failed type conversion for filter values

### Exception Handling in Controllers

```java
@RestControllerAdvice
public class QueryExceptionHandler {
    
    @ExceptionHandler(InvalidFilterCriteriaException.class)
    public ResponseEntity<ErrorResponse> handleInvalidFilter(InvalidFilterCriteriaException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("Invalid filter criteria", ex.getMessage()));
    }
    
    @ExceptionHandler(PropertyNotFoundException.class)
    public ResponseEntity<ErrorResponse> handlePropertyNotFound(PropertyNotFoundException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("Property not found", ex.getMessage()));
    }
}
```

## Best Practices

### Repository Design

1. **Type-Safe Criteria**
   - Use `FilterCriteria` static factory methods
   - Validate input parameters before processing

2. **Query Complexity**
   - Break down complex queries into smaller, manageable parts
   - Use projections to optimize data retrieval

3. **Performance Monitoring**
   - Monitor query execution times
   - Set up alerts for slow queries

### API Design

1. **Consistent Endpoints**
   - Use consistent URL patterns across controllers
   - Implement proper HTTP status codes

2. **Input Validation**
   - Validate search parameters
   - Provide meaningful error messages

3. **Security Considerations**
   - Implement proper authentication and authorization
   - Validate user permissions before data access

### Testing

1. **Integration Testing**
   - Test complex filter combinations
   - Verify pagination and sorting behavior

2. **Performance Testing**
   - Test with realistic data volumes
   - Verify index effectiveness

## Limitations

1. **Database Support**
   - Currently optimized for PostgreSQL
   - May require adjustments for other databases

2. **Collection Filtering**
   - Limited to one level of collection nesting
   - Complex collection queries may impact performance

3. **Query Complexity**
   - Very complex filters may generate suboptimal SQL
   - Consider native queries for extremely complex scenarios

## Troubleshooting

### Common Issues

1. **Performance Problems**
   - Check execution plan for generated SQL
   - Verify indexes are being used
   - Consider query optimization techniques

2. **Type Conversion Errors**
   - Ensure proper type mapping in filter criteria
   - Check data types in entity definitions

3. **Missing Properties**
   - Verify property names match entity field names
   - Check column mappings in entity annotations

### Debugging

Enable SQL logging for query analysis:
```properties
logging.level.org.springframework.jdbc.core=TRACE
logging.level.io.preboot.query=DEBUG
```

---
layout: documentation
title: "Secure Data Access"
subtitle: "Multi-tenant data isolation and RBAC for repositories with the preboot-securedata module."
permalink: /docs/modules/secure-data/
---
# Secure Data Module

## Overview

The Secure Data module provides a robust multi-tenant data access control layer for Spring Data JDBC applications. It ensures data isolation between tenants and implements role-based access control (RBAC) at the repository level.

**This module extends preboot-query**, inheriting all its powerful filtering, projection, and query capabilities while adding comprehensive security features.

### Key Features

- **Multi-tenant Data Isolation**: Automatic filtering by tenant ID
- **Role-based Access Control (RBAC)**: Fine-grained permissions at entity level
- **Automatic Audit Fields**: Built-in created/modified tracking
- **UUID Entity Support**: Full support for UUID-based operations with security
- **Integration with preboot-query**: All filtering and projection features work with security
- **Events System**: Comprehensive event handling for repository operations
- **Runtime Metadata Caching**: Optimized performance through metadata caching
- **Custom Security Contexts**: Flexible security context implementations

## Module Dependencies

This module **depends on preboot-query** and extends its functionality:

```xml
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-query</artifactId>
</dependency>
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-securedata</artifactId>
</dependency>
```

**Relationship**: preboot-securedata → preboot-query → Spring Data JDBC

## Architecture

### Core Components

1. **Security Annotations**
    - `@SecureAccess`: Defines read/write access rules
    - `@AccessRule`: Specifies role-based permissions
    - `@Tenant`: Marks the tenant ID field in an entity
    - `@DisableSecureEntity`: Disables security checks for specific entities

2. **Audit Annotations**
    - `@CreatedAt`: Automatic creation timestamp (Instant)
    - `@CreatedBy`: Automatic creation user tracking (UUID)
    - `@ModifiedAt`: Automatic modification timestamp (Instant)
    - `@ModifiedBy`: Automatic modification user tracking (UUID)

3. **Security Context**
    - `SecurityContext`: Interface defining security information (user ID, tenant ID, roles, permissions)
    - `SecurityContextProvider`: Interface for obtaining the current security context

4. **Repository Layer**
    - `SecureRepository`: Base interface for secure repositories (extends FilterableRepository)
    - `SecureRepositoryImpl`: Implementation providing security filtering and tenant isolation
    - `SecureUuidRepository`: Extended interface for entities with UUID fields
    - `SecureUuidRepositoryImpl`: Implementation providing UUID-specific operations with security

5. **Events System**
    - `SecureRepositoryEvent`: Sealed interface for repository events
    - Before/After events for Create, Update, Delete operations

6. **Metadata System**
    - `SecureEntityMetadata`: Contains entity security configuration
    - `SecureEntityMetadataCache`: Caches entity metadata to avoid reflection overhead

## Integration with preboot-query

The Secure Data module seamlessly integrates with preboot-query, providing security on top of all query features:

### Available Query Features with Security

All preboot-query features work with security constraints automatically applied:

- **Dynamic Filtering**: All filter operations respect tenant boundaries
- **Complex Queries**: AND/OR conditions work with security constraints
- **Projections**: Custom projections include security filtering
- **Pagination & Sorting**: Secure pagination and sorting
- **UUID Operations**: UUID-based queries with tenant isolation
- **Aggregate References**: Referenced entities respect security rules
- **Nested Filtering**: Collection filtering with tenant boundaries

### How Security Integration Works

When you use secure repositories, security constraints are automatically added to all queries:

```java
// Your query
SearchParams params = SearchParams.criteria(
    FilterCriteria.eq("status", "ACTIVE")
).build();

// Automatically becomes (internally)
SearchParams secureParams = SearchParams.criteria(
    FilterCriteria.eq("status", "ACTIVE"),
    FilterCriteria.eq("tenantId", currentTenantId)  // Added automatically
).build();
```

## Configuration

### 1. Standard Entity Configuration

```java
@Table("documents")
@SecureAccess(
    read = @AccessRule(roles = {"VIEWER", "EDITOR"}),
    write = @AccessRule(roles = {"EDITOR"})
)
public class Document {
    @Id
    private Long id;

    @Tenant
    private UUID tenantId;

    private String content;
    
    // Audit fields
    @CreatedBy
    private UUID createdBy;
    
    @CreatedAt
    private Instant createdAt;
    
    @ModifiedBy
    private UUID modifiedBy;
    
    @ModifiedAt
    private Instant modifiedAt;
}
```

### 2. UUID Entity Configuration

```java
@Table("documents")
@SecureAccess(
    read = @AccessRule(roles = {"VIEWER", "EDITOR"}),
    write = @AccessRule(roles = {"EDITOR"})
)
public class Document implements HasUuid {
    @Id
    private Long id;
    
    private UUID uuid;
    
    @Tenant
    private UUID tenantId;
    
    private String content;
    
    // Audit fields are automatically populated
    @CreatedBy
    private UUID createdBy;
    
    @CreatedAt
    private Instant createdAt;
    
    @ModifiedBy
    private UUID modifiedBy;
    
    @ModifiedAt
    private Instant modifiedAt;
}
```

### 3. Repository Setup

**Standard Secure Repository**:
```java
// Define the repository interface
public interface DocumentRepository extends SecureRepository<Document, Long> {
}

// Implement the repository
@Repository
public class DocumentRepositoryImpl extends SecureRepositoryImpl<Document, Long> {
    public DocumentRepositoryImpl(SecureRepositoryContext context) {
        super(context, Document.class);
    }
}
```

**UUID-based Secure Repository**:
```java
// Define the repository interface
public interface DocumentRepository extends SecureUuidRepository<Document, Long> {
}

// Implement the repository
@Repository
public class DocumentRepositoryImpl extends SecureUuidRepositoryImpl<Document, Long> {
    public DocumentRepositoryImpl(SecureRepositoryContext context) {
        super(context, Document.class);
    }
}
```

### 4. Security Context Implementation

```java
@Component
public class YourSecurityContextProvider implements SecurityContextProvider {
    @Override
    public SecurityContext getCurrentContext() {
        // Return your security context implementation
        return new SecurityContext() {
            @Override
            public UUID getUserId() {
                return getCurrentUserId(); // Your implementation
            }
            
            @Override
            public UUID getTenantId() {
                return getCurrentTenantId(); // Your implementation
            }
            
            @Override
            public Set<String> getRoles() {
                return getCurrentUserRoles(); // Your implementation
            }
            
            @Override
            public Set<String> getPermissions() {
                return getCurrentUserPermissions(); // Your implementation
            }
        };
    }
}
```

## Audit Fields

The module provides automatic audit field population for tracking entity lifecycle.

### Supported Audit Fields

1. **@CreatedBy**: Automatically set to current user ID when entity is created
2. **@CreatedAt**: Automatically set to current timestamp when entity is created
3. **@ModifiedBy**: Automatically updated to current user ID when entity is modified
4. **@ModifiedAt**: Automatically updated to current timestamp when entity is modified

### Audit Field Requirements

- `@CreatedBy` and `@ModifiedBy` fields must be of type `UUID`
- `@CreatedAt` and `@ModifiedAt` fields must be of type `Instant`

### Example with Full Audit Support

```java
@Table("audit_documents")
@SecureAccess(
    read = @AccessRule(roles = {"USER"}),
    write = @AccessRule(roles = {"EDITOR"})
)
public class AuditDocument {
    @Id
    private Long id;
    
    @Tenant
    private UUID tenantId;
    
    private String title;
    private String content;
    
    // Audit fields - automatically populated
    @CreatedBy
    private UUID createdBy;
    
    @CreatedAt
    private Instant createdAt;
    
    @ModifiedBy
    private UUID modifiedBy;
    
    @ModifiedAt
    private Instant modifiedAt;
    
    // Getters and setters
}
```

### Usage with Audit Fields

```java
@Service
public class DocumentService {
    private final DocumentRepository documentRepository;
    
    public Document createDocument(String title, String content) {
        Document document = new Document();
        document.setTitle(title);
        document.setContent(content);
        // tenantId, createdBy, and createdAt are automatically set
        
        return documentRepository.save(document);
    }
    
    public Document updateDocument(Long id, String newContent) {
        Document document = documentRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Document not found"));
        
        document.setContent(newContent);
        // modifiedBy and modifiedAt are automatically updated
        
        return documentRepository.save(document);
    }
}
```

## Usage Examples

### Basic CRUD Operations

**Standard Repository**:
```java
@Service
public class DocumentService {
    private final DocumentRepository documentRepository;
    
    public DocumentService(DocumentRepository documentRepository) {
        this.documentRepository = documentRepository;
    }
    
    public Document createDocument(Document document) {
        // tenantId and audit fields are automatically set
        return documentRepository.save(document);
    }
    
    public List<Document> findDocuments() {
        // Results are automatically filtered by tenant
        return documentRepository.findAll(SearchParams.empty())
            .getContent();
    }
    
    public List<Document> findActiveDocuments() {
        // Security constraints are automatically added
        SearchParams params = SearchParams.criteria(
            FilterCriteria.eq("status", "ACTIVE")
        ).build();
        
        return documentRepository.findAll(params).getContent();
    }
}
```

**UUID Repository**:
```java
@Service
public class UuidDocumentService {
    private final DocumentRepository documentRepository;
    
    public UuidDocumentService(DocumentRepository documentRepository) {
        this.documentRepository = documentRepository;
    }
    
    public Document findByUuid(UUID uuid) {
        // Results are filtered by tenant automatically
        return documentRepository.findByUuid(uuid)
            .orElseThrow(() -> new EntityNotFoundException("Document not found"));
    }
    
    public Document createDocument(Document document) {
        // UUID, tenantId, and audit fields are automatically set
        return documentRepository.save(document);
    }
    
    public void deleteByUuid(UUID uuid) {
        // Will only delete if document belongs to current tenant
        documentRepository.deleteByUuid(uuid);
    }
    
    public boolean existsByUuid(UUID uuid) {
        // Will only check within current tenant's scope
        return documentRepository.existsByUuid(uuid);
    }
}
```

### Advanced Querying with Security

All toolbox-query features work seamlessly with security:

```java
@Service
public class AdvancedDocumentService {
    private final DocumentRepository documentRepository;
    
    // Complex filtering with security
    public List<Document> findRecentActiveDocuments() {
        LocalDateTime weekAgo = LocalDateTime.now().minusWeeks(1);
        
        SearchParams params = SearchParams.criteria(
            FilterCriteria.eq("status", "ACTIVE"),
            FilterCriteria.gt("createdAt", weekAgo.toString())
        ).sortField("createdAt")
        .sortDirection(Sort.Direction.DESC)
        .build();
        
        // Automatically filtered by tenant
        return documentRepository.findAll(params).getContent();
    }
    
    // Using projections with security
    public Page<DocumentSummary> getDocumentSummaries(int page, int size) {
        SearchParams params = SearchParams.builder()
            .page(page)
            .size(size)
            .sortField("modifiedAt")
            .sortDirection(Sort.Direction.DESC)
            .build();
        
        // Projection results are tenant-filtered
        return documentRepository.findAllProjectedBy(params, DocumentSummary.class);
    }
    
    // Complex conditions with security
    public List<Document> findDocumentsByComplexCriteria() {
        SearchParams params = SearchParams.criteria(
            FilterCriteria.or(Arrays.asList(
                FilterCriteria.and(Arrays.asList(
                    FilterCriteria.eq("status", "ACTIVE"),
                    FilterCriteria.like("title", "Report%")
                )),
                FilterCriteria.eq("priority", "HIGH")
            ))
        ).build();
        
        return documentRepository.findAll(params).getContent();
    }
}

public interface DocumentSummary {
    UUID getUuid();
    String getTitle();
    Instant getCreatedAt();
    
    @Value("#{target.createdBy}")
    UUID getCreatedBy();
    
    @Value("#{target.title + ' (Created: ' + target.createdAt + ')'}")
    String getDisplayName();
}
```

### Multi-tenant Operations

```java
@Service
public class MultiTenantDocumentService {
    private final DocumentRepository documentRepository;
    private final SecurityContextProvider securityContextProvider;
    
    public void demonstrateTenantIsolation() {
        UUID originalTenant = securityContextProvider.getCurrentContext().getTenantId();
        
        // All operations are automatically tenant-filtered
        List<Document> tenantDocuments = documentRepository
            .findAll(SearchParams.empty())
            .getContent();
        
        System.out.println("Found " + tenantDocuments.size() + 
                          " documents for tenant: " + originalTenant);
        
        // Each tenant sees only their own data
        tenantDocuments.forEach(doc -> 
            assert doc.getTenantId().equals(originalTenant)
        );
    }
    
    public Page<Document> searchDocumentsSecurely(String searchTerm, Pageable pageable) {
        SearchParams params = SearchParams.builder()
            .filters(List.of(FilterCriteria.like("content", searchTerm)))
            .page(pageable.getPageNumber())
            .size(pageable.getPageSize())
            .build();
        
        // Search is automatically scoped to current tenant
        return documentRepository.findAll(params);
    }
}
```

## UUID Repository Support

The module provides comprehensive UUID support with security features.

### UUID Repository Features

All standard features work with UUID repositories, plus:

- **Automatic UUID Generation**: UUIDs are generated automatically if not provided
- **UUID-based Operations**: findByUuid, existsByUuid, deleteByUuid
- **Tenant-scoped UUID Operations**: All UUID operations respect tenant boundaries
- **Security Integration**: Full integration with access control and audit features

### UUID Entity Example

```java
@Table("secure_orders")
@SecureAccess(
    read = @AccessRule(roles = {"USER", "ADMIN"}),
    write = @AccessRule(roles = {"ADMIN"})
)
public class SecureOrder implements HasUuid {
    @Id
    private Long id;
    
    private UUID uuid;  // Business identifier
    
    @Tenant
    private UUID tenantId;  // Tenant isolation
    
    private String orderNumber;
    private BigDecimal amount;
    private String status;
    
    // Audit fields
    @CreatedBy
    private UUID createdBy;
    
    @CreatedAt
    private Instant createdAt;
    
    @ModifiedBy
    private UUID modifiedBy;
    
    @ModifiedAt
    private Instant modifiedAt;
    
    // Collection with security
    @MappedCollection(idColumn = "order_id")
    private Set<SecureOrderItem> orderItems;
}
```

### UUID Operations with Security

```java
@Service
public class SecureOrderService {
    private final SecureOrderRepository orderRepository;
    
    public SecureOrder createOrder(SecureOrder order) {
        // UUID, tenantId, and audit fields set automatically
        return orderRepository.save(order);
    }
    
    public Optional<SecureOrder> findByBusinessId(UUID uuid) {
        // Only returns order if it belongs to current tenant
        return orderRepository.findByUuid(uuid);
    }
    
    public Page<SecureOrder> findOrdersWithItems() {
        SearchParams params = SearchParams.criteria(
            FilterCriteria.isNotNull("orderItems.productCode")
        ).build();
        
        // Nested filtering with tenant security
        return orderRepository.findAll(params);
    }
    
    public boolean orderExistsForTenant(UUID uuid) {
        // Checks existence within current tenant only
        return orderRepository.existsByUuid(uuid);
    }
    
    public void cancelOrder(UUID uuid) {
        SecureOrder order = orderRepository.findByUuid(uuid)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
        
        order.setStatus("CANCELLED");
        // modifiedBy and modifiedAt updated automatically
        orderRepository.save(order);
    }
}
```

## Events System

The module provides a comprehensive event system for repository operations.

### Event Types

The sealed interface `SecureRepositoryEvent<T>` defines the following events:

1. **Create Events**
    - `BeforeCreateEvent<T>`: Fired before an entity is created
    - `AfterCreateEvent<T>`: Fired after an entity is successfully created

2. **Update Events**
    - `BeforeUpdateEvent<T>`: Fired before an entity is updated
    - `AfterUpdateEvent<T>`: Fired after an entity is successfully updated

3. **Delete Events**
    - `BeforeDeleteEvent<T>`: Fired before an entity is deleted
    - `AfterDeleteEvent<T>`: Fired after an entity is successfully deleted

### Event Handler Implementation

```java
@Component
public class DocumentEventHandler {
    
    @EventHandler(typeParameter = Document.class)
    public void handleBeforeCreate(SecureRepositoryEvent.BeforeCreateEvent<Document> event) {
        Document document = event.getEntity();
        System.out.println("About to create document: " + document.getTitle());
        
        // Custom validation or processing
        validateDocumentContent(document);
    }

    @EventHandler(typeParameter = Document.class)
    public void handleAfterCreate(SecureRepositoryEvent.AfterCreateEvent<Document> event) {
        Document document = event.getEntity();
        System.out.println("Created document with ID: " + document.getId());
        
        // Post-creation tasks
        notifySubscribers(document);
        logAuditEvent("DOCUMENT_CREATED", document);
    }
    
    @EventHandler(typeParameter = Document.class)
    public void handleBeforeUpdate(SecureRepositoryEvent.BeforeUpdateEvent<Document> event) {
        Document document = event.getEntity();
        
        // Pre-update processing
        validateUpdatePermissions(document);
        archivePreviousVersion(document);
    }
    
    @EventHandler(typeParameter = Document.class)
    public void handleAfterDelete(SecureRepositoryEvent.AfterDeleteEvent<Document> event) {
        Document document = event.getEntity();
        
        // Cleanup tasks
        cleanupRelatedResources(document);
        logAuditEvent("DOCUMENT_DELETED", document);
    }
    
    private void validateDocumentContent(Document document) {
        // Custom validation logic
    }
    
    private void notifySubscribers(Document document) {
        // Notification logic
    }
    
    private void logAuditEvent(String action, Document document) {
        // Audit logging
    }
}
```

### Advanced Event Handling

```java
@Component
public class MultiEntityEventHandler {
    
    // Handle events for different entity types
    @EventHandler(typeParameter = Document.class)
    public void handleDocumentCreate(SecureRepositoryEvent.AfterCreateEvent<Document> event) {
        updateDocumentStatistics();
    }
    
    @EventHandler(typeParameter = Order.class)  
    public void handleOrderCreate(SecureRepositoryEvent.AfterCreateEvent<Order> event) {
        processOrderWorkflow(event.getEntity());
    }
    
    // Generic handler for audit logging
    @EventHandler(typeParameter = Object.class)
    public void handleAnyEntityUpdate(SecureRepositoryEvent.AfterUpdateEvent<Object> event) {
        logGenericAuditEvent("ENTITY_UPDATED", event.getEntity());
    }
    
    private void updateDocumentStatistics() {
        // Update statistics
    }
    
    private void processOrderWorkflow(Order order) {
        // Business logic
    }
    
    private void logGenericAuditEvent(String action, Object entity) {
        // Generic audit logging
    }
}
```

### Event Handler Best Practices

1. **Type Safety**
    - Always specify the correct entity type in `@EventHandler(typeParameter = ...)`
    - Use separate handlers for different entity types
    - Handle each event type explicitly for clarity

2. **Performance Considerations**
    - Keep event handlers lightweight
    - Avoid blocking operations in handlers
    - Consider asynchronous processing for heavy operations

3. **Error Handling**
    - Handle exceptions appropriately in event handlers
    - Consider the impact of handler failures on the main operation
    - Log errors for debugging

## Security Considerations

### Tenant Isolation

- **Automatic Filtering**: Every entity with a `@Tenant` field is automatically filtered by tenant
- **Cross-tenant Protection**: Attempts to access data from different tenants are blocked
- **Automatic Tenant Assignment**: Tenant ID is automatically set on new entities
- **Bulk Operation Restrictions**: Bulk operations like deleteAll() are restricted for security
- **UUID Operations**: UUID-based operations respect tenant boundaries
- **Collection Security**: Nested collections and references respect tenant isolation

### Access Control

- **Role-based Permissions**: Fine-grained access control at entity level
- **Read/Write Separation**: Different access rules for read and write operations
- **Runtime Validation**: Access rules are evaluated against current security context
- **Exception Handling**: Failed access attempts throw `SecureDataException`
- **Inheritance**: Security rules apply to all repository operations including UUID operations

### Security Best Practices

1. **Entity Design**
    - Use `@Tenant(required = false)` for shared/global entities
    - Keep tenant IDs immutable after creation
    - Implement proper UUID handling for entities requiring global identification
    - Consider using composite indexes for tenant_id and uuid columns

2. **Repository Implementation**
    - Always extend secure repository implementations
    - Use `SearchParams` for complex queries to ensure security filtering
    - Avoid direct SQL queries that might bypass security
    - Test tenant isolation thoroughly

3. **Security Context Management**
    - Implement proper user session management
    - Ensure tenant ID is always available when required
    - Cache security context appropriately for performance
    - Handle security context switching carefully

4. **Performance Optimization**
    - Index tenant_id columns appropriately
    - Use covering indexes for frequently queried tenant data
    - Consider database-specific optimizations for UUID operations
    - Monitor query performance with security constraints

## Testing

The module provides comprehensive testing utilities and examples.

### Testing Standard Repositories

```java
@SpringBootTest
@Import({TestSecurityConfig.class, TestContainersConfig.class})
@Transactional
@Sql("/secure-test-data.sql")
class DocumentRepositoryTest {
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private TestSecurityContextHolder securityContextHolder;
    
    private static final UUID TENANT_1 = UUID.fromString("11111111-1111-1111-1111-111111111111");
    private static final UUID TENANT_2 = UUID.fromString("22222222-2222-2222-2222-222222222222");
    
    @BeforeEach
    void setUp() {
        securityContextHolder.setCurrentContext(
            new TestSecurityContext(TENANT_1)
        );
    }
    
    @Test
    void findAll_ShouldRespectTenantIsolation() {
        List<Document> documents = documentRepository
            .findAll(SearchParams.empty())
            .getContent();
            
        assertThat(documents)
            .allMatch(doc -> TENANT_1.equals(doc.getTenantId()));
    }
    
    @Test
    void save_ShouldSetAuditFields() {
        Document document = new Document();
        document.setTitle("Test Document");
        
        Document saved = documentRepository.save(document);
        
        assertThat(saved.getCreatedBy()).isNotNull();
        assertThat(saved.getCreatedAt()).isNotNull();
        assertThat(saved.getTenantId()).isEqualTo(TENANT_1);
    }
    
    @Test
    void complexQuery_ShouldRespectSecurity() {
        SearchParams params = SearchParams.criteria(
            FilterCriteria.like("title", "Test%"),
            FilterCriteria.eq("status", "ACTIVE")
        ).build();
        
        List<Document> results = documentRepository
            .findAll(params)
            .getContent();
            
        assertThat(results)
            .allMatch(doc -> TENANT_1.equals(doc.getTenantId()));
    }
}
```

### Testing UUID Repositories

```java
@SpringBootTest
@Import({TestSecurityConfig.class, TestContainersConfig.class})
@Transactional
@Sql("/secure-uuid-test-data.sql")
class UuidDocumentRepositoryTest {
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private TestSecurityContextHolder securityContextHolder;
    
    private static final UUID TENANT_1 = UUID.fromString("11111111-1111-1111-1111-111111111111");
    private static final UUID TENANT_2 = UUID.fromString("22222222-2222-2222-2222-222222222222");
    
    @BeforeEach
    void setUp() {
        securityContextHolder.setCurrentContext(
            new TestSecurityContext(TENANT_1)
        );
    }
    
    @Test
    void findByUuid_ShouldRespectTenantIsolation() {
        // Arrange
        Document document = new Document();
        document.setTitle("Test Doc");
        Document saved = documentRepository.save(document);
        UUID uuid = saved.getUuid();
        
        // Act & Assert
        assertThat(documentRepository.findByUuid(uuid))
            .isPresent()
            .map(Document::getTenantId)
            .hasValue(TENANT_1);
    }
    
    @Test
    void deleteByUuid_ShouldOnlyDeleteForCurrentTenant() {
        // Arrange
        Document document = new Document();
        document.setTitle("To Delete");
        Document saved = documentRepository.save(document);
        UUID uuid = saved.getUuid();
        
        // Switch tenant
        securityContextHolder.setCurrentContext(
            new TestSecurityContext(TENANT_2)
        );
        
        // Act - try to delete from wrong tenant
        documentRepository.deleteByUuid(uuid);
        
        // Assert - document still exists for original tenant
        securityContextHolder.setCurrentContext(
            new TestSecurityContext(TENANT_1)
        );
        assertThat(documentRepository.findByUuid(uuid)).isPresent();
    }
    
    @Test
    void save_ShouldGenerateUuidAndSetAuditFields() {
        Document document = new Document();
        document.setTitle("UUID Test");
        
        Document saved = documentRepository.save(document);
        
        assertThat(saved.getUuid()).isNotNull();
        assertThat(saved.getCreatedBy()).isNotNull();
        assertThat(saved.getCreatedAt()).isNotNull();
        assertThat(saved.getTenantId()).isEqualTo(TENANT_1);
    }
}
```

### Testing Events

```java
@SpringBootTest
@Import({TestSecurityConfig.class, TestContainersConfig.class, EventTestConfig.class})
@Transactional
class EventsTest {
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private DocumentEventCollector eventCollector;
    
    @BeforeEach
    void setUp() {
        eventCollector.clear();
    }
    
    @Test
    void save_ShouldTriggerCreateEvents() {
        Document document = new Document();
        document.setTitle("Event Test");
        
        documentRepository.save(document);
        
        assertThat(eventCollector.getEvents())
            .hasSize(2)
            .extracting("class")
            .containsExactly(
                SecureRepositoryEvent.BeforeCreateEvent.class,
                SecureRepositoryEvent.AfterCreateEvent.class
            );
    }
}
```

### Test Configuration

```java
@TestConfiguration
public class TestSecurityConfig {
    @Bean
    @Primary
    public TestSecurityContextHolder securityContextHolder() {
        return new TestSecurityContextHolder();
    }

    @Bean
    public EventPublisher eventPublisher(LocalEventHandlerRepository repository) {
        return new LocalEventPublisher(repository);
    }
}

@Component
public class TestSecurityContextHolder implements SecurityContextProvider {
    private SecurityContext currentContext;

    public void setCurrentContext(SecurityContext context) {
        this.currentContext = context;
    }

    @Override
    public SecurityContext getCurrentContext() {
        return currentContext;
    }
}
```

## Performance Optimization

### Metadata Caching

- **Automatic Caching**: Entity metadata is cached automatically using `ConcurrentHashMap`
- **Thread Safety**: Cache is thread-safe for concurrent access
- **No Reflection Overhead**: Metadata is computed once per entity type
- **Memory Efficient**: Only essential metadata is cached

### Query Performance

1. **Index Strategy**
    - Always index tenant_id columns
    - Create composite indexes for (tenant_id, frequently_filtered_column)
    - Index UUID columns with appropriate database-specific optimizations
    - Consider partial indexes for tenant-specific queries

2. **Security Constraint Optimization**
    - Security filters are added at the SQL level, not in memory
    - Database query planner can optimize tenant-filtered queries
    - Use EXPLAIN to analyze query execution plans

3. **Audit Field Performance**
    - Audit fields are set during entity lifecycle, not via separate queries
    - No additional database round trips for audit operations
    - Consider indexing audit timestamp fields for reporting queries

### Database Optimization

1. **PostgreSQL-Specific Optimizations**
    - Use UUID-specific column types and indexes
    - Consider row-level security policies for additional tenant isolation
    - Use partial indexes for active/common tenant data

2. **Connection and Transaction Management**
    - Use appropriate transaction boundaries
    - Consider read-only transactions for query-only operations
    - Pool connections efficiently for multi-tenant applications

## Troubleshooting

### Common Issues

1. **Missing Tenant ID**
    - **Symptom**: `SecureDataException: "Failed to set tenant ID"`
    - **Solution**: Ensure SecurityContextProvider returns valid tenant ID
    - **Debug**: Check security context implementation and session management

2. **Access Denied Errors**
    - **Symptom**: `SecureDataException: "Access denied"`
    - **Solution**: Verify user roles match @SecureAccess configuration
    - **Debug**: Log current user roles and entity access requirements

3. **UUID Not Found**
    - **Symptom**: Empty Optional from findByUuid
    - **Check**: Entity exists and belongs to current tenant
    - **Debug**: Verify tenant isolation is working correctly

4. **Performance Issues**
    - **Symptom**: Slow queries with security constraints
    - **Solution**:
        - Verify indexes on tenant_id columns
        - Add indexes on UUID columns
        - Consider composite indexes (tenant_id, uuid)
        - Analyze query execution plans

5. **Event Handler Issues**
    - **Symptom**: Events not firing or wrong handlers called
    - **Solution**: Check @EventHandler typeParameter matches exactly
    - **Debug**: Verify event publisher configuration and handler registration

### Debugging

Enable comprehensive logging:
```properties
# Query debugging
logging.level.org.springframework.jdbc.core=TRACE
logging.level.io.preboot.query=DEBUG

# Security debugging  
logging.level.io.preboot.securedata=DEBUG

# Event debugging
logging.level.io.preboot.eventbus=DEBUG
```

### Migration from Non-Secure Repositories

1. **Step-by-step Migration**
   ```java
   // Before: Standard repository
   public interface DocumentRepository extends FilterableRepository<Document, Long> {}
   
   // After: Secure repository  
   public interface DocumentRepository extends SecureRepository<Document, Long> {}
   
   // Update implementation
   @Repository
   public class DocumentRepositoryImpl extends SecureRepositoryImpl<Document, Long> {
       public DocumentRepositoryImpl(SecureRepositoryContext context) {
           super(context, Document.class);
       }
   }
   ```

2. **Add Security Annotations**
   ```java
   @Table("documents")
   @SecureAccess(read = @AccessRule(roles = {"USER"}))  // Add security
   public class Document {
       @Tenant  // Add tenant field
       private UUID tenantId;
       
       // Existing fields...
   }
   ```

3. **Database Schema Updates**
   ```sql
   -- Add tenant column
   ALTER TABLE documents ADD COLUMN tenant_id UUID NOT NULL;
   
   -- Add audit columns (optional)
   ALTER TABLE documents ADD COLUMN created_by UUID;
   ALTER TABLE documents ADD COLUMN created_at TIMESTAMP;
   ALTER TABLE documents ADD COLUMN modified_by UUID;
   ALTER TABLE documents ADD COLUMN modified_at TIMESTAMP;
   
   -- Add indexes
   CREATE INDEX idx_documents_tenant_id ON documents(tenant_id);
   CREATE INDEX idx_documents_uuid ON documents(uuid);
   ```

## Advanced Use Cases

### Custom Security Rules

```java
@Table("projects")
@SecureAccess(
    read = @AccessRule(roles = {"PROJECT_VIEWER", "PROJECT_MANAGER"}),
    write = @AccessRule(roles = {"PROJECT_MANAGER", "ADMIN"})
)
public class Project implements HasUuid {
    @Id
    private Long id;
    
    private UUID uuid;
    
    @Tenant
    private UUID tenantId;
    
    private String name;
    private String status;
    private UUID ownerId;  // Additional ownership layer
    
    @CreatedBy
    private UUID createdBy;
    
    @CreatedAt
    private Instant createdAt;
}
```

### Multi-level Security with Business Logic

```java
@Service
public class ProjectService {
    private final ProjectRepository projectRepository;
    private final SecurityContextProvider securityContextProvider;
    
    public List<Project> findUserProjects() {
        UUID currentUserId = securityContextProvider.getCurrentContext().getUserId();
        
        // Combine security filtering with business logic
        SearchParams params = SearchParams.criteria(
            FilterCriteria.or(Arrays.asList(
                FilterCriteria.eq("ownerId", currentUserId),
                FilterCriteria.eq("createdBy", currentUserId)
            ))
        ).build();
        
        return projectRepository.findAll(params).getContent();
    }
}
```

### Integration with External Security Systems

```java
@Component
public class JwtSecurityContextProvider implements SecurityContextProvider {
    private final JwtTokenProvider jwtTokenProvider;
    
    @Override
    public SecurityContext getCurrentContext() {
        String token = getCurrentJwtToken();
        Claims claims = jwtTokenProvider.parseClaims(token);
        
        return new SimpleSecurityContext(
            UUID.fromString(claims.getSubject()),
            UUID.fromString(claims.get("tenantId", String.class)),
            extractRoles(claims),
            extractPermissions(claims)
        );
    }
    
    private Set<String> extractRoles(Claims claims) {
        // Extract roles from JWT claims
        return new HashSet<>(claims.get("roles", List.class));
    }
}
```
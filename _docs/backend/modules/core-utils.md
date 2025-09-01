---
layout: documentation
title: "Core Utilities"
subtitle: "Essential utilities and tools for Spring Boot applications, including TTL collections and JSON helpers."
permalink: /docs/backend/modules/core-utils/
section: backend
---
# PreBoot Core

## Overview

The PreBoot Core module provides essential utilities and tools for Spring Boot applications. This module includes thread-safe collections with TTL functionality, JSON processing utilities, transaction management wrappers, and bean validation helpers.

## Quick Start

### 1. Add Dependency

Add the following dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-core</artifactId>
    <version>${preboot.version}</version>
</dependency>
```

### 2. Enable Auto-Configuration

The module automatically configures itself when added to a Spring Boot application. No additional configuration is required.

### 3. Start Using

```java
@Service
public class UserService {
    
    private final TTLMap<String, User> userCache = new TTLMap<>(300); // 5 minutes TTL
    private final JsonMapper jsonMapper = JsonMapperFactory.createJsonMapper();
    
    @Autowired
    private TransactionWrapper transactionWrapper;
    
    public User getUserWithCache(String userId) {
        return userCache.computeIfAbsent(userId, id -> 
            transactionWrapper.doInTransaction(() -> userRepository.findById(id))
        );
    }
}
```

## Core Components

### TTLMap - Time-To-Live Map

A thread-safe map implementation where entries automatically expire after a specified time period.

#### When to Use
- Caching frequently accessed data that can become stale
- Rate limiting implementations
- Session management
- Temporary data storage that should auto-cleanup

#### Basic Usage

```java
// Create a map with 60-second default TTL
TTLMap<String, UserSession> sessionMap = new TTLMap<>(60);

// Store with default TTL
sessionMap.put("session123", userSession);

// Store with custom TTL (30 seconds)
sessionMap.put("tempSession", userSession, 30);

// Retrieve (returns null if expired)
UserSession session = sessionMap.get("session123");

// Check existence (accounts for expiration)
boolean exists = sessionMap.containsKey("session123");

// Clean up when done
sessionMap.close(); // Important: removes from cleanup service
```

#### Advanced Usage

```java
// Using try-with-resources pattern
try (TTLMap<String, CacheData> cache = new TTLMap<>(120)) {
    cache.put("key1", data1);
    cache.put("key2", data2, 300); // Custom TTL
    
    // Process data
    processData(cache);
} // Automatically cleaned up
```

#### Key Features
- **Thread-safe**: Can be safely used across multiple threads
- **Shared cleanup service**: All TTLMap instances share a single cleanup thread
- **Automatic expiration**: Entries are automatically removed when expired
- **Custom TTL per entry**: Each entry can have its own expiration time
- **Memory efficient**: Expired entries are actively cleaned up

#### Best Practices
- Always call `close()` when done with a TTLMap instance
- Use appropriate TTL values based on your data freshness requirements
- Monitor memory usage when storing large objects
- Consider using try-with-resources for automatic cleanup

### JSON Processing

A flexible JSON processing framework built on Jackson with sensible defaults for Spring Boot applications.

#### When to Use
- Serializing/deserializing objects to/from JSON
- Working with REST APIs
- Configuration file processing
- Data transformation between systems

#### Basic Usage

```java
JsonMapper jsonMapper = JsonMapperFactory.createJsonMapper();

// Serialize object to JSON
String json = jsonMapper.toJson(myObject);

// Deserialize JSON to object
MyClass object = jsonMapper.fromJson(json, MyClass.class);

// Access underlying ObjectMapper for advanced features
ObjectMapper objectMapper = jsonMapper.getObjectMapper();
```

#### Working with Java Time API

```java
public class Event {
    private LocalDateTime timestamp;
    private ZonedDateTime scheduledTime;
    // getters/setters
}

Event event = new Event();
event.setTimestamp(LocalDateTime.now());
event.setScheduledTime(ZonedDateTime.now());

// Serializes using ISO-8601 format
String json = jsonMapper.toJson(event);
// {"timestamp":"2024-01-15T10:30:00","scheduledTime":"2024-01-15T10:30:00+01:00[Europe/Warsaw]"}

// Deserializes back to objects
Event deserialized = jsonMapper.fromJson(json, Event.class);
```

#### Configuration Features
- **JSR-310 support**: Full Java Time API support with ISO-8601 formatting
- **Parameter names**: Supports constructor parameter names without @JsonProperty
- **Lenient deserialization**: Ignores unknown properties by default
- **Zone ID serialization**: Includes timezone information in serialized dates
- **No timestamp mode**: Dates are serialized as strings, not timestamps

#### Error Handling

```java
try {
    MyObject obj = jsonMapper.fromJson(invalidJson, MyObject.class);
} catch (JsonParsingException e) {
    logger.error("Failed to parse JSON: {}", e.getMessage(), e);
    // Handle parsing error appropriately
}
```

### Transaction Management

A functional wrapper around Spring's transaction management, providing cleaner code and better testability.

#### When to Use
- When you need programmatic transaction control
- For complex business operations spanning multiple service calls
- When you need to ensure operations run in new transactions
- For better separation of transaction logic from business logic

#### Basic Usage

```java
@Service
public class OrderService {
    
    @Autowired
    private TransactionWrapper transactionWrapper;
    
    public Order processOrder(OrderRequest request) {
        return transactionWrapper.doInTransaction(() -> {
            Order order = createOrder(request);
            updateInventory(order);
            sendNotification(order);
            return order;
        });
    }
    
    public void processInBackground(Order order) {
        transactionWrapper.doInTransaction(() -> {
            updateOrderStatus(order);
            logOrderProcessing(order);
        });
    }
}
```

#### New Transaction Usage

```java
public class AuditService {
    
    @Autowired
    private TransactionWrapper transactionWrapper;
    
    public void auditOperation(String operation, String details) {
        // Always runs in a new transaction, even if current transaction fails
        transactionWrapper.doAlwaysInNewTransaction(() -> {
            AuditLog log = new AuditLog(operation, details);
            auditRepository.save(log);
        });
    }
    
    public String processWithAudit(String data) {
        return transactionWrapper.doAlwaysInNewTransaction(() -> {
            String result = processData(data);
            auditOperation("PROCESS", "Data processed successfully");
            return result;
        });
    }
}
```

#### Key Features
- **Functional interface**: Clean, readable code without transaction annotations
- **Return values**: Supports both void and return-value operations
- **New transaction support**: Force operations to run in separate transactions
- **Spring integration**: Full integration with Spring's transaction management
- **Exception handling**: Proper transaction rollback on exceptions

#### Transaction Behavior
- `doInTransaction()`: Participates in existing transaction or creates new one
- `doAlwaysInNewTransaction()`: Always creates a new transaction (REQUIRES_NEW)

### Bean Validation

Utility for validating Java beans using Jakarta Validation with simplified API.

#### When to Use
- Validating domain objects before persistence
- Input validation in service layers
- Data validation in batch processing
- API request validation

#### Basic Usage

```java
public class User {
    @NotNull(message = "Username cannot be null")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    private String username;
    
    @Email(message = "Email must be valid")
    @NotNull(message = "Email cannot be null")
    private String email;
    
    @Min(value = 18, message = "Age must be at least 18")
    private Integer age;
}

// Validate an object
User user = new User();
user.setUsername("john");
user.setEmail("invalid-email");
user.setAge(16);

try {
    BeanValidator.validate(user);
} catch (ConstraintViolationException e) {
    e.getConstraintViolations().forEach(violation -> 
        System.out.println(violation.getPropertyPath() + ": " + violation.getMessage())
    );
}
```

#### Service Layer Integration

```java
@Service
public class UserService {
    
    public User createUser(UserCreateRequest request) {
        // Validate request
        BeanValidator.validate(request);
        
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        
        // Validate domain object before saving
        BeanValidator.validate(user);
        
        return userRepository.save(user);
    }
}
```

#### Custom Validation Groups

```java
public class User {
    @NotNull(groups = Creation.class)
    private String username;
    
    @NotNull(groups = {Creation.class, Update.class})
    private String email;
    
    public interface Creation {}
    public interface Update {}
}

// Validate with specific group
BeanValidator.validate(user, Creation.class);
```

## Configuration

### Dependencies

The module requires the following Spring Boot starters to be present in your application:

**Required:**
- `spring-boot-starter` - Core Spring Boot functionality
- `spring-boot-starter-json` - For JSON processing features

**Optional (for specific features):**
- `spring-tx` - For TransactionWrapper functionality
- `spring-boot-starter-validation` - For BeanValidator functionality

### Auto-Configuration

The module automatically configures when added to your Spring Boot application classpath. The auto-configuration:

- Scans the `io.preboot` package for components
- Registers the `TransactionWrapper` bean
- Sets up component scanning for all preboot modules

No manual configuration is required unless you want to customize behavior.

## Best Practices

### General Guidelines

1. **Dependency Management**: Use the preboot BOM for consistent versioning
2. **Error Handling**: Always handle exceptions appropriately for your use case
3. **Resource Management**: Clean up resources (especially TTLMap instances)
4. **Testing**: All components are designed to be easily testable

### TTLMap Best Practices

```java
// Good: Using try-with-resources
try (TTLMap<String, Data> cache = new TTLMap<>(300)) {
    // Use cache
    processWithCache(cache);
} // Automatically cleaned up

// Good: Explicit cleanup in service
@PreDestroy
public void cleanup() {
    if (cache != null) {
        cache.close();
    }
}

// Avoid: Not cleaning up
TTLMap<String, Data> cache = new TTLMap<>(300);
// ... use cache but never call close()
```

### JSON Processing Best Practices

```java
// Good: Reuse JsonMapper instances
@Service
public class DataService {
    private final JsonMapper jsonMapper = JsonMapperFactory.createJsonMapper();
    
    public String serialize(Object obj) {
        return jsonMapper.toJson(obj);
    }
}

// Avoid: Creating new instances repeatedly
public String serialize(Object obj) {
    return JsonMapperFactory.createJsonMapper().toJson(obj); // Inefficient
}
```

### Transaction Management Best Practices

```java
// Good: Keep transactions focused
public Order processOrder(OrderRequest request) {
    return transactionWrapper.doInTransaction(() -> {
        Order order = createOrder(request);
        updateInventory(order);
        return order;
    });
}

// Good: Use new transactions for independent operations
public void processOrderWithAudit(OrderRequest request) {
    Order order = processOrder(request);
    
    // Audit in separate transaction
    transactionWrapper.doAlwaysInNewTransaction(() -> 
        auditService.logOrderCreation(order)
    );
}
```

## Error Handling

### Common Exceptions

| Exception | Cause | Handling Strategy |
|-----------|-------|------------------|
| `JsonParsingException` | Invalid JSON format or mapping issues | Log error, return default value or re-throw as business exception |
| `ConstraintViolationException` | Bean validation failures | Extract violation messages, return validation errors to user |

### Exception Handling Examples

```java
// JSON parsing with error handling
public Optional<User> parseUser(String json) {
    try {
        return Optional.of(jsonMapper.fromJson(json, User.class));
    } catch (JsonParsingException e) {
        logger.warn("Failed to parse user JSON: {}", e.getMessage());
        return Optional.empty();
    }
}

// Validation with error collection
public List<String> validateUser(User user) {
    try {
        BeanValidator.validate(user);
        return Collections.emptyList();
    } catch (ConstraintViolationException e) {
        return e.getConstraintViolations()
                .stream()
                .map(violation -> violation.getPropertyPath() + ": " + violation.getMessage())
                .collect(Collectors.toList());
    }
}
```

## Troubleshooting

### Common Issues

**TTLMap entries not expiring**
- Check if `close()` was called prematurely
- Verify TTL values are reasonable (not too short/long)
- Monitor system clock changes

**JSON parsing failures**
- Ensure proper Jackson modules are on classpath
- Check for circular references in objects
- Verify date/time formats match expected patterns

**Transaction not working**
- Ensure `@EnableTransactionManagement` is present
- Check if transaction manager bean is configured
- Verify method calls are going through Spring proxies

**Validation not working**
- Ensure Jakarta Validation implementation is on classpath
- Check if validation annotations are properly placed
- Verify constraint validator implementations are available

### Performance Considerations

- **TTLMap**: Cleanup runs every second - consider impact with many instances
- **JsonMapper**: Instances are thread-safe and should be reused
- **TransactionWrapper**: Adds minimal overhead, safe for frequent use
- **BeanValidator**: Creates validator factory per call - consider caching for high-frequency validation

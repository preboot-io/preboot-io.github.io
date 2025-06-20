---
layout: documentation
title: "Observability"
subtitle: "Utilities for monitoring application performance, such as the @WarnOnSlowCall aspect."
permalink: /docs/modules/observability/
---
# Slow Call Warning Documentation

## Overview
The Slow Call Warning system is a monitoring utility that helps developers identify methods that take longer than expected to execute. It consists of two main components:

- `@WarnOnSlowCall`: An annotation that marks methods for execution time monitoring
- `SlowCallWarningAspect`: An aspect that implements the monitoring logic using Spring AOP

This utility is particularly useful for:
- Identifying performance bottlenecks in production
- Monitoring critical method execution times
- Setting performance baselines for important operations
- Early detection of degrading performance

## Installation

Add the following files to your project:
- `WarnOnSlowCall.java` in your observability package
- `SlowCallWarningAspect.java` in the same package

Ensure you have the following dependencies in your project:
```xml
<dependencies>
    <dependency>
        <groupId>io.preboot</groupId>
        <artifactId>preboot-observability</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

## Usage Examples

### Basic Usage
```java
@Service
public class UserService {
    
    @WarnOnSlowCall
    public User findUserById(Long id) {
        // Method implementation
        return userRepository.findById(id);
    }
}
```
In this example, a warning will be logged if the method takes longer than 1000ms (default threshold) to execute.

### Custom Threshold
```java
@Service
public class ReportService {
    
    @WarnOnSlowCall(threshold = 5000)
    public Report generateComplexReport(LocalDate startDate, LocalDate endDate) {
        // Complex report generation logic
        return report;
    }
}
```
This will log a warning if the report generation takes longer than 5000ms.

### Multiple Methods Monitoring
```java
@RestController
public class ProductController {
    
    @WarnOnSlowCall(threshold = 200)
    @GetMapping("/products")
    public List<Product> getAllProducts() {
        return productService.findAll();
    }
    
    @WarnOnSlowCall(threshold = 100)
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }
}
```

## Warning Output Format

When a method exceeds its threshold, a warning message is logged with the following format:
```
WARN  c.a.t.o.SlowCallWarningAspect - methodName(param1, param2) executed in XXXms ----- attention
```

Example:
```
WARN  c.a.t.o.SlowCallWarningAspect - getProduct(123) executed in 150ms ----- attention
```

## Best Practices

1. **Choose Appropriate Thresholds**
    - Consider the expected execution time for each method
    - Account for normal variations in system load
    - Set different thresholds for different types of operations

2. **Strategic Placement**
    - Apply to critical business operations
    - Monitor external service calls
    - Watch database operations
    - Track API endpoint performance

3. **Log Management**
    - Configure log aggregation to collect these warnings
    - Set up alerts for frequent warnings
    - Analyze trends in execution times

4. **Performance Impact**
    - The aspect adds minimal overhead
    - Safe for production use
    - Consider the number of annotated methods

## Troubleshooting

### Common Issues

1. **Annotation Not Working**
    - Ensure Spring AOP is properly configured
    - Verify the aspect is being picked up by component scanning
    - Check that the method is public (AOP limitation)

2. **Missing Logs**
    - Verify logging configuration
    - Check log levels (WARN level must be enabled)
    - Ensure the aspect is in the correct package

## Technical Details

### Annotation Properties
- `threshold`: Long value in milliseconds (default: 1000)
- Runtime retention
- Applicable to methods only

### Aspect Features
- Uses Around advice
- Captures method parameters
- Handles null parameters gracefully
- Thread-safe execution time calculation

## Performance Considerations

The SlowCallWarningAspect is designed to have minimal impact on application performance:
- Lightweight time measurement using `System.currentTimeMillis()`
- Lazy parameter string construction (only when threshold is exceeded)
- No synchronization overhead
- Warning logging only when threshold is exceeded

## Integration Tips

### With Monitoring Systems
```java
@Service
public class MonitoredService {
    
    @WarnOnSlowCall(threshold = 500)
    @Timed(value = "service.operation") // Compatible with Micrometer
    public void importantOperation() {
        // Implementation
    }
}
```

### With Existing Logging
```java
@Service
public class LoggedService {
    
    @WarnOnSlowCall(threshold = 300)
    @Slf4j
    public void criticalOperation() {
        log.info("Starting critical operation");
        // Implementation
        log.info("Completed critical operation");
    }
}
```

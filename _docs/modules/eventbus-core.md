---
layout: documentation
title: "Core EventBus"
subtitle: "A lightweight, in-memory event bus for synchronous and asynchronous communication."
permalink: /docs/modules/eventbus-core/
---
# preboot-eventbus-core

## Overview
The `preboot-eventbus-core` is a lightweight, Spring Boot-compatible event bus implementation that provides both synchronous and asynchronous event publishing capabilities. It allows for easy event-driven communication between components in a Spring application while maintaining loose coupling.

## Features
- Synchronous and asynchronous event publishing
- Annotation-based event handlers
- Priority-based handler execution
- Generic event type parameter filtering
- Automatic handler discovery through Spring context
- Configurable error handling for missing handlers
- Thread-safe event publishing

## Installation

Add the following dependency to your project:

```xml
<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-eventbus-core</artifactId>
    <version>{version}</version>
</dependency>
```

## Core Components

### Event Handler
Events are handled using the `@EventHandler` annotation on methods. The handler method must:
- Be in a public class
- Have exactly one parameter (the event type)
- Be a public method

```java
public class UserEventHandler {
    @EventHandler(priority = 100)
    public void handleUserCreated(UserCreatedEvent event) {
        // Handle the event
    }
}
```

The `priority` parameter is optional (default = 0). Higher priority handlers are executed first.

#### Generic Event Handling
For events with generic type parameters, you can use the `typeParameter` attribute in the `@EventHandler` annotation to filter events based on their generic type:

```java
public class GenericEventHandler {
    @EventHandler(typeParameter = String.class)
    public void handleStringEvent(DataEvent<String> event) {
        // Only handles DataEvent<String>
    }

    @EventHandler(typeParameter = Integer.class)
    public void handleIntegerEvent(DataEvent<Integer> event) {
        // Only handles DataEvent<Integer>
    }
}
```

### Generic Events
To create an event with generic type parameters, implement the `GenericEvent<T>` interface:

```java
public class DataEvent<T> implements GenericEvent<T> {
    private final T data;

    public DataEvent(T data) {
        this.data = data;
    }

    @Override
    public T getTypeParameter() {
        return data;
    }
}
```

### Event Publishers

#### Synchronous Publishing
The `LocalEventPublisher` provides synchronous event publishing:

```java
@Service
public class UserService {
    private final EventPublisher eventPublisher;

    public UserService(EventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void createUser(String username) {
        // Business logic
        eventPublisher.publish(new UserCreatedEvent(username));
    }

    public void processData(String data) {
        // Publishing generic event
        eventPublisher.publish(new DataEvent<>(data));
    }
}
```

#### Asynchronous Publishing
The `LocalAsynchronousEventPublisher` handles events asynchronously using a provided executor:

```java
@Service
public class NotificationService {
    private final AsynchronousEventPublisher asyncEventPublisher;

    public NotificationService(AsynchronousEventPublisher asyncEventPublisher) {
        this.asyncEventPublisher = asyncEventPublisher;
    }

    public void sendNotification(String message) {
        asyncEventPublisher.publish(new NotificationEvent(message));
    }
}
```

## Configuration

### Basic Setup
```java
@Configuration
public class EventBusConfig {
    
    @Bean
    public LocalEventHandlerRepository localEventHandlerRepository(ApplicationContext applicationContext) {
        return new LocalEventHandlerRepository(applicationContext);
    }
    
    @Bean
    public EventPublisher eventPublisher(LocalEventHandlerRepository repository) {
        return new LocalEventPublisher(repository);
    }
    
    @Bean
    public AsynchronousEventPublisher asyncEventPublisher(
            LocalEventHandlerRepository repository,
            @Qualifier("eventExecutor") Executor executor) {
        return new LocalAsynchronousEventPublisher(repository, executor);
    }
    
    @Bean("eventExecutor")
    public Executor eventExecutor() {
        return Executors.newFixedThreadPool(5);
    }
}
```

### Configuring Error Handling
Missing handlers are ignored by default, you can annotate an event payload class with @ExceptionIfNoHandler to throw an exception if no handler is found:

```java
@ExceptionIfNoHandler
public record CustomEvent() {}
```

## Error Handling

### NoEventHandlerException
Thrown when no handler is found for an event (if @ExceptionIfNoHandler is used on the event class):

```java
try {
    eventPublisher.publish(new CustomEvent());
} catch (NoEventHandlerException e) {
    // Handle missing handler
}
```

### EventPublishException
Thrown when an error occurs during event handling:

```java
try {
    eventPublisher.publish(new CustomEvent());
} catch (EventPublishException e) {
    // Handle event processing error
}
```

## Best Practices

1. **Event Class Design**
   - Make events immutable
   - Include all necessary data in the event object
   - Use meaningful names that end with "Event"
   - For generic events, implement the GenericEvent interface
   - Ensure getTypeParameter() returns the actual type parameter instance

```java
// Simple event
public record UserCreatedEvent(
    String username,
    LocalDateTime createdAt
) {}

// Generic event
public record DataProcessedEvent<T>(
    T data,
    LocalDateTime processedAt
) implements GenericEvent<T> {
    @Override
    public T getTypeParameter() {
        return data;
    }
}
```

2. **Handler Design**
   - Keep handlers focused on a single responsibility
   - Avoid long-running operations in synchronous handlers
   - Use asynchronous publishing for time-consuming tasks
   - Handle exceptions appropriately
   - Use typeParameter in @EventHandler when handling generic events
   - Consider creating separate handlers for different generic type parameters

3. **Priority Usage**
   - Use priorities sparingly
   - Document priority values in your codebase
   - Consider using constants for priority values

4. **Testing**
   - Write unit tests for event handlers
   - Test both synchronous and asynchronous scenarios
   - Verify handler execution order when using priorities
   - Test generic event type parameter filtering

## Example: Complete Usage Scenario with Generic Events

```java
// Generic event class
public record ProcessingEvent<T>(
    String processId,
    T data,
    LocalDateTime timestamp
) implements GenericEvent<T> {
    @Override
    public T getTypeParameter() {
        return data;
    }
}

// Event handlers
@Service
public class ProcessingEventHandler {
    private final NotificationService notificationService;
    
    @EventHandler(priority = 100, typeParameter = String.class)
    public void handleStringProcessing(ProcessingEvent<String> event) {
        // Process string data
        notificationService.notifyStringProcessing(event.processId());
    }

    @EventHandler(priority = 100, typeParameter = Integer.class)
    public void handleIntegerProcessing(ProcessingEvent<Integer> event) {
        // Process integer data
        notificationService.notifyIntegerProcessing(event.processId());
    }
}

// Event publisher usage
@Service
public class ProcessingService {
    private final EventPublisher eventPublisher;
    
    public void processString(String data) {
        ProcessingEvent<String> event = new ProcessingEvent<>(
            UUID.randomUUID().toString(),
            data,
            LocalDateTime.now()
        );
        eventPublisher.publish(event);
    }

    public void processInteger(Integer data) {
        ProcessingEvent<Integer> event = new ProcessingEvent<>(
            UUID.randomUUID().toString(),
            data,
            LocalDateTime.now()
        );
        eventPublisher.publish(event);
    }
}
```

## Thread Safety Considerations
- The event bus is thread-safe for publishing events
- Handler registration is performed on first use
- Handlers are stored in a thread-safe manner
- Asynchronous publishing uses the provided executor for thread management

## Performance Considerations
- Handler scanning is deferred until first use
- Handlers are cached after initial scanning
- Priority sorting is done during registration, not during publishing
- Consider using appropriate executor configurations for asynchronous publishing

## Limitations
- Events must have at least one handler (unless configured to ignore)
- Handler methods must have exactly one parameter
- Handler classes must be public
- No built-in event persistence
- No distributed event publishing (local JVM only)
- Generic event type parameter filtering only works with GenericEvent implementations

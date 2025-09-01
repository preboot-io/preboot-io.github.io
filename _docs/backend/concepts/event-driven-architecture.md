---
layout: documentation
title: "Event-Driven Architecture"
subtitle: "Learn how PreBoot leverages events for building loosely coupled and scalable systems."
permalink: /docs/backend/concepts/event-driven-architecture/
section: backend
---
## Overview

Event-Driven Architecture (EDA) is a fundamental paradigm in the PreBoot framework, designed to promote loose coupling and enhance scalability. Instead of components calling each other directly, they communicate by producing and consuming events. This allows different modules of your application to react to changes in the system without being tightly bound to the source of the change.

PreBoot provides a comprehensive event system that serves as the backbone for communication between its various modules, from user authentication to data persistence.

**Key Benefits of EDA in PreBoot:**
* **Decoupling:** Modules don't need direct knowledge of other modules. For example, the authentication module (`preboot-auth`) publishes a `UserAccountCreatedEvent`, and the notifications module can listen to it to send a welcome email, without the auth module knowing anything about email sending.
* **Extensibility:** You can easily add new functionality by creating new event listeners without modifying existing code. For instance, you could add a handler that assigns a user to a trial plan upon the `UserAccountCreatedEvent`.
* **Asynchronous Workflows:** Long-running tasks can be offloaded to asynchronous handlers, preventing blocking of the main application thread and improving responsiveness.
* **Reliability:** For critical operations, events can be persisted as tasks in a database, ensuring they are processed even in case of an application failure.

## Core Components of the Event System

The event system in PreBoot is primarily powered by the `preboot-eventbus-core` module and consists of three main components.

### 1. Events

An event is a simple, immutable Plain Old Java Object (POJO) or a Java Record that represents a significant occurrence in the application.

* **Simple Event:** A straightforward data carrier.
    ```java
    // Represents a user being successfully created
    public record UserAccountCreatedEvent(UUID tenantId, UUID userAccountId, String email, String username) {}
    ```
* **Generic Event:** For handling events with generic type parameters, PreBoot uses a `GenericEvent<T>` interface. This is useful for creating reusable event structures.
    ```java
    // An event that can carry any type of data
    public record DataProcessedEvent<T>(T data, LocalDateTime processedAt) implements GenericEvent<T> {
        @Override
        public T getTypeParameter() {
            return data;
        }
    }
    ```

### 2. Event Publishers

Publishers are responsible for sending events to the event bus. PreBoot offers different publishers depending on the required processing model.

* **`EventPublisher` (Synchronous):**
    * This is the standard, synchronous publisher (`LocalEventPublisher`).
    * When an event is published, all its handlers are executed immediately and sequentially within the same thread.
    * **Use case:** When you need immediate and consistent processing before the original method continues. For example, validating data with a `BeforeCreateEvent` listener.

    ```java
    @Service
    public class UserService {
        private final EventPublisher eventPublisher;

        // ... constructor ...

        public void createUser(String username) {
            // ... business logic ...
            // This event is published and handled instantly
            eventPublisher.publish(new UserCreatedEvent(...));
        }
    }
    ```

* **`AsynchronousEventPublisher`:**
    * This publisher (`LocalAsynchronousEventPublisher`) executes handlers in a separate thread pool.
    * The `publish` method returns immediately, not waiting for handlers to complete.
    * **Use case:** For long-running tasks like sending emails, processing images, or calling external APIs, where you don't want to block the user-facing thread.

* **`TaskPublisher` (Reliable & Persistent):**
    * Provided by the `preboot-eventbus-tasks` module, this publisher persists events as tasks in a database before they are processed.
    * This guarantees that the event will be handled even if the application crashes right after publishing.
    * It supports features like automatic retries with backoff policies and a dead-letter queue for tasks that repeatedly fail.
    * **Use case:** For critical business operations that must not be lost, such as payment processing, order fulfillment, or generating invoices.

### 3. Event Handlers

An Event Handler is a method that listens for and processes a specific type of event.

* Handlers are defined by annotating a public method with `@EventHandler`.
* The method must accept exactly one parameter: the event object it is supposed to handle.
* Handlers are automatically discovered by PreBoot from your Spring-managed beans.
* You can control the execution order of handlers for the same event using the `priority` attribute.

```java
@Component
public class UserNotificationHandler {

    @EventHandler(priority = 100)
    public void handleUserCreation(UserAccountCreatedEvent event) {
        // Logic to send a welcome email or notification
        log.info("New user created: {}. Sending welcome email.", event.username());
    }

    // Example of handling a repository event from preboot-securedata
    @EventHandler
    public void handleDataUpdate(SecureRepositoryEvent.AfterUpdateEvent<MyEntity> event) {
        // Logic to clear a related cache
        log.info("Entity updated. Invalidating cache for ID: {}", event.getEntity().getId());
    }
}
```

## Common Event-Driven Workflows in PreBoot

Here are some examples of how PreBoot modules use the event system to communicate:

1.  **New User Registration (`preboot-auth` -> `preboot-notifications`):**
    * A user registers through an API endpoint.
    * The `preboot-auth` module creates a user and publishes a `UserAccountActivationTokenGeneratedEvent`.
    * A handler in the `preboot-notifications` module (or your custom implementation) listens for this event and sends an activation email to the user. The auth module is completely decoupled from the notification logic.

2.  **Data Auditing (`preboot-securedata`):**
    * When an entity is created, updated, or deleted using a `SecureRepository`, events like `AfterCreateEvent`, `AfterUpdateEvent`, and `AfterDeleteEvent` are published.
    * You can create handlers to listen for these events to implement a comprehensive audit trail, log changes to an external system, or trigger other business processes without cluttering your repository or service logic.

3.  **Asynchronous File Export (`preboot-exporters`):**
    * A user requests a large data export.
    * Instead of making the user wait, you can start the process and publish an `ExportRequestEvent`.
    * An asynchronous handler can perform the heavy lifting of generating the file. Once complete, it can publish an `ExportCompletedEvent`.
    * Another handler can listen for the completion event to send a WebSocket message or an email with a download link to the user.

## Best Practices

* **Make Events Immutable:** Events should be simple data carriers representing a fact that has already occurred. Avoid putting mutable objects or business logic inside them.
* **Keep Handlers Focused:** Each handler should have a single responsibility. If you need to perform multiple independent actions, create multiple handlers.
* **Choose the Right Publisher:**
    * Use the synchronous `EventPublisher` for immediate, transactional logic.
    * Use the `AsynchronousEventPublisher` for non-critical, long-running tasks.
    * Use the persistent `TaskPublisher` for critical operations that cannot be lost.
```
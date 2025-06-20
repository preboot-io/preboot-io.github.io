---
layout: documentation
title: "Persistent Tasks"
subtitle: "A database-backed, reliable task execution system with retry and dead-letter queue support."
permalink: /docs/modules/eventbus-tasks/
---
# preboot-eventbus-tasks

## Overview
The `preboot-eventbus-tasks` module provides a robust, database-backed task execution system for Spring Boot applications. It enables reliable event processing with features like retry mechanisms, dead letter queues, and deduplication of tasks. The module is particularly useful for handling asynchronous operations that require persistence and reliability.

## Features
- Persistent task storage using JDBC (supports PostgreSQL and H2)
- Automatic task retry with configurable backoff policies
- Dead letter queue support with configurable policies
- Task deduplication using optional hash values
- Task runner heartbeat monitoring
- Stalled task recovery
- Transaction management
- Priority-based task execution
- JSON serialization of task payloads

## Core Components

### TaskPublisher
Responsible for creating and storing tasks in the database.

```java
@Service
public class YourService {
    private final TaskPublisher taskPublisher;

    public YourService(TaskPublisher taskPublisher) {
        this.taskPublisher = taskPublisher;
    }

    public void scheduleTask(YourEvent event) {
        // Simple publishing
        taskPublisher.publishTask(event);

        // Publishing with deduplication
        String hash = HashUtils.getHash(Map.of("key", event.getId()));
        taskPublisher.publishTask(event, hash);
    }
}
```

### TaskRunner
Executes stored tasks and manages their lifecycle.

```java
@Service
public class TaskExecutionService {
    private final TaskRunner taskRunner;

    public TaskExecutionService(TaskRunner taskRunner) {
        this.taskRunner = taskRunner;
    }

    @Scheduled(fixedDelay = 1000)
    public void executeTask() {
        taskRunner.runTask();
    }

    @Scheduled(fixedDelay = 30000)
    public void updateHeartbeat() {
        taskRunner.updateHeartbeat();
    }

    @Scheduled(fixedDelay = 60000)
    public void checkStalledTasks() {
        taskRunner.retrieveStalledTasks(Instant.now().minus(Duration.ofMinutes(5)));
    }
}
```

## Configuration

### Basic Setup
```java
@Configuration
public class EventBusTasksConfig {
    
    @Bean
    public TaskConfigFactory taskConfigFactory(JdbcTemplate jdbcTemplate, JsonMapper jsonMapper) {
        return new TaskConfigFactory(jdbcTemplate, jsonMapper);
    }
    
    @Bean
    public TaskRepository taskRepository(TaskConfigFactory factory) {
        return factory.createTaskRepository("your_tasks_table");
    }
    
    @Bean
    public TaskPublisher taskPublisher(TaskConfigFactory factory, TaskRepository taskRepository) {
        return factory.createTaskPublisher(taskRepository);
    }
    
    @Bean
    public TaskRunner taskRunner(
            TaskConfigFactory factory,
            EventPublisher eventPublisher,
            TaskRepository taskRepository) {
        return factory.createTaskRunner(
                eventPublisher,
                taskRepository,
                new TimeBasedDeadQueuePolicy(Duration.ofDays(1)),
                new ConstantBackOffPolicy(Duration.ofMinutes(5))
        );
    }
}
```

### Database Schema
The module automatically creates the required database table with the following structure:

```sql
CREATE TABLE your_tasks_table (
    id SERIAL PRIMARY KEY,
    type VARCHAR(255),
    payload TEXT,
    created_at TIMESTAMP WITH TIME ZONE,
    next_run_at TIMESTAMP WITH TIME ZONE,
    started_at TIMESTAMP WITH TIME ZONE,
    fail_count INT,
    error_message TEXT,
    error_stack_trace TEXT,
    completed BOOLEAN,
    completed_at TIMESTAMP WITH TIME ZONE,
    dead BOOLEAN,
    optional_hash TEXT,
    executor_instance_id TEXT,
    heartbeat TIMESTAMP WITH TIME ZONE
);
```

## BackOff Policies

### ConstantBackOffPolicy
Implements a fixed delay between retry attempts:

```java
BackOffPolicy policy = new ConstantBackOffPolicy(Duration.ofMinutes(5));
```

### ExpandingTimeOfBackOffPolicy
Implements an exponential backoff with randomization:

```java
BackOffPolicy policy = new ExpandingTimeOfBackOffPolicy(
    Duration.ofMinutes(1),  // Initial backoff
    30,                     // Random seconds to add
    2,                      // Multiplier for each retry
    60                      // Max backoff in minutes
);
```

## Dead Queue Policies

### TimeBasedDeadQueuePolicy
Moves tasks to dead letter queue based on their age:

```java
DeadQueuePolicy policy = new TimeBasedDeadQueuePolicy(Duration.ofDays(1));
```

## Task Deduplication
The module supports task deduplication using hash values:

```java
public void scheduleUniqueTask(OrderEvent event) {
    String hash = HashUtils.getHash(Map.of(
        "orderId", event.getOrderId(),
        "type", event.getType()
    ));
    try {
        taskPublisher.publishTask(event, hash);
    } catch (TaskHashExistsException e) {
        // Task with this hash already exists
    }
}
```

## Error Handling

### TaskHashExistsException
Thrown when attempting to publish a task with a hash that already exists:

```java
try {
    taskPublisher.publishTask(event, hash);
} catch (TaskHashExistsException e) {
    log.warn("Duplicate task detected: {}", hash);
}
```

## Best Practices

1. **Task Design**
    - Make task classes immutable
    - Include all necessary data in the task object
    - Use meaningful names
    - Implement proper JSON serialization

```java
public class OrderProcessingTask {
    private final String orderId;
    private final String status;
    private final Instant timestamp;

    @JsonCreator
    public OrderProcessingTask(
        @JsonProperty("orderId") String orderId,
        @JsonProperty("status") String status,
        @JsonProperty("timestamp") Instant timestamp
    ) {
        this.orderId = orderId;
        this.status = status;
        this.timestamp = timestamp;
    }

    // Getters
}
```

2. **Runner Configuration**
    - Configure appropriate heartbeat intervals
    - Set reasonable backoff policies
    - Configure dead letter queue policy based on business requirements
    - Monitor stalled tasks regularly

3. **Database Considerations**
    - Regular maintenance of the tasks table
    - Index optimization
    - Archiving completed tasks
    - Monitoring table growth

4. **Error Handling**
    - Implement proper error handling in task handlers
    - Monitor dead letter queue
    - Set up alerts for repeated failures

## Monitoring

1. **Key Metrics to Monitor**
    - Number of pending tasks
    - Task failure rates
    - Dead letter queue size
    - Task processing time
    - Stalled tasks count

2. **Health Checks**
    - Runner heartbeat status
    - Database connectivity
    - Task processing latency

## Limitations
- Single database support (no distributed task queue)
- No built-in task prioritization beyond execution order
- No task cancellation mechanism
- No built-in task scheduling (relies on next_run_at)
- No built-in task result storage
- No built-in task monitoring UI

## Transaction Management
The module uses Spring's transaction management with `REQUIRES_NEW` propagation level for task operations to ensure isolation:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void markAsCompleted(final Long taskId) {
    // Implementation
}
```

## Testing
The module includes support for testing with H2 database:

```java
@Test
void shouldPublishAndProcessTask() {
    TaskRepository repository = factory.createTaskRepositoryOnH2("test_tasks");
    TaskPublisher publisher = factory.createTaskPublisher(repository);
    
    publisher.publishTask(new TestTask("test"));
    
    // Verify task execution
    assertThat(repository.hasPendingTasks()).isTrue();
}
```

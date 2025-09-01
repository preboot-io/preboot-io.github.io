---
layout: documentation
title: "Sending Notifications"
subtitle: "A unified interface for sending email and WebSocket notifications with template support."
permalink: /docs/backend/modules/notifications/
section: backend
---
# preboot-notifications

## Overview
The `preboot-notifications` module provides a unified interface for sending notifications through different channels (email and WebSocket) in Spring Boot applications. It features template support, attachment handling, event publishing, and flexible configuration options.

## Features
- Email notifications with template support
- WebSocket messaging
- Attachment handling
- Event publishing for notification tracking
- Support for multiple email recipients
- UTF-8 encoding support
- Bean validation integration
- Configurable email disable switch
- Error handling with custom exceptions

## Core Components

### NotificationApi
The main interface for sending notifications:

```java
@Service
public class YourService {
    private final NotificationApi notificationApi;

    public void notifyUser(UserEvent event) {
        // Send email
        notificationApi.sendEmail(EmailParam.builder(
            "sender@example.com",
            "receiver@example.com",
            "Important Update",
            "notification-template")
            .addTemplateParam("userName", event.getUserName())
            .build());

        // Send WebSocket message
        notificationApi.sendWebSocketMessage(WebSocketMessage.builder()
            .receiverId(event.getUserId())
            .destination("/topic/notifications")
            .payload(event)
            .build());
    }
}
```

### Email Notifications

#### Basic Email
```java
EmailParam email = EmailParam.builder(
    "sender@example.com",
    "receiver@example.com",
    "Welcome!",
    "welcome-template")
    .build();

notificationApi.sendEmail(email);
```

#### Email with Template Parameters
```java
EmailParam email = EmailParam.builder(
    "sender@example.com",
    "receiver@example.com",
    "Order Confirmation",
    "order-template")
    .addTemplateParam("orderNumber", "ORD-123")
    .addTemplateParam("customerName", "John Doe")
    .addTemplateParam("amount", "$99.99")
    .build();

notificationApi.sendEmail(email);
```

#### Email with Attachments
```java
Map<String, byte[]> attachments = new HashMap<>();
attachments.put("invoice.pdf", pdfBytes);
attachments.put("receipt.pdf", receiptBytes);

EmailParam email = EmailParam.builder(
    "sender@example.com",
    "receiver@example.com",
    "Your Invoice",
    "invoice-template")
    .attachments(attachments)
    .build();

notificationApi.sendEmail(email);
```

#### Multiple Recipients
```java
EmailParam email = EmailParam.builder(
    "sender@example.com",
    "primary@example.com",
    "Team Update",
    "update-template")
    .copyToEmail("manager1@example.com,manager2@example.com")
    .build();

notificationApi.sendEmail(email);
```

### WebSocket Messaging

#### Broadcast Message
```java
WebSocketMessage message = WebSocketMessage.builder()
    .destination("/topic/announcements")
    .payload(new Announcement("System maintenance scheduled"))
    .build();

notificationApi.sendWebSocketMessage(message);
```

#### Direct Message
```java
WebSocketMessage message = WebSocketMessage.builder()
    .receiverId(userId)
    .destination("/queue/private-messages")
    .payload(new PrivateMessage("Hello!"))
    .build();

notificationApi.sendWebSocketMessage(message);
```

### Event Handling

The module publishes events that can be listened to:

```java
@Component
public class NotificationEventHandler {
    
    @EventHandler
    public void onEmailSent(EmailSentEvent event) {
        // Handle successful email sending
        log.info("Email sent to {} with subject: {}", 
            event.getReceiver(), 
            event.getSubject());
    }

    @EventHandler
    public void onInvalidRecipient(InvalidEmailRecipientEvent event) {
        // Handle invalid recipient
        log.error("Invalid email recipient: {}", 
            event.getReceiver());
    }
}
```

## Configuration

### Application Properties
```yaml
app:
  configuration:
    disableEmails: false  # Set to true to disable email sending

spring:
  mail:
    host: smtp.example.com
    port: 587
    username: your-username
    password: your-password
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true

  websocket:
    endpoints: /ws
    allowed-origins: "*"
    user-destination-prefix: /user
```

### Java Configuration
```java
@Configuration
public class NotificationConfig {
    
    @Bean
    public JavaMailSender mailSender() {
        JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
        mailSender.setHost(host);
        mailSender.setPort(port);
        mailSender.setUsername(username);
        mailSender.setPassword(password);
        
        Properties props = mailSender.getJavaMailProperties();
        props.put("mail.transport.protocol", "smtp");
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");
        
        return mailSender;
    }
}
```

## Error Handling

### Mail Send Failure
```java
try {
    notificationApi.sendEmail(emailParam);
} catch (MailSendFailureException e) {
    log.error("Failed to send email: {}", e.getMessage());
    // Handle failure
}
```

### Validation Errors
```java
try {
    notificationApi.sendEmail(invalidEmailParam);
} catch (ValidationException e) {
    log.error("Invalid email parameters: {}", e.getMessage());
    // Handle validation failure
}
```

## Best Practices

1. **Template Management**
    - Keep templates in a centralized location
    - Use meaningful template names
    - Include proper error messages in templates
    - Version templates if needed

2. **Email Sending**
    - Always validate email addresses
    - Use meaningful subject lines
    - Include proper error messages
    - Handle attachments properly
    - Consider email size limits

3. **WebSocket Messages**
    - Use appropriate destinations
    - Keep payload size reasonable
    - Handle disconnections gracefully
    - Implement proper security

4. **Error Handling**
    - Log all notification failures
    - Implement retry mechanisms for critical notifications
    - Monitor notification success rates
    - Handle template errors gracefully

5. **Security**
    - Validate all input
    - Sanitize template parameters
    - Implement rate limiting
    - Use secure protocols

## Testing

### Email Testing
```java
@Test
void shouldSendEmailWithAttachments() {
    // Given
    EmailParam email = EmailParam.builder(
        "test@example.com",
        "recipient@example.com",
        "Test Subject",
        "test-template")
        .addTemplateParam("key", "value")
        .build();

    // When
    notificationApi.sendEmail(email);

    // Then
    verify(mailSender).send(any(MimeMessage.class));
    verify(eventPublisher).publish(any(EmailSentEvent.class));
}
```

### WebSocket Testing
```java
@Test
void shouldSendWebSocketMessage() {
    // Given
    WebSocketMessage message = WebSocketMessage.builder()
        .receiverId("user123")
        .destination("/topic/test")
        .payload("test message")
        .build();

    // When
    notificationApi.sendWebSocketMessage(message);

    // Then
    verify(simpMessagingTemplate).convertAndSendToUser(
        eq("user123"),
        eq("/topic/test"),
        eq("test message")
    );
}
```

## Limitations
- No built-in rate limiting (in Pro)
- No message queuing (in Pro)
- No retry mechanism (in Pro)

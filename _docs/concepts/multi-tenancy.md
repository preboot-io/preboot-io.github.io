---
layout: documentation
title: "Multi-tenancy"
subtitle: "Understanding the core multi-tenancy architecture in PreBoot."
permalink: /docs/concepts/multi-tenancy/
---

## Overview

Multi-tenancy is a core architectural principle of the PreBoot framework, enabling a single instance of an application to serve multiple customers (or **tenants**) in a secure and isolated manner. Instead of deploying a separate application for each customer, PreBoot provides the tools to partition data and configuration, ensuring that tenants can only access their own information.

This approach is fundamental for building modern Software-as-a-Service (SaaS) applications and is deeply integrated into two key modules:
1.  **`preboot-auth`**: Manages user identity, their association with tenants, and the active tenant context for a session.
2.  **`preboot-securedata`**: Automatically enforces data isolation at the database repository level, ensuring that queries only return data belonging to the current active tenant.

Together, these modules provide a comprehensive solution for multi-tenancy.

## The PreBoot Tenant Model

The relationship between users and tenants is simple yet powerful:

* **Users and Tenants**: A `User` represents an individual person, while a `Tenant` typically represents a distinct organization or client using your application.
* **Flexible Associations**: A single user account can be a member of one or more tenants. This is useful for consultants, support staff, or users who work across multiple organizations.
* **Active Tenant Context**: For any given session, a user operates within the context of one "active" tenant. All actions, from API calls to data queries, are performed on behalf of this active tenant.
* **Scoped Roles and Permissions**: A user's roles (e.g., "ADMIN", "EDITOR") and permissions are assigned within the scope of a specific tenant. This means a user can be an Admin in one tenant but only a Viewer in another.

## Pillar 1: Tenant-Aware Authentication

The `preboot-auth` module is responsible for managing which tenant a user is currently acting as.

### Session and JWT Context

When a user logs in, they receive a JSON Web Token (JWT) that is tied to a server-side session. This session holds the user's identity and, crucially, their **active tenant context**.

### Switching Tenants

If a user belongs to multiple tenants, they can switch their active context. The typical flow is:
1.  User logs in and receives a JWT for their default or last-used tenant.
2.  The frontend can call `/api/auth/my-tenants` to get a list of all tenants accessible to the user.
3.  The user selects a different tenant from the list.
4.  The frontend calls `POST /api/auth/use-tenant` with the chosen `tenantId`.
5.  The backend validates access and, if successful, issues a **new JWT** containing the new active tenant context.

### Programmatic Access to Tenant Context

Inside your services, you can programmatically access the current tenant ID from the user's session using the `TenantResolver` interface.

```java
@Service
@RequiredArgsConstructor
public class MyTenantAwareService {

    private final TenantResolver tenantResolver;

    public void performTenantSpecificAction() {
        UUID currentTenantId = tenantResolver.getCurrentTenant(); // Resolves tenant ID from the session
        if (currentTenantId != null) {
            // Logic specific to the current tenant...
            System.out.println("Operating under tenant: " + currentTenantId);
        }
    }
}
```

## Pillar 2: Automatic Data Isolation

While `preboot-auth` manages the context, `preboot-securedata` is responsible for enforcing data isolation at the persistence layer. It seamlessly extends the `preboot-query` module to make data access tenant-aware.

### The `@Tenant` Annotation

To enable data isolation for an entity, you simply annotate the field that holds the tenant identifier with `@Tenant`.

```java
@Table("documents")
@SecureAccess(read = @AccessRule(roles = {"VIEWER", "EDITOR"}))
public class Document {
    @Id
    private Long id;

    @Tenant // This field links the document to a specific tenant
    private UUID tenantId;

    private String title;
    private String content;

    // ... other fields and audit annotations ...
}
```

### Automatic Query Filtering

This is the core of data isolation in PreBoot. When you use a `SecureRepository` (which extends the standard `FilterableRepository`), every query is **automatically and transparently modified** to include a `WHERE` clause that filters by the current user's active `tenantId`.

You, as a developer, write your queries as if you were in a single-tenant environment.

**What you write:**
```java
// Find all documents with status "PUBLISHED"
SearchParams params = SearchParams.criteria(
    FilterCriteria.eq("status", "PUBLISHED")
).build();

// Your code is clean and unaware of multi-tenancy details
Page<Document> documents = documentRepository.findAll(params);
```

**What PreBoot executes (conceptually):**
```sql
-- The currentTenantId is retrieved from the SecurityContext
SELECT * FROM documents WHERE status = 'PUBLISHED' AND tenant_id = 'the-current-users-tenant-id';
```

This automatic filtering applies to all repository methods: `findById`, `findAll`, `count`, `existsByUuid`, projections, and complex nested queries.

### Automatic Tenant ID Assignment

When you create and save a new entity using a `SecureRepository`, the field annotated with `@Tenant` is automatically populated with the ID of the current active tenant. You don't need to set it manually.

```java
public Document createDocument(String title, String content) {
    Document document = new Document();
    document.setTitle(title);
    document.setContent(content);

    // No need to call document.setTenantId(...)
    // It will be set automatically upon saving.
    return documentRepository.save(document);
}
```

## Summary

Multi-tenancy in PreBoot is a first-class citizen, not an afterthought. The combination of `preboot-auth` and `preboot-securedata` provides a complete and robust solution:
1.  **Authentication (`preboot-auth`)** establishes *who* the user is and *which tenant* they are acting for.
2.  **Data Isolation (`preboot-securedata`)** uses that context to automatically enforce that the user can only see and manipulate data belonging to their active tenant.

By using `SecureRepository` for your entities, you get enterprise-grade data isolation out of the box, allowing you to focus on writing business logic.

```
---
layout: documentation
title: Multi-Tenancy in PreBoot
subtitle: Learn how to implement and configure multi-tenant applications
permalink: /docs/multi-tenancy/
---

# Multi-Tenancy

Multi-tenancy is a core feature of PreBoot that allows your application to serve multiple clients (tenants) from a single deployment. PreBoot provides several strategies for implementing multi-tenancy, with built-in support for tenant isolation, custom domains, and resource quotas.

## What is Multi-Tenancy?

Multi-tenancy is an architecture where a single instance of software serves multiple tenants. Each tenant's data is isolated and remains invisible to other tenants. In SaaS applications, each of your customers is typically a tenant with their own users, data, and often custom configuration.

## Multi-Tenancy Approaches

PreBoot supports multiple approaches to multi-tenancy:

### 1. Database-per-Tenant

This approach provides the strongest isolation by giving each tenant their own database.

```java
@Configuration
public class MultiTenancyConfig {
    @Bean
    public TenantDatabaseConfig tenantDatabaseConfig() {
        return TenantDatabaseConfig.builder()
            .withStrategy(TenantDatabaseStrategy.SEPARATE_DATABASE)
            .withConnectionPooling(true)
            .withMaxConnectionsPerTenant(10)
            .build();
    }
}
```

### 2. Schema-per-Tenant

This approach uses a single database but separate schemas for each tenant.

```java
@Configuration
public class MultiTenancyConfig {
    @Bean
    public TenantDatabaseConfig tenantDatabaseConfig() {
        return TenantDatabaseConfig.builder()
            .withStrategy(TenantDatabaseStrategy.SEPARATE_SCHEMA)
            .withSchemaPrefix("tenant_")
            .build();
    }
}
```

### 3. Shared Schema with Discriminator Column

This approach uses a shared database and schema but adds a tenant discriminator column to each table.

```java
@Configuration
public class MultiTenancyConfig {
    @Bean
    public TenantDatabaseConfig tenantDatabaseConfig() {
        return TenantDatabaseConfig.builder()
            .withStrategy(TenantDatabaseStrategy.DISCRIMINATOR_COLUMN)
            .withDiscriminatorColumn("tenant_id")
            .build();
    }
}
```

## Tenant Resolution

PreBoot offers multiple ways to identify the current tenant:

### Domain-based Resolution

```java
@Configuration
public class TenantResolutionConfig {
    @Bean
    public TenantResolver tenantResolver() {
        return TenantResolver.builder()
            .withDomainStrategy()
            .withDomainMapping("example.com", "default")
            .withDomainMapping("client1.example.com", "client1")
            .withDomainMapping("client2.example.com", "client2")
            .build();
    }
}
```

### Header-based Resolution

```java
@Configuration
public class TenantResolutionConfig {
    @Bean
    public TenantResolver tenantResolver() {
        return TenantResolver.builder()
            .withHeaderStrategy("X-Tenant-ID")
            .build();
    }
}
```

### Path-based Resolution

```java
@Configuration
public class TenantResolutionConfig {
    @Bean
    public TenantResolver tenantResolver() {
        return TenantResolver.builder()
            .withPathStrategy()
            .withPathPrefix("/tenants/")
            .build();
    }
}
```

## Tenant-Aware Repositories

PreBoot automatically applies tenant filtering to your repositories. You don't need to explicitly filter by tenant in your queries.

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    // This will automatically filter by the current tenant
    List<Customer> findByLastName(String lastName);
}
```

## Resource Quotas

In Professional Edition, you can set up resource quotas for tenants:

```java
@Configuration
public class TenantQuotasConfig {
    @Bean
    public TenantQuotaManager tenantQuotaManager() {
        return TenantQuotaManager.builder()
            .withQuota("storage", 5, QuotaUnit.GIGABYTES)
            .withQuota("users", 100, QuotaUnit.COUNT)
            .withQuota("api_requests", 10000, QuotaUnit.COUNT_PER_DAY)
            .build();
    }
}
```

## Advanced Configuration

For advanced cases, you can implement custom tenant-aware components:

```java
@Component
public class CustomTenantFilter implements TenantFilter {
    private final TenantContext tenantContext;
    
    @Autowired
    public CustomTenantFilter(TenantContext tenantContext) {
        this.tenantContext = tenantContext;
    }
    
    @Override
    public Predicate apply(Root<?> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
        // Custom tenant filtering logic
        String currentTenant = tenantContext.getCurrentTenant();
        return cb.equal(root.get("tenantId"), currentTenant);
    }
}
```

## Best Practices

1. **Choose the right multi-tenancy approach** for your needs. Database-per-tenant offers the best isolation but is more resource-intensive.

2. **Set appropriate resource quotas** to prevent any tenant from consuming excessive resources.

3. **Use tenant-aware caching** to avoid cache poisoning between tenants.

4. **Implement proper error handling** for tenant-specific errors, like quota exceeded events.

5. **Consider tenant provisioning and onboarding** processes as part of your application design.

## Conclusion

PreBoot's multi-tenancy support makes it easy to build SaaS applications that can serve multiple customers from a single deployment. By choosing the right multi-tenancy approach and configuring tenant resolution, you can create applications that scale efficiently while maintaining proper data isolation.
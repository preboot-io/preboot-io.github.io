---
layout: documentation
title: "Customization Guide"
subtitle: "Adapt the PreBoot Reference App for your specific needs"
permalink: /docs/reference-app/customization/
section: reference-app
---

# Customization Guide

The PreBoot Reference App is designed to be a starting point for your own SaaS application. This guide shows you how to customize and extend it for your specific business needs.

## Getting Started with Customization

### 1. Fork the Repository

```bash
# Fork the reference app repository
git clone https://github.com/your-username/preboot-refapp
cd preboot-refapp

# Add upstream remote for updates
git remote add upstream https://github.com/preboot-io/preboot-refapp

# Create your customization branch
git checkout -b customization/my-saas-app
```

### 2. Rename the Application

```bash
# Update package names in backend
find backend/src -name "*.java" -exec sed -i 's/io.preboot.refapp/com.yourcompany.yourapp/g' {} \;

# Update frontend package.json
sed -i 's/"name": "preboot-refapp"/"name": "your-saas-app"/g' frontend/package.json

# Update configuration files
sed -i 's/preboot-refapp/your-saas-app/g' backend/src/main/resources/application.yml
```

## Backend Customization

### Adding New Entities

Create your business entities using PreBoot patterns:

```java
// entities/Customer.java
@Entity
@Table(name = "customers")
@TenantFiltered  // Automatic tenant filtering
@AuditLogged     // Automatic audit logging
public class Customer {
    
    @Id
    @GeneratedValue
    private UUID id;
    
    @TenantId
    @Column(name = "tenant_id")
    private UUID tenantId;
    
    @NotBlank
    @Column(name = "name")
    private String name;
    
    @Email
    @Column(name = "email")
    private String email;
    
    @Column(name = "phone")
    private String phone;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    private CustomerStatus status = CustomerStatus.ACTIVE;
    
    @CreationTimestamp
    @Column(name = "created_at")
    private Instant createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    // Constructors, getters, setters...
}

// repositories/CustomerRepository.java
@Repository
public interface CustomerRepository extends SecureJpaRepository<Customer, UUID> {
    
    // These queries are automatically tenant-filtered
    Page<Customer> findByNameContainingIgnoreCase(String name, Pageable pageable);
    
    List<Customer> findByStatus(CustomerStatus status);
    
    @Query("SELECT c FROM Customer c WHERE c.email = ?1")
    Optional<Customer> findByEmail(String email);
}

// services/CustomerService.java
@Service
@Transactional
public class CustomerService {
    
    @Autowired
    private CustomerRepository customerRepository;
    
    @EventPublisher
    private EventBus eventBus;
    
    public Customer create(CreateCustomerRequest request) {
        Customer customer = Customer.builder()
            .name(request.getName())
            .email(request.getEmail())
            .phone(request.getPhone())
            .status(CustomerStatus.ACTIVE)
            .build();
        
        customer = customerRepository.save(customer);
        
        // Publish domain event
        eventBus.publish(new CustomerCreatedEvent(customer));
        
        return customer;
    }
    
    public Page<Customer> search(String query, Pageable pageable) {
        if (StringUtils.hasText(query)) {
            return customerRepository.findByNameContainingIgnoreCase(query, pageable);
        }
        return customerRepository.findAll(pageable);
    }
}
```

### Custom API Endpoints

```java
// controllers/CustomerController.java
@RestController
@RequestMapping("/api/customers")
@PreAuthorize("hasRole('USER')")
public class CustomerController {
    
    @Autowired
    private CustomerService customerService;
    
    @GetMapping
    @PreAuthorize("hasPermission('customer', 'read')")
    public Page<Customer> getCustomers(
            @RequestParam(required = false) String search,
            Pageable pageable) {
        return customerService.search(search, pageable);
    }
    
    @PostMapping
    @PreAuthorize("hasPermission('customer', 'write')")
    public ResponseEntity<Customer> createCustomer(
            @RequestBody @Valid CreateCustomerRequest request) {
        Customer customer = customerService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(customer);
    }
    
    @GetMapping("/{id}")
    @PreAuthorize("hasPermission('customer', 'read')")
    public Customer getCustomer(@PathVariable UUID id) {
        return customerService.findById(id)
            .orElseThrow(() -> new CustomerNotFoundException(id));
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasPermission('customer', 'write')")
    public Customer updateCustomer(@PathVariable UUID id, 
                                 @RequestBody @Valid UpdateCustomerRequest request) {
        return customerService.update(id, request);
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasPermission('customer', 'delete')")
    public ResponseEntity<Void> deleteCustomer(@PathVariable UUID id) {
        customerService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Database Migrations

```sql
-- db/changelog/changes/add-customers-table.sql
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT uk_customers_tenant_email UNIQUE (tenant_id, email)
);

-- Indexes for performance
CREATE INDEX idx_customers_tenant_id ON customers(tenant_id);
CREATE INDEX idx_customers_status ON customers(tenant_id, status);
CREATE INDEX idx_customers_name ON customers(tenant_id, name);

-- Update changelog master file
-- db/changelog/db.changelog-master.xml
<databaseChangeLog>
    <!-- existing changesets -->
    
    <include file="db/changelog/changes/add-customers-table.sql"/>
</databaseChangeLog>
```

## Frontend Customization

### New Pages and Components

```tsx
// pages/CustomersPage.tsx
import React from 'react';
import { DataTable, PageHeader, CreateButton } from '@preboot/ui';
import { useSecureQuery, useTenant } from '@preboot/ui';

function CustomersPage() {
  const { currentTenant } = useTenant();
  const { data: customers, loading, error, refetch } = useSecureQuery('/customers');

  const columns = [
    { key: 'name', label: 'Name', sortable: true },
    { key: 'email', label: 'Email', sortable: true },
    { key: 'phone', label: 'Phone' },
    { key: 'status', label: 'Status', filterable: true },
    { key: 'createdAt', label: 'Created', sortable: true, type: 'date' }
  ];

  const actions = [
    {
      label: 'Edit',
      icon: 'edit',
      onClick: (customer) => navigate(`/customers/${customer.id}/edit`)
    },
    {
      label: 'Delete',
      icon: 'trash',
      onClick: (customer) => handleDelete(customer.id),
      confirm: 'Are you sure you want to delete this customer?',
      condition: (customer) => customer.status !== 'DELETED'
    }
  ];

  const handleDelete = async (customerId) => {
    try {
      await api.delete(`/customers/${customerId}`);
      refetch();
      toast.success('Customer deleted successfully');
    } catch (error) {
      toast.error('Failed to delete customer');
    }
  };

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <div className="customers-page">
      <PageHeader 
        title="Customers"
        subtitle={`${customers.length} customers in ${currentTenant.name}`}
        actions={
          <CreateButton 
            onClick={() => navigate('/customers/new')}
            label="Add Customer"
          />
        }
      />
      
      <DataTable
        data={customers}
        columns={columns}
        actions={actions}
        searchable
        exportable
        onRowClick={(customer) => navigate(`/customers/${customer.id}`)}
      />
    </div>
  );
}

export default CustomersPage;
```

### Custom Forms

```tsx
// components/forms/CustomerForm.tsx
import React from 'react';
import { useForm, FormProvider } from 'react-hook-form';
import { FormField, FormActions, Card } from '@preboot/ui';

interface CustomerFormProps {
  customer?: Customer;
  onSubmit: (data: CustomerFormData) => void;
  onCancel: () => void;
  loading?: boolean;
}

function CustomerForm({ customer, onSubmit, onCancel, loading }: CustomerFormProps) {
  const methods = useForm<CustomerFormData>({% raw %}{
    defaultValues: customer || {
      name: '',
      email: '',
      phone: '',
      status: 'ACTIVE'
    }
  }{% endraw %});

  const {% raw %}{ handleSubmit, formState: { errors } }{% endraw %} = methods;

  return (
    <FormProvider {...methods}>
      <Card>
        <form onSubmit={handleSubmit(onSubmit)}>
          <div className="form-grid">
            <FormField
              name="name"
              label="Customer Name"
              rules={% raw %}{{ required: 'Name is required' }}{% endraw %}
              placeholder="Enter customer name"
            />
            
            <FormField
              name="email"
              type="email"
              label="Email Address"
              rules={% raw %}{{ 
                required: 'Email is required',
                pattern: {
                  value: /\S+@\S+\.\S+/,
                  message: 'Invalid email address'
                }
              }}{% endraw %}
              placeholder="customer@example.com"
            />
            
            <FormField
              name="phone"
              label="Phone Number"
              placeholder="(555) 123-4567"
            />
            
            <FormField
              name="status"
              type="select"
              label="Status"
              options={[
                { value: 'ACTIVE', label: 'Active' },
                { value: 'INACTIVE', label: 'Inactive' },
                { value: 'PROSPECT', label: 'Prospect' }
              ]}
            />
          </div>
          
          <FormActions>
            <button type="button" onClick={onCancel} disabled={loading}>
              Cancel
            </button>
            <button type="submit" className="primary" disabled={loading}>
              {loading ? 'Saving...' : customer ? 'Update Customer' : 'Create Customer'}
            </button>
          </FormActions>
        </form>
      </Card>
    </FormProvider>
  );
}

export default CustomerForm;
```

### Navigation and Routing

```tsx
// components/navigation/AppNavigation.tsx
import { Navigation } from '@preboot/ui';

const navigationItems = [
  {
    label: 'Dashboard',
    path: '/dashboard',
    icon: 'dashboard'
  },
  {
    label: 'Customers',
    path: '/customers',
    icon: 'users',
    permission: 'customer:read'
  },
  {
    label: 'Orders',
    path: '/orders',
    icon: 'shopping-cart',
    permission: 'order:read'
  },
  {
    label: 'Reports',
    path: '/reports',
    icon: 'chart-bar',
    permission: 'report:read'
  },
  {
    label: 'Settings',
    path: '/settings',
    icon: 'cog',
    permission: 'setting:read'
  }
];

// App.tsx - Add your routes
import { Routes, Route } from 'react-router-dom';
import { ProtectedRoute } from '@preboot/ui';

function App() {
  return (
    <Routes>
      <Route path="/customers" element={
        <ProtectedRoute permission="customer:read">
          <CustomersPage />
        </ProtectedRoute>
      } />
      <Route path="/customers/new" element={
        <ProtectedRoute permission="customer:write">
          <CreateCustomerPage />
        </ProtectedRoute>
      } />
      <Route path="/customers/:id" element={
        <ProtectedRoute permission="customer:read">
          <CustomerDetailsPage />
        </ProtectedRoute>
      } />
      <Route path="/customers/:id/edit" element={
        <ProtectedRoute permission="customer:write">
          <EditCustomerPage />
        </ProtectedRoute>
      } />
    </Routes>
  );
}
```

## Configuration Customization

### Environment Variables

```bash
# .env.production
REACT_APP_NAME=Your SaaS App
REACT_APP_API_BASE_URL=https://api.yoursaas.com
REACT_APP_WS_URL=wss://api.yoursaas.com/ws

# Backend application-prod.yml
spring:
  application:
    name: your-saas-app
  
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
  
  mail:
    host: ${SMTP_HOST}
    port: ${SMTP_PORT}
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}

your-app:
  features:
    enable-analytics: true
    enable-export: true
    max-users-per-tenant: 100
  
  integrations:
    stripe:
      public-key: ${STRIPE_PUBLIC_KEY}
      secret-key: ${STRIPE_SECRET_KEY}
```

### Custom Permissions

```java
// config/SecurityConfig.java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public PermissionEvaluator customPermissionEvaluator() {
        return new CustomPermissionEvaluator();
    }
}

// security/CustomPermissionEvaluator.java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {
    
    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        PrebootUser user = (PrebootUser) auth.getPrincipal();
        String permissionStr = (String) permission;
        
        // Your custom permission logic
        switch (permissionStr) {
            case "customer:advanced":
                return user.hasRole("ADMIN") || user.getTenant().getPlan().equals("ENTERPRISE");
            case "report:financial":
                return user.hasPermission("financial:read") && user.hasRole("MANAGER", "ADMIN");
            default:
                return user.hasPermission(permissionStr);
        }
    }
}
```

## Deployment Customization

### Custom Docker Configuration

```dockerfile
# Dockerfile.custom
FROM openjdk:17-jdk-slim

# Add your custom configurations
COPY custom-configs/ /app/config/
COPY your-app.jar /app/app.jar

# Custom startup script
COPY startup.sh /app/startup.sh
RUN chmod +x /app/startup.sh

EXPOSE 8080
ENTRYPOINT ["/app/startup.sh"]
```

### Infrastructure as Code

```yaml
# terraform/main.tf
resource "aws_ecs_service" "your_saas_app" {
  name            = "your-saas-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.your_app.arn
  desired_count   = var.app_replica_count

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }
  
  # Your custom configuration...
}
```

## Best Practices

### 1. Keep Core PreBoot Updates
```bash
# Regularly sync with upstream
git fetch upstream
git merge upstream/main

# Resolve conflicts in your customizations
```

### 2. Modular Customizations
- Keep custom code in separate packages
- Use feature flags for optional functionality
- Document your customizations thoroughly

### 3. Testing Custom Features
```java
// tests/integration/CustomerIntegrationTest.java
@SpringBootTest
@TestPropertySource(properties = {"spring.profiles.active=test"})
class CustomerIntegrationTest extends BaseIntegrationTest {
    
    @Test
    void shouldCreateCustomerWithTenantIsolation() {
        // Test your custom functionality
        Customer customer = customerService.create(createRequest);
        
        assertThat(customer.getTenantId()).isEqualTo(getCurrentTenantId());
        assertThat(customer.getName()).isEqualTo(createRequest.getName());
    }
}
```

### 4. Performance Monitoring
```java
// Add custom metrics
@Component
public class CustomerMetrics {
    
    @EventListener
    @Timed("customer.creation.time")
    public void onCustomerCreated(CustomerCreatedEvent event) {
        meterRegistry.counter("customers.created", 
            "tenant", event.getTenantId()).increment();
    }
}
```

This customization approach allows you to build upon the solid PreBoot foundation while creating a unique SaaS application tailored to your specific business requirements.
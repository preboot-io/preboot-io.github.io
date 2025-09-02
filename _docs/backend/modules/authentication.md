---
layout: documentation
title: "Authentication & RBAC"
subtitle: "JWT-based authentication, multi-tenancy, and role-based access control with the preboot-auth module."
permalink: /docs/backend/modules/authentication/
section: backend
---
# PreBoot Auth Module Documentation

## Overview

The `preboot-auth` module is a comprehensive authentication and authorization library designed for Spring Boot applications. It provides enterprise-grade security features with multi-tenant architecture, JWT-based session management, and flexible user management capabilities.

### Key Features

-   **JWT-based Authentication**: Secure token-based authentication with configurable expiration.
-   **Multi-tenant Architecture**: Users can belong to multiple tenants with dynamic switching.
-   **Role-based Access Control (RBAC)**: Hierarchical permission system with tenant-specific and user-specific roles and permissions.
-   **Session Management**: Device fingerprinting, remember-me functionality, and session tracking.
-   **User Lifecycle Management**: Account creation, activation (email-based), password reset workflows.
-   **Email Integration**: Built-in email templates for activation and password reset, using `preboot-notifications`.
-   **Event-driven Architecture**: Comprehensive event system for system integration.
-   **REST API**: Ready-to-use endpoints for all authentication operations.
-   **Security Features**: CORS configuration, optional CSRF protection, device fingerprinting.

### Architecture

The module follows a multi-layered architecture:

```
┌─────────────────────────────────────────────┐
│                REST Layer                   │
│  (Controllers, Security Filters)           │
├─────────────────────────────────────────────┤
│                API Layer                    │
│  (Public Interfaces, DTOs, Events)         │
├─────────────────────────────────────────────┤
│               Service Layer                 │
│  (Use Cases, Business Logic)               │
├─────────────────────────────────────────────┤
│              Repository Layer               │
│  (Data Access, Persistence)                │
└─────────────────────────────────────────────┘
```


---
## Installation

### Maven Dependency

Add the starter dependency to your `pom.xml`:

```xml
<dependency>
   <groupId>io.preboot</groupId>
   <artifactId>preboot-auth-starter</artifactId>
   <version>${preboot.version}</version>
</dependency>
```


The `preboot-auth-starter` conveniently includes `preboot-auth-api`, `preboot-auth-core`, and `preboot-auth-emails`.

### Individual Modules

For fine-grained control, you can include specific modules:

```xml
<dependency>
   <groupId>io.preboot</groupId>
   <artifactId>preboot-auth-api</artifactId>
   <version>${preboot.version}</version>
</dependency>

<dependency>
   <groupId>io.preboot</groupId>
   <artifactId>preboot-auth-core</artifactId>
   <version>${preboot.version}</version>
   <scope>runtime</scope> </dependency>

<dependency>
    <groupId>io.preboot</groupId>
    <artifactId>preboot-auth-emails</artifactId>
    <version>${preboot.version}</version>
</dependency>
```


### Database Setup

The module requires PostgreSQL and uses Liquibase for schema management. Add the following dependencies to your project:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```


Create your master Liquibase changelog that includes the auth module schema. This ensures that the auth module's schema is created before your application's specific schema changes.

**Example: `src/main/resources/db/changelog/db.changelog.master.xml`**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="classpath:db/changelog/db-changelog-preboot-auth.xml"/>
    <include file="db/changelog/db.changelog.app.xml"/>
</databaseChangeLog>
```


Configure Spring Boot to use your master changelog in your `application.yml`:
```yaml
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog.master.xml # Path to your master changelog
    enabled: true
  datasource: # Ensure your datasource is configured
    url: jdbc:postgresql://localhost:5432/your_database
    username: your_username
    password: your_password
    driver-class-name: org.postgresql.Driver
```


The auth schema will be automatically created or updated by Liquibase when the application starts.

---
## Configuration

### Minimal Configuration

A minimal configuration requires the database connection and a JWT secret.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/your_database
    username: your_username
    password: your_password
  liquibase: # Ensure Liquibase is configured as shown in Database Setup
    change-log: classpath:db/changelog/db.changelog.master.xml

preboot:
  security:
    jwt-secret: "your-secure-jwt-secret-key-minimum-32-characters-long" # IMPORTANT: Use a strong, unique secret
```

**Security Note**: The `jwt-secret` must be at least 256 bits (32 characters for the HS256 algorithm used). It should be stored securely, for example, using environment variables or a secrets management tool, not hardcoded in version control for production environments.

### Complete Configuration Options

The module offers various configuration properties to customize its behavior. These are typically set in your `application.yml` or `application.properties` file.

```yaml
preboot:
  security: # Properties from io.preboot.auth.spring.AuthSecurityProperties
    # JWT Configuration (Required)
    jwt-secret: "your-secure-jwt-secret-key-minimum-32-characters-long" # Must be at least 256 bits (32 UTF-8 chars for HS256).

    # Session Configuration
    session-timeout-minutes: 15                    # Default: 15. Standard session duration in minutes.
    long-session-timeout-days: 15                  # Default: 15. Used when "rememberMe" is true during login, duration in days.

    # Token Expiration for specific workflows
    password-reset-token-timeout-in-days: 2        # Default: 2. Validity duration for password reset tokens in days.
    activation-token-timeout-in-days: 30           # Default: 30. Validity duration for account activation tokens in days.

    # CORS Configuration
    cors-allowed-origins:                          # Default: Empty list, which results in ["*"] (allows all origins) in CorsConfigurationSource. For production, specify your frontend domains.
      - http://localhost:3000
      - https://your-frontend-domain.com

    # Security Features
    enable-csrf: false                             # Default: false. CSRF protection. If true, Spring Security's CSRF protection is enabled.

    # Additional Public Endpoints
    # These endpoints are added to the module's defaults.
    # Default public endpoints (from AuthSecurityConfiguration.DEFAULT_PERMIT_ALL):
    # - /api/auth/login
    # - /api/auth/password/reset-request
    # - /api/auth/password/reset
    # - /api/auth/activation
    # - /api/auth/registration
    public-endpoints:
      - /api/public/**
      - /api/health/**
      - /v3/api-docs/** # Example for OpenAPI/Swagger
      - /swagger-ui/** # Example for OpenAPI/Swagger

  accounts:
    # Default User Settings (for new user accounts)
    default-timezone: "UTC"                        # Default: "UTC". Timezone assigned if not specified during creation.
    default-language: "en"                         # Default: "en". Language assigned if not specified during creation.
    default-roles:                                 # Default: ["ADMIN"]. Roles assigned to the first user created via registration that also creates a new tenant.
      - "ADMIN"
      # - "USER" # Example to add more default roles

    # Registration Settings
    registration-enabled: true                     # Default: true. Allows users to register via the /api/auth/registration endpoint.
    assign-demo-role-to-new-tenants: true          # Default: true. If true, tenants created via API or registration will automatically be assigned the "DEMO" role (from TenantService.DEMO_TENANT_ROLE). This can be used to identify or apply specific logic to new/trial tenants.

  auth-emails:
    # Sender and URLs (Required if email functionality is active)
    sender-email: "noreply@your-domain.com"        # Email address used to send authentication-related emails.
    password-reset-url: "https://your-frontend-domain.com/reset-password" # Base URL for the password reset page (token will be appended as a query parameter: ?token=...).
    password-activation-url: "https://your-frontend-domain.com/activate-account" # Base URL for the account activation page (token will be appended as query parameter: ?token=...).

    # Email Logo
    logo-path: "svg/logo.svg"                      # Classpath location for the SVG logo used in emails.
                                                   # Defaults to "svg/logo.svg".
                                                   # If the configured logo is not found, it attempts to load "svg/logo.svg" (the default).
                                                   # If neither is found, no logo will be embedded.

    # Email Template Customization
    # Template names are relative to `classpath:templates/` (e.g., "custom-activation.email" refers to `classpath:templates/custom-activation.email.html`).
    # Ensure your custom templates are Thymeleaf compatible if you override them.
    account-activation-email-template: "account-activation.email" # Default: "account-activation.email".
    account-activation-email-subject: "Account Activation"      # Default: "Account Activation".
    forgotten-password-reset-email-template: "account-reset-password.email" # Default: "account-reset-password.email".
    forgotten-password-reset-email-subject: "Password Reset"      # Default: "Password Reset".
```

---
## Quick Start

Once the module is installed and configured, several REST endpoints become available for common authentication tasks.

### 1. Basic Authentication

```bash
# Login: Authenticate a user and receive a JWT.
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123",
  "rememberMe": false # Optional, defaults to false. Set to true for a long-lived session.
}

# Response (200 OK):
# {
#   "token": "<jwt-token>"
# }

# Get Current User Info: Retrieve details of the authenticated user.
GET /api/auth/me
Authorization: Bearer <jwt-token>

# Response (200 OK): UserAccountInfo DTO (see DTOs section)

# Logout: Invalidate the current user's session.
POST /api/auth/logout
Authorization: Bearer <jwt-token>

# Response (200 OK or 204 No Content)

# Refresh Session: Obtain a new JWT for an active session.
POST /api/auth/refresh
Authorization: Bearer <jwt-token>

# Response (200 OK):
# {
#   "token": "<new-jwt-token>"
# }
```


### 2. User Registration

If `preboot.accounts.registration-enabled` is `true` (default), new users can register via the `RegistrationController`. This typically creates a new tenant and an inactive user account within it. An activation email is then sent.

```bash
# Register New User and Tenant
POST /api/auth/registration
Content-Type: application/json

{
  "username": "newuser",
  "email": "newuser@example.com",
  "language": "en",                 # Optional, defaults to preboot.accounts.default-language
  "timezone": "UTC",                # Optional, defaults to preboot.accounts.default-timezone
  "tenantName": "My New Organization" # Name for the new tenant
}

# Response (200 OK or void/204 No Content typically)
# Internally, this uses CreateTenantAndInactiveUserAccountRequest.
# Default roles (from preboot.accounts.default-roles) and empty permissions are assigned.
```

If `preboot-auth-emails` is configured, an account activation email will be sent to the user. The user will set their password during the activation step.

### 3. Account Activation

After registration or manual creation of an inactive user, the account must be activated.

```bash
# Activate Account and Set Password
POST /api/auth/activation
Content-Type: application/json

{
  "token": "jwt-activation-token-from-email", # Token received via activation email
  "password": "chosenSecurePassword123"       # User chooses their password here
}

# Response (204 No Content on success)
```


### 4. Password Reset

Workflow for users who have forgotten their password.

```bash
# Step 1: Request Password Reset Token
POST /api/auth/password/reset-request
Content-Type: application/json

{
  "email": "user@example.com"
}

# Response (204 No Content on success, even if email doesn't exist to prevent enumeration)
# An email with a reset link will be sent if the user exists and emails are configured.

# Step 2: Reset Password using Token
POST /api/auth/password/reset
Content-Type: application/json

{
  "token": "jwt-reset-token-from-email",    # Token received via password reset email
  "newPassword": "newSecurePassword123"
}

# Response (204 No Content on success)
```


---
## Core Concepts

### Multi-tenancy

The system is designed with a multi-tenant architecture:

-   **Users**: Can belong to one or more tenants.
-   **Tenants**: Represent distinct organizational units or clients. Each tenant has its own isolated set of user associations and roles.
-   **Roles & Permissions**: User roles are assigned within the scope of a specific tenant. Permissions can be derived from these roles or assigned directly. Tenant-level roles (e.g., "DEMO") can also influence effective permissions.
-   **Active Tenant Context**: A user with access to multiple tenants operates within one "active" tenant context at a time for most operations. They can switch this context using the `/api/auth/use-tenant` endpoint. The `AuthApiImpl.setCurrentUserTenant` method handles this, checking if the user has access to the tenant and if the tenant is active.
-   **Data Isolation**: Most data, including user management by tenant admins, is scoped to the active tenant.
-   **Super Admin Tenant**: A special tenant UUID `00000000-0000-0000-0000-000000000000` (defined as `Tenant.SUPER_ADMIN_TENANT`) is used for system-level administration by users with the `super-admin` role.

#### Tenant Operations (for Authenticated Users)

```bash
# Get My Tenants: Retrieve the list of tenants the current user belongs to.
GET /api/auth/my-tenants
Authorization: Bearer <jwt-token>

# Response (200 OK): List<TenantInfo> (see DTOs section)

# Switch Active Tenant: Change the user's active tenant context for the current session.
# This will issue a new JWT reflecting the new tenant context.
POST /api/auth/use-tenant
Authorization: Bearer <jwt-token>
Content-Type: application/json

{
  "tenantId": "tenant-uuid-to-switch-to" # UUID of the tenant to activate
}

# Response (200 OK): AuthResponse (containing a new JWT token for the new tenant context)
```

#### Programmatic Tenant Context Access

Within your application's services, you can resolve the current active tenant ID using the `TenantResolver` interface. The default implementation is `SessionTenantResolver`.

```java
import io.preboot.auth.api.resolver.TenantResolver;
import org.springframework.stereotype.Service;
import java.util.UUID;
import lombok.RequiredArgsConstructor; // if using Lombok

@Service
@RequiredArgsConstructor // if using Lombok
public class MyTenantAwareService {

    private final TenantResolver tenantResolver;

    public void performTenantSpecificAction() {
        UUID currentTenantId = tenantResolver.getCurrentTenant(); // Resolves tenant ID from the current session
        if (currentTenantId != null) {
            // Logic specific to currentTenantId
            System.out.println("Operating under tenant: " + currentTenantId);
        } else {
            // Handle cases where tenant context is not available (e.g., system operations, public endpoints)
            System.out.println("No active tenant context.");
        }
    }
}
```


### Role-Based Access Control (RBAC)

Access to resources and operations is controlled using an RBAC model:

-   **User Roles**: Named collections of permissions (e.g., "ADMIN", "USER", "EDITOR"). Roles are assigned to users *within a specific tenant* (`user_account_roles` table).
-   **Role Permissions**: Granular authorizations (e.g., "CREATE_DOCUMENT", "DELETE_USER") associated with roles. These are defined system-wide in `user_account_role_permissions`.
-   **Custom User Permissions**: Permissions can be assigned directly to a user *within a specific tenant context* (`user_account_permissions` table).
-   **Tenant-Level Roles**: Roles defined directly on a tenant (in `tenant_roles` table, e.g., "DEMO" automatically assigned if `preboot.accounts.assign-demo-role-to-new-tenants` is true). These act as tenant-wide attributes. The names of these tenant-level roles are added to the `permissions` set in `UserAccountInfo` for users active within that tenant.
-   **Scope**: User role assignments and custom permissions are always scoped to a specific tenant.
-   **Security Annotations**: For declarative access control at controller or service method levels.
    -   `@SuperAdminRoleAccessGuard`: Restricts access to users with the `super-admin` role.
    -   `@TenantAdminRoleAccessGuard`: Restricts access to users with the `ADMIN` role within their currently active tenant.
    -   Spring Security's `@PreAuthorize` for more complex expression-based access control.

#### Security Annotations Example

```java
import io.preboot.auth.api.guard.SuperAdminRoleAccessGuard;
import io.preboot.auth.api.guard.TenantAdminRoleAccessGuard;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
// ... other imports

@RestController
@RequestMapping("/api/data")
public class DataController {

    // Requires the user to have the 'super-admin' role (system-level).
    @GetMapping("/system-report")
    @SuperAdminRoleAccessGuard // Checks for hasRole('super-admin')
    public String getSystemReport() {
        // ... logic for super-admin ...
        return "System Report Data";
    }

    // Requires the user to have the 'ADMIN' role within their currently active tenant.
    @GetMapping("/tenant-report")
    @TenantAdminRoleAccessGuard // Checks for hasRole('ADMIN')
    public String getTenantReport() {
        // ... logic for tenant admin ...
        return "Tenant Report Data for current tenant";
    }

    // Method-level security using Spring Security expressions.
    // Requires 'EDITOR' role OR 'EDIT_CONTENT' authority (permission) in the current tenant context.
    @PreAuthorize("hasRole('EDITOR') or hasAuthority('EDIT_CONTENT')") //
    @PostMapping("/content")
    public String updateContent(@RequestBody String content) {
        // ... logic to update content ...
        return "Content updated";
    }
}
```
**Note**: In Spring Security, `hasRole('X')` checks for an authority named `ROLE_X`. `hasAuthority('Y')` checks for an authority named `Y`. `guard-auth` adds `ROLE_` prefix to roles when creating `GrantedAuthority` objects in `JwtAuthenticationFilter`.

### Session Management

The module implements robust session management:

-   **JWT (JSON Web Tokens)**: Used for stateless authentication. Each token is tied to a server-side session entry in `user_account_sessions`.
-   **Device Fingerprinting**: Enhances security by generating a fingerprint using `DeviceFingerprintService`. It combines request headers (e.g., User-Agent, Accept-Language, Sec-Ch-Ua-Platform), IP address, and optionally client-side information from `ClientInfo` DTO (if provided to the service, like screen dimensions, timezone, canvas/WebGL fingerprints). This fingerprint is stored with the session and verified during session refresh.
-   **Configurable Timeouts**: Separate timeouts for standard sessions (`session-timeout-minutes`) and "remember-me" extended sessions (`long-session-timeout-days`).
-   **Session Tracking**: Active sessions are stored in the `user_account_sessions` database table.
-   **Session Invalidation**: Logout explicitly invalidates the session on the server-side by setting its `expires_at` to the current time.
-   **`SessionAwareAuthentication`**: Custom Spring Security `Authentication` object that holds the `UserAccountInfo` principal and the `sessionId`.

#### Accessing Session Information Programmatically (Server-Side)

Within a secured endpoint, you can access details about the current user's session:

```java
import io.preboot.auth.core.spring.SessionAwareAuthentication;
import io.preboot.auth.api.dto.UserAccountInfo;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.UUID;

@RestController
public class MySessionInfoController {

    @GetMapping("/api/my-session-details")
    public SessionDetailsResponse getCurrentSessionDetails() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (authentication instanceof SessionAwareAuthentication sessionAuth) { //
            UserAccountInfo userPrincipal = (UserAccountInfo) sessionAuth.getPrincipal(); //
            UUID sessionId = sessionAuth.getSessionId(); //
            UUID currentTenantId = userPrincipal.tenantId(); // TenantId from UserAccountInfo reflects the session's tenant

            return new SessionDetailsResponse(
                userPrincipal.uuid(),
                sessionId,
                currentTenantId,
                userPrincipal.username()
            );
        }
        throw new IllegalStateException("No active SessionAwareAuthentication found.");
    }

    record SessionDetailsResponse(UUID userId, UUID sessionId, UUID tenantId, String username) {}
}
```

---
## API Reference

The `preboot-auth-api` module defines the public interfaces and DTOs for interacting with the authentication and user management services.

### `AuthApi` Interface

(`io.preboot.auth.api.AuthApi`)
Main interface for authentication and session operations.

-   `AuthResponse login(PasswordLoginRequest request, HttpServletRequest servletRequest)`: Authenticates a user, creates a session, and returns a JWT.
-   `void logout(final HttpServletRequest servletRequest)`: Invalidates the current user's session.
-   `AuthResponse refreshSession(final HttpServletRequest servletRequest)`: Refreshes the current session and returns a new JWT; validates device fingerprint.
-   `UserAccountInfo getCurrentUserAccount(final HttpServletRequest servletRequest)`: Retrieves info about the currently authenticated user.
-   `List<TenantInfo> getCurrentUserTenants(HttpServletRequest servletRequest)`: Gets tenants accessible to the current user.
-   `AuthResponse setCurrentUserTenant(UUID tenantUUID, HttpServletRequest servletRequest)`: Sets a new active tenant for the session and returns a new JWT.

### `UserAccountManagementApi` Interface

(`io.preboot.auth.api.UserAccountManagementApi`)
Interface for user account and tenant-related user management.

-   `UserAccountInfo getUserAccount(UUID userAccountId, final UUID tenantId)`: Retrieves user details within a tenant context.
-   `UserAccountInfo createUserAccountForTenant(CreateInactiveUserAccountRequest request)`: Creates an inactive user in an existing tenant; triggers activation.
-   `UserAccountInfo createTenantAndUserAccount(CreateTenantAndInactiveUserAccountRequest request)`: Creates a new tenant and an initial inactive user; triggers activation.
-   `void updatePassword(UpdatePasswordCommand command)`: Allows a user to update their own password.
-   `UserAccountInfo addRole(UUID userId, UUID tenantId, String roleName)`: Assigns a role to a user within a tenant.
-   `UserAccountInfo removeRole(UUID userId, UUID tenantId, String roleName)`: Removes a role from a user within a tenant.
-   `Page<UserAccountInfo> getUserAccountsInfo(SearchParams queryParams, final UUID tenantId)`: Gets a paginated list of users for a tenant, with search.
-   `Page<UserAccountInfo> getAllUserAccountsInfo(SearchParams params)`: Gets a paginated list of all users across all tenants (super-admin).
-   `void removeUser(UUID userId, final UUID tenantId)`: Removes a user's association with a tenant; may delete the user if it's their last tenant.

### `UserAccountSessionManagementApi` Interface

(`io.preboot.auth.api.UserAccountSessionManagementApi`)
Interface for administrative session management.

-   `void cleanExpiredSessions(Instant threshold)`: Deletes sessions expired before the threshold.

### Key Data Transfer Objects (DTOs)

#### Core DTOs

##### `UserAccountInfo`
(`io.preboot.auth.api.dto.UserAccountInfo`)
Detailed user information within their active tenant context.
-   `uuid`: User's unique identifier.
-   `username`: Display name.
-   `email`: User's email.
-   `roles`: Assigned roles in the current active tenant.
-   `permissions`: Permissions derived from user's roles, direct tenant-level roles (like "DEMO"), and custom permissions.
-   `customPermissions`: Additional permissions assigned directly to the user in the current tenant.
-   `active`: Account activation status.
-   `tenantId`: UUID of the current active tenant.
-   `tenantName`: Name of the current active tenant.
-   `getAllPermissions()`: Method to get a consolidated set of all `permissions` and `customPermissions`.

##### `TenantInfo`
(`io.preboot.auth.api.dto.TenantInfo`)
Information about a tenant a user can access.
-   `uuid`: Tenant's unique identifier.
-   `name`: Tenant name.
-   `lastUsedAt`: Timestamp of when the user last used this tenant.
-   `active`: Tenant status.

##### `AuthResponse`
(`io.preboot.auth.api.dto.AuthResponse`)
Response for authentication operations, containing the JWT.
-   `token`: JWT token (NotBlank).

#### Key Request & Other DTOs

##### `PasswordLoginRequest`
(`io.preboot.auth.api.dto.PasswordLoginRequest`)
-   `email`: User's email (NotBlank, Email).
-   `password`: User's password (NotBlank).
-   `rememberMe`: Boolean, if true, requests a long-lived session.

##### `ActivateAccountRequest`
(`io.preboot.auth.api.dto.ActivateAccountRequest`)
-   `token`: Activation token (NotBlank).
-   `password`: User's chosen password (NotBlank).

##### `CreateInactiveUserAccountRequest`
(`io.preboot.auth.api.dto.CreateInactiveUserAccountRequest`)
-   `username`: Username (NotBlank).
-   `email`: User's email (NotBlank, Email).
-   `language`: Optional.
-   `timezone`: Optional.
-   `roles`: Set of roles for the user in the tenant.
-   `permissions`: Set of custom permissions.
-   `tenantId`: UUID of the tenant.

##### `CreateTenantAndInactiveUserAccountRequest`
(`io.preboot.auth.api.dto.CreateTenantAndInactiveUserAccountRequest`)
Used for registration, creating a new tenant and an inactive user.
-   `username`: Username (NotBlank).
-   `email`: User's email (NotBlank, Email).
-   `language`: Optional.
-   `timezone`: Optional.
-   `roles`: Roles for the user in the new tenant.
-   `permissions`: Custom permissions in the new tenant.
-   `tenantName`: Name for the new tenant (NotBlank).

##### `CreateTenantRequest`
(`io.preboot.auth.api.dto.CreateTenantRequest`)
-   `name`: Tenant name (NotBlank).
-   `active`: Tenant active status (NotNull).

##### `RequestPasswordResetRequest`
(`io.preboot.auth.api.dto.RequestPasswordResetRequest`)
-   `email`: User's email (NotBlank, Email).

##### `ResetPasswordRequest`
(`io.preboot.auth.api.dto.ResetPasswordRequest`)
-   `token`: Password reset token (NotBlank).
-   `newPassword`: User's new password (NotBlank).

##### `TenantResponse`
(`io.preboot.auth.api.dto.TenantResponse`)
Details about a tenant.
-   `uuid`: Tenant UUID.
-   `name`: Tenant name.
-   `roles`: Set of roles defined at the tenant level (e.g., "DEMO").
-   `active`: Tenant active status.
-   `demo`: Convenience flag, true if "DEMO" is in tenant roles.

##### `TenantUpdateRequest`
(`io.preboot.auth.api.dto.TenantUpdateRequest`)
-   `tenantId`: Tenant UUID (NotNull).
-   `name`: New tenant name (NotBlank).
-   `active`: New active status (NotNull).

##### `TenantUserAssignRequest`
(`io.preboot.auth.api.dto.TenantUserAssignRequest`)
Used by tenant admins to update user roles.
-   `userUuid`: UUID of the user (NotNull).
-   `roles`: New set of roles for the user (NotNull).

##### `UpdatePasswordCommand`
(`io.preboot.auth.api.dto.UpdatePasswordCommand`)
-   `userId`: User's own UUID (NotNull).
-   `currentPassword`: User's current password (NotBlank).
-   `newPassword`: User's new password (NotBlank).

##### `UseTenantRequest`
(`io.preboot.auth.api.dto.UseTenantRequest`)
-   `tenantId`: Tenant UUID to switch to (NotNull).

##### `ClientInfo`
(`io.preboot.auth.api.dto.ClientInfo`)
Contains client-side information used by `DeviceFingerprintService` to enhance device fingerprinting. It's not typically a direct request body for login/refresh but is used if its data is available to the service.
-   `userAgent`, `platform`, `language`, `screenWidth`, `screenHeight`, `timezone`, `plugins`, `cookiesEnabled`, `canvas` (fingerprint), `webgl` (fingerprint).

---
## REST Endpoints Documentation

Base path: `/api/auth`. Super-admin: `/api/super-admin`. Tenant admin: `/api/tenant`.

### Authentication Endpoints
Provider: `io.preboot.auth.core.rest.AuthController`

| Method | Endpoint                            | Description                                      | Request Body             | Response Type      | Exception Handling (HTTP Status) |
|--------|-------------------------------------|--------------------------------------------------|--------------------------|--------------------|------------------------------------|
| POST   | `/api/auth/login`                   | Authenticate user.                        | `PasswordLoginRequest`   | `AuthResponse`     | `PasswordInvalidException` (401), `UserAccountNotFoundException` (401) |
| POST   | `/api/auth/logout`                  | Logout current user.                      | -                        | 200 OK / 204       | -                                  |
| POST   | `/api/auth/refresh`                 | Refresh session token.                    | -                        | `AuthResponse`     | `SessionFingerprintException` (401), `SessionExpiredException` (401), `SessionNotFoundException` (401) |
| GET    | `/api/auth/me`                      | Get current user's info.                  | -                        | `UserAccountInfo`  | (Uses session exceptions from refresh) |
| GET    | `/api/auth/my-tenants`              | Get accessible tenants.                   | -                        | `List<TenantInfo>` | (Uses session exceptions from refresh) |
| POST   | `/api/auth/use-tenant`              | Switch active tenant.                     | `UseTenantRequest`       | `AuthResponse`     | `TenantAccessDeniedException` (403), (Uses session exceptions from refresh) |

### User Registration & Activation Endpoints
Providers: `RegistrationController`, `AccountActivationController`

| Method | Endpoint                            | Description                                       | Request Body (see details below)         | Response Status | Exception Handling (HTTP Status) |
|--------|-------------------------------------|---------------------------------------------------|------------------------------------------|-----------------|------------------------------------|
| POST   | `/api/auth/registration`            | Register new user & tenant.                   | `RegistrationController.RegisterTenantAndInactiveUserAccountRequest` | 200 OK (void)   | `ResponseStatusException` (403 if disabled) |
| POST   | `/api/auth/activation`              | Activate account & set password.              | `ActivateAccountRequest`                 | 204 No Content  | `InvalidActivationTokenException` (401), `PasswordResetTokenExpiredException` (401) |

*Note on `/api/auth/registration` Request Body*: The actual DTO is `io.preboot.auth.core.rest.RegistrationController.RegisterTenantAndInactiveUserAccountRequest(username, email, language, timezone, tenantName)`. It internally creates a `CreateTenantAndInactiveUserAccountRequest` using default roles from `AuthAccountProperties`.

### Password Reset Endpoints
Provider: `PasswordResetController`

| Method | Endpoint                            | Description                             | Request Body                  | Response Status | Exception Handling (HTTP Status) |
|--------|-------------------------------------|-----------------------------------------|-------------------------------|-----------------|------------------------------------|
| POST   | `/api/auth/password/reset-request`  | Request password reset token.           | `RequestPasswordResetRequest` | 204 No Content  | `UserAccountNotFoundException` (404) |
| POST   | `/api/auth/password/reset`          | Reset password using token.             | `ResetPasswordRequest`        | 204 No Content  | `InvalidPasswordResetTokenException` (401), `PasswordResetTokenExpiredException` (401), `JwtException` (401) |

### Super Admin: User Management Endpoints
Provider: `io.preboot.auth.core.rest.SuperAdminUserController` (Requires `super-admin` role via `@SuperAdminRoleAccessGuard`)

| Method | Endpoint                                                | Description                                       | Request Body                        | Response Type        | Exception Handling (HTTP Status) |
|--------|---------------------------------------------------------|---------------------------------------------------|-------------------------------------|----------------------|------------------------------------|
| POST   | `/api/super-admin/users/search`                         | Search all users.                         | `SearchRequest`                     | `Page<UserAccountInfo>`| -                                  |
| POST   | `/api/super-admin/users/tenant/{tenantId}`              | Create inactive user in a tenant.           | `CreateInactiveUserAccountRequest`  | `UserAccountInfo`    | `UserAccountNotFoundException` (404) |
| GET    | `/api/super-admin/users/{userId}/tenant/{tenantId}`     | Get user details in a tenant.             | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404) |
| POST   | `/api/super-admin/users/tenant/{tenantId}/search`       | Search users in a tenant.                 | `SearchRequest`                     | `Page<UserAccountInfo>`| -                                  |
| DELETE | `/api/super-admin/users/{userId}/tenant/{tenantId}`     | Remove user from tenant.                  | -                                   | 204 No Content       | `UserAccountNotFoundException` (404) |
| POST   | `/api/super-admin/users/{userId}/tenant/{tenantId}/roles/{roleName}` | Add role to user in tenant.        | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404) |
| DELETE | `/api/super-admin/users/{userId}/tenant/{tenantId}/roles/{roleName}` | Remove role from user in tenant.   | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404) |

### Super Admin: Tenant Management Endpoints
Provider: `io.preboot.auth.core.rest.SuperAdminTenantController` (Requires `super-admin` role via `@SuperAdminRoleAccessGuard`)

| Method | Endpoint                                      | Description                                  | Request Body           | Response Type         |
|--------|-----------------------------------------------|----------------------------------------------|------------------------|-----------------------|
| POST   | `/api/super-admin/tenants/search`             | Search tenants.                              | `SearchRequest`        | `Page<TenantResponse>`|
| GET    | `/api/super-admin/tenants/{tenantId}`         | Get tenant details.                          | -                      | `TenantResponse`      |
| POST   | `/api/super-admin/tenants`                    | Create tenant.                               | `CreateTenantRequest`  | `TenantResponse` (201)|
| PUT    | `/api/super-admin/tenants/{tenantId}`         | Update tenant.                               | `TenantUpdateRequest`  | `TenantResponse`      |
| DELETE | `/api/super-admin/tenants/{tenantId}`         | Delete tenant.                               | -                      | 204 No Content        |
| POST   | `/api/super-admin/tenants/{tenantId}/roles/{roleName}` | Add role to tenant (e.g., "DEMO").       | -                      | 201 Created           |
| DELETE | `/api/super-admin/tenants/{tenantId}/roles/{roleName}` | Remove role from tenant.                 | -                      | 204 No Content        |

### Tenant Admin: User Management Endpoints
Provider: `io.preboot.auth.core.rest.TenantUserAdminController` (Requires `ADMIN` role within active tenant via `@TenantAdminRoleAccessGuard`)

| Method | Endpoint                                      | Description                                         | Request Body                        | Response Type        | Exception Handling (HTTP Status) |
|--------|-----------------------------------------------|-----------------------------------------------------|-------------------------------------|----------------------|------------------------------------|
| POST   | `/api/tenant/users/search`                    | Search users in current tenant.             | `SearchRequest`                     | `Page<UserAccountInfo>`| `UserAccountNotFoundException` (404) |
| GET    | `/api/tenant/users/{userId}`                  | Get user details in current tenant.       | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404) |
| POST   | `/api/tenant/users`                           | Create inactive user in current tenant.     | `CreateInactiveUserAccountRequest`  | `UserAccountInfo` (201)| `UserAccountNotFoundException` (404), `ResponseStatusException` (403 for restricted roles) |
| PUT    | `/api/tenant/users/{userId}/roles`            | Update user roles in current tenant.        | `TenantUserAssignRequest`           | `UserAccountInfo`    | `UserAccountNotFoundException` (404), `ResponseStatusException` (403 for self-admin role removal or restricted roles) |
| DELETE | `/api/tenant/users/{userId}`                  | Remove user from current tenant.            | -                                   | 204 No Content       | `UserAccountNotFoundException` (404), `ResponseStatusException` (403 for self-removal) |
| POST   | `/api/tenant/users/{userId}/roles/{roleName}` | Add role to user in current tenant.         | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404), `ResponseStatusException` (403 for restricted roles) |
| DELETE | `/api/tenant/users/{userId}/roles/{roleName}` | Remove role from user in current tenant.    | -                                   | `UserAccountInfo`    | `UserAccountNotFoundException` (404), `ResponseStatusException` (403 for self-admin role removal) |

**Notes for Tenant Admin User Management:**
-   Uses `TenantUserService` for role validation (cannot assign restricted roles like "super-admin") and user verification.
-   Tenant admin cannot remove their own "ADMIN" role or remove themselves from the tenant.

---
## Email Integration

The `preboot-auth-emails` module, if included and configured, provides automatic email notifications for account activation and password reset requests. It listens for events from `preboot-auth-core` and uses `io.preboot.notifications.spi.NotificationApi`.

### Account Activation Email
-   **Triggered by**: `UserAccountActivationTokenGeneratedEvent`.
-   **Purpose**: Verify email, allow account activation and password setting.
-   **Content**: Activation link, token expiration info (from `preboot.security.activation-token-timeout-in-days`).
-   **Key Config**: `sender-email`, `password-activation-url`, `account-activation-email-template`/`subject`, `logo-path`.

### Password Reset Email
-   **Triggered by**: `UserAccountPasswordResetTokenGeneratedEvent`.
-   **Purpose**: Securely reset forgotten password.
-   **Content**: Reset link, token expiration info (from `preboot.security.password-reset-token-timeout-in-days`, converted to hours in template).
-   **Key Config**: `sender-email`, `password-reset-url`, `forgotten-password-reset-email-template`/`subject`, `logo-path`.

### Email Templates
-   Uses Thymeleaf. Default HTML templates: `account-activation.email.html`, `account-reset-password.email.html`, `styles.html`.
-   **Customization**: Override by placing same-named files in `src/main/resources/templates/` or configure different names.
-   **Logo**: `logoBase64` (Base64 SVG) available to templates. Configured via `preboot.auth-emails.logo-path`. Fallback to `classpath:svg/logo.svg` if configured/default is not found. If neither found, no logo.

---
## Event System

The module publishes domain events for auth, user, and tenant lifecycles. Consume with an event bus implementation (e.g., `preboot-eventbus-core`).

### Published Events (`io.preboot.auth.api.event.*`)

#### User Account Events
-   `UserAccountCreatedEvent(UUID tenantId, UUID userAccountId, String email, String username)`: New user associated with a tenant.
-   `UserAccountActivatedEvent(UUID userAccountId, String email, String username)`: User account activated.
-   `UserAccountRemovedEvent(String email, String username)`: User account completely removed.
-   `UserAccountPasswordUpdatedEvent(UUID userAccountId, String email, String username)`: User's password changed.
-   `UserAccountActivationTokenGeneratedEvent(UUID userAccountId, String email, String username, String token)`: Activation token generated.
-   `UserAccountPasswordResetTokenGeneratedEvent(UUID userAccountId, String username, String email, String token)`: Password reset token generated.

#### Tenant Events
-   `TenantCreatedEvent(UUID tenantUuid, String name)`: New tenant created.
-   `TenantUpdatedEvent(UUID tenantUuid, String name)`: Tenant details updated.
-   `TenantDeletedEvent(UUID tenantUuid)`: Tenant deleted.

#### User-Tenant Relationship Events
-   `UserAccountAssignedToTenantEvent(UUID userUuid, UUID tenantUuid)`: (Defined, but not explicitly published by current core user creation which uses `UserAccountCreatedEvent`).
-   `UserAccountRemovedFromTenantEvent(UUID userUuid, UUID tenantUuid)`: User disassociated from a tenant.

### Event Handling Example

```java
import io.preboot.eventbus.EventHandler;
import io.preboot.auth.api.event.UserAccountCreatedEvent;
import io.preboot.auth.api.event.TenantCreatedEvent;
import org.springframework.stereotype.Service;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class CustomAuthAuditLogger {

    @EventHandler
    public void handleUserCreatedInTenant(UserAccountCreatedEvent event) { //
        log.info("User {} (Email: {}) created with ID {} in tenant {}.",
            event.username(), event.email(), event.userAccountId(), event.tenantId());
    }

    @EventHandler
    public void handleTenantCreation(TenantCreatedEvent event) { //
        log.info("Tenant '{}' created with ID {}.", event.name(), event.tenantUuid());
    }
}
```

---
## Advanced Usage

### Custom User Provisioning Service Example

```java
import io.preboot.auth.api.UserAccountManagementApi;
import io.preboot.auth.api.dto.CreateInactiveUserAccountRequest;
import io.preboot.auth.api.dto.preboot.UserAccountInfo;
import io.preboot.auth.api.resolver.TenantResolver;
import io.preboot.auth.core.spring.AuthAccountProperties;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import java.util.Set;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class EmployeeProvisioningService {

    private final UserAccountManagementApi userAccountManagementApi; //
    private final TenantResolver tenantResolver; //
    private final AuthAccountProperties authAccountProperties; //

    public UserAccountInfo onboardNewEmployee(String email, String username, String departmentRole) {
        UUID currentTenantId = tenantResolver.getCurrentTenant(); //
        if (currentTenantId == null) {
            throw new IllegalStateException("Cannot provision employee: No active tenant context.");
        }

        Set<String> rolesToAssign = determineRolesForDepartment(departmentRole);
        String defaultLanguage = authAccountProperties.getDefaultLanguage(); //
        String defaultTimezone = authAccountProperties.getDefaultTimezone(); //

        CreateInactiveUserAccountRequest request = new CreateInactiveUserAccountRequest( //
            username, email, defaultLanguage, defaultTimezone,
            rolesToAssign, Set.of(), currentTenantId
        );

        return userAccountManagementApi.createUserAccountForTenant(request); //
    }

    private Set<String> determineRolesForDepartment(String departmentRole) {
        return switch (departmentRole.toUpperCase()) { //
            case "IT_SUPPORT" -> Set.of("USER", "HELPDESK_AGENT");
            case "SALES" -> Set.of("USER", "SALES_REP", "VIEW_REPORTS");
            default -> Set.of("USER");
        };
    }
}
```
**Note**: Ensure `@Order` and matchers are distinct or correctly prioritized with `preboot-auth-core`'s chain.

---
## Database Schema

Managed by Liquibase (`db-changelog-preboot-auth.xml`). Key tables (PostgreSQL specific syntax in changelog):

#### `user_accounts`
Core user info.
-   `id` (BIGSERIAL PK), `uuid` (UUID UNIQUE NOT NULL), `username` (VARCHAR), `email` (VARCHAR UNIQUE NOT NULL), `language` (VARCHAR), `timezone` (VARCHAR), `active` (BOOLEAN), `version` (BIGINT for optimistic locking), `reset_token_version` (INT).
    - `active` is typically false initially.
    - `reset_token_version` increments on password reset to invalidate old tokens.

#### `tenants`
Tenant information.
-   `id` (BIGSERIAL PK), `uuid` (UUID UNIQUE NOT NULL), `name` (VARCHAR UNIQUE NOT NULL), `created_at` (TIMESTAMPTZ), `attributes` (JSONB), `active` (BOOLEAN DEFAULT TRUE).

#### `user_account_credentials`
Hashed passwords. FK `user_accounts` (BIGINT) to `user_accounts(id)`.
-   `id` (BIGSERIAL PK), `user_accounts` (BIGINT FK), `type` (VARCHAR, e.g., "PASSWORD"), `attribute` (VARCHAR, hashed password).

#### `user_account_sessions`
Active user sessions. FK `user_account_id` to `user_accounts(uuid)`, `impersonated_by` to `user_accounts(uuid)`.
-   `id` (BIGSERIAL PK), `session_id` (UUID UNIQUE NOT NULL), `user_account_id` (UUID FK), `impersonated_by` (UUID FK, optional), `credential_type` (VARCHAR), `agent` (VARCHAR), `ip` (VARCHAR), `device_fingerprint` (VARCHAR), `created_at` (TIMESTAMPTZ), `expires_at` (TIMESTAMPTZ), `remember_me` (BOOLEAN), `tenant_id` (UUID).

#### `user_account_devices`
(Less used in core logic beyond fingerprinting). FK `user_accounts` to `user_accounts(id)`.
-   `id` (BIGSERIAL PK), `user_accounts` (BIGINT FK), `name` (VARCHAR), `device_fingerprint` (VARCHAR), `created_at` (TIMESTAMPTZ).

### Security and Relationship Tables

#### `user_account_roles`
User-role mapping within tenants. FK `user_accounts` to `user_accounts(id)`.
-   `id` (BIGSERIAL PK), `user_accounts` (BIGINT FK), `name` (VARCHAR, role name), `tenant_id` (UUID NOT NULL).

#### `user_account_permissions`
Direct user permissions within tenants. FK `user_accounts` to `user_accounts(id)`.
-   `id` (BIGSERIAL PK), `user_accounts` (BIGINT FK), `name` (VARCHAR, permission string), `tenant_id` (UUID NOT NULL).

#### `user_account_role_permissions`
System-wide role-to-permission definitions.
-   `id` (BIGSERIAL PK), `role` (VARCHAR, role name), `name` (VARCHAR, permission string).

#### `user_account_tenants`
Links users to their member tenants. FK `user_account_uuid` to `user_accounts(uuid)`, `tenant_uuid` to `tenants(uuid)`.
-   `id` (BIGSERIAL PK), `user_account_uuid` (UUID FK), `tenant_uuid` (UUID FK), `last_used_at` (TIMESTAMPTZ).

#### `tenant_roles`
Tenant-level roles (e.g., "DEMO").
-   `id` (BIGSERIAL PK), `tenant_id` (UUID NOT NULL), `role_name` (VARCHAR NOT NULL). `UNIQUE(tenant_id, role_name)`.

### Views

#### `user_accounts_info_view`
Read-only view for efficient querying of user lists with aggregated details. Used by `GetUserAccountUseCase` for `getUserAccountsInfo` and `getAllUserAccountsInfo`.
The view combines data from `user_accounts`, `tenants`, `user_account_roles`, `user_account_permissions`, and `user_account_role_permissions` using Common Table Expressions (CTEs) named `combined_permissions` and `tenant_users` internally in its definition.
It provides:
-   User details (`id`, `uuid`, `username`, `email`, `active`).
-   Associated `tenant_id` and `tenant_name`.
-   Comma-separated string of `roles` for the user within that tenant.
-   Comma-separated string of effective `permissions` (derived from user's roles in that tenant, tenant-level roles, and direct user permissions in that tenant).

---
## Default Setup (via Liquibase)

### Super Admin Account

Liquibase creates a super-admin account:
-   **Email**: `super-admin@system.local`
-   **Default Password**: `changeme` **(CRITICAL: Change immediately post-deployment!)**
-   **Role**: `super-admin`
-   **Tenant Association**: Nil UUID tenant `00000000-0000-0000-0000-000000000000` (conceptual "Administration" tenant).

### Default Permissions for `super-admin` Role

Granted via `user_account_role_permissions`:
-   `ADMIN_ACCESS`
-   `USER_MANAGEMENT`
-   `SYSTEM_CONFIGURATION`

---
## Security Considerations

### JWT Security
-   **Strong Secrets**: Use a unique, strong `jwt-secret` (min. 32 chars for HS256). Store securely.
-   **Token Expiration**: Configure appropriate timeouts.
-   **HTTPS Only**: Transmit JWTs over HTTPS.
-   **Secret Rotation**: Implement policy for `jwt-secret` rotation.

### Session Security
-   **Device Fingerprinting**: Enabled by default.
-   **Logout**: Ensure proper server-side session invalidation.
-   **Session Store Security**: Secure the session database.

### Password Security
-   **Hashing**: Uses Spring Security's delegating `PasswordEncoder` (typically BCrypt).
-   **Password Reset Tokens**: JWTs with short expiration; `reset_token_version` invalidates old tokens.
-   **Account Activation Tokens**: JWTs with configurable expiration.

### CORS Configuration
-   Specify allowed origins; avoid `"*"` in production.

### CSRF Protection
-   Disabled by default (`preboot.security.enable-csrf: false`). Enable if needed for traditional form submissions.

### Input Validation
-   DTOs use Jakarta Bean Validation constraints.

### Least Privilege
-   Assign roles/permissions based on least privilege. Review regularly.

---
## Troubleshooting

### Common Issues & Solutions

#### 401 Unauthorized / 403 Forbidden Errors
-   **Invalid/Expired JWT**: Check token format, expiration (`session-timeout-minutes`, `long-session-timeout-days`), server-side invalidation.
-   **Incorrect JWT Secret**: Ensure consistency across instances.
-   **User Inactive/Not Found**: Account status.
-   **Session Integrity**: `SessionExpiredException`, `SessionFingerprintException`, `SessionNotFoundException`.
-   **Permissions (403)**: Missing roles/permissions for endpoint guards.
-   **Public Endpoint Config**: Ensure intended public paths are configured.
-   **Tenant Access**: User lacks access to tenant or tenant is inactive.

#### Database Connection or Schema Problems
-   **Datasource Config**: Verify `spring.datasource.*`.
-   **Liquibase Errors**: Check startup logs. Ensure `db-changelog-preboot-auth.xml` runs before app changelogs. DB user needs DDL/DML privileges.

#### Email Not Sending (`preboot-auth-emails`)
-   **Dependencies**: Check for `preboot-auth-emails` and a `preboot-notifications-*-starter` (or `NotificationApi` bean).
-   **Config**: Verify `preboot.auth-emails.*` (sender, URLs) and underlying notifier (e.g., SMTP) settings.
-   **Templates**: Check Thymeleaf templates in `classpath:templates/` and logs for errors.
-   **Event System**: Ensure `AuthEventsHandler` receives events.
-   **Logo Path**: Verify `preboot.auth-emails.logo-path` and SVG file existence.

### Performance Optimization

#### Database Indexes
-   Changelog creates indexes. Add custom ones if needed based on query patterns.

#### Session Cleanup
-   `user_account_sessions` can grow. Use `UserAccountSessionManagementApi.cleanExpiredSessions(Instant threshold)` in a scheduled job.

**Example Scheduled Job:**
```java
import io.preboot.auth.api.UserAccountSessionManagementApi;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class ExpiredSessionCleanupTask {
    private final UserAccountSessionManagementApi sessionManagementApi; //

    @Scheduled(cron = "0 0 3 * * ?") // Daily at 3 AM
    public void cleanupVeryOldExpiredSessions() {
        Instant cleanupThreshold = Instant.now().minus(30, ChronoUnit.DAYS); //
        log.info("Cleaning expired sessions older than {}.", cleanupThreshold);
        sessionManagementApi.cleanExpiredSessions(cleanupThreshold); //
    }
}
```
Enable with `@EnableScheduling`.

#### Query Optimization
-   `user_accounts_info_view` is optimized for user listings.
-   Prefer `UserAccountManagementApi.getUserAccountsInfo()` or `getAllUserAccountsInfo()`.

---
## Best Practices

### Development
1.  **API First**: Use public interfaces from `preboot-auth-api`.
2.  **Configuration**: Prefer `application.yml` for customization.
3.  **Events for Customizations**: Use events for side-effects.
4.  **Tenant Awareness**: Use `TenantResolver`.
5.  **Security Annotations**: Use guards and `@PreAuthorize`.

### Production
1.  **Secrets Management**: Store `jwt-secret`, DB credentials securely (env vars, secrets manager). **No version control.**
2.  **HTTPS**: Enforce for all communication.
3.  **Monitoring & Logging**: Log auth events, failures, errors. Monitor session table growth, cleanup job.
4.  **Password Policies**: Enforce strong passwords client-side.
5.  **Updates**: Keep `preboot-auth` and dependencies updated.
6.  **Change Super Admin Password**: **Immediately change default `super-admin` password.**

## Related Documentation

### Frontend Integration
- [Authentication Components](/docs/frontend/components/authentication/) - Ready-to-use login and registration forms
- [useAuth Hook](/docs/frontend/hooks/use-auth/) - React hook for authentication state management

### Complete Example
- [Reference App Authentication Flow](/docs/reference-app/authentication-flow/) - End-to-end authentication implementation
- [Reference App Overview](/docs/reference-app/overview/) - See authentication in a complete application

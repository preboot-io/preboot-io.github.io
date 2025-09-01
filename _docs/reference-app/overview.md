---
layout: documentation
title: "Reference App Overview"
subtitle: "Complete SaaS application showcasing PreBoot integration"
permalink: /docs/reference-app/overview/
section: reference-app
---

# Reference App Overview

The PreBoot Reference App is a complete, production-ready SaaS application that demonstrates how to integrate all PreBoot components. It serves as both a learning tool and a starting point for your own applications.

## What's Included

### Backend Features
- **Multi-tenant architecture** with automatic data isolation
- **JWT authentication** with role-based access control
- **RESTful APIs** following enterprise patterns
- **Event-driven updates** with WebSocket notifications
- **Audit logging** and observability
- **Database migrations** with Liquibase
- **Email notifications** for account activation

### Frontend Features
- **Responsive React UI** built with PreBoot components
- **Authentication flows** (login, registration, password reset)
- **Tenant switching** for multi-tenant users
- **Data tables** with filtering and pagination
- **Real-time updates** via WebSocket integration
- **Form validation** and error handling
- **TypeScript** for type safety

### Integration Examples
- **Full authentication flow** from frontend to backend
- **Multi-tenant data operations** with automatic filtering
- **Event-driven architecture** with real-time UI updates
- **Error handling** and user feedback patterns
- **Production deployment** configuration

## Architecture Overview

```
Frontend (React)     Backend (Spring Boot)     Database (PostgreSQL)
┌─────────────────┐  ┌───────────────────────┐  ┌─────────────────────┐
│ PreBoot UI      │  │ PreBoot Core          │  │ Multi-tenant Schema │
│ Components      │  │ - Authentication      │  │ - User accounts     │
│ - LoginForm     │  │ - Multi-tenancy      │  │ - Tenant isolation  │
│ - DataTable     │  │ - Secure Data        │  │ - Audit logs        │
│ - TenantSwitcher│  │ - Event Bus          │  │ - Business data     │
│                 │  │ - Notifications      │  │                     │
└─────────────────┘  └───────────────────────┘  └─────────────────────┘
         │                        │                         │
         └────── HTTP + WebSocket ─────── JDBC ──────────────┘
```

## Use Cases Demonstrated

1. **SaaS User Management**
   - User registration with email activation
   - Multi-tenant user assignment
   - Role-based access control
   - Profile management

2. **Data Management**
   - CRUD operations with automatic tenant filtering
   - Bulk operations with security constraints
   - Data export and import
   - Audit trail tracking

3. **Real-time Features**
   - WebSocket notifications
   - Live data updates
   - User activity tracking
   - System status monitoring

## Getting Started

The fastest way to explore PreBoot is to run the Reference App locally:

1. [Quick Start Guide](/docs/reference-app/quick-start/) - Run in 10 minutes
2. [Architecture Deep Dive](/docs/reference-app/architecture/) - Understand the structure
3. [Customization Guide](/docs/reference-app/customization/) - Make it your own

## Next Steps

- Clone and run the Reference App locally
- Explore the codebase to understand patterns
- Read the architecture documentation
- Customize it for your specific use case
- Deploy to production using our guides
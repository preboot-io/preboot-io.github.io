---
layout: documentation
title: "Introduction"
subtitle: "Discover what PreBoot is and how it can help you accelerate SaaS application development."
permalink: /docs/
section: getting-started
---
# Introduction to PreBoot

Welcome to the official documentation for PreBoot, a comprehensive framework for Java developers building SaaS applications. This guide will help you get started with PreBoot and explore its features in depth.

## What is PreBoot?

PreBoot is a framework built on top of Spring Boot that provides a solid foundation for developing multi-tenant SaaS applications. It offers a collection of ready-to-use components and patterns that solve common challenges in SaaS development:

- **Multi-tenancy** - Serve multiple customers from a single deployment
- **Authentication & Authorization** - Secure your application with built-in security features
- **Event-Driven Architecture** - Build scalable, reactive applications
- **CQRS Implementation** - Clean separation of commands and queries
- **Task Queue** - Background job processing and scheduling
- **Frontend Components** - UI components tailored for SaaS applications

## PreBoot Architecture

PreBoot consists of three main components that work together:

### 1. Backend (Core)
A modular Spring Boot library providing enterprise SaaS patterns:
- **Multi-tenant data isolation** with automatic filtering
- **JWT authentication** with role-based access control  
- **Event-driven architecture** for scalable communication
- **Secure data access** with audit trails
- **Production-ready modules** for common SaaS needs

[Explore Backend Documentation →](/docs/backend/concepts/multi-tenancy/)

### 2. Frontend (UI)
React components and hooks integrated with the backend:
- **Authentication components** (login, registration, tenant switching)
- **Smart data tables** with filtering and pagination
- **Form components** with validation
- **Multi-tenant context** management
- **TypeScript support** for better developer experience

[Explore Frontend Documentation →](/docs/frontend/getting-started/)

### 3. Reference App
A complete working example showing full integration:
- **Complete SaaS application boilerplate** built with PreBoot
- **Best practices implementation** from real-world usage
- **Starting point** for your own SaaS application

[Explore Reference App →](/docs/reference-app/overview/)

## Recommended Learning Path

1. **Start with Reference App** - See everything working together
2. **Explore Backend concepts** - Understand the core architecture  
3. **Study Frontend components** - Learn the UI integration
4. **Customize for your needs** - Build your own SaaS application

## Next Steps
- [Installation Guide](/docs/getting-started/installation/) - Set up your development environment
- [Quick Start](/docs/getting-started/quick-start/) - Get running in 10 minutes
- [Reference App Overview](/docs/reference-app/overview/) - See the full stack in action

## Why Choose PreBoot?

Building SaaS applications involves implementing many common patterns repeatedly.
PreBoot saves you time by providing these patterns out of the box, allowing you to focus
on your unique business logic.

### Benefits

- **Speed up development** by not writing SaaS boilerplate again
- **Avoid common pitfalls** with proven patterns
- **Scale efficiently** with proper architecture from day one
- **Maintain clean code** with organized components
- **Focus on your business logic** instead of boilerplate

## Getting Started

Ready to start building with PreBoot?
Check out the [Quick Start Guide](/docs/getting-started/quick-start/) to get up and running in minutes.

## Community & Support

PreBoot is backed by an active community of developers:

- **GitHub**: Explore the source code [PreBoot](https://github.com/preboot-io/preboot), [PreBoot UI](https://github.com/preboot-io/preboot-ui), [PreBoot RefApp](https://github.com/preboot-io/preboot-refapp)
- **Discord**: Join our community for discussions and support

If you encounter any issues or have questions, please don't hesitate to reach out through one of these channels.

## Contributing

PreBoot is an open-source project, and we welcome contributions from the community. Whether it's fixing bugs, improving documentation, or adding new features, your help is appreciated.

See the [GitHub repository](https://github.com/preboot-io/preboot) to learn how you can get involved.

## Next Steps
- [Quick Start Guide](/docs/getting-started/quick-start/) - Get running in 10 minutes  
- [Reference App Overview](/docs/reference-app/overview/) - See the full stack in action
- [Backend Documentation](/docs/backend/concepts/multi-tenancy/) - Deep dive into concepts
- [Frontend Documentation](/docs/frontend/getting-started/) - Learn the UI components

---
layout: documentation
title: "Quick Start"
subtitle: "Build your first application with PreBoot in minutes."
permalink: /docs/getting-started/quick-start/
section: getting-started
---

# Quick Start Guide

The fastest way to understand PreBoot is to run the Reference App - a complete SaaS application built with all PreBoot components.

## Prerequisites

- Java 21+
- Node.js 16+  
- PostgreSQL database
- Git

## 1. Clone the Reference App

```bash
git clone https://github.com/preboot-io/preboot-refapp
cd preboot-refapp
```

## 2. Set up the Database

Create a `docker-compose.yml` file in your project root:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:17.4
    container_name: preboot-postgres
    environment:
      POSTGRES_DB: preboot_refapp
      POSTGRES_USER: preboot_user
      POSTGRES_PASSWORD: preboot_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    
  maildev:
    image: maildev/maildev
    container_name: preboot-maildev
    ports:
      - "1080:1080"  # Web UI
      - "1025:1025"  # SMTP server
    
volumes:
  postgres_data:
```

Start the services:

```bash
docker-compose up -d
```

Your database and mail server will be available at:
- **PostgreSQL**: `localhost:5432` (user: `preboot_user`, password: `preboot_password`)
- **MailDev UI**: http://localhost:1080 (for viewing development emails)

## 3. Run the Backend

```bash
cd backend
./mvnw spring-boot:run
```

The backend will start on http://localhost:8080

## 4. Run the Frontend

```bash
# In a new terminal
cd frontend
npm install
npm start
```

The frontend will start on http://localhost:5173

## 5. Explore the Application

1. **Register a new account** - Creates your tenant automatically
2. **Activate via email** - Check Maildev web interface for activation link - http://localhost:1080
3. **Explore multi-tenancy** - Create additional users and tenants
4. **Test the API** - Use built-in admin panels and forms

## What You'll See

- **Complete authentication flow** with JWT and sessions
- **Multi-tenant data isolation** - each tenant sees only their data
- **Real-time updates** via WebSocket integration  
- **Production-ready patterns** like audit logging and error handling
- **Responsive UI** built with PreBoot components

## Next Steps

- [Reference App Architecture](/docs/reference-app/architecture/) - Understand how it works
- [Customize the RefApp](/docs/reference-app/customization/) - Make it your own
- [Backend Documentation](/docs/backend/concepts/multi-tenancy/) - Deep dive into concepts
- [Frontend Documentation](/docs/frontend/getting-started/) - Learn the UI components

## Need Help?

- [Join our Discord](https://discord.gg/Ux7ruzctZ5) for community support
- [GitHub Discussions](https://github.com/preboot-io/preboot/discussions) for questions
- [Report issues](https://github.com/preboot-io/preboot-refapp/issues) if something doesn't work

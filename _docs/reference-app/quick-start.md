---
layout: documentation
title: "Reference App Quick Start"
subtitle: "Get the complete PreBoot example running in minutes"
permalink: /docs/reference-app/quick-start/
section: reference-app
---

# Reference App Quick Start

Get the complete PreBoot Reference App running locally to see all components working together.

## Prerequisites

- **Java 17+** - For running the Spring Boot backend
- **Node.js 16+** - For the React frontend
- **PostgreSQL 12+** - Database server
- **Git** - For cloning the repository

## 1. Clone the Repository

```bash
git clone https://github.com/preboot-io/preboot-refapp
cd preboot-refapp
```

## 2. Database Setup

### Option A: Local PostgreSQL

```bash
# Create database
createdb preboot_refapp

# Create user (optional)
createuser -s preboot_user
```

### Option B: Docker PostgreSQL

```bash
docker run --name preboot-postgres \
  -e POSTGRES_DB=preboot_refapp \
  -e POSTGRES_USER=preboot_user \
  -e POSTGRES_PASSWORD=preboot_pass \
  -p 5432:5432 -d postgres:15
```

## 3. Backend Configuration

Create `backend/src/main/resources/application-local.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/preboot_refapp
    username: preboot_user
    password: preboot_pass
  
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
  
  mail:
    host: localhost
    port: 1025
    # For development, use MailHog or similar

preboot:
  auth:
    jwt:
      secret: your-secret-key-here-make-it-long-and-random
      expiration: 86400000
  
  tenant:
    default-admin-email: admin@example.com
```

## 4. Start the Backend

```bash
cd backend
./mvnw spring-boot:run -Dspring.profiles.active=local
```

The backend will start on `http://localhost:8080`

**What happens on startup:**
- Database schema created automatically via Liquibase
- Default admin user and tenant created
- Email notifications configured (check console for activation links)

## 5. Frontend Setup

In a new terminal:

```bash
cd frontend
npm install
```

Create `frontend/.env.local`:

```env
REACT_APP_API_BASE_URL=http://localhost:8080/api
REACT_APP_WS_URL=ws://localhost:8080/ws
```

## 6. Start the Frontend

```bash
npm start
```

The frontend will start on `http://localhost:3000`

## 7. First Login

1. **Open** http://localhost:3000
2. **Register** a new account (creates your tenant automatically)
3. **Check console logs** for email activation link
4. **Click activation link** to activate your account
5. **Login** with your credentials

## 8. Explore the Features

### Multi-tenant Data
- Create users and assign them to different tenants
- Switch between tenants to see data isolation
- Test role-based access control

### Real-time Updates
- Open multiple browser tabs
- Make changes in one tab, see updates in others
- WebSocket notifications work automatically

### Data Management
- Use the data tables with filtering and sorting
- Import/export data to test bulk operations
- Check audit logs for all changes

## Development Tools

### Hot Reload
- Backend: Uses Spring Boot DevTools for hot reload
- Frontend: React development server provides live reload

### Database Management
- **H2 Console**: `http://localhost:8080/h2-console` (if using H2)
- **pgAdmin**: For PostgreSQL management
- **Liquibase**: Database changes in `src/main/resources/db/changelog/`

### API Documentation
- **OpenAPI/Swagger**: `http://localhost:8080/swagger-ui.html`
- **API Docs**: `http://localhost:8080/api-docs`

## Troubleshooting

### Port Already in Use
```bash
# Check what's using port 8080
lsof -i :8080

# Kill the process if needed
kill -9 <PID>
```

### Database Connection Issues
```bash
# Test PostgreSQL connection
psql -h localhost -U preboot_user -d preboot_refapp

# Check if PostgreSQL is running
pg_ctl status
```

### Frontend Build Issues
```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

## Next Steps

- [Architecture Overview](/docs/reference-app/architecture/) - Understand how it all works
- [Authentication Flow](/docs/reference-app/authentication-flow/) - Deep dive into auth
- [Customization Guide](/docs/reference-app/customization/) - Make it your own
- [Backend Concepts](/docs/backend/concepts/multi-tenancy/) - Learn core patterns
- [Frontend Components](/docs/frontend/getting-started/) - Explore UI components

## Need Help?

- **Discord**: [Join our community](https://discord.gg/Ux7ruzctZ5)
- **GitHub Issues**: [Report problems](https://github.com/preboot-io/preboot-refapp/issues)
- **Documentation**: [Browse all docs](/docs/)
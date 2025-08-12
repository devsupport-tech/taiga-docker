# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the official Docker deployment repository for Taiga, an open-source project management platform. The deployment uses Docker Compose to orchestrate multiple services including PostgreSQL, RabbitMQ, Nginx, and various Taiga components.

## Essential Commands

### Starting Taiga
```bash
./launch-taiga.sh
```

### Managing Taiga Backend
```bash
# Run Django management commands
./taiga-manage.sh [COMMAND]

# Create superuser (after initial setup)
./taiga-manage.sh createsuperuser

# Run migrations
./taiga-manage.sh migrate

# Collect static files
./taiga-manage.sh collectstatic
```

### Docker Compose Operations
```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f [service-name]

# Restart a specific service
docker compose restart [service-name]
```

## Architecture & Services

The deployment consists of these interconnected services defined across `docker-compose.yml` and `docker-compose-inits.yml`:

- **taiga-db**: PostgreSQL 12.3 database
- **taiga-back**: Django backend API server
- **taiga-async**: Celery worker for async tasks
- **taiga-front**: Angular frontend application
- **taiga-events**: WebSocket server for real-time updates
- **taiga-protected**: Service for protected media file serving
- **taiga-gateway**: Nginx reverse proxy (port 9000)
- **taiga-async-rabbitmq**: RabbitMQ for async task queue
- **taiga-events-rabbitmq**: RabbitMQ for event system
- **taiga-manage**: Utility service for Django management commands (in docker-compose-inits.yml)

## Configuration Structure

### Environment Variables (.env)
Primary configuration is managed through the `.env` file with these key sections:
- **URLs**: TAIGA_SCHEME, TAIGA_DOMAIN, SUBPATH, WEBSOCKETS_SCHEME
- **Security**: SECRET_KEY (must be changed in production)
- **Database**: POSTGRES_USER, POSTGRES_PASSWORD
- **Email**: EMAIL_BACKEND, SMTP configuration
- **RabbitMQ**: RABBITMQ_USER, RABBITMQ_PASS, RABBITMQ_VHOST
- **Attachments**: ATTACHMENTS_MAX_AGE
- **Telemetry**: ENABLE_TELEMETRY

### Advanced Configuration
For advanced setups, you can:
1. Map custom `config.py` for backend (uncomment volume mapping in docker-compose.yml)
2. Map custom `conf.json` for frontend (uncomment volume mapping in docker-compose.yml)

## Critical Deployment Patterns

### Subdomain vs Subpath Deployment
The system supports two deployment patterns configured via environment variables:

**Subdomain** (default):
- TAIGA_DOMAIN=taiga.mycompany.com
- SUBPATH=""

**Subpath**:
- TAIGA_DOMAIN=mycompany.com
- SUBPATH="/taiga"

### Service Dependencies
Services have specific startup dependencies:
- taiga-back and taiga-async wait for taiga-db health check
- taiga-events depends on taiga-events-rabbitmq
- taiga-gateway depends on taiga-front, taiga-back, and taiga-events

### Volume Persistence
Data is persisted in named Docker volumes:
- taiga-static-data: Static files
- taiga-media-data: User uploads
- taiga-db-data: PostgreSQL data
- taiga-async-rabbitmq-data: Async queue data
- taiga-events-rabbitmq-data: Events queue data

## Common Development Tasks

### Enabling Features
Features are enabled by adding environment variables to docker-compose.yml:
- Public registration: PUBLIC_REGISTER_ENABLED
- OAuth: ENABLE_GITHUB_AUTH, ENABLE_GITLAB_AUTH
- Integrations: ENABLE_SLACK
- Importers: ENABLE_GITHUB_IMPORTER, ENABLE_JIRA_IMPORTER, ENABLE_TRELLO_IMPORTER

### Debugging
```bash
# Check service status
docker compose ps

# View specific service logs
docker compose logs -f taiga-back

# Access Django shell
./taiga-manage.sh shell

# Check RabbitMQ management UI
# Access at http://localhost:15672 (if management plugin exposed)
```

## Important Notes

1. The SECRET_KEY in .env must be changed to a secure value before production deployment
2. When switching between subdomain and subpath configurations, change TAIGA_SECRET_KEY to force token refresh
3. The nginx gateway runs on port 9000 by default
4. Email defaults to console backend; configure SMTP for production
5. Both docker-compose.yml and docker-compose-inits.yml read from the same .env file
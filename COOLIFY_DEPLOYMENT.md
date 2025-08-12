# Deploying Taiga with Coolify

This guide explains how to deploy Taiga using Coolify with the provided Docker Compose configuration.

## Prerequisites

- Coolify v4 installed and running
- A domain name (for production)
- SMTP credentials (for email notifications)

## Deployment Steps

### 1. Create New Resource in Coolify

1. Go to your Coolify dashboard
2. Click "New Resource" → "Docker Compose"
3. Name your deployment (e.g., "Taiga Project Management")

### 2. Configure Docker Compose

Copy the contents of `coolify-docker-compose.yml` into the Docker Compose editor in Coolify.

### 3. Set Environment Variables

In Coolify's environment variables section, add the following required variables:

```bash
# CRITICAL: Generate secure values for production
SECRET_KEY=<generate-a-long-random-string>
POSTGRES_PASSWORD=<secure-database-password>
RABBITMQ_PASS=<secure-rabbitmq-password>
RABBITMQ_ERLANG_COOKIE=<unique-erlang-cookie>

# Domain configuration
TAIGA_DOMAIN=taiga.yourdomain.com
TAIGA_SCHEME=https
WEBSOCKETS_SCHEME=wss

# Email configuration (for production)
EMAIL_BACKEND=smtp
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
EMAIL_DEFAULT_FROM=noreply@yourdomain.com
```

> **Tip**: Use the `coolify.env.example` file as a template for all available options.

### 4. Configure Networking

#### Option A: Let Coolify Handle SSL (Recommended)
1. In Coolify, set the domain for your application
2. Enable "Force HTTPS"
3. Coolify will automatically provision SSL certificates

#### Option B: External Reverse Proxy
If you're using an external reverse proxy:
1. Point your proxy to `http://your-server-ip:9000`
2. Configure SSL termination at your proxy
3. Set appropriate headers (see nginx example in README.md)

### 5. Deploy

1. Click "Deploy" in Coolify
2. Monitor the deployment logs
3. Wait for all services to be healthy (this may take 2-3 minutes)

### 6. Create Admin User

Once deployed, create your admin user:

1. In Coolify, go to your Taiga deployment
2. Click on "Terminal" or "Execute Command"
3. Select the `taiga-back` container
4. Run:
   ```bash
   python manage.py createsuperuser
   ```
5. Follow the prompts to create your admin account

### 7. Access Taiga

- **Frontend**: https://taiga.yourdomain.com
- **Admin Panel**: https://taiga.yourdomain.com/admin/
- **API**: https://taiga.yourdomain.com/api/v1/

## Configuration Options

### Enable Public Registration

Add to environment variables:
```bash
PUBLIC_REGISTER_ENABLED=True
```

### Enable GitHub OAuth

Add to environment variables:
```bash
ENABLE_GITHUB_AUTH=True
GITHUB_API_CLIENT_ID=your-github-client-id
GITHUB_API_CLIENT_SECRET=your-github-secret
GITHUB_CLIENT_ID=your-github-client-id
```

### Enable GitLab OAuth

Add to environment variables:
```bash
ENABLE_GITLAB_AUTH=True
GITLAB_API_CLIENT_ID=your-gitlab-client-id
GITLAB_API_CLIENT_SECRET=your-gitlab-secret
GITLAB_CLIENT_ID=your-gitlab-client-id
GITLAB_URL=https://gitlab.com
```

## Troubleshooting

### Services Not Starting

Check the logs in Coolify:
1. Go to your deployment
2. Click "Logs"
3. Select the failing service
4. Look for error messages

Common issues:
- **Database connection failed**: Check POSTGRES_PASSWORD matches in all services
- **RabbitMQ connection failed**: Ensure RABBITMQ_USER and RABBITMQ_PASS are consistent
- **Frontend not loading**: Verify TAIGA_DOMAIN and TAIGA_SCHEME are correct

### Cannot Create Superuser

If the command fails:
1. Ensure the database is fully initialized
2. Try restarting the `taiga-back` service
3. Check database connectivity

### WebSocket Connection Issues

If real-time updates aren't working:
1. Verify WEBSOCKETS_SCHEME matches your TAIGA_SCHEME (wss for https, ws for http)
2. Check that your reverse proxy supports WebSocket upgrades
3. Ensure the taiga-events service is running

### Email Not Sending

For email issues:
1. Verify EMAIL_BACKEND is set to "smtp" (not "console")
2. Check SMTP credentials are correct
3. For Gmail, use an app-specific password, not your regular password
4. Review logs in the taiga-back service

## Backup and Restore

### Backup

To backup your Taiga data:

1. **Database**: 
   ```bash
   docker exec taiga-db pg_dump -U taiga taiga > taiga_backup.sql
   ```

2. **Media files**: 
   ```bash
   docker run --rm -v taiga-media-data:/data -v $(pwd):/backup alpine tar czf /backup/taiga-media.tar.gz -C /data .
   ```

3. **Static files** (optional, can be regenerated):
   ```bash
   docker run --rm -v taiga-static-data:/data -v $(pwd):/backup alpine tar czf /backup/taiga-static.tar.gz -C /data .
   ```

### Restore

1. **Database**:
   ```bash
   docker exec -i taiga-db psql -U taiga taiga < taiga_backup.sql
   ```

2. **Media files**:
   ```bash
   docker run --rm -v taiga-media-data:/data -v $(pwd):/backup alpine tar xzf /backup/taiga-media.tar.gz -C /data
   ```

## Performance Tuning

For production deployments with many users:

1. **Increase PostgreSQL resources** in docker-compose:
   ```yaml
   taiga-db:
     # ...
     deploy:
       resources:
         limits:
           memory: 2G
         reservations:
           memory: 1G
   ```

2. **Scale async workers** if needed (add to environment):
   ```bash
   CELERY_WORKER_CONCURRENCY=4
   ```

3. **Monitor RabbitMQ** queues through the management interface

## Security Considerations

1. **Always use HTTPS** in production
2. **Change all default passwords** before deploying
3. **Use strong SECRET_KEY** (minimum 50 characters, random)
4. **Restrict database access** to internal network only
5. **Regular backups** are essential
6. **Keep images updated** with latest security patches

## Support

- **Documentation**: https://docs.taiga.io/
- **Community**: https://community.taiga.io/
- **Issues**: https://github.com/taigaio/taiga-docker/issues
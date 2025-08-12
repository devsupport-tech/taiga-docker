# Deployment Instructions for taiga.css1.clientsupport.services

## Quick Start Deployment

### 1. Prepare Environment Variables

```bash
# Copy the production environment template
cp .env.production .env

# Generate secure values
echo "SECRET_KEY=$(openssl rand -hex 50)"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32)"
echo "RABBITMQ_PASS=$(openssl rand -base64 32)"
echo "RABBITMQ_ERLANG_COOKIE=$(openssl rand -hex 20)"
```

Edit `.env` and replace the placeholder values with the generated secure values.

### 2. Deploy with Coolify

#### Option A: Via Coolify UI
1. Create new resource → Docker Compose
2. Copy contents of `docker-compose.yaml`
3. Set environment variables from `.env`
4. Deploy

#### Option B: Direct Docker Compose
```bash
# Start all services
docker compose up -d

# Monitor startup
docker compose logs -f

# Wait for services to be healthy (2-3 minutes)
docker compose ps
```

### 3. Create Admin User

Once all services are running:

```bash
# Using the helper script
./taiga-manage.sh createsuperuser

# Or directly with docker
docker compose exec taiga-back python manage.py createsuperuser
```

### 4. Access Taiga

- **Main Application**: https://taiga.css1.clientsupport.services
- **Admin Panel**: https://taiga.css1.clientsupport.services/admin/
- **API Endpoint**: https://taiga.css1.clientsupport.services/api/v1/

## Essential Configuration

### Required Environment Variables

These MUST be changed from defaults before production deployment:

| Variable | Description | How to Generate |
|----------|-------------|-----------------|
| `SECRET_KEY` | Django secret key | `openssl rand -hex 50` |
| `POSTGRES_PASSWORD` | Database password | `openssl rand -base64 32` |
| `RABBITMQ_PASS` | RabbitMQ password | `openssl rand -base64 32` |
| `RABBITMQ_ERLANG_COOKIE` | RabbitMQ cluster cookie | `openssl rand -hex 20` |

### Email Configuration

For production email notifications:

```env
EMAIL_BACKEND=smtp
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=app-specific-password
EMAIL_DEFAULT_FROM=noreply@taiga.css1.clientsupport.services
```

## SSL/TLS Configuration

### If Coolify Manages SSL
Coolify will automatically provision Let's Encrypt certificates for `taiga.css1.clientsupport.services`.

### If Using External Reverse Proxy

Add to your nginx configuration:

```nginx
server {
    server_name taiga.css1.clientsupport.services;
    
    location / {
        proxy_pass http://your-server-ip:9000;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
    }
    
    location /events {
        proxy_pass http://your-server-ip:9000/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
    
    listen 443 ssl;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
}
```

## Post-Deployment Checklist

- [ ] All services show as healthy: `docker compose ps`
- [ ] Admin user created successfully
- [ ] Can login at https://taiga.css1.clientsupport.services
- [ ] Email sending works (test password reset)
- [ ] WebSocket events work (real-time updates)
- [ ] File uploads work properly
- [ ] SSL certificate is valid

## Maintenance Commands

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f taiga-back
```

### Restart Services
```bash
# All services
docker compose restart

# Specific service
docker compose restart taiga-back
```

### Database Backup
```bash
# Backup
docker exec taiga-db pg_dump -U taiga taiga > backup_$(date +%Y%m%d).sql

# Restore
docker exec -i taiga-db psql -U taiga taiga < backup_20240101.sql
```

### Update Taiga
```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d
```

## Troubleshooting

### Services Won't Start
```bash
# Check logs for errors
docker compose logs taiga-db
docker compose logs taiga-back

# Verify environment variables
docker compose config
```

### Cannot Access Frontend
1. Check nginx gateway: `docker compose logs taiga-gateway`
2. Verify domain in .env matches: `taiga.css1.clientsupport.services`
3. Ensure port 9000 is accessible

### Database Connection Issues
```bash
# Test database connection
docker compose exec taiga-db psql -U taiga -d taiga -c "SELECT 1"

# Check database logs
docker compose logs taiga-db
```

### Email Not Working
```bash
# Test email configuration
docker compose exec taiga-back python manage.py sendtestemail your@email.com

# Check email backend setting
grep EMAIL_BACKEND .env
```

## Security Hardening

1. **Firewall Rules**: Only expose port 9000 (or use Coolify's proxy)
2. **Regular Updates**: Pull latest images monthly
3. **Backup Strategy**: Daily database backups, weekly media backups
4. **Monitor Logs**: Check for unauthorized access attempts
5. **Secrets Rotation**: Change passwords quarterly

## Support

- **Taiga Documentation**: https://docs.taiga.io/
- **Community Forum**: https://community.taiga.io/
- **GitHub Issues**: https://github.com/taigaio/taiga-docker/issues
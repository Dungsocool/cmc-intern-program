# VM Deployment Guide with Nginx and Certbot

## Overview

This setup utilizes **2 layers of Nginx**:

- **Nginx in Docker** (port 80 → mapped to 3000): Serves frontend static files + proxies API calls to the backend container
- **Nginx on VM**: Reverse proxy + SSL termination with Certbot
- **Docker** to run the backend (Go API) and database (PostgreSQL)

### Why 2 layers of Nginx?

1. **Nginx in Docker container**:
   - Serves React static files (HTML, CSS, JS)
   - Proxies `/api/*` requests to the backend container
   - Portable and consistent across all environments

2. **Nginx on VM**:
   - SSL/TLS termination with Let's Encrypt (Certbot)
   - Rate limiting, DDoS protection
   - Load balancing (if scaling to multiple containers)
   - Centralized logging and monitoring

## Architecture

```
Internet (HTTPS) → Nginx (VM) + Certbot SSL
                        ↓
                   (HTTP internal)
                        ↓
            ┌───────────┴───────────┐
            ↓                       ↓
    Frontend:3000              Backend:8080
    (Nginx container)          (Go API)
    - Static files                 ↓
    - Proxy /api/*          PostgreSQL:5432
```

**Request Flow**:

1. Browser → `https://domain.com/` → Nginx VM (SSL) → Nginx container (port 3000) → Serve index.html
2. Browser → `https://domain.com/api/health` → Nginx VM → Nginx container `/api/*` → Backend:8080

## Step 1: Preparing the VM

### Requirements

- Ubuntu 20.04+ or Debian 11+
- Docker and Docker Compose installed
- Domain name pointed to the VM IP (for SSL)

### Installing Docker

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify
docker --version
docker-compose --version
```

### Installing Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Installing Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

## Step 2: Deploying the Application

### Clone the Repository

```bash
cd /opt
sudo git clone https://github.com/dinhmanhtan/cmc-intern-program.git
sudo chown -R $USER:$USER cmc-intern-program
cd cmc-intern-program/app/session7-deployment
```

### Configuring Environment Variables

```bash
# Create .env for backend (optional, can also use docker-compose.yml directly)
cat > backend/.env << EOF
DB_HOST=db
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=YOUR_SECURE_PASSWORD_HERE
DB_NAME=mini_asm
EOF
```

### Updating docker-compose.yml

```bash
# Modify passwords in docker-compose.yml
nano docker-compose.yml

# Change:
# POSTGRES_PASSWORD: postgres@123
# DB_PASSWORD: postgres@123
#
# To a stronger password
```

### Starting Services

```bash
# Build and start containers
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

## Step 3: Configuring Nginx

### Creating Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/mini-asm
```

File content (replace `your-domain.com` with your actual domain):

```nginx
# Mini ASM - Nginx Configuration
upstream frontend {
    server localhost:3000;
}

upstream backend {
    server localhost:8080;
}

server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;

    # Redirect HTTP to HTTPS (will be enabled after obtaining SSL)
    # return 301 https://$server_name$request_uri;

    # Client max body size
    client_max_body_size 10M;

    # Frontend
    location / {
        # Proxy all requests to the Nginx container
        # The Nginx container will serve static files OR proxy /api/* to the backend
        proxy_pass http://frontend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # NO NEED for a separate /api/ proxy here
    # Because Nginx container (frontend:3000) already handles /api/* and proxies to the backend
    # Nginx VM only needs to forward ALL requests to the frontend container

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Logging
    access_log /var/log/nginx/mini-asm-access.log;
    error_log /var/log/nginx/mini-asm-error.log;
}
```

### Activating the Site

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/mini-asm /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm /etc/nginx/sites-enabled/default

# Test config
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

## Step 4: Configuring SSL with Certbot

### Obtaining the SSL Certificate

```bash
# Run Certbot
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# Follow the instructions:
# 1. Enter email
# 2. Agree to terms of service
# 3. Choose redirect HTTP to HTTPS (option 2)
```

Certbot will automatically:

- Fetch the SSL certificate from Let's Encrypt
- Update Nginx config
- Set up auto-renewal

### Verifying SSL

```bash
# Check certificates
sudo certbot certificates

# Test renewal
sudo certbot renew --dry-run
```

### Auto-renewal

Certbot automatically creates a cron job or systemd timer. Verify:

```bash
# Systemd timer
sudo systemctl status certbot.timer

# Or cron
sudo crontab -l
```

## Step 5: Updating Frontend Configuration

Update the frontend to use `/api` instead of the direct URL:

```bash
# File: frontend/src/services/api.js
# Ensure API calls use the /api prefix:
# const API_BASE = '/api'
```

Rebuild frontend if necessary:

```bash
cd /opt/cmc-intern-program/app/session7-deployment
docker-compose build frontend
docker-compose up -d frontend
```

## Step 6: Testing

### Test Locally

```bash
# Health check
curl http://localhost:8080/health

# Frontend
curl http://localhost:3000
```

### Test via Nginx

```bash
# HTTP (before SSL)
curl http://your-domain.com
curl http://your-domain.com/api/health

# HTTPS (after SSL)
curl https://your-domain.com
curl https://your-domain.com/api/health
```

### Test on Browser

1. Open `https://your-domain.com`
2. Check the SSL certificate (padlock icon in the address bar)
3. Test application functionalities

## Step 7: Monitoring & Logs

### Docker Logs

```bash
# View all logs
cd /opt/cmc-intern-program/app/session7-deployment
docker-compose logs -f

# Log specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f db
```

### Nginx Logs

```bash
# Access logs
sudo tail -f /var/log/nginx/mini-asm-access.log

# Error logs
sudo tail -f /var/log/nginx/mini-asm-error.log
```

### System Resources

```bash
# Container stats
docker stats

# Disk usage
docker system df
df -h
```

## Step 8: Backup & Maintenance

### Database Backup

```bash
# Create backup directory
mkdir -p /opt/backups/mini-asm

# Backup database
docker-compose exec -T db pg_dump -U postgres mini_asm > /opt/backups/mini-asm/backup_$(date +%Y%m%d_%H%M%S).sql

# Or use an automated script
cat > /opt/backup-db.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/backups/mini-asm"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cd /opt/cmc-intern-program/app/session7-deployment
docker-compose exec -T db pg_dump -U postgres mini_asm > "$BACKUP_DIR/backup_$TIMESTAMP.sql"
# Keep only the last 7 days
find "$BACKUP_DIR" -name "backup_*.sql" -mtime +7 -delete
EOF

chmod +x /opt/backup-db.sh

# Add to crontab (daily at 2 AM)
echo "0 2 * * * /opt/backup-db.sh" | sudo crontab -
```

### Update Application

```bash
cd /opt/cmc-intern-program/app/session7-deployment

# Pull latest code
git pull origin main

# Rebuild and restart
docker-compose build
docker-compose up -d

# Verify
docker-compose ps
```

### SSL Certificate Renewal

Certbot automatically renews certificates, but you can trigger it manually:

```bash
sudo certbot renew
sudo systemctl reload nginx
```

## Troubleshooting

### Error: "502 Bad Gateway"

```bash
# Check if containers are running
docker-compose ps

# Check logs
docker-compose logs backend
docker-compose logs frontend

# Restart containers
docker-compose restart
```

### Error: "Connection Refused"

```bash
# Check listening ports
sudo netstat -tulpn | grep -E ':3000|:8080'

# Check firewall
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### SSL Certificate Errors

```bash
# Check certificate status
sudo certbot certificates

# Force manual renewal
sudo certbot renew --force-renewal

# Restart Nginx
sudo systemctl restart nginx
```

### Database Connection Issues

```bash
# Verify database responsiveness
docker-compose exec db psql -U postgres -d mini_asm -c "SELECT 1;"

# Restart database container
docker-compose restart db

# Check database logs
docker-compose logs db
```

## Security Checklist

- [ ] Change default passwords
- [ ] Set up firewall (UFW)
- [ ] Enable Fail2ban
- [ ] Regular security updates
- [ ] Regular database backups
- [ ] Monitor logs
- [ ] Limit SSH access
- [ ] Use secure SSL settings

## Firewall Setup

```bash
# Install UFW
sudo apt install ufw -y

# Allow SSH (IMPORTANT!)
sudo ufw allow ssh

# Allow HTTP & HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

## Performance Tuning

### Nginx

```bash
# Edit nginx.conf
sudo nano /etc/nginx/nginx.conf

# Increase worker_connections
events {
    worker_connections 2048;
}

# Enable caching
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;
}
```

### Docker Resources

```bash
# Limit resources in docker-compose.yml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
```

## Useful Commands

```bash
# Restart all services
docker-compose restart

# Stop all services
docker-compose down

# Start with build
docker-compose up -d --build

# View container IPs
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mini-asm-backend

# Clean up old images
docker system prune -a

# Nginx reload
sudo systemctl reload nginx

# Nginx restart
sudo systemctl restart nginx
```

## Support

If you encounter issues:

1. Check logs: `docker-compose logs` and `/var/log/nginx/`
2. Verify config: `sudo nginx -t`
3. Check ports: `sudo netstat -tulpn`
4. Review firewall: `sudo ufw status`

---

**Note**: Replace `your-domain.com` with your actual domain in all configurations!

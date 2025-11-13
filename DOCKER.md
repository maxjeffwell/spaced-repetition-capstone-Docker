# Docker Deployment Guide

This guide covers deploying the Spaced Repetition Capstone application using Docker and Docker Compose.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Development](#development)
- [Production](#production)
- [Docker Images](#docker-images)
- [Troubleshooting](#troubleshooting)
- [Database Management](#database-management)

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- MongoDB Atlas account (or external MongoDB instance)
- 4GB+ RAM recommended
- Ports available: 80, 8080, 5000

## Quick Start

### 1. Initial Setup

```bash
# Clone the repository
git clone <repository-url>
cd spaced-repetition-capstone

# Copy environment template
cp .env.example .env
```

### 2. Configure Environment

Edit `.env` and set your MongoDB Atlas connection string and JWT secret:

```bash
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/spaced_repetition?appName=spaced-repetition
JWT_SECRET=your-secure-jwt-secret-here
```

### 3. Start Services

```bash
# Pull and start services
docker-compose up -d

# View logs
docker-compose logs -f

# Check status
docker-compose ps
```

### 4. Access Application

- **Application**: http://localhost
- **API**: http://localhost/api
- **Health Check**: http://localhost/health
- **Server Direct**: http://localhost:8080/api
- **Client Direct**: http://localhost:5000

## Configuration

### Environment Variables

#### Required

- `MONGODB_URI` - MongoDB Atlas connection string
- `JWT_SECRET` - Secret key for JWT signing

#### Optional

- `NODE_ENV` - Environment (development/production) [default: development]
- `JWT_EXPIRY` - JWT expiration time [default: 7d]
- `CLIENT_ORIGIN` - Allowed CORS origins [default: http://localhost:80,http://localhost:3000]
- `REACT_APP_API_BASE_URL` - API endpoint for client [default: http://localhost/api]
- `VERSION` - Docker image version tag [default: latest]

### MongoDB Atlas Setup

1. **Create Account**: Sign up at https://www.mongodb.com/cloud/atlas
2. **Create Cluster**: Free tier (M0) works great for development
3. **Database Access**: Create database user with read/write permissions
4. **Network Access**: Add your IP or allow from anywhere (0.0.0.0/0) for development
5. **Get Connection String**: Click "Connect" → "Connect your application" → Copy connection string
6. **Update .env**: Replace `<username>`, `<password>`, and `<database>` in connection string

## Development

### Local Builds

By default, `docker-compose.override.yml` builds images locally instead of pulling from Docker Hub.

```bash
# Build and start
docker-compose up --build -d

# Rebuild specific service
docker-compose build server
docker-compose up -d server

# Force rebuild everything
docker-compose build --no-cache
docker-compose up -d
```

### Development Mode with Hot Reloading

For active development with hot reloading:

```bash
# Server (nodemon)
cd spaced-repetition-capstone-server
docker build --target development -t spaced-repetition-server:dev .
docker run -p 8080:8080 -v $(pwd):/app -e MONGODB_URI="your-connection-string" spaced-repetition-server:dev

# Client (React dev server)
cd spaced-repetition-capstone-client
docker build --target development -t spaced-repetition-client:dev .
docker run -p 3000:3000 -v $(pwd)/src:/app/src spaced-repetition-client:dev
```

### Viewing Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f server
docker-compose logs -f client
docker-compose logs -f nginx

# Last 100 lines
docker-compose logs --tail=100

# Since specific time
docker-compose logs --since 2024-01-01T10:00:00
```

### Shell Access

```bash
# Server container
docker exec -it spaced-repetition-server sh

# Client container
docker exec -it spaced-repetition-client sh

# Nginx container
docker exec -it spaced-repetition-nginx sh
```

## Production

### Production Configuration

Create `.env.production`:

```bash
NODE_ENV=production
MONGODB_URI=mongodb+srv://prod_user:prod_pass@cluster.mongodb.net/spaced_repetition_prod?appName=spaced-repetition
JWT_SECRET=<strong-random-secret-min-32-chars>
JWT_EXPIRY=7d
CLIENT_ORIGIN=https://yourdomain.com
REACT_APP_API_BASE_URL=https://yourdomain.com/api
VERSION=v1.0.0
```

### Deploy Production

```bash
# Use production compose file
docker-compose -f docker-compose.prod.yml up -d

# With specific version
VERSION=v1.0.0 docker-compose -f docker-compose.prod.yml up -d

# View production logs
docker-compose -f docker-compose.prod.yml logs -f
```

### Production Features

- **Resource Limits**: CPU and memory limits enforced
- **Security Hardening**:
  - Read-only root filesystems
  - Non-root users
  - No new privileges
  - Minimal capabilities
- **Log Rotation**: Max 10MB per file, 3 files retained
- **Health Checks**: Automatic restart on failure
- **Restart Policy**: `unless-stopped`

### Updating Production

```bash
# Pull latest images
docker-compose -f docker-compose.prod.yml pull

# Restart with new images (zero-downtime)
docker-compose -f docker-compose.prod.yml up -d

# Or with specific version
VERSION=v1.0.1 docker-compose -f docker-compose.prod.yml up -d --no-deps server
```

## Docker Images

### Pre-built Images

Multi-platform images (linux/amd64, linux/arm64) available on Docker Hub:

- `maxjeffwell/spaced-repetition-capstone-server:latest`
- `maxjeffwell/spaced-repetition-capstone-client:latest`

### Building Images Manually

```bash
# Server
cd spaced-repetition-capstone-server
docker build -t maxjeffwell/spaced-repetition-capstone-server:latest .

# Client
cd spaced-repetition-capstone-client
docker build --build-arg REACT_APP_API_BASE_URL=/api \
  -t maxjeffwell/spaced-repetition-capstone-client:latest .
```

### Multi-platform Builds

```bash
# Setup buildx
docker buildx create --name multiplatform --use
docker buildx inspect --bootstrap

# Build and push
cd spaced-repetition-capstone-server
docker buildx build --platform linux/amd64,linux/arm64 \
  -t maxjeffwell/spaced-repetition-capstone-server:latest --push .

cd ../spaced-repetition-capstone-client
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg REACT_APP_API_BASE_URL=/api \
  -t maxjeffwell/spaced-repetition-capstone-client:latest --push .
```

## Troubleshooting

### Common Issues

#### Port Already in Use

```bash
# Check what's using the port
sudo lsof -i :80
sudo lsof -i :8080
sudo lsof -i :5000

# Kill process
sudo kill -9 <PID>

# Or change ports in docker-compose.yml
```

#### MongoDB Connection Failed

```bash
# Verify connection string
docker exec -it spaced-repetition-server sh
echo $MONGODB_URI

# Test connection
npm install -g mongosh
mongosh "$MONGODB_URI"

# Check MongoDB Atlas Network Access settings
```

#### Server Health Check Failing

```bash
# Check server logs
docker-compose logs server

# Manual health check
curl http://localhost:8080/api

# Enter container and debug
docker exec -it spaced-repetition-server sh
node -e "require('http').get('http://localhost:8080/api', (r) => console.log(r.statusCode))"
```

#### Client Not Loading

```bash
# Check client logs
docker-compose logs client

# Verify build
docker exec -it spaced-repetition-client sh
ls -la /usr/share/nginx/html

# Check nginx logs
docker-compose logs nginx
```

#### ML Models Not Loading

```bash
# Verify models in client
docker exec -it spaced-repetition-client sh
ls -la /usr/share/nginx/html/models/

# Check browser console for errors
# Models should be in: public/models/
```

### Container Management

```bash
# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Remove everything including images
docker-compose down --rmi all -v

# Restart specific service
docker-compose restart server

# Rebuild and restart
docker-compose up -d --build --force-recreate server
```

### Performance Issues

```bash
# Check resource usage
docker stats

# Increase memory limits in docker-compose.prod.yml
deploy:
  resources:
    limits:
      memory: 1G  # Increase this

# Clear Docker cache
docker system prune -a
docker builder prune
```

## Database Management

### MongoDB Atlas Best Practices

1. **Use Connection Pooling**: Default pool settings in config are optimized
2. **Enable Backups**: Configure automated backups in Atlas
3. **Monitor Performance**: Use Atlas monitoring dashboard
4. **Index Optimization**: Ensure proper indexes on User collection
5. **Separate Environments**: Use different clusters for dev/staging/prod

### Backup and Restore

Since using MongoDB Atlas:

```bash
# Atlas handles backups automatically
# To restore, use Atlas UI: Clusters → Backup → Restore

# For manual backup:
mongodump --uri="$MONGODB_URI" --out=/backup

# For manual restore:
mongorestore --uri="$MONGODB_URI" /backup
```

## Security Considerations

1. **Never commit `.env` files**: Always in .gitignore
2. **Use strong JWT secrets**: Minimum 32 characters, random
3. **Rotate secrets regularly**: Update JWT_SECRET periodically
4. **MongoDB Atlas Security**:
   - Use strong database passwords
   - Restrict network access
   - Enable audit logs
5. **Update images regularly**: Pull latest security patches

## Next Steps

- [Kubernetes Deployment](KUBERNETES.md) - Deploy to Kubernetes
- [CI/CD Setup](CICD.md) - Automated builds and deployments
- [README](README.md) - Project overview

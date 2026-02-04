# Strapi Setup Guide

## Overview
Strapi is a leading open-source headless CMS that's 100% JavaScript, fully customizable, and developer-first. This guide will help you deploy Strapi using the provided YML configuration.

**Deployment Method**: This configuration uses the official Node.js Docker image and automatically installs Strapi on first run in production mode. This approach ensures you always get a working installation without relying on unofficial Docker images.

## Basic Information
- **Default Port**: 1337
- **Docker Image**: node:20-alpine (Official Node.js image)
- **Node.js Version**: 20.x (required by Strapi)
- **Data Directory**: /opt/appdata/strapi
- **Default Database**: SQLite (for simplicity)
- **Deployment Method**: Strapi is installed on first run using the official create-strapi-app
- **Note**: Since Strapi doesn't provide pre-built Docker images, this configuration auto-installs Strapi in the Node.js container

## Prerequisites
- Docker installed and running
- PlexGuide or compatible Ansible setup
- Domain configured for Traefik reverse proxy

## Installation Steps

1. **Deploy Strapi**
   ```bash
   ansible-playbook /opt/communityapps/apps/strapi.yml
   ```

2. **Wait for Initial Setup**
   - First deployment will take 5-10 minutes as Strapi is installed and built
   - Check logs: `docker logs -f strapi`
   - Wait for message: "Project created successfully!" then "Server started"
   - Production build takes longer than development mode

3. **Access Strapi**
   - Navigate to: `https://strapi.yourdomain.com/admin`
   - First-time setup will prompt you to create an admin account

## Environment Variables

### Required Security Secrets
⚠️ **IMPORTANT**: You must change these default values before production use!

The following secrets should be generated as random strings:
- `APP_KEYS`: Comma-separated random strings (minimum 2)
- `API_TOKEN_SALT`: Random string
- `ADMIN_JWT_SECRET`: Random string
- `TRANSFER_TOKEN_SALT`: Random string
- `JWT_SECRET`: Random string

### How to Generate Secure Secrets

**Using OpenSSL (Linux/Mac):**
```bash
openssl rand -base64 32
```

**Using Node.js:**
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Generate 5 different random strings and update the `pg_env` section in the YML file.

### Current Configuration

```yaml
pg_env:
  PUID: '1000'                          # User ID for file permissions
  PGID: '1000'                          # Group ID for file permissions
  TZ: 'America/New_York'                # Timezone (change as needed)
  APP_KEYS: 'toBeModified1,toBeModified2'  # ⚠️ CHANGE THIS
  API_TOKEN_SALT: 'toBeModified'        # ⚠️ CHANGE THIS
  ADMIN_JWT_SECRET: 'toBeModified'      # ⚠️ CHANGE THIS
  TRANSFER_TOKEN_SALT: 'toBeModified'   # ⚠️ CHANGE THIS
  JWT_SECRET: 'toBeModified'            # ⚠️ CHANGE THIS
  NODE_ENV: 'production'                # Environment mode
  DATABASE_CLIENT: 'sqlite'             # Database type (sqlite/mysql/postgres)
  DATABASE_FILENAME: '/opt/app/data.db' # SQLite database location
  HOST: '0.0.0.0'                       # Listen on all interfaces
  PORT: '1337'                          # Strapi port
```

## Database Configuration

### Using SQLite (Default)
The default configuration uses SQLite, which is suitable for:
- Development environments
- Small to medium websites
- Single-server deployments

No additional configuration needed!

### Using MySQL/MariaDB
To use MySQL/MariaDB instead, modify the `pg_env` section:

```yaml
pg_env:
  # ... other variables ...
  DATABASE_CLIENT: 'mysql'
  DATABASE_HOST: 'mysql_host'
  DATABASE_PORT: '3306'
  DATABASE_NAME: 'strapi'
  DATABASE_USERNAME: 'strapi_user'
  DATABASE_PASSWORD: 'secure_password'
  DATABASE_SSL: 'false'
```

Remove the `DATABASE_FILENAME` variable when using MySQL.

### Using PostgreSQL
To use PostgreSQL, modify the `pg_env` section:

```yaml
pg_env:
  # ... other variables ...
  DATABASE_CLIENT: 'postgres'
  DATABASE_HOST: 'postgres_host'
  DATABASE_PORT: '5432'
  DATABASE_NAME: 'strapi'
  DATABASE_USERNAME: 'strapi_user'
  DATABASE_PASSWORD: 'secure_password'
  DATABASE_SSL: 'false'
```

Remove the `DATABASE_FILENAME` variable when using PostgreSQL.

## Volume Mounts

The configuration mounts the following directories:

```yaml
pg_volumes:
  - '/etc/localtime:/etc/localtime:ro'      # System time (read-only)
  - '/opt/appdata/strapi:/opt/app'          # Strapi data directory
  - '{{path.stdout}}:{{path.stdout}}'       # PlexGuide path
```

### Data Persistence
All Strapi data (uploads, database, configuration) is stored in:
```
/opt/appdata/strapi/
```

### Backup Recommendations
Regularly backup the following:
- `/opt/appdata/strapi/data.db` - SQLite database (if using SQLite)
- `/opt/appdata/strapi/public/uploads/` - Uploaded media files
- `/opt/appdata/strapi/.env` - Environment configuration

## Post-Installation

### First-Time Setup
1. Access Strapi at `https://strapi.yourdomain.com/admin`
2. Create your admin account
3. Configure your content types
4. Set up API permissions

### Security Recommendations
1. ✅ Change all default secrets in the YML file
2. ✅ Use strong passwords for admin accounts
3. ✅ Enable HTTPS (handled by Traefik)
4. ✅ Regularly update Strapi to latest version
5. ✅ Configure API permissions restrictively
6. ✅ Use external database (MySQL/PostgreSQL) for production

### Performance Optimization
1. Use PostgreSQL or MySQL for better performance
2. Enable caching in Strapi settings
3. Configure CDN for media files
4. Use external storage providers (AWS S3, Cloudinary, etc.)

## Customization

### Changing Port
To use a different external port, modify:
```yaml
extport: '1337'  # Change to your desired port
```

### Changing Timezone
Update the TZ variable to your timezone:
```yaml
TZ: 'Europe/London'  # or 'Asia/Tokyo', 'Australia/Sydney', etc.
```

### Using Different Node.js Version
To use a specific Node.js version:
```yaml
image: 'node:20-alpine'  # Use Node.js 20 (current default, required)
image: 'node:22-alpine'  # Use Node.js 22 (also supported)
```

**Important**: Strapi requires Node.js >=20.0.0 <=24.x.x. Do not use Node.js 18 or older.

**Note**: Strapi is automatically installed on first run. The installation persists in `/opt/appdata/strapi`, so subsequent restarts are fast.

## Troubleshooting

### Initial Installation Taking Long

The first deployment installs and builds Strapi from scratch, which can take 5-10 minutes:

1. **Monitor progress:**
   ```bash
   docker logs -f strapi
   ```

2. **What you'll see:**
   - Installing dependencies
   - Creating Strapi project
   - Building admin panel (takes longest)
   - "Project created successfully!" when done
   - "Server started" when ready

3. **If stuck for more than 15 minutes:**
   ```bash
   docker stop strapi
   docker rm strapi
   # Clean data directory
   sudo rm -rf /opt/appdata/strapi/*
   # Redeploy
   ansible-playbook /opt/communityapps/apps/strapi.yml
   ```

### Container Won't Start
1. Check Docker logs:
   ```bash
   docker logs strapi
   ```

2. Verify file permissions:
   ```bash
   sudo chown -R 1000:1000 /opt/appdata/strapi
   ```

3. Ensure all required secrets are set

### Cannot Access Admin Panel
1. Verify Traefik is running
2. Check domain DNS configuration
3. Verify port mapping: `docker ps | grep strapi`
4. Check Strapi logs for errors

### Database Connection Issues
1. Verify database credentials
2. Ensure database container is running
3. Check network connectivity between containers

### Permission Denied Errors
```bash
sudo chown -R 1000:1000 /opt/appdata/strapi
sudo chmod -R 755 /opt/appdata/strapi
```

## Additional Resources

- **Official Documentation**: https://docs.strapi.io
- **Docker Installation Guide**: https://docs.strapi.io/cms/installation/docker
- **Community Forum**: https://forum.strapi.io
- **GitHub Repository**: https://github.com/strapi/strapi
- **Discord Community**: https://discord.strapi.io

## Building a Custom Docker Image (Advanced)

For production environments, you may want to build your own custom Strapi Docker image instead of using the community image.

### Create a Dockerfile

Create a `Dockerfile` in your Strapi project directory:

```dockerfile
FROM node:20-alpine
RUN apk update && apk add --no-cache build-base gcc autoconf automake zlib-dev libpng-dev vips-dev git
ENV NODE_ENV=production
WORKDIR /opt/app
COPY package*.json ./
RUN npm install --production
COPY . .
RUN npm run build
EXPOSE 1337
CMD ["npm", "start"]
```

### Build and Push

```bash
docker build -t your-registry/strapi:latest .
docker push your-registry/strapi:latest
```

### Update YML Configuration

Update the `image` field in [strapi.yml](strapi.yml):
```yaml
image: 'your-registry/strapi:latest'
```

## Upgrading Strapi

To upgrade to a newer version:

1. Stop the container
2. Backup your data
3. Update the image version in the YML file
4. Redeploy using the playbook
5. Check logs for migration messages

```bash
docker stop strapi
# Backup /opt/appdata/strapi
ansible-playbook /opt/communityapps/apps/strapi.yml
docker logs -f strapi
```

## Support

For issues specific to this deployment configuration, please refer to the Apps-Community repository. For Strapi-specific issues, consult the official Strapi documentation.

---
**Note**: This is a community-maintained configuration. Always review and adjust security settings based on your specific requirements.

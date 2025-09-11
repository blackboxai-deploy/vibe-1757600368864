# Deployment Guide

This guide covers different deployment options for the Audio Video Separator application.

## Vercel (Recommended)

Vercel provides the easiest deployment option with automatic builds and previews.

### Prerequisites

- GitHub repository with your code
- Vercel account ([vercel.com](https://vercel.com))

### Steps

1. **Connect Repository**
   ```bash
   # Push your code to GitHub
   git add .
   git commit -m "Initial commit"
   git push origin main
   ```

2. **Import Project**
   - Visit [vercel.com](https://vercel.com)
   - Click "Import Project"
   - Select your GitHub repository
   - Configure project settings

3. **Environment Variables**
   Set the following environment variables in Vercel:
   ```
   NEXT_PUBLIC_APP_URL=https://your-domain.vercel.app
   MAX_FILE_SIZE=104857600
   PROCESSING_TIMEOUT=300000
   ```

4. **Deploy**
   - Vercel automatically builds and deploys
   - Get your live URL
   - Set up custom domain (optional)

### Automatic Deployments

Vercel automatically deploys when you push to your main branch:
- **Production** - Deploys from `main` branch
- **Preview** - Deploys from feature branches and PRs

## Docker Deployment

Deploy using Docker for containerized environments.

### Dockerfile

```dockerfile
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY pnpm-lock.yaml ./

# Install pnpm
RUN npm install -g pnpm

# Install dependencies
RUN pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build application
RUN pnpm build

# Production stage
FROM node:18-alpine AS runner

WORKDIR /app

# Create app user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

# Set permissions
USER nextjs

# Expose port
EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# Start application
CMD ["node", "server.js"]
```

### Docker Compose

```yaml
version: '3.8'

services:
  audio-separator:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MAX_FILE_SIZE=104857600
      - PROCESSING_TIMEOUT=300000
    volumes:
      - ./uploads:/app/uploads
      - ./outputs:/app/outputs
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - audio-separator
    restart: unless-stopped
```

### Build and Run

```bash
# Build image
docker build -t audio-separator .

# Run container
docker run -p 3000:3000 \
  -e NODE_ENV=production \
  -e MAX_FILE_SIZE=104857600 \
  audio-separator

# Or use docker-compose
docker-compose up -d
```

## Traditional VPS/Server

Deploy to a traditional VPS or dedicated server.

### Prerequisites

- Ubuntu 20.04+ or similar Linux distribution
- Node.js 18+
- Nginx (recommended)
- SSL certificate (Let's Encrypt recommended)

### Installation Steps

1. **Prepare Server**
   ```bash
   # Update system
   sudo apt update && sudo apt upgrade -y
   
   # Install Node.js
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt-get install -y nodejs
   
   # Install pnpm
   npm install -g pnpm
   
   # Install PM2 for process management
   npm install -g pm2
   ```

2. **Clone and Build**
   ```bash
   # Clone repository
   git clone https://github.com/yourusername/audio-video-separator.git
   cd audio-video-separator
   
   # Install dependencies
   pnpm install --frozen-lockfile
   
   # Build application
   pnpm build
   ```

3. **Configure PM2**
   ```bash
   # Create PM2 ecosystem file
   cat > ecosystem.config.js << EOF
   module.exports = {
     apps: [{
       name: 'audio-separator',
       script: 'npm',
       args: 'start',
       cwd: '/path/to/audio-video-separator',
       instances: 'max',
       exec_mode: 'cluster',
       env: {
         NODE_ENV: 'production',
         PORT: 3000,
         MAX_FILE_SIZE: '104857600'
       }
     }]
   }
   EOF
   
   # Start application
   pm2 start ecosystem.config.js
   
   # Save PM2 configuration
   pm2 save
   pm2 startup
   ```

4. **Configure Nginx**
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;
       return 301 https://$server_name$request_uri;
   }
   
   server {
       listen 443 ssl http2;
       server_name your-domain.com;
   
       ssl_certificate /path/to/ssl/cert.pem;
       ssl_certificate_key /path/to/ssl/key.pem;
   
       client_max_body_size 100M;
   
       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_cache_bypass $http_upgrade;
           proxy_read_timeout 300s;
           proxy_send_timeout 300s;
       }
       
       location /uploads/ {
           alias /path/to/audio-video-separator/uploads/;
           expires 1d;
           add_header Cache-Control "public, immutable";
       }
   }
   ```

5. **SSL Certificate**
   ```bash
   # Install Certbot
   sudo apt install certbot python3-certbot-nginx
   
   # Get SSL certificate
   sudo certbot --nginx -d your-domain.com
   ```

## AWS Deployment

Deploy on AWS using various services.

### AWS Amplify

1. **Connect Repository**
   - Go to AWS Amplify Console
   - Connect your GitHub repository
   - Configure build settings

2. **Build Configuration**
   ```yaml
   version: 1
   applications:
     - frontend:
         phases:
           preBuild:
             commands:
               - npm install -g pnpm
               - pnpm install --frozen-lockfile
           build:
             commands:
               - pnpm build
         artifacts:
           baseDirectory: .next
           files:
             - '**/*'
         cache:
           paths:
             - node_modules/**/*
   ```

### AWS ECS (Container Service)

1. **Create Task Definition**
2. **Set up ECS Service**
3. **Configure Load Balancer**
4. **Set up Auto Scaling**

## Environment Configuration

### Production Environment Variables

```bash
# Application
NODE_ENV=production
NEXT_PUBLIC_APP_URL=https://your-domain.com
PORT=3000

# File Processing
MAX_FILE_SIZE=104857600
UPLOAD_DIR=/app/uploads
OUTPUT_DIR=/app/outputs
PROCESSING_TIMEOUT=300000

# Security
JWT_SECRET=your-secure-jwt-secret
API_RATE_LIMIT=100

# Optional: Database
DATABASE_URL=postgresql://user:pass@localhost:5432/audioapp
REDIS_URL=redis://localhost:6379

# Optional: Cloud Storage
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
S3_BUCKET_NAME=your-bucket
```

## Monitoring and Logging

### Health Checks

Create health check endpoints:

```javascript
// pages/api/health.js
export default function handler(req, res) {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version
  })
}
```

### Logging

Configure structured logging:

```javascript
// lib/logger.js
import winston from 'winston'

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
})

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }))
}

export default logger
```

## Performance Optimization

### CDN Configuration

Set up CloudFlare or AWS CloudFront for:
- Static asset caching
- Global content delivery
- DDoS protection
- SSL termination

### Caching Strategy

```javascript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/api/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'no-store, no-cache, must-revalidate'
          }
        ]
      },
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Cache-Control', 
            value: 'public, max-age=31536000, immutable'
          }
        ]
      }
    ]
  }
}
```

## Backup and Recovery

### Database Backups

```bash
# PostgreSQL backup
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d_%H%M%S).sql

# Automated daily backups
0 2 * * * /path/to/backup-script.sh
```

### File Backups

```bash
# Sync uploads to S3
aws s3 sync /app/uploads s3://your-backup-bucket/uploads/

# Automated backup script
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/uploads.tar.gz /app/uploads
tar -czf $BACKUP_DIR/outputs.tar.gz /app/outputs
```

## Troubleshooting

### Common Issues

1. **File Upload Timeouts**
   - Increase `client_max_body_size` in Nginx
   - Set `PROCESSING_TIMEOUT` environment variable
   - Configure proxy timeouts

2. **Memory Issues**
   - Increase Node.js memory limit: `--max-old-space-size=4096`
   - Monitor memory usage with PM2
   - Implement file cleanup

3. **SSL Certificate Issues**
   - Verify certificate installation
   - Check certificate expiration
   - Test SSL configuration

### Monitoring

```bash
# Check application status
pm2 status

# View logs
pm2 logs audio-separator

# Monitor resources
pm2 monit

# Restart application
pm2 restart audio-separator
```

## Security Considerations

### Firewall Configuration

```bash
# UFW firewall rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### Regular Updates

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Node.js dependencies
pnpm audit
pnpm update

# Update PM2
pm2 update
```

For additional deployment questions, please refer to our [GitHub Discussions](https://github.com/yourusername/audio-video-separator/discussions) or create an issue.
I understand! Instead of adding it to the repository, I'll provide you with the complete markdown content that you can save and send to yourself or share via email.

Here's the complete **SALI IDP Deployment and Integration Guide** in markdown format:

```markdown
# SALI IDP: Deployment and Integration Guide

## Table of Contents
1. [EC2 Deployment Guide](#ec2-deployment-guide)
2. [Application Integration Guide](#application-integration-guide)

---

# EC2 Deployment Guide for SALI IDP

This is a **NestJS-based Identity Provider (IDP)** application that provides OAuth2/OIDC authentication, passwordless OTP, WebAuthn, and multi-factor authentication. Here's how to deploy it on AWS EC2:

## Pre-Deployment Setup

### 1. Prepare AWS Resources

```bash
# Create EC2 instance (Ubuntu 22.04 LTS recommended)
# - Instance type: t3.medium or larger (2GB+ RAM needed)
# - Security group: Allow ports 22 (SSH), 80, 443, 5432 (optional if RDS used)
# - Storage: 30GB+ EBS volume

# Create RDS PostgreSQL 16 instance (or use EC2-hosted Postgres)
# Create ElastiCache Redis cluster (or use EC2-hosted Redis)

# Create environment file with secrets
# Store in AWS Secrets Manager or Parameter Store
```

### 2. EC2 Instance Setup

```bash
# SSH into your EC2 instance
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Node.js 22 (LTS)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install Docker and Docker Compose (optional, for containerized approach)
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu

# Install PM2 (process manager)
sudo npm install -g pm2
```

---

## Deployment Method 1: Direct Node.js (Recommended for EC2)

### 1. Clone & Setup Application

```bash
# Clone repository
cd /opt
sudo git clone https://github.com/AbsoluteRisktec/sali-idp.git
cd sali-idp

# Create .env file from example
sudo cp .env.example .env

# Edit configuration for production
sudo nano .env
```

### 2. Configure Environment Variables

```bash
# Critical variables to update in .env:
APP_NAME=SALI-ID API
NODE_ENV=production
PORT=3100
CLIENT_URL=https://your-domain.com

# Database - Point to RDS/external PostgreSQL
DB_HOST=your-rds-endpoint.rds.amazonaws.com
DB_PORT=5432
DB_USERNAME=<your-db-user>
DB_PASSWORD=<your-db-password>
DB_NAME=sali_idp_prod

# Redis - Point to ElastiCache or EC2-hosted Redis
REDIS_HOST=your-redis-endpoint.elasticache.amazonaws.com
REDIS_PORT=6379
REDIS_PASSWORD=<your-redis-password>

# Email Configuration (SMTP)
MAIL_HOST=smtp.sendgrid.net  # or your email provider
MAIL_PORT=587
MAIL_USER=apikey
MAIL_PASSWORD=<your-sendgrid-api-key>
MAIL_FROM="SALI-ID <no-reply@your-domain.com>"

# JWT Secrets (generate with: openssl rand -hex 32)
JWT_ACCESS_SECRET=<64-char-hex-key>
JWT_REFRESH_SECRET=<64-char-hex-key>
JWT_ISSUER=https://id.your-domain.com
JWT_AUDIENCE=sali-client
SIGNING_ALG=RS256

# Encryption key for credentials (generate with: openssl rand -hex 32)
CREDENTIAL_ENCRYPTION_KEY=<64-char-hex-key>

# WebAuthn configuration
WEBAUTHN_RP_ID=id.your-domain.com
WEBAUTHN_RP_NAME=SALI
WEBAUTHN_ORIGIN=https://your-domain.com

# Optional: Internal API token for /orgs management
INTERNAL_API_TOKEN=<64-char-hex-key>
```

### 3. Install Dependencies & Build

```bash
# Install production dependencies
npm ci

# Build the application
npm run build

# Verify build
ls -la dist/
```

### 4. Run Database Migrations

```bash
# Run TypeORM migrations
npm run migration:run

# Verify migrations applied
npm run migration:show
```

### 5. Setup PM2 (Process Manager)

```bash
# Create PM2 ecosystem file
sudo nano ecosystem.config.js
```

```javascript
module.exports = {
  apps: [
    {
      name: 'sali-idp',
      script: 'dist/main.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
      },
      error_file: './logs/error.log',
      out_file: './logs/out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true,
      watch: false,
      ignore_watch: ['node_modules', 'dist'],
      max_memory_restart: '1G',
      autorestart: true,
      max_restarts: 10,
      min_uptime: '10s',
    },
  ],
};
```

```bash
# Create logs directory
mkdir -p logs

# Start application with PM2
sudo pm2 start ecosystem.config.js

# Setup auto-restart on system boot
sudo pm2 startup
sudo pm2 save

# Monitor logs
sudo pm2 logs sali-idp

# Check status
sudo pm2 status
```

---

## Deployment Method 2: Docker on EC2 (Container-based)

### 1. Build Docker Image

```bash
# Clone repository
cd /opt
git clone https://github.com/AbsoluteRisktec/sali-idp.git
cd sali-idp

# Create .env for Docker
cp .env.example .env
# Edit .env with production values (same as above)

# Build image
docker build -t sali-idp:latest .

# Tag for ECR (optional)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
docker tag sali-idp:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/sali-idp:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/sali-idp:latest
```

### 2. Run with Docker Compose

```bash
# Create production docker-compose file
nano docker-compose.prod.yml
```

```yaml
version: '3.8'
services:
  sali-idp:
    image: sali-idp:latest
    container_name: sali-idp-prod
    restart: always
    ports:
      - "3100:3100"
    environment:
      NODE_ENV: production
      PORT: 3100
      DB_HOST: your-rds-endpoint.rds.amazonaws.com
      DB_PORT: 5432
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: sali_idp_prod
      REDIS_HOST: your-redis-endpoint
      REDIS_PORT: 6379
      # Add other env variables
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3100/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - sali-network
    logging:
      driver: awslogs
      options:
        awslogs-group: /ecs/sali-idp
        awslogs-region: us-east-1
        awslogs-stream-prefix: ecs

networks:
  sali-network:
    driver: bridge
```

```bash
# Start services
docker-compose -f docker-compose.prod.yml up -d

# View logs
docker-compose -f docker-compose.prod.yml logs -f sali-idp

# Stop services
docker-compose -f docker-compose.prod.yml down
```

---

## Setup Reverse Proxy (Nginx)

```bash
# Install Nginx
sudo apt install -y nginx

# Create Nginx config
sudo nano /etc/nginx/sites-available/sali-idp
```

```nginx
upstream sali_idp {
    server 127.0.0.1:3100;
}

server {
    listen 80;
    server_name id.your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name id.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/id.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/id.your-domain.com/privkey.pem;
    
    # HSTS header (important for IDP security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    client_max_body_size 10M;

    location / {
        proxy_pass http://sali_idp;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/sali-idp /etc/nginx/sites-enabled/

# Test Nginx config
sudo nginx -t

# Start Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Get SSL certificate with Certbot
sudo apt install -y certbot python3-certbot-nginx
sudo certbot certonly --nginx -d id.your-domain.com
```

---

## Setup SSL/TLS Certificate

```bash
# Using Let's Encrypt (automatic renewal)
sudo certbot certonly --nginx -d id.your-domain.com

# Automatic renewal via cron
sudo certbot renew --dry-run
sudo systemctl enable certbot.timer
```

---

## Database Migration on Production

```bash
# Run migrations before starting application
npm run migration:run

# Verify migrations applied
npm run migration:show

# Seed data (optional)
npm run seed
```

---

## Health Checks & Monitoring

```bash
# Health check endpoint
curl https://id.your-domain.com/health

# Metrics endpoint (Prometheus format)
curl https://id.your-domain.com/metrics

# Setup CloudWatch monitoring
# In AWS Console: Create CloudWatch alarms for:
# - High memory/CPU usage
# - Database connectivity issues
# - HTTP error rates
```

---

## Security Hardening

```bash
# 1. Configure firewall (Security Group in AWS)
# Allow: 22 (SSH), 80 (HTTP), 443 (HTTPS)
# Deny: All other inbound traffic

# 2. Disable SSH password auth
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart ssh

# 3. Setup fail2ban
sudo apt install -y fail2ban
sudo systemctl enable fail2ban

# 4. Store secrets in AWS Secrets Manager
# Retrieve at runtime instead of .env files

# 5. Enable VPC encryption for RDS/Redis
# Use private subnets for databases
```

---

## Backup & Disaster Recovery

```bash
# PostgreSQL automated backups
# In AWS RDS console: Enable automated backups (retention 7+ days)

# Application code backup
cd /opt/sali-idp && tar -czf backup-$(date +%Y%m%d).tar.gz .

# Backup to S3
aws s3 cp backup-*.tar.gz s3://your-backup-bucket/
```

---

## Deployment Checklist

- [ ] EC2 instance created and configured
- [ ] RDS PostgreSQL database created
- [ ] ElastiCache Redis cluster created
- [ ] Security Groups configured (ports 22, 80, 443)
- [ ] `.env` file created with all secrets
- [ ] Dependencies installed (`npm ci`)
- [ ] Application built (`npm run build`)
- [ ] Database migrations run (`npm run migration:run`)
- [ ] Application tested locally
- [ ] PM2/Docker started and verified
- [ ] Nginx reverse proxy configured
- [ ] SSL certificate installed (Let's Encrypt)
- [ ] Health endpoint responding
- [ ] Logs being collected
- [ ] Monitoring/alerts configured
- [ ] Backup strategy implemented

---

## Useful Commands

```bash
# Restart application
sudo pm2 restart sali-idp

# View logs
sudo pm2 logs sali-idp

# Check status
sudo pm2 status

# Stop application
sudo pm2 stop sali-idp

# Rebuild & restart
npm run build && sudo pm2 restart sali-idp
```

---

# Application Integration Guide

## How to Connect Your Applications to SALI IDP

This IDP is an **OAuth2/OIDC-compliant identity provider**. It uses the **Authorization Code Flow with PKCE** for user authentication and supports multiple token grants for service-to-service communication.

---

## Integration Overview

### Three Integration Patterns:

1. **User Login (Authorization Code Flow)** — Web/mobile apps that need users to authenticate
2. **Service-to-Service (Client Credentials)** — Backend services that need machine-to-machine access
3. **Token Exchange** — Microservices that need to delegate user context across services

---

## Pattern 1: User Login Integration (Web/Mobile Apps)

This is the **most common pattern** for connecting web and mobile applications.

### Step 1: Register Your Application as an OAuth Client

First, register your app with the IDP using the **internal `/internal/clients` API** (operator only):

```bash
curl -X POST https://id.your-domain.com/internal/clients \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: <your-INTERNAL_API_TOKEN>" \
  -d '{
    "client_id": "my-web-app",
    "name": "My Web Application",
    "type": "public",
    "redirect_uris": [
      "https://my-app.com/auth/callback",
      "https://my-app.com/admin/callback"
    ],
    "allowed_scopes": ["openid", "profile", "email"],
    "allowed_audiences": ["my-web-app"],
    "allowed_grants": ["authorization_code", "refresh_token"],
    "first_party": false
  }'
```

**Response:**
```json
{
  "client": {
    "clientId": "my-web-app",
    "name": "My Web Application",
    "type": "public",
    "redirectUris": ["https://my-app.com/auth/callback"],
    "allowedScopes": ["openid", "profile", "email"],
    "allowedAudiences": ["my-web-app"],
    "allowedGrants": ["authorization_code", "refresh_token"],
    "firstParty": false
  }
}
```

### Step 2: Implement Login Flow in Your Frontend

```typescript
// React example using the Authorization Code Flow with PKCE

import { useEffect } from 'react';
import { sha256 } from 'js-sha256';

// Generate PKCE values
function generatePKCE() {
  const codeVerifier = generateRandomString(128);
  const codeChallenge = base64URLEncode(
    sha256.digest(codeVerifier)
  );
  return { codeVerifier, codeChallenge };
}

function generateRandomString(length: number): string {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
  let result = '';
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

function base64URLEncode(buffer: Uint8Array): string {
  return btoa(String.fromCharCode(...buffer))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

// Start Login
export function LoginButton() {
  const handleLogin = () => {
    const { codeVerifier, codeChallenge } = generatePKCE();
    const state = generateRandomString(32);
    
    // Store for callback
    sessionStorage.setItem('pkce_verifier', codeVerifier);
    sessionStorage.setItem('oauth_state', state);

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: 'my-web-app',
      redirect_uri: `${window.location.origin}/auth/callback`,
      scope: 'openid profile email offline_access',
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      state: state,
    });

    window.location.href = 
      `https://id.your-domain.com/authorize?${params.toString()}`;
  };

  return <button onClick={handleLogin}>Login with SALI</button>;
}

// Callback Handler
export function AuthCallback() {
  useEffect(() => {
    const handleCallback = async () => {
      const params = new URLSearchParams(window.location.search);
      const code = params.get('code');
      const state = params.get('state');
      const error = params.get('error');

      if (error) {
        console.error('Auth error:', error, params.get('error_description'));
        return;
      }

      if (state !== sessionStorage.getItem('oauth_state')) {
        console.error('State mismatch');
        return;
      }

      const codeVerifier = sessionStorage.getItem('pkce_verifier');

      // Exchange code for tokens
      const response = await fetch('https://id.your-domain.com/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          grant_type: 'authorization_code',
          code: code!,
          redirect_uri: `${window.location.origin}/auth/callback`,
          client_id: 'my-web-app',
          code_verifier: codeVerifier!,
        }).toString(),
      });

      const tokens = await response.json();
      
      // Store tokens
      localStorage.setItem('access_token', tokens.access_token);
      localStorage.setItem('id_token', tokens.id_token);
      localStorage.setItem('refresh_token', tokens.refresh_token);

      // Redirect to app
      window.location.href = '/dashboard';
    };

    handleCallback();
  }, []);

  return <div>Processing login...</div>;
}
```

### Step 3: Use Access Token to Call Protected APIs

```typescript
// Make authenticated requests
async function getProtectedData() {
  const accessToken = localStorage.getItem('access_token');

  const response = await fetch('https://api.my-app.com/data', {
    headers: {
      'Authorization': `Bearer ${accessToken}`,
    },
  });

  return response.json();
}
```

### Step 4: Token Validation in Your Backend

```typescript
// NestJS example - validate access tokens

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { jwtVerify } from 'jose';

@Injectable()
export class JwtService {
  private jwksUrl = 'https://id.your-domain.com/.well-known/jwks.json';
  private publicKeys: Map<string, any> = new Map();

  async validateToken(token: string) {
    try {
      // Fetch JWKS (cache it)
      const jwks = await this.getJWKS();
      
      // Verify JWT signature
      const verified = await jwtVerify(token, jwks);
      
      return verified.payload;
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private async getJWKS() {
    if (this.publicKeys.size === 0) {
      const response = await fetch(this.jwksUrl);
      const { keys } = await response.json();
      
      keys.forEach((key: any) => {
        this.publicKeys.set(key.kid, key);
      });
    }
    return this.publicKeys;
  }
}

// Use in your controller
import { Controller, Get, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './jwt-auth.guard';

@Controller('data')
export class DataController {
  @Get()
  @UseGuards(JwtAuthGuard)
  getData() {
    return { data: 'sensitive information' };
  }
}
```

### Step 5: Refresh Token Flow (Keep User Logged In)

```typescript
// Refresh access token when expired
async function refreshAccessToken() {
  const refreshToken = localStorage.getItem('refresh_token');

  const response = await fetch('https://id.your-domain.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken!,
      client_id: 'my-web-app',
    }).toString(),
  });

  const tokens = await response.json();
  localStorage.setItem('access_token', tokens.access_token);
  localStorage.setItem('refresh_token', tokens.refresh_token);

  return tokens.access_token;
}
```

---

## Pattern 2: Service-to-Service Authentication (Client Credentials)

For backend services that need to call each other's APIs without a user.

### Step 1: Register Confidential Client

```bash
curl -X POST https://id.your-domain.com/internal/clients \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: <your-INTERNAL_API_TOKEN>" \
  -d '{
    "client_id": "order-service",
    "name": "Order Service",
    "type": "confidential",
    "allowed_scopes": ["read:products", "write:orders"],
    "allowed_audiences": ["inventory-service"],
    "allowed_grants": ["client_credentials"]
  }'
```

**Response:** (Store the `client_secret` securely!)
```json
{
  "client": {
    "clientId": "order-service",
    "type": "confidential",
    "allowedScopes": ["read:products", "write:orders"],
    "allowedAudiences": ["inventory-service"]
  },
  "client_secret": "aBcDeFgHiJkLmNoPqRsT"
}
```

### Step 2: Get Service Token

```bash
curl -X POST https://id.your-domain.com/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u order-service:aBcDeFgHiJkLmNoPqRsT \
  -d 'grant_type=client_credentials&resource=inventory-service&scope=read:products'
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMyJ9...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "read:products"
}
```

### Step 3: Call Protected Service with Token

```typescript
// Node.js example
async function callInventoryService() {
  const token = await getServiceToken();

  const response = await fetch('https://inventory-service.com/products', {
    headers: {
      'Authorization': `Bearer ${token}`,
    },
  });

  return response.json();
}
```

---

## Pattern 3: Token Exchange (Microservices Delegation)

For a service that has a user's token and needs to call another service on behalf of that user.

### Step 1: Register Exchange Client

```bash
curl -X POST https://id.your-domain.com/internal/clients \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: <your-INTERNAL_API_TOKEN>" \
  -d '{
    "client_id": "api-gateway",
    "name": "API Gateway",
    "type": "confidential",
    "allowed_grants": ["token_exchange"],
    "exchange_audiences": ["payment-service", "notification-service"]
  }'
```

### Step 2: Exchange User Token for Target Service

```typescript
// API Gateway exchanges user token for payment-service token
async function exchangeTokenForPaymentService(userToken: string) {
  const response = await fetch('https://id.your-domain.com/token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'Authorization': `Basic ${btoa('api-gateway:client-secret')}`,
    },
    body: new URLSearchParams({
      grant_type: 'urn:ietf:params:oauth:grant-type:token-exchange',
      subject_token: userToken,
      subject_token_type: 'urn:ietf:params:oauth:token-type:access_token',
      resource: 'payment-service',
      audience: 'payment-service',
    }).toString(),
  });

  return response.json();
}

// Now use the delegated token to call payment-service
async function processPayment(userToken: string, amount: number) {
  const delegatedToken = await exchangeTokenForPaymentService(userToken);

  const response = await fetch('https://payment-service.com/process', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${delegatedToken.access_token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ amount }),
  });

  return response.json();
}
```

---

## OpenID Connect (OIDC) Discovery

Your clients can auto-discover the IDP configuration:

```bash
# Get OIDC metadata
curl https://id.your-domain.com/.well-known/openid-configuration

# Get signing keys (JWKS)
curl https://id.your-domain.com/.well-known/jwks.json
```

---

## Common Token Claims Reference

Tokens issued by the IDP contain these claims:

```json
{
  "access_token": {
    "sub": "user-id",
    "aud": "my-web-app",
    "client_id": "my-web-app",
    "azp": "my-web-app",
    "scope": "openid profile email",
    "user_id": "user-123",
    "company_id": "company-456",
    "tenant_id": "tenant-789",
    "roles": ["admin", "editor"],
    "ent": {
      "product-key": "tier-1"
    },
    "iat": 1700000000,
    "exp": 1700900000
  },
  "id_token": {
    "sub": "user-123",
    "aud": "my-web-app",
    "azp": "my-web-app",
    "nonce": "random-nonce",
    "auth_time": 1700000000,
    "iat": 1700000000,
    "exp": 1700900000
  }
}
```

---

## Logout Flow

```bash
# Logout (revokes session and refresh tokens)
curl -X POST https://id.your-domain.com/logout \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "id_token_hint=<id_token>&post_logout_redirect_uri=https://my-app.com/logged-out"
```

---

## Introspection & Revocation

```bash
# Check if a token is still valid
curl -X POST https://id.your-domain.com/introspect \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u my-web-app: \
  -d "token=<access_token>"

# Revoke a token immediately
curl -X POST https://id.your-domain.com/revoke \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u my-web-app: \
  -d "token=<access_token>&token_type_hint=access_token"
```

---

## Integration Checklist

### For Each Application:

- [ ] Register OAuth client via `/internal/clients`
- [ ] Store client credentials securely (use environment variables)
- [ ] Implement authorization code flow with PKCE
- [ ] Cache and refresh access tokens
- [ ] Validate token signatures using JWKS
- [ ] Handle token expiration and refresh
- [ ] Implement logout/revocation
- [ ] Test with the IDP in staging first

### Security Best Practices:

- ✅ **Always use HTTPS** for all OAuth endpoints
- ✅ **Use PKCE** for public clients (mandatory in this IDP)
- ✅ **Store refresh tokens** securely (HttpOnly cookies or secure storage)
- ✅ **Rotate client secrets** periodically via `/internal/clients/{clientId}/secrets/rotate`
- ✅ **Validate token audience** (`aud` claim) matches your service
- ✅ **Use short-lived access tokens** (15 minutes by default)
- ✅ **Validate JWT signature** before trusting claims
- ✅ **Verify `iss` (issuer)** matches your IDP URL

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **`invalid_request`** | Missing required parameters (client_id, redirect_uri, code_challenge) |
| **`invalid_client`** | Unknown client_id or wrong secret |
| **`invalid_grant`** | Authorization code expired, already used, or PKCE verification failed |
| **`invalid_scope`** | Requested scope not allowed for this client |
| **Token validation fails** | JWKS might be cached; refresh JWKS or check token expiration |
| **Redirect URI mismatch** | Ensure redirect_uri exactly matches registered value |

---

## Summary

This IDP acts as a **central authentication hub** for all your applications, providing single sign-on (SSO) across your product suite, along with service-to-service authentication for your backend infrastructure.

Use the **Authorization Code Flow (Pattern 1)** for user-facing applications, **Client Credentials (Pattern 2)** for backend services, and **Token Exchange (Pattern 3)** for microservices that need to delegate user context.
```

---

## How to Use This Guide

You can now:

1. **Copy and save** this entire markdown content to a file named `DEPLOYMENT_AND_INTEGRATION_GUIDE.md`
2. **Email it** to yourself or your team for reference
3. **Import it** into Notion, Confluence, or any documentation tool
4. **Share** it with team members involved in deployment and integration

**To save locally:**
```bash
# On your computer, create a file and paste the content
# Or use this command if on macOS/Linux:
cat > SALI_IDP_Guide.md << 'EOF'
[paste the entire markdown content above]
EOF
```

Would you like me to help you with anything else related to the SALI IDP deployment or integration?

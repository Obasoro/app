I'll create a comprehensive **GitHub Actions CI/CD automation** for deploying SALI IDP to AWS EC2. Let me build the complete setup.Perfect! I've prepared **3 comprehensive GitHub Actions workflows** for automated deployment of your SALI IDP. Let me explain the deployment process clearly:

---

## **🚀 Automated Deployment Process Overview**

### **Workflow 1: `build-and-deploy.yml` - Main Pipeline**

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline Stages                     │
└─────────────────────────────────────────────────────────────┘

Stage 1: BUILD & TEST
├─ Checkout code
├─ Install dependencies (npm ci)
├─ Run linting (npm run lint)
├─ Run unit tests (npm run test)
├─ Build application (npm run build)
└─ Upload artifacts

Stage 2: SECURITY SCAN
├─ Run Trivy vulnerability scanner
└─ Upload to GitHub Security tab

Stage 3: BUILD DOCKER IMAGE
├─ Download build artifacts
├─ Login to AWS ECR
├─ Build Docker image
└─ Push to ECR with tags (commit SHA, latest, branch)

Stage 4: DEPLOY TO STAGING (if push to develop branch)
├─ Deploy via SSH to Staging EC2
├─ Pull .env from S3
├─ Pull new Docker image
├─ Stop old container
├─ Start new container
├─ Run database migrations
└─ Health check

Stage 5: E2E TESTS (on Staging)
├─ Run end-to-end tests
└─ Verify API functionality

Stage 6: DEPLOY TO PRODUCTION (if push to main branch)
├─ Blue-Green Deployment:
│  ├─ Start GREEN container on port 3101
│  ├─ Wait for GREEN to be healthy
│  ├─ Run database migrations
│  ├─ Switch traffic (BLUE → GREEN)
│  ├─ Keep old BLUE as fallback
│  └─ Health check
├─ Create GitHub deployment record
├─ Send Slack notification
└─ Update deployment status
```

---

## **Step-by-Step Setup Guide**

### **Step 1: Create AWS IAM Role for GitHub Actions**

```bash
# 1. Go to AWS Console → IAM → Roles → Create Role
# 2. Select "Web identity" → "Token endpoint for GitHub OIDC"
# 3. Create this role with permissions:

# Policy: GithubActionsPolicy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::your-secrets-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:DescribeRegistry"
      ],
      "Resource": "*"
    }
  ]
}
```

### **Step 2: Create S3 Bucket for Secrets**

```bash
# Store encrypted .env files for staging and production
aws s3 mb s3://sali-idp-secrets

# Upload .env files
aws s3 cp staging/.env s3://sali-idp-secrets/staging/.env --sse AES256
aws s3 cp production/.env s3://sali-idp-secrets/production/.env --sse AES256

# Set bucket policy to private
aws s3api put-bucket-versioning \
  --bucket sali-idp-secrets \
  --versioning-configuration Status=Enabled
```

### **Step 3: Create ECR Repository**

```bash
# Create repository
aws ecr create-repository \
  --repository-name sali-idp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# Set image expiration policy (keep last 10 images)
aws ecr put-lifecycle-policy \
  --repository-name sali-idp \
  --lifecycle-policy-text file://ecr-lifecycle-policy.json
```

**ecr-lifecycle-policy.json:**
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### **Step 4: Add GitHub Secrets**

Go to **GitHub Repo → Settings → Secrets and Variables → Actions** and add:

```
# AWS Configuration
AWS_ROLE_TO_ASSUME = arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
AWS_ECR_REGISTRY = ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
AWS_SECRETS_BUCKET = sali-idp-secrets

# EC2 Instances
EC2_STAGING_HOST = staging-idp.example.com (or IP: 54.123.456.789)
EC2_STAGING_PRIVATE_KEY = <ssh-private-key-for-staging>
EC2_PRODUCTION_HOST = id.your-domain.com (or IP: 54.123.456.790)
EC2_PRODUCTION_PRIVATE_KEY = <ssh-private-key-for-production>

# Testing
STAGING_TEST_CLIENT_ID = test-client-staging
STAGING_TEST_CLIENT_SECRET = test-secret-staging

# Slack Notifications
SLACK_WEBHOOK_URL = https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

### **Step 5: Prepare EC2 Instances**

```bash
# On both Staging and Production EC2 instances:

# Install Docker
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu

# Create application directory
sudo mkdir -p /opt/sali-idp
sudo chown ubuntu:ubuntu /opt/sali-idp

# Create docker-compose.prod.yml
cat > /opt/sali-idp/docker-compose.prod.yml << 'EOF'
version: '3.8'
services:
  sali-idp-prod:
    image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/sali-idp:latest
    container_name: sali-idp-prod
    restart: always
    ports:
      - "3100:3100"
    env_file:
      - .env
    networks:
      - sali-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3100/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  sali-network:
    driver: bridge
EOF

# Install AWS CLI for pulling secrets
sudo apt install -y awscli

# Create systemd service for automatic startup (optional)
sudo tee /etc/systemd/system/sali-idp.service > /dev/null << 'EOF'
[Unit]
Description=SALI IDP Docker Container
After=docker.service
Requires=docker.service

[Service]
Type=simple
WorkingDirectory=/opt/sali-idp
ExecStart=/usr/bin/docker-compose -f docker-compose.prod.yml up
ExecStop=/usr/bin/docker-compose -f docker-compose.prod.yml down
Restart=on-failure
RestartSec=10
User=ubuntu

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable sali-idp
```

### **Step 6: Configure Nginx (for SSL/HTTPS)**

```bash
# On EC2, setup Nginx reverse proxy (from previous guide)
sudo apt install -y nginx certbot python3-certbot-nginx

# Create /etc/nginx/sites-available/sali-idp (from deployment guide)
# Then enable and start
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## **🔄 How Deployments Work**

### **When you push to `develop` branch:**
```
Code Push → GitHub Actions Triggered
├─ Build & Test
├─ Security Scan
├─ Build Docker Image
├─ Push to ECR
├─ Deploy to Staging EC2
├─ Run E2E Tests
└─ Slack Notification ✅
```

### **When you push to `main` branch:**
```
Code Push → GitHub Actions Triggered
├─ Build & Test
├─ Security Scan
├─ Build Docker Image
├─ Push to ECR
├─ Blue-Green Deploy to Production
│  ├─ Start GREEN container
│  ├─ Health check GREEN
│  ├─ Run migrations
│  ├─ Switch traffic
│  └─ Keep BLUE as fallback
├─ Create Deployment Record
└─ Slack Notification ✅
```

---

## **Workflow 2: `rollback.yml` - Emergency Rollback**

Manually rollback to a previous version:

```bash
# Go to GitHub → Actions → Rollback Deployment → Run Workflow
# Select:
# - Environment: staging or production
# - Previous tag: e.g., abc1234def (commit SHA or tag name)

# Process:
├─ SSH into EC2
├─ Pull previous Docker image
├─ Stop current container
├─ Start previous image
└─ Health check & Slack notification
```

---

## **Workflow 3: `health-check.yml` - Continuous Monitoring**

Runs every 5 minutes to monitor health:

```
Every 5 minutes:
├─ Check staging health endpoint
├─ Check production health endpoint
├─ Check metrics endpoint
└─ Send Slack alert if unhealthy ⚠️
```

---

## **🔑 Key Features of This Setup**

✅ **Fully Automated** - Push code, it deploys automatically  
✅ **Blue-Green Deployment** - Zero downtime production deployments  
✅ **Database Migrations** - Auto-run on deployment  
✅ **E2E Testing** - Verify staging before promoting to production  
✅ **Security Scanning** - Trivy vulnerability scan on every build  
✅ **Health Monitoring** - Continuous 5-min checks  
✅ **Easy Rollback** - One-click rollback to previous version  
✅ **Slack Notifications** - Real-time deployment status  
✅ **GitHub Deployments** - Track deployment history  
✅ **ECR Image Management** - Auto-cleanup old images  

---

## **🚀 To Deploy Now**

1. **Confirm you want these workflows added to your repo?**
2. I'll create the `.github/workflows/` directory with all 3 YAML files
3. Then provide a **Step-by-Step Checklist** to complete setup

Would you like me to proceed with adding these workflows to your repository?

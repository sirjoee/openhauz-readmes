# Deployment Guide

This project uses GitHub Actions for **fully automated** continuous integration and deployment to EC2 running Amazon Linux.

## 🚀 **Automated Deployment Features**

- ✅ **Automatic repository cloning** using GitHub token
- ✅ **Automatic environment variable setup** from GitHub secrets
- ✅ **Zero-touch deployment** - just push to main branch
- ✅ **Private repository support** with token authentication
- ✅ **Complete CI/CD pipeline** with testing and deployment

## Setup Instructions

### 1. GitHub Secrets Configuration

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

#### **SSH Connection Secrets:**

- `EC2_HOST`: Your EC2 instance public IP or domain
- `EC2_USERNAME`: SSH username (`ec2-user` for Amazon Linux)
- `EC2_SSH_KEY`: Your private SSH key content (the entire .pem file content)
- `EC2_SSH_PORT`: SSH port (usually `22`)

#### **Repository Access Secret:**

- `GH_TOKEN`: GitHub Personal Access Token (for private repository access)

#### **Environment Variable Secrets:**

- `DB_HOST`: Database host (e.g., `your-db.rds.amazonaws.com`)
- `DB_PORT`: Database port (usually `5432`)
- `DB_USER`: Database username
- `DB_PASSWORD`: Database password
- `DB_NAME`: Database name
- `JWT_SECRET`: JWT secret key for authentication
- `SENDGRID_API_KEY`: SendGrid API key for emails
- `SENDGRID_FROM_EMAIL`: From email address
- `FRONTEND_URL`: Frontend application URL ⚠️ **Use HTTPS URL after SSL setup**
- `RAPIDAPI_KEY`: RapidAPI key (if using external APIs)
- `RAPIDAPI_HOST`: RapidAPI host
- `CLOUDINARY_CLOUD_NAME`: Cloudinary cloud name (for file uploads)
- `CLOUDINARY_API_KEY`: Cloudinary API key
- `CLOUDINARY_API_SECRET`: Cloudinary API secret

### 2. Creating GitHub Personal Access Token

1. Go to **GitHub.com** → Your profile → **Settings**
2. Click **Developer settings** → **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token (classic)**
4. Configure the token:
   - **Note**: `OpenHauz Backend Deployment`
   - **Expiration**: Choose your preference
   - **Scopes**: Select `repo` (Full control of private repositories)
5. Copy the generated token and add it as `GH_TOKEN` secret

### 3. EC2 Instance Prerequisites (Amazon Linux)

Your Amazon Linux EC2 instance needs the following installed:

```bash
# Update system packages
sudo yum update -y

# Install development tools
sudo yum groupinstall -y "Development Tools"

# Install Node.js 18.x (recommended for production)
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs

# Verify Node.js installation
node --version
npm --version

# Install Yarn globally
sudo npm install -g yarn

# Verify Yarn installation
yarn --version

# Install PM2 globally for process management
sudo yarn global add pm2

# Install Git (usually pre-installed on Amazon Linux)
sudo yum install -y git

# Install nginx for reverse proxy (REQUIRED for HTTPS)
sudo yum install -y nginx

# Create logs directory
mkdir -p /home/ec2-user/logs

# Set up PM2 to start on system boot
pm2 startup
# Follow the instructions provided by the command above
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
```

### 4. Amazon Linux Security Groups

Configure your EC2 Security Group to allow:

- **SSH (22)**: Your IP address only
- **HTTP (80)**: 0.0.0.0/0 (REQUIRED for Let's Encrypt certificate verification)
- **HTTPS (443)**: 0.0.0.0/0 (for secure traffic)
- **Custom (4000)**: 0.0.0.0/0 (temporary - can be removed after HTTPS setup)

## 🔒 **HTTPS Setup (Production Ready)**

### **Option 1: Automated HTTPS Setup (Recommended)**

Use the provided script to automatically set up HTTPS:

```bash
# SSH into your EC2 instance
ssh -i your-key.pem ec2-user@your-ec2-ip

# Navigate to your application directory
cd /home/ec2-user/openhauz-backend

# Run the HTTPS setup script
./scripts/setup-https.sh api.yourdomain.com
```

### **Option 2: Manual HTTPS Setup**

#### **Step 1: Domain Setup**

1. **Purchase a domain** from a registrar (Route 53, Namecheap, GoDaddy)
2. **Create DNS A record**:
   - **Name**: `api` (for `api.yourdomain.com`)
   - **Type**: `A`
   - **Value**: Your EC2 Public IP
   - **TTL**: `300` (5 minutes)

#### **Step 2: Install SSL Certificate**

```bash
# SSH into your EC2 instance
ssh -i your-key.pem ec2-user@your-ec2-ip

# Install nginx (if not already installed)
sudo yum install -y nginx

# Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Install certbot for Let's Encrypt SSL certificates
sudo yum install -y python3-pip
sudo pip3 install certbot certbot-nginx

# Get SSL certificate (replace with your domain)
sudo certbot --nginx -d api.yourdomain.com

# Set up automatic renewal
echo "0 0,12 * * * /usr/local/bin/certbot renew --quiet" | sudo crontab -
```

#### **Step 3: Configure Nginx (Automatic with certbot)**

The certbot command automatically creates the HTTPS configuration, but here's what it looks like:

```nginx
# /etc/nginx/conf.d/openhauz.conf
server {
    listen 80;
    server_name api.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/api.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomain.com/privkey.pem;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

### **Step 4: Update GitHub Secrets for HTTPS**

After HTTPS is set up, update these GitHub secrets:

1. **FRONTEND_URL**: Change from `http://...` to `https://your-frontend-domain.com`
2. **Any other URLs**: Ensure they use HTTPS where applicable

### **Step 5: Remove Direct Port Access (Security)**

After HTTPS is working, remove port 4000 from your Security Group:

1. Go to **EC2 Console** → **Security Groups**
2. Find your instance's security group
3. **Remove the rule** for **Custom TCP Port 4000**
4. **Keep only**: SSH (22), HTTP (80), HTTPS (443)

## 🚀 **Automated Deployment Workflow**

### **How It Works**

1. **Push to main branch** triggers the GitHub Actions workflow
2. **CI Pipeline runs**:
   - Installs dependencies with Yarn
   - Builds the application
   - Runs tests (if available)
3. **CD Pipeline executes** (only on main branch):
   - Connects to EC2 via SSH
   - **First-time setup** (if repository doesn't exist):
     - Clones repository using GitHub token
     - Creates `.env` file from GitHub secrets
   - **Every deployment**:
     - Runs the deployment script
     - Pulls latest changes
     - Installs dependencies
     - Builds application
     - Runs database migrations
     - Restarts application with PM2

### **What Gets Automated**

- ✅ **Repository cloning** (first time only)
- ✅ **Environment variables** creation from secrets
- ✅ **Dependency installation** with Yarn
- ✅ **Application building** and compilation
- ✅ **Database migrations** execution
- ✅ **Process management** with PM2
- ✅ **Zero-downtime restarts**

### **Manual Steps (One-time Setup)**

You only need to:

1. **Set up EC2 instance** with required software
2. **Configure GitHub secrets** with all environment variables
3. **Set up domain and HTTPS** for production security
4. **Push to main branch** - everything else is automated!

## Monitoring and Management

### PM2 Commands

```bash
# Check application status
pm2 status

# View logs
pm2 logs openhauz-backend

# Restart application
pm2 restart openhauz-backend

# Stop application
pm2 stop openhauz-backend

# View real-time logs
pm2 logs openhauz-backend --lines 50

# Monitor CPU and memory usage
pm2 monit
```

### System Monitoring (Amazon Linux)

```bash
# Check system resources
htop
free -h
df -h

# Check nginx status
sudo systemctl status nginx

# Check nginx logs
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Check system logs
sudo journalctl -u nginx -f
```

### Database Migrations

```bash
# Run migrations manually
yarn migration:run

# Generate new migration
yarn migration:generate src/database/migrations/YourMigrationName

# Revert last migration
yarn migration:revert
```

### Yarn Commands

```bash
# Install dependencies
yarn install

# Install dependencies (production only)
yarn install --production

# Add a new dependency
yarn add package-name

# Add a dev dependency
yarn add -D package-name

# Remove a dependency
yarn remove package-name

# Update dependencies
yarn upgrade

# Check for outdated packages
yarn outdated

# Clean yarn cache
yarn cache clean
```

## Amazon Linux Specific Considerations

### Firewall Configuration

```bash
# Check if firewalld is running (Amazon Linux 2)
sudo systemctl status firewalld

# If using firewalld, configure it
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Performance Tuning

```bash
# Increase file descriptor limits for Node.js
echo "ec2-user soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "ec2-user hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Optimize TCP settings for high traffic
echo "net.core.somaxconn = 1024" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 1024" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Security Considerations

1. **EC2 Security Groups**: Restrict access to necessary ports only
2. **GitHub Secrets**: Never commit sensitive data to git
3. **SSH Keys**: Use key-based authentication, disable password auth
4. **Database**: Use SSL connections and VPC security groups
5. **SSL Certificates**: Use HTTPS with Let's Encrypt certificates
6. **Regular Updates**: Keep Amazon Linux packages updated
7. **Token Security**: Rotate GitHub tokens periodically

## Troubleshooting

### Common Deployment Issues

1. **GitHub token expired**: Regenerate and update `GH_TOKEN` secret
2. **SSH connection failed**: Verify EC2_SSH_KEY and security groups
3. **Environment variables missing**: Check all required secrets are set
4. **Database connection failed**: Verify database credentials and security groups
5. **PM2 startup issues**: Ensure PM2 startup script is configured

### GitHub Actions Debugging

```bash
# Check workflow logs in GitHub Actions tab
# Look for specific error messages in the deployment step
# Verify all required secrets are configured
```

### Application Debugging

```bash
# SSH into EC2 and check logs
ssh -i your-key.pem ec2-user@your-ec2-ip

# Check PM2 logs
pm2 logs openhauz-backend

# Check if .env file was created properly
ls -la /home/ec2-user/openhauz-backend/.env

# Verify environment variables
pm2 show openhauz-backend | grep -A 20 "env:"
```

### Yarn Specific Issues

```bash
# Clear yarn cache if having dependency issues
yarn cache clean

# Reinstall node_modules if corrupted
rm -rf node_modules yarn.lock
yarn install

# Check yarn integrity
yarn check

# Update yarn to latest version
yarn set version stable
```

### Logs Locations

- **Application logs**: `/home/ec2-user/logs/`
- **PM2 logs**: `~/.pm2/logs/`
- **Nginx logs**: `/var/log/nginx/`
- **System logs**: `/var/log/messages`
- **GitHub Actions logs**: Repository → Actions tab

## Rollback Procedure

If deployment fails, GitHub Actions will stop, but if you need to rollback manually:

```bash
# SSH into EC2
ssh -i your-key.pem ec2-user@your-ec2-ip

# Navigate to app directory
cd /home/ec2-user/openhauz-backend

# Check previous commits
git log --oneline -10

# Rollback to previous commit
git reset --hard <previous-commit-hash>

# Reinstall dependencies for the rolled back version
yarn install --frozen-lockfile

# Restart application
pm2 restart openhauz-backend

# Check application status
pm2 status
pm2 logs openhauz-backend --lines 20
```

## 🌐 **Accessing Your API**

After successful deployment, your API will be accessible at:

### **HTTP (Initial Setup)**

- **Direct IP**: `http://YOUR-EC2-IP:4000/graphql`

### **HTTPS (Production)**

- **Secure Domain**: `https://api.yourdomain.com/graphql` ✅ **Recommended**
- **Auto-redirect**: HTTP traffic automatically redirects to HTTPS

## 🔒 **HTTPS Verification**

After setting up HTTPS, verify your setup:

```bash
# Test HTTPS certificate
curl -I https://api.yourdomain.com

# Test GraphQL endpoint
curl -X POST https://api.yourdomain.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name } } }"}'

# Check SSL certificate details
openssl s_client -connect api.yourdomain.com:443 -servername api.yourdomain.com
```

## Initial Setup Checklist

- [ ] Launch Amazon Linux EC2 instance
- [ ] Configure Security Groups (SSH, HTTP, HTTPS)
- [ ] Install Node.js, Yarn, PM2, Git, nginx on EC2
- [ ] Create GitHub Personal Access Token
- [ ] Add all required GitHub secrets (SSH + Environment variables)
- [ ] **Set up domain name** pointing to EC2 IP
- [ ] **Run HTTPS setup script** or configure SSL manually
- [ ] **Update FRONTEND_URL** secret to use HTTPS
- [ ] **Remove port 4000** from Security Group (after HTTPS works)
- [ ] Configure PM2 startup
- [ ] **Push to main branch** - Full automated deployment!

## 🔒 **HTTPS-Specific Troubleshooting**

### SSL Certificate Issues

```bash
# Check certificate status
sudo certbot certificates

# Renew certificates manually
sudo certbot renew --dry-run

# Check nginx SSL configuration
sudo nginx -t

# View SSL logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

### Common HTTPS Problems

1. **Domain not pointing to EC2**: Verify DNS A record
2. **Certificate request failed**: Check Security Groups allow port 80
3. **Mixed content warnings**: Ensure all URLs use HTTPS
4. **Certificate expired**: Set up automatic renewal with cron
5. **Nginx configuration errors**: Run `sudo nginx -t` to check syntax

### HTTPS Health Check

```bash
# Test SSL certificate validity
echo | openssl s_client -servername api.yourdomain.com -connect api.yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates

# Check SSL rating (external tool)
# Visit: https://www.ssllabs.com/ssltest/analyze.html?d=api.yourdomain.com

# Verify HSTS header
curl -I https://api.yourdomain.com | grep -i strict
```

## 🎉 **That's It!**

Once you complete the initial setup, every push to the main branch will:

1. ✅ **Automatically test** your code
2. ✅ **Deploy to EC2** without any manual intervention
3. ✅ **Update environment variables** from GitHub secrets
4. ✅ **Restart your application** with zero downtime
5. ✅ **Run database migrations** automatically

### 🔒 **Your Production API is now:**

- **Secure**: HTTPS with Let's Encrypt SSL certificates
- **Professional**: Custom domain with automatic certificate renewal
- **Automated**: Zero-touch deployment pipeline
- **Scalable**: PM2 process management with monitoring

Your deployment pipeline is now **fully automated and production-ready with HTTPS**! 🚀

## 📋 **Quick Reference**

### **API Endpoints**

- **Development**: `http://EC2-IP:4000/graphql`
- **Production**: `https://api.yourdomain.com/graphql`

### **Key Commands**

```bash
# Deploy manually
./scripts/deploy.sh

# Setup HTTPS
./scripts/setup-https.sh api.yourdomain.com

# Check PM2 status
pm2 status

# Check SSL certificates
sudo certbot certificates

# Restart nginx
sudo systemctl restart nginx
```

### **Important Files**

- **Environment**: `/home/ec2-user/openhauz-backend/.env`
- **Nginx Config**: `/etc/nginx/conf.d/openhauz.conf`
- **SSL Certificates**: `/etc/letsencrypt/live/api.yourdomain.com/`
- **PM2 Config**: `ecosystem.config.js`
- **Logs**: `/home/ec2-user/logs/` and `~/.pm2/logs/`

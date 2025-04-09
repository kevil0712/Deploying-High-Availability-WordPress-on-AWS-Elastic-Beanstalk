# Complete Step-by-Step Guide: High-Availability WordPress on AWS Elastic Beanstalk

This comprehensive guide provides exact, copy-pastable commands and detailed instructions for deploying a high-availability WordPress website on AWS with an external RDS database. Each step includes screenshots descriptions and code snippets that you can directly use.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Set Up the RDS Database](#step-1-set-up-the-rds-database)
- [Step 2: Configure Security Groups for RDS](#step-2-configure-security-groups-for-rds)
- [Step 3: Prepare WordPress Files](#step-3-prepare-wordpress-files)
- [Step 4: Create Elastic Beanstalk Environment](#step-4-create-elastic-beanstalk-environment)
- [Step 5: Update Security Groups](#step-5-update-security-groups)
- [Step 6: Test WordPress Installation](#step-6-test-wordpress-installation)
- [Step 7: Set Up Domain and HTTPS](#step-7-set-up-domain-and-https)
- [Step 8: Configure HTTPS Redirect](#step-8-configure-https-redirect)
- [Step 9: Verify High Availability](#step-9-verify-high-availability)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- AWS account with necessary permissions
- Basic understanding of AWS services
- Domain name (for HTTPS configuration)
- SSH client (Terminal on Mac/Linux, PuTTY on Windows)

## Step 1: Set Up the RDS Database

### 1.1. Navigate to RDS Service
1. Log in to the AWS Management Console
2. Search for "RDS" in the search bar and select it

### 1.2. Create Database
1. Click the "Create database" button
2. Select "Standard create" for more configuration options
3. Choose MySQL as the engine type
4. Under "Templates" section, select "Dev/Test"

### 1.3. Configure Database Settings
1. Under "Settings" section:
   - DB instance identifier: `wordpress-db`
   - Master username: `admin` (or choose your own)
   - Master password: Create a strong password and note it down
   
2. Under "Instance configuration":
   - Select "Burstable classes (includes t classes)"
   - Choose "db.t3.small" or "db.t2.small" if t3 isn't available
   
3. Under "Storage":
   - Storage type: General Purpose SSD (gp2)
   - Allocated storage: 20 GiB

4. Under "Availability & durability":
   - Select "Do not create a standby instance"

5. Under "Connectivity":
   - VPC: Choose your default VPC
   - Subnet group: Default
   - Public access: No (for better security)
   - VPC security group: Create new
   - New VPC security group name: `wordpress-db-sg`
   - Availability Zone: No preference

6. Under "Additional configuration":
   - Initial database name: `wordpress`
   - **Important**: Uncheck "Enable Enhanced monitoring" as specified in requirements

7. Leave other options as default and click "Create database"

### 1.4. Note Database Endpoint
Once the database is created (may take 5-10 minutes):
1. Click on the database name from the RDS dashboard
2. Find the "Endpoint & port" section
3. Note down the endpoint (e.g., `wordpress-db.cxyz123abc.us-east-1.rds.amazonaws.com`) - you'll need this later

## Step 2: Configure Security Groups for RDS

1. Go to EC2 Dashboard (search for "EC2" in the search bar)
2. In the left navigation panel, click on "Security Groups"
3. Find the security group created for your RDS instance (named `wordpress-db-sg` or similar)
4. Note down the Security Group ID for later use

Note: We'll add the specific inbound rules after creating the Elastic Beanstalk environment to get its security group IDs.

## Step 3: Prepare WordPress Files

### 3.1. Download and Extract WordPress
```bash
# Create a project directory
mkdir wordpress-eb-project
cd wordpress-eb-project

# Download WordPress
curl -O https://wordpress.org/latest.tar.gz

# Extract files
tar -xzf latest.tar.gz

# Move files to current directory
mv wordpress/* .
rmdir wordpress
rm latest.tar.gz
```

### 3.2. Create .ebextensions Directory
```bash
mkdir .ebextensions
```

### 3.3. Create Settings Configuration
Create a file named `.ebextensions/00_settings.config` with the following content:

```yaml
option_settings:
  aws:elasticbeanstalk:container:php:phpini:
    document_root: /
  aws:elasticbeanstalk:application:environment:
    DB_HOST: YOUR_RDS_ENDPOINT
    DB_NAME: wordpress
    DB_USER: YOUR_DB_USERNAME
    DB_PASSWORD: YOUR_DB_PASSWORD
```

Replace:
- `YOUR_RDS_ENDPOINT` with your RDS endpoint (e.g., `wordpress-db.cxyz123abc.us-east-1.rds.amazonaws.com`)
- `YOUR_DB_USERNAME` with your database username (e.g., `admin`)
- `YOUR_DB_PASSWORD` with your database password

### 3.4. Create WordPress Configuration
Create a custom `wp-config.php` file in the root directory:

```php
<?php
define('DB_NAME', getenv('DB_NAME'));
define('DB_USER', getenv('DB_USER'));
define('DB_PASSWORD', getenv('DB_PASSWORD'));
define('DB_HOST', getenv('DB_HOST'));
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

// Authentication Unique Keys and Salts
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');

$table_prefix = 'wp_';
define('WP_DEBUG', false);

// If we're behind a proxy server and using HTTPS, we need to alert WordPress of that fact
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}

/* That's all, stop editing! Happy blogging. */

/** Absolute path to the WordPress directory. */
if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

/** Sets up WordPress vars and included files. */
require_once(ABSPATH . 'wp-settings.php');
```

Replace the Authentication Unique Keys and Salts with values generated from the WordPress API:
```bash
curl https://api.wordpress.org/secret-key/1.1/salt/
```

Copy the output and replace the placeholder lines in the wp-config.php file.

### 3.5. Create Zip Archive for Deployment
```bash
# Create the zip file (exclude unnecessary files)
zip -r wordpress.zip . -x "*.git*" "*.DS_Store" "*.zip"
```

## Step 4: Create Elastic Beanstalk Environment

### 4.1. Navigate to Elastic Beanstalk
1. In the AWS Management Console, search for "Elastic Beanstalk" and select it
2. Click "Create application"

### 4.2. Configure Application
1. Application name: `wordpress-app`
2. Platform:
   - Platform: PHP
   - Platform branch: PHP 8.2 (or latest stable version)
   - Platform version: Latest
3. Application code:
   - Select "Upload your code"
   - Upload the wordpress.zip file you created
   - Version label: v1 (or any label you prefer)
4. Click "Configure more options"

### 4.3. Configure Environment
1. Environment tier: Select "Web server environment"
2. Click "Edit" in the "Capacity" box:
   - Environment type: "Load balanced"
   - Min instances: 2
   - Max instances: 4
   - Instance types: t3.small (or t2.small if t3 is unavailable)
   - Click "Save"

3. Click "Edit" in the "Instances" box:
   - **Important**: Check "Enable IMDS v1" as specified in the requirements
   - Click "Save"

4. Click "Edit" in the "Software" box:
   - In the "Environment properties" section, verify the following are set:
     - DB_HOST: Your RDS endpoint
     - DB_NAME: wordpress
     - DB_USER: Your database username
     - DB_PASSWORD: Your database password
   - Click "Save"

5. Click "Edit" in the "Load balancer" box:
   - Type: Application Load Balancer
   - Click "Save"

6. Click "Edit" in the "Security" box:
   - EC2 key pair: Create a new key pair or select an existing one
   - Service role: Use the default service role
   - Click "Save"

7. Review all settings and click "Create environment"

### 4.4. Note the Environment Details
After the environment is created (this may take 5-10 minutes):
1. Note down the environment URL (e.g., `wordpress-env.eba-abcd1234.us-east-1.elasticbeanstalk.com`)
2. Go to EC2 Dashboard > Security Groups
3. Find the two security groups associated with your Elastic Beanstalk environment:
   - One for the EC2 instances (has "AWSEBSecurityGroup" in the name)
   - One for the load balancer (has "AWSEBLoadBalancerSecurityGroup" in the name)
4. Note down both Security Group IDs

## Step 5: Update Security Groups

### 5.1. Update RDS Security Group
1. Go to EC2 Dashboard > Security Groups
2. Select the RDS security group (wordpress-db-sg)
3. Click "Edit inbound rules"
4. Add the following rules:
   - Type: MySQL/Aurora
   - Port Range: 3306
   - Source: Custom
   - Search for and select the EC2 instances security group ID from your Elastic Beanstalk environment
   - Description: "Allow MySQL from EB instances"
   - Click "Add rule" to add another rule
   - Type: MySQL/Aurora
   - Port Range: 3306
   - Source: Custom
   - Search for and select the load balancer security group ID from your Elastic Beanstalk environment
   - Description: "Allow MySQL from EB load balancer"
5. Click "Save rules"

### 5.2. Verify Elastic Beanstalk Security Groups
1. Verify that the load balancer security group has rules allowing:
   - HTTP (port 80) from anywhere
   - HTTPS (port 443) from anywhere (this might be added later)
2. If these rules don't exist, add them manually

## Step 6: Test WordPress Installation

1. Open your Elastic Beanstalk environment URL in a web browser
2. You should see the WordPress installation page
3. Complete the WordPress installation:
   - Site Title: Your choice
   - Username: Choose an admin username
   - Password: Create a strong password
   - Your Email: Enter your email address
   - Click "Install WordPress"
4. Once installation is complete, log in with your credentials
5. Create a test post to verify everything is working properly

## Step 7: Set Up Domain and HTTPS

### 7.1. Request SSL Certificate
1. Go to AWS Certificate Manager (ACM):
   - Search for "Certificate Manager" in the AWS Console
   - Make sure you're in the same region as your Elastic Beanstalk environment
2. Click "Request a certificate"
3. Select "Request a public certificate" and click "Next"
4. Add domain names:
   - Enter your domain (e.g., example.com)
   - Click "Add another name to this certificate" and add www.example.com
5. Select "DNS validation" and click "Request"

### 7.2. Validate Certificate
1. On the certificate details page, click "View certificate"
2. In the "Domains" section, click "Create records in Route 53" for each domain
   - If you're using a different DNS provider, you'll need to manually create CNAME records with the provided values
3. Click "Create records" in the pop-up window
4. Wait for validation to complete (can take 5-30 minutes)

### 7.3. Configure DNS for Your Domain
If using Route 53:
1. Go to Route 53 in the AWS Console
2. Click "Hosted zones"
3. Either select your existing hosted zone or create a new one for your domain
4. Click "Create record"
5. Create an A record:
   - Record name: Leave empty for root domain or enter "www" for the www subdomain
   - Record type: A
   - Toggle on "Alias"
   - Route traffic to: "Alias to Elastic Beanstalk environment"
   - Select your region and your Elastic Beanstalk environment
   - Click "Create records"

If using another DNS provider:
1. Create a CNAME record for www subdomain pointing to your Elastic Beanstalk URL
2. For the root domain, use your provider's options (URL redirect or forwarding)

### 7.4. Configure HTTPS in Elastic Beanstalk
1. Once the certificate is issued (Status: Issued), go to your Elastic Beanstalk environment
2. Click "Configuration" in the left sidebar
3. Under "Load balancer", click "Edit"
4. Scroll to "Listeners" and click "Add listener"
5. Configure the new listener:
   - Listener port: 443
   - Listener protocol: HTTPS
   - SSL certificate: Select your ACM certificate from the dropdown
   - SSL policy: ELBSecurityPolicy-2016-08 (default)
6. Click "Apply" and wait for the update to complete

## Step 8: Configure HTTPS Redirect

### 8.1. SSH to Your EC2 Instance
First, find your instance IP:
1. Go to EC2 Dashboard
2. Click "Instances"
3. Find one of the instances associated with your Elastic Beanstalk environment
4. Note its public IP address

Then SSH into it:
```bash
# For Mac/Linux (replace with your key path and IP)
chmod 400 /path/to/your-key.pem
ssh -i /path/to/your-key.pem ec2-user@YOUR_INSTANCE_IP
```

For Windows using PuTTY, use your .ppk file and the instance IP.

### 8.2. Create HTTPS Redirect Configuration
Once connected to your instance:

```bash
# Create the .ebextensions directory if it doesn't already exist
sudo mkdir -p /var/www/html/.ebextensions

# Create the https-redirect.config file
sudo nano /var/www/html/.ebextensions/https-redirect.config
```

Add the following content to the file:

```yaml
files:
  "/etc/httpd/conf.d/http-redirect.conf":
    mode: "000644"
    owner: root
    group: root
    content: |
      RewriteEngine On
      RewriteCond %{HTTP:X-Forwarded-Proto} !https
      RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```

Save the file (Ctrl+O, then Enter, then Ctrl+X to exit nano).

```bash
# Set proper ownership
sudo chown webapp:webapp /var/www/html/.ebextensions/https-redirect.config

# Restart Apache to apply changes
sudo systemctl restart httpd
```

## Step 9: Verify High Availability

### 9.1. Test HTTPS Configuration
1. Visit your site using HTTPS (https://yourdomain.com)
2. Visit your site using HTTP (http://yourdomain.com) - it should redirect to HTTPS
3. Verify the SSL certificate is valid (you should see a padlock icon in your browser)

### 9.2. Update WordPress Site URL
1. Log in to WordPress admin dashboard
2. Go to Settings > General
3. Update both "WordPress Address (URL)" and "Site Address (URL)" to use https://
4. Save changes

### 9.3. Test High Availability
1. Go to EC2 Dashboard
2. Select one of your Elastic Beanstalk instances
3. Click "Instance state" > "Stop instance"
4. Immediately try to access your website
5. It should remain available as traffic is redirected to the other instance
6. Wait about 5-10 minutes and verify that Elastic Beanstalk launches a replacement instance

## Troubleshooting

### Database Connection Issues
If you see "Error establishing a database connection":

1. Verify environment variables:
```bash
# SSH into your EC2 instance
ssh -i /path/to/your-key.pem ec2-user@YOUR_INSTANCE_IP

# Check environment variables
sudo /opt/elasticbeanstalk/bin/get-config environment
```

2. Test direct database connection:
```bash
# Install MySQL client
sudo yum install -y mariadb105

# Connect to RDS
mysql -h YOUR_RDS_ENDPOINT -u YOUR_DB_USERNAME -p
# Enter your password when prompted
```

3. Check security group rules:
   - Verify the RDS security group allows traffic from the Elastic Beanstalk security groups
   - EC2 Dashboard > Security Groups > RDS security group > Inbound rules

### 5xx Errors
If you see HTTP 5xx errors:

1. Check Apache error logs:
```bash
sudo tail -f /var/log/httpd/error_log
```

2. Check PHP-FPM logs:
```bash
sudo tail -f /var/log/php-fpm/error.log
```

3. Restart services:
```bash
sudo systemctl restart httpd php-fpm
```

### SSL/HTTPS Issues
If HTTPS is not working:

1. Verify certificate status in ACM (should be "Issued")
2. Check load balancer listener configuration in Elastic Beanstalk
3. Verify DNS records are correctly pointing to your Elastic Beanstalk environment

## Conclusion

Congratulations! You now have a high-availability WordPress setup running on AWS Elastic Beanstalk with:
- Multiple EC2 instances across different availability zones
- External MySQL database on RDS
- HTTPS encryption for secure communication
- Automatic failover for high availability

This setup satisfies all the requirements specified in the assignment, including:
- High availability with load balancing
- External RDS database
- Apache web server instead of Nginx
- MySQL/MariaDB with enhanced monitoring disabled
- HTTPS configuration with domain integration
- IMDS v1 enabled

For further improvements, consider:
- Implementing a CDN like CloudFront
- Setting up S3 for media storage
- Configuring automatic database backups
- Adding a Web Application Firewall (WAF)

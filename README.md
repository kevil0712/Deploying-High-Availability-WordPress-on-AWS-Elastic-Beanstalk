# Detailed Guide: Deploying High-Availability WordPress on AWS Elastic Beanstalk

This guide provides step-by-step instructions for deploying a high-availability WordPress website with an external Amazon RDS database to Elastic Beanstalk, including HTTPS configuration.

## Prerequisites

- AWS account with necessary permissions
- AWS CLI installed and configured on your local machine
- Domain name for HTTPS configuration
- Basic understanding of WordPress and AWS services

## Step 1: Set Up the RDS Database

1. Log in to the AWS Management Console
2. Navigate to the RDS service
3. Click "Create database"
4. Choose MySQL as the database engine
5. Select "Dev/Test" template
6. Under "Settings":
   - Enter a DB instance identifier (e.g., wordpress-db)
   - Set master username and password (save these credentials)
7. Under "Instance configuration":
   - Choose Burstable classes (t3.small or t2.small if t3 isn't available)
8. Under "Storage":
   - Configure storage as needed (20 GB is usually sufficient to start)
9. Under "Availability & durability":
   - Choose "Do not create a standby instance" for testing
10. Under "Connectivity":
    - Create a new VPC or use an existing one
    - Create a new security group or use an existing one
11. Expand "Additional configuration":
    - Specify an initial database name (e.g., wordpress)
    - **Uncheck "Enhanced monitoring"**
12. Click "Create database" and wait for it to become available

## Step 2: Configure Security Group for RDS

1. Go to the EC2 Dashboard
2. Navigate to "Security Groups"
3. Find the security group associated with your RDS instance
4. Edit inbound rules
5. Note: We'll add specific rules after creating our Elastic Beanstalk environment

## Step 3: Prepare WordPress Files

1. Download WordPress from https://wordpress.org/download/
2. Extract the files to a local directory
3. Create a new `.ebextensions` directory in the WordPress root
4. Inside this directory, create a file named `00_settings.config` with:

```yaml
option_settings:
  aws:elasticbeanstalk:container:php:phpini:
    document_root: /
  aws:elasticbeanstalk:application:environment:
    DB_HOST: [RDS-ENDPOINT]
    DB_NAME: wordpress
    DB_USER: [DB-USERNAME]
    DB_PASSWORD: [DB-PASSWORD]
```

5. Create a file `wp-config.php` in the WordPress root with:

```php
<?php
define('DB_NAME', getenv('DB_NAME'));
define('DB_USER', getenv('DB_USER'));
define('DB_PASSWORD', getenv('DB_PASSWORD'));
define('DB_HOST', getenv('DB_HOST'));
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

// Authentication Unique Keys and Salts - replace these with values from https://api.wordpress.org/secret-key/1.1/salt/
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

6. Replace the salt values with unique strings (generate them from https://api.wordpress.org/secret-key/1.1/salt/)
7. Zip all WordPress files into a single zip file

## Step 4: Create Elastic Beanstalk Environment

1. Open the AWS Management Console
2. Navigate to Elastic Beanstalk
3. Click "Create application"
4. Enter an application name (e.g., wordpress-app)
5. Under "Platform":
   - Choose "PHP" as the platform
   - Select the latest PHP platform branch
   - Choose "Apache" as the web server (as specified)
6. Under "Application code", choose "Upload your code" and upload the zip file
7. Click "Configure more options"

## Step 5: Configure Environment Details

1. In the configuration page:
   - Choose "High availability" for Environment tier
   - Under "Instances":
     - Select t3.small or t2.small as the instance type
     - **Enable IMDS v1** as specified
   - Under "Capacity":
     - Set Environment type to "Load balanced"
     - Min instances: 2
     - Max instances: 4
   - Under "Security":
     - Select an EC2 key pair (create one if needed)
     - Choose an IAM instance profile with appropriate permissions
   - Under "Software":
     - Set environment variables:
       - DB_HOST: [Your-RDS-Endpoint]
       - DB_NAME: wordpress
       - DB_USER: [Your-DB-Username]
       - DB_PASSWORD: [Your-DB-Password]

2. Click "Create environment" and wait for it to be created

## Step 6: Configure Security Group for Elastic Beanstalk

1. Once the environment is created, go to the EC2 Dashboard
2. Find the security group for your Elastic Beanstalk environment's load balancer and EC2 instances
3. Note both security group IDs
4. Edit inbound rules for the load balancer security group to allow:
   - HTTP (port 80)
   - HTTPS (port 443)

## Step 7: Update RDS Security Group

1. Go back to the RDS security group
2. Add new inbound rules allowing MySQL/MariaDB (port 3306) from:
   - The Elastic Beanstalk EC2 instances security group
   - The Elastic Beanstalk load balancer security group (if needed)

## Step 8: Test Initial Deployment

1. Access your Elastic Beanstalk environment URL
2. You should see the WordPress installation page
3. Complete the WordPress installation process:
   - Set up site title, admin username, password, and email
   - Click "Install WordPress"
4. Verify you can create posts and pages

## Step 9: Set Up Domain and HTTPS

1. **Request SSL Certificate**:
   - Go to AWS Certificate Manager (ACM)
   - Click "Request a certificate"
   - Choose "Request a public certificate"
   - Add your domain names (e.g., example.com and www.example.com)
   - Choose DNS validation
   - Click "Request"

2. **Validate Certificate**:
   - Follow the instructions to validate domain ownership
   - This typically involves adding CNAME records to your DNS
   - Wait for the certificate to be issued

3. **Configure DNS in your domain registrar**:
   - Add a CNAME record for the www subdomain pointing to your Elastic Beanstalk URL
   - For the root domain, use your registrar's options (A record or redirection)

4. **Configure HTTPS in Elastic Beanstalk**:
   - Go to your Elastic Beanstalk environment
   - Click "Configuration"
   - Under "Load Balancer", click "Edit"
   - Add a listener:
     - Port: 443
     - Protocol: HTTPS
     - SSL Certificate: Select your ACM certificate
   - Apply changes

## Step 10: Configure HTTPS Redirect

1. SSH into your EC2 instance:
```bash
ssh -i /path/to/your-key.pem ec2-user@your-instance-ip
```

2. Create the `.ebextensions` directory if it doesn't exist:
```bash
sudo mkdir -p /var/www/html/.ebextensions
```

3. Create a file for HTTPS redirection:
```bash
sudo nano /var/www/html/.ebextensions/https-redirect.config
```

4. Add the following content:
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

5. Save the file and set permissions:
```bash
sudo chown webapp:webapp /var/www/html/.ebextensions/https-redirect.config
```

6. Restart Apache:
```bash
sudo systemctl restart httpd
```

## Step 11: Verify High Availability

1. Test your site by accessing both HTTP and HTTPS versions (HTTP should redirect to HTTPS)
2. Test the high-availability by stopping one EC2 instance:
   - Go to EC2 console
   - Select one of your instances
   - Choose "Instance state" > "Stop instance"
   - Immediately try to access your website
   - It should remain available as traffic redirects to the other instance
   - Wait to verify that Elastic Beanstalk automatically replaces the stopped instance

## Step 12: Final WordPress Configuration

1. Log in to WordPress admin dashboard
2. Go to Settings > General
3. Update WordPress Address (URL) and Site Address (URL) to use https://
4. Save changes
5. Install essential plugins for security and performance
6. Configure WordPress settings as needed

## Troubleshooting Common Issues

### Database Connection Issues
- Verify security group settings allow traffic from EC2 to RDS
- Check environment variables match your RDS settings
- Test direct database connection from EC2 instance with: `mysql -h [RDS-ENDPOINT] -u [USERNAME] -p`

### 5xx Errors
- Check Apache error logs: `sudo tail -f /var/log/httpd/error_log`
- Check PHP-FPM logs: `sudo tail -f /var/log/php-fpm/error.log`
- Verify WordPress files have correct permissions
- Restart Apache and PHP-FPM: `sudo systemctl restart httpd php-fpm`

### SSL/HTTPS Issues
- Verify certificate is properly issued in ACM
- Check load balancer listener configuration
- Ensure DNS records are correctly pointing to your Elastic Beanstalk environment

## Maintenance Tips

- Regularly update WordPress core, themes, and plugins
- Monitor RDS and EC2 metrics for performance issues
- Consider implementing a CDN (CloudFront) for better performance
- Set up regular database backups (RDS snapshots)

## Additional Resources

- AWS Elastic Beanstalk documentation
- WordPress documentation
- Amazon RDS documentation
- AWS Certificate Manager documentation

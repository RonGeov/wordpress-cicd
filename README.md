# Wordpress Deployment Using Github Actions

This repo contains automated deployment process for a WordPress website
using Nginx as the web server, LEMP (Linux, Nginx, MySQL, PHP) stack, and GitHub
Actions as the CI/CD automation tool. The deployment process should follow security
best practices and ensure optimal performance of the website.


## Installation

* Login to any cloud provider (AWS, GCP, Azure, DO etc..). I am doing this with GCP.

* Create an instance, Make sure SSH, HTTP and HTTPS is allowed in Firewall while creating the instance

* Access the instance using SSH.


* Make sure the system packages are up to date:
```bash
sudo apt update
sudo apt upgrade
```


## Installing Nginx webserver

* Intsall Nginx
```bash
sudo apt install nginx
```
* Enable nginx on ufw firewall
```bash
sudo ufw allow 'Nginx Full'
```

* Verify ufw settings using
```bash
sudo ufw status
```
Note: if ufw not enabled enable it using command `sudo ufw enable` but make sure OpenSSH is also allowed in ufw apps before enabling ufw.


## Installing and Configuring MySQL

* Install mysql
```bash
sudo apt install mysql-server
```

* Configure mysql root password
```bash
sudo mysql_secure_installation
```

* Create a database and databse user for wordpress website
```bash
sudo mysql -u root -p

CREATE DATABASE your_databse_name;
CREATE USER 'your_username'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL ON your_database_name.* TO 'your_username'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
*replace your_username your_database_name and your_password*

## Install PHP

* To install the php8.1-fpm and php-mysql packages, run:
```bash
sudo apt install php8.1-fpm php-mysql
```

## NGINX Configuration and SSL installation

* Create a configuration file for your website
```bash
sudo vi /etc/nginx/sites-available/your_domain
```
* Copy the below configuration to the file
```bash
server {
    listen 80;
    server_name your_domain www.your_domain;
    root /var/www/your_domain;

    access_log /var/log/nginx/yourdomain_access.log;
    error_log /var/log/nginx/yourdomain_error.log;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
     }

    location ~ /\.ht {
        deny all;
    }

}
```
*Replace your_domain with your domain name and 'var/www/your_domain' with your website files path*

* Create a symbolic link and test Nginx configuration:
```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
sudo unlink /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

* Create a test index.html file in your document root and test working

* Install lets encrypt ssl
```bash
sudo apt install certbot python3-certbot-nginx
```

* Generate and secure website using certbot SSL
```bash
sudo certbot --nginx -d your_domain
```

* Test SSL certificate using https://yourdomain

* A cron file is automatically added during the installation of Certbot and we can find it in the `/etc/cron.d/certbot` directory. In case it’s not available, we need to create it. and add below lines.
```bash
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
0 */12 * * * root certbot -q renew --nginx
```

* Additional Security using Nginx configuration file by adding below blocks in server block
```bash
    # Protection against click-jacking
    add_header X-Frame-Options “SAMEORIGIN”;

    # Protection against MIME-TYPE sniffing
    add_header X-Content-Type-Options “nosniff”;

    # Enable gzip compression for static files
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types 
        text/plain 
        text/css 
        application/json 
        application/javascript 
        text/xml 
        application/xml 
        application/xml+rss 
        text/javascript 
        image/*;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;

    }

    # Enable Browser caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 360d;
        add_header Cache-Control “public”;
    }

    # Enable Keep-Alive Connections
    keepalive_requests 100;

    # Deny access to sensitive files
    location ~* (?:\.(?:bak|conf|dist|fla|in[ci]|log|psd|sh|sql|sw[op])|~)$ {
        deny all;
    }
```

## WordPress Installation and Configuration

* Run below commands to set
```bash
sudo mkdir -p /path/to/your/project
sudo usermod -aG your_user www-data
sudo chown -R your_user:www-data /path/to/your/project
sudo chmod -R 775 /path/to/your/project
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
cp wp-config-sample.php wp-config.php
```
*replace your_user with your unix user and /path/to/your/project with project path*

* Configure the wp-config.php file with your MySQL credentials

```bash
sudo vi wp-config.php
```
* make sure to change this for mysql connection
```bash
define('DB_NAME', 'your_databse_name');
define('DB_USER', 'your_user_name');
define('DB_PASSWORD', 'your_password');
```
* Make sure wordpress is working fine

## Setting up Github Actions for deployment

*  Create a .github/workflows directory.
* Inside this directory, create a YAML file (e.g., deploy.yml) for your GitHub Actions workflow:

```bash
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Install Composer dependencies
        run: |
          composer install --no-dev --optimize-autoloader

      - name: Run integrated tests
        run: |
          # Add commands to run your integrated tests here

      - name: Copy files to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.PRODUCTION_SERVER_HOST }}
          username: ${{ secrets.PRODUCTION_SERVER_USERNAME }}
          key: ${{ secrets.PRODUCTION_SERVER_SSH_KEY }}
          source: "."  # Adjust this based on your project structure
          target: "path/to/your/project"
```

### Setting up secrets in your GitHub repository settings for PRODUCTION_SERVER_HOST, PRODUCTION_SERVER_USERNAME, and PRODUCTION_SERVER_SSH_KEY. These will be used for secure SSH authentication.

* Generate an SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

* Go to .ssh folder and verify the key is generated with the given name or not
```bash
cd ~/.ssh
ls
```
* Adding the Public Key to authorized_keys
```bash
cat .ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
*on some versions of openssh you have to use ~/.ssh/authorized_keys2*

* Add the private key to your repository’s secrets :

Setting up secrets in your GitHub repository settings is essential for securely storing sensitive information such as SSH credentials. Here's how you can set up secrets for PRODUCTION_SERVER_HOST, PRODUCTION_SERVER_USERNAME, and PRODUCTION_SERVER_SSH_KEY:

* Navigate to your GitHub Repository

* Access Repository Settings: Click on the "Settings" tab near the top-right corner of your repository page.

* Manage Secrets: In the left sidebar, click on "Secrets and variables" under the "Security" section.

* Add a New Secret: Click on the "New repository secret" button.

* Add the Secrets: For each secret, you'll need to provide a name and the corresponding value.

* Secret Name: PRODUCTION_SERVER_HOST Secret Value: The IP address or hostname of your server.

* Secret Name: PRODUCTION_SERVER_USERNAME Secret Value: The SSH username to access your server.

* Secret Name: PRODUCTION_SERVER_USERNAME Secret Value: Your private SSH key.Paste the entire key, including line breaks.

on your server, you can copy the private key from
```bash
cat ~/.ssh/id_rsa
```

## Additional Features that can be implemented

### Security

* ModSecurity WAF can be enabled with nginx for additional security. For steps refer to https://www.linode.com/docs/guides/securing-nginx-with-modsecurity/

### Monitoring 

* Setup website and server monitoring like new relic, site24x7 etc.. or on premise monitoring tools you have like prometheus, nagios etc.. I have set up monitoring using new relic.

### Backup

* Setup backups for website files and database and setup cron job to run it periodically

Create a new file, for example, backup.sh:
```bash
#!/bin/bash

# Set the paths
BACKUP_DIR="/path/to/backup"
WEBSITE_DIR="/path/to/your/wordpress/installation"
DB_NAME="your_database_name"
DB_USER="your_database_user"
DB_PASSWORD="your_database_password"

# Create a backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Backup the database
mysqldump -u$DB_USER -p$DB_PASSWORD $DB_NAME > $BACKUP_DIR/db_backup_$(date +\%Y\%m\%d).sql

# Backup the website files
tar -czf $BACKUP_DIR/website_backup_$(date +\%Y\%m\%d).tar.gz -C $WEBSITE_DIR .

# Remove backups older than 7 days (adjust as needed)
find $BACKUP_DIR -name "db_backup_*" -type f -mtime +7 -exec rm {} \;
find $BACKUP_DIR -name "website_backup_*" -type f -mtime +7 -exec rm {} \;
```

Make the script executable:
```bash
chmod +x backup.sh
```

Open the crontab for editing:
```bash
crontab -e
```
Add the below line to run backups At 00:00 on every Sunday. Adjust timings for your convenience
```bash
0 0 * * 0 /path/to/backup.sh
```
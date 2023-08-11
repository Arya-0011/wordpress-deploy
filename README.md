
# wordpress-deployment

The task is to set up an automated deployment process for a WordPress website
using Nginx as the web server, LEMP (Linux, Nginx, MySQL, PHP) stack, and GitHub
Actions as the CI/CD automation tool. The deployment process should follow security
best practices and ensure optimal performance of the website.

## Tech Stack

**Cloud Provider:** AWS

**Server:** Ubuntu 22

**Local:** Windows 11

**ssh connection:** putty

**Editor:** nano

**Project Framework:** Wordpress

**Database:** MySql

**WebServer:** NGINX

**VCS (Version Control):** git

**Automated Deployment Process:** GitHub Action
## Installation

* Login to AWS account and choose the region you want I have choosen us-east-1

* create keypair, choose pem or ppk. I have choosn pem, Make sure to download that keypair

* Launch an instance, choose rule for ALLOW SSH from anywhere and HTTPS in security group

```bash
  ssh ubuntu@your_server_ip
```
*you can find the server private or public ip on EC2 management console by click on running instance*


* Update the system packages:
```bash
sudo apt update
sudo apt upgrade
```

* Install necessary packages:
```bash
sudo apt install nginx mysql-server php-fpm php-mysql unzip
```

* Configure MySQL (secure installation, create database and user):

```bash
sudo mysql_secure_installation
sudo mysql -u root -p
```
*default password wuold be root or just press enter
*choose the option carefully

```bash
CREATE DATABASE wordpress;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL ON wordpress.* TO 'wordpressuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
*replace you username and password*

**NGINX Configuratio**
```bash
sudo nano /etc/nginx/sites-available/your_domain
```

```bash
server {
    listen 80;
    server_name assignment www.assignment;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name assignment www.assignment;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```


* I dont have my domain so I use self signed ssl_certificate
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.cr
```


* Create a symbolic link and test Nginx configuration:
```bash
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

**WordPress Installation and Configuration**

```bash
sudo mkdir -p /var/www/wordpress
curl -O https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz --strip-components=1
cp wp-config-sample.php wp-config.php
```

* Configure the wp-config.php file with your MySQL credentials

```bash
sudo nano wp-config.php
```

```bash
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpressuser');
define('DB_PASSWORD', 'your_password');
```

## Deployment

*  create a .github/workflows directory.
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
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '7.4' # Adjust version as needed

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y unzip
          composer install --no-dev --optimize-autoloader

      - name: Deploy to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ./
          target: /var/www/wordpress/
```

* Setting up secrets in your GitHub repository settings for SERVER_IP, SERVER_USER, and SSH_PRIVATE_KEY. These will be used for secure SSH authentication.

Step 1: Generate an SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
you can add passphrase, I didn't

Go to .ssh folder and verify the key is generated with the given name or not
```bash
cd ~/.ssh
ls
```
Step 2: Adding the Public Key to authorized_keys

The easiest way is to use a cat command to append github-actions.pub into authorized_keys
```bash
cat github-actions.pub >> ~/.ssh/authorized_keys
```

Step 3: Adding the private key to your repository’s secrets

Setting up secrets in your GitHub repository settings is essential for securely storing sensitive information such as SSH credentials. Here's how you can set up secrets for SERVER_IP, SERVER_USER, and SSH_PRIVATE_KEY:

* Navigate to your GitHub Repository:
* Go to the GitHub repository where you want to set up the secrets.

* Access Repository Settings: Click on the "Settings" tab near the top-right corner of your repository page.

* Manage Secrets: In the left sidebar, click on "Secrets and variables" under the "Security" section.

* Add a New Secret: Click on the "New repository secret" button.

* Add the Secrets: For each secret, you'll need to provide a name and the corresponding value.

* Secret Name: SERVER_IP Secret Value: The IP address of your server.

* Secret Name: SERVER_USER Secret Value: The SSH username to access your server.

* Secret Name: SSH_PRIVATE_KEY Secret Value: Your private SSH key. This is a multiline value, so you'll need to paste the entire key, including line breaks.

on your server, you can copy the private key from here
```bash
cat ~/.ssh/github-actions
```


## Instruction for more

I have added manual approval approch also with github Action.

* When code is pushed to master ot main deploy.yml will be trigger and first it will deploy to staging server first then It waits for approval to deploy to production where only authorized person can approve that.

```bash
name: Deploy to Production

on:
  push:
    branches:
      - master

jobs:
  deploy_staging:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "7.4"

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y unzip
          composer install --no-dev --optimize-autoloader

      - name: Deploy to staging server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.STAGING_SERVER_IP }}
          username: ${{ secrets.STAGING_SERVER_USER }}
          key: ${{ secrets.STAGING_SSH_PRIVATE_KEY }}
          source: ./
          target: /var/www/inceptive/

  manual_approval:
    needs: [deploy_staging]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Manual Approval
        id: approval
        uses: trstringer/manual-approval@v1
        with:
          approve-message: "Approve deployment to production"
          deny-message: "Deny deployment to production"
          secret: ${{ secrets.MANUAL_APPROVAL_SECRET }}
          approvers: "Arya-0011"

  deploy_production:
    needs: [manual_approval]
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "7.4"

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y unzip
          composer install --no-dev --optimize-autoloader

      - name: Deploy to production server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ./
          target: /var/www/inceptive/

```



# Digital Ocean Server Setup with Keycloak SSO

This guide provides a streamlined approach to setting up a **Digital Ocean droplet** with **Rocky Linux 10**, configuring **Keycloak** for Single Sign-On (SSO), and deploying three separate applications: **Drupal 11**, a **Django** project, and a generic **PHP** application.

**A Note on Placeholders:** Throughout this guide, you'll see placeholders like `your_droplet_ip`, `your_username`, and `your_domain.com`. Remember to replace these with your actual values.

## Page 1: Infrastructure & Keycloak Foundation

### 1. Digital Ocean Droplet & Initial Setup

First, we'll create the server and perform essential security hardening.

**A. Create the Droplet:**

1. **Log in** to your Digital Ocean control panel.
    
2. Click **Create > Droplets**.
    
3. **Operating System:** Choose **Rocky Linux 10**.
    
4. **Plan:** Select a plan that meets your needs (e.g., Basic Shared CPU with 2GB+ RAM).

5. IP: IPv6 should be enabled along with IPv4
    
6. **Datacenter:** Choose a region close to your users.
    
7. **Authentication:** Add your **SSH key**. This is more secure than using a password.
    
8. **Finalize:** Set a hostname and click **Create Droplet**.
    

B. Initial Server Hardening:

Connect to your new droplet as the root user.

```
ssh root@your_droplet_ip
```

Create a new user and grant administrative privileges.

```
# Create the user and set a strong password
adduser your_username
passwd your_username

# Add the user to the 'wheel' group to grant sudo access
usermod -aG wheel your_username
```

Copy your SSH key to the new user to allow direct SSH access.

```
# Copy SSH key for passwordless login
rsync --archive --chown=your_username:your_username ~/.ssh /home/your_username
```

Disable root SSH login for better security. Edit `/etc/ssh/sshd_config` and change `PermitRootLogin yes` to `PermitRootLogin no`.

```
# Apply the change
sudo systemctl restart sshd
```

Log out and log back in as your new user: `ssh your_username@your_droplet_ip`.

**C. Configure the Firewall:**

```
# Enable and start the firewall
sudo systemctl enable --now firewalld

# Allow essential services permanently
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp # For Keycloak
sudo firewall-cmd --permanent --add-service=ssh

# Apply the new rules
sudo firewall-cmd --reload
```

**D. Update System & Install Core Components:**

```
# Update all system packages
sudo dnf update -y

# Install EPEL and Remi repositories for up-to-date packages
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
sudo dnf module enable php:remi-8.3 -y

# Install Apache, PHP, MariaDB, Python, and other tools
sudo dnf install httpd php php-cli php-mysqlnd php-gd php-xml php-mbstring php-json php-fpm mariadb-server python3 python3-pip unzip wget -y

# Enable and start core services
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now mariadb

# Secure your database installation
sudo mysql_secure_installation
```

### 2. Keycloak Installation & Configuration

Keycloak requires Java. We'll install OpenJDK and then set up Keycloak.

**A. Install Java & Keycloak:**

```
# Install Java JDK 17
sudo dnf install java-17-openjdk-devel -y

# Navigate to /opt for the installation
cd /opt

# Download the latest stable Keycloak (check keycloak.org/downloads for new versions)
sudo wget https://github.com/keycloak/keycloak/releases/download/24.0.4/keycloak-24.0.4.zip
sudo unzip keycloak-24.0.4.zip
sudo mv keycloak-24.0.4 keycloak

# Create a dedicated user for Keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
sudo chown -R keycloak:keycloak /opt/keycloak
```

B. Configure and Run Keycloak:

For initial setup, you can run Keycloak in development mode.

```
# Start Keycloak to create an initial admin user
# The --auto-build flag is for development; we'll create a service for production
/opt/keycloak/bin/kc.sh start-dev --http-port=8080
```

Access `http://your_droplet_ip:8080`, create an admin user, and log in to the **Administration Console**.

C. Create a Systemd Service for Production:

Create the service file /etc/systemd/system/keycloak.service:

```
[Unit]
Description=Keycloak Authorization Server
After=network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized --http-port=8080
LimitNOFILE=102400
LimitNPROC=102400
TimeoutStartSec=600
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

**Note:** For a true production setup, you should configure Keycloak to use your MariaDB database instead of the default H2 database and set up an Apache reverse proxy with SSL.

## Page 2: Application Deployment & SSO Integration

### 1. Drupal 11 Setup & SSO

**A. Create Database:**

```
-- Log in to MariaDB
sudo mysql -u root -p

-- Create database and user
CREATE DATABASE drupaldb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON drupaldb.* TO 'drupaluser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**B. Install Drupal & Configure Apache:**

```
# Install Drupal using Composer for better dependency management
cd /var/www/
sudo composer create-project drupal/recommended-project drupal
sudo chown -R apache:apache /var/www/drupal
sudo chmod -R 755 /var/www/drupal/web
```

Create an Apache virtual host at `/etc/httpd/conf.d/drupal.conf`:

```
<VirtualHost *:80>
    ServerName your_drupal_domain.com
    DocumentRoot /var/www/drupal/web

    <Directory /var/www/drupal/web>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/drupal_error.log
    CustomLog /var/log/httpd/drupal_access.log combined
</VirtualHost>
```

Restart Apache: `sudo systemctl restart httpd`. Then, navigate to `http://your_drupal_domain.com` to complete the web installation.

**C. Integrate Keycloak SSO:**

1. Install Module: In your Drupal directory (/var/www/drupal), run:
    
    sudo composer require drupal/keycloak
    
2. **Enable Module:** In the Drupal admin UI, go to **Extend** and enable the **Keycloak** module.
    
3. **Create Keycloak Client:**
    
    - In the Keycloak Admin Console, go to **Clients > Create client**.
        
    - **Client ID:** `drupal`
        
    - **Client authentication:** ON
        
    - **Valid redirect URIs:** `http://your_drupal_domain.com/keycloak/oauth2/callback`
        
    - **Web origins:** `http://your_drupal_domain.com`
        
    - Save, then copy the **Client Secret** from the **Credentials** tab.
        
4. **Configure Drupal Module:**
    
    - In Drupal, go to **Configuration > Web services > Keycloak**.
        
    - Enter your Keycloak URL (`http://your_droplet_ip:8080`), Realm (`master`), Client ID, and Client Secret.
        

### 2. Django Project Setup & SSO

**A. Create Database:**

```
sudo mysql -u root -p
CREATE DATABASE djangodb;
CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'another_secure_password';
GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**B. Set Up Django Project with Gunicorn:**

```
# Create project directory and virtual environment
sudo mkdir /var/www/django_project
sudo chown your_username:your_username /var/www/django_project
cd /var/www/django_project
python3 -m venv venv
source venv/bin/activate

# Install Django, Gunicorn, and a robust OIDC library
pip install django gunicorn mozilla-django-oidc mysqlclient

# Create project and configure database in settings.py
django-admin startproject mysite .
# ... edit mysite/settings.py to configure your MariaDB database ...
python manage.py migrate
python manage.py createsuperuser
deactivate
```

C. Configure Apache as a Reverse Proxy:

Create /etc/httpd/conf.d/django.conf:

```
<VirtualHost *:80>
    ServerName your_django_domain.com
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
```

For this to work, you'll run Gunicorn listening on port 8000. Create a `systemd` service for Gunicorn to manage it properly. Restart Apache after setup.

**D. Integrate Keycloak SSO with `mozilla-django-oidc`:**

1. **Create Keycloak Client:**
    
    - **Client ID:** `django`
        
    - **Valid redirect URIs:** `http://your_django_domain.com/oidc/callback/`
        
    - Copy the **Client Secret**.
        
2. **Configure Django (`settings.py`):**
    
    ```
    # settings.py
    INSTALLED_APPS = [
        # ...
        'mozilla_django_oidc',
    ]
    
    AUTHENTICATION_BACKENDS = [
        'mozilla_django_oidc.auth.OIDCAuthenticationBackend',
        'django.contrib.auth.backends.ModelBackend',
    ]
    
    OIDC_RP_CLIENT_ID = "django"
    OIDC_RP_CLIENT_SECRET = "your_client_secret_from_keycloak"
    OIDC_OP_AUTHORIZATION_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/auth"
    OIDC_OP_TOKEN_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/token"
    OIDC_OP_USER_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/userinfo"
    OIDC_RP_SIGN_ALGO = "RS256"
    OIDC_OP_JWKS_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/certs"
    
    LOGIN_REDIRECT_URL = "/"
    LOGOUT_REDIRECT_URL = "/"
    ```
    
3. **Update URLs (`urls.py`):**
    
    ```
    # urls.py
    from django.urls import path, include
    
    urlpatterns = [
        path('admin/', admin.site.urls),
        path('oidc/', include('mozilla_django_oidc.urls')),
        # ... your other app urls
    ]
    ```
    

Restart Gunicorn and Apache to apply changes.

## Page 3: PHP Application & Documentation Strategy

### 1. Generic PHP Project Setup

A. Deploy Application:

Place your PHP application files in a dedicated directory.

```
sudo mkdir /var/www/php_app
# Copy your PHP application files to this directory
sudo chown -R apache:apache /var/www/php_app
sudo chmod -R 755 /var/www/php_app
```

B. Configure Apache:

Create /etc/httpd/conf.d/php_app.conf:

```
<VirtualHost *:80>
    ServerName your_php_app_domain.com
    DocumentRoot /var/www/php_app

    <Directory /var/www/php_app>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Restart Apache: `sudo systemctl restart httpd`.

### 2. PHP Keycloak SSO Integration

We'll use a standard OIDC client library for PHP.

1. **Install Library:**
    
    ```
    cd /var/www/php_app
    sudo composer require jumbojett/openid-connect-php
    ```
    
2. **Create Keycloak Client:**
    
    - **Client ID:** `php-app`
        
    - **Valid redirect URIs:** `http://your_php_app_domain.com/callback.php`
        
    - Copy the **Client Secret**.
        
3. Example PHP Code:
    
    Create a file login.php to initiate the SSO flow:
    
    ```
    <?php
    require 'vendor/autoload.php';
    use Jumbojett\OpenIDConnectClient;
    
    $oidc = new OpenIDConnectClient(
        'http://your_droplet_ip:8080/realms/master', // Keycloak provider URL
        'php-app',                                  // Client ID
        'your_client_secret_from_keycloak'          // Client Secret
    );
    
    // This triggers the authentication flow
    $oidc->authenticate();
    
    // Store user info in the session
    $_SESSION['user_info'] = $oidc->requestUserInfo();
    
    // Redirect to a protected page
    header("Location: /profile.php");
    exit();
    ```
    
    Create a `profile.php` to display user data:
    
    ```
    <?php
    session_start();
    if (empty($_SESSION['user_info'])) {
        header("Location: /login.php");
        exit();
    }
    $userInfo = $_SESSION['user_info'];
    echo "<h1>Welcome, " . htmlspecialchars($userInfo->name) . "</h1>";
    echo "<p>Email: " . htmlspecialchars($userInfo->email) . "</p>";
    ?>
    ```
    

### 3. Documentation & Version Control Strategy

**A. Hosting Documentation on GitHub:**

1. **Create Repository:** Create a new public repository on GitHub (e.g., `my-internship-tasks`).
    
2. **Use Markdown:** Write your documentation in Markdown (`.md`) files. This is simple, clean, and renders beautifully on GitHub.
    
3. **Structure:** Organize your documentation logically.
    
    ```
    my-internship-tasks/
    ├── README.md              # Project overview
    ├── 01-server-setup.md     # Droplet and hardening
    ├── 02-keycloak-setup.md   # Keycloak details
    ├── 03-drupal-integration.md
    └── 04-django-integration.md
    ```
    
4. **Commit and Push:** Regularly commit your changes and push them to GitHub. `git add .`, `git commit -m "Add Drupal setup guide"`, `git push`.
    

B. Ensuring Daily GitHub Commits:

To maintain a consistent commit history for project tracking, you can use a GitHub Actions workflow to create an empty commit each day.

Create the file `.github/workflows/daily-commit.yml` in your repository:

```
name: Daily Empty Commit

on:
  schedule:
    # Runs every day at a specific time (e.g., 5:30 UTC)
    - cron: '30 5 * * *'
  workflow_dispatch: # Allows manual triggering from the Actions tab

jobs:
  daily_commit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure Git
        run: |
          git config user.name "GitHub Actions Bot"
          git config user.email "actions@github.com"

      - name: Create empty commit and push
        run: |
          git commit --allow-empty -m "Automated daily commit to track activity"
          git push
```

Commit this file to your repository. This workflow will now run automatically, helping you maintain a visible activity streak.
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
    
5. **IP:** IPv6 should be enabled along with IPv4.
    
6. **Datacenter:** Choose a region close to your users.
    
7. **Authentication:** Add your **SSH key**. This is more secure than using a password.
    
8. **Finalize:** Set a hostname and click **Create Droplet**.
    

**B. Initial Server Hardening:**

Connect to your new Droplet as the root user.

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
sudo dnf install [https://rpms.remirepo.net/enterprise/remi-release-10.rpm](https://rpms.remirepo.net/enterprise/remi-release-10.rpm) -y
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
sudo wget [https://github.com/keycloak/keycloak/releases/download/24.0.4/keycloak-24.0.4.zip](https://github.com/keycloak/keycloak/releases/download/24.0.4/keycloak-24.0.4.zip)
sudo unzip keycloak-24.0.4.zip
sudo mv keycloak-24.0.4 keycloak

# Create a dedicated user for Keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
sudo chown -R keycloak:keycloak /opt/keycloak
```

**B. Configure and Run Keycloak:**

For initial setup, you can run Keycloak in development mode.

```
# Start Keycloak to create an initial admin user
# The --auto-build flag is for development; we'll create a service for production
/opt/keycloak/bin/kc.sh start-dev --http-port=8080
```

Access `http://your_droplet_ip:8080`, create an admin user, and log in to the **Administration Console**.

**C. Create a Systemd Service for Production:**

Create the service file `/etc/systemd/system/keycloak.service`:

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

**Note:** For a proper production setup, you should configure Keycloak to use your MariaDB database instead of the default H2 database and set up an Apache reverse proxy with SSL.

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
sudo dnf install composer -y
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

Restart Apache: `sudo systemctl restart httpd`. Then, navigate `http://your_drupal_domain.com` to complete the web installation.

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

**C. Configure Apache as a Reverse Proxy:**

Create `/etc/httpd/conf.d/django.conf`:

```
<VirtualHost *:80>
    ServerName your_django_domain.com
    ProxyPass / [http://127.0.0.1:8000/](http://127.0.0.1:8000/)
    ProxyPassReverse / [http://127.0.0.1:8000/](http://127.0.0.1:8000/)
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
    from django.contrib import admin
    
    urlpatterns = [
        path('admin/', admin.site.urls),
        path('oidc/', include('mozilla_django_oidc.urls')),
        # ... your other app urls
    ]
    ```
    

Restart Gunicorn and Apache to apply changes.

## Page 3: PHP Application & Final Submission

### 1. Generic PHP Project Setup

**A. Deploy Application:**

Place your PHP application files in a dedicated directory.

```
sudo mkdir /var/www/php_app
# Copy your PHP application files to this directory
sudo chown -R apache:apache /var/www/php_app
sudo chmod -R 755 /var/www/php_app
```

**B. Configure Apache:**

Create `/etc/httpd/conf.d/php_app.conf`:

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
        
3. **Example PHP Code:**
    
    Create a file `login.php` To initiate the SSO flow:
    
    ```
    <?php
    require 'vendor/autoload.php';
    use Jumbojett\OpenIDConnectClient;
    
    session_start();
    
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
    
    Create a `profile.php` To display user data:
    
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
    

### 3. Submission and Evaluation

This section clarifies the submission process and how your work will be evaluated.

**A. Evaluation Criteria**

Your task will be judged based on the following criteria:

- **Correctness & Functionality:** The entire setup must be fully functional. This includes the Digital Ocean droplet, a secure server environment, a working Keycloak instance, and all three applications (Drupal, Django, and PHP) correctly integrated with Keycloak SSO. The evaluation team will test the SSO flow on each application.
    
- **Documentation Quality:** The documentation must be clear, comprehensive, and easy to follow.
    
- **Screenshots as Proof:** You must include screenshots in your documentation to visually confirm key steps, such as firewall status, service status, Keycloak client configuration, and successful SSO logins on each application.
    
- **Live Demo:** A live, publicly accessible environment is required for evaluation.
    

**B. Mode of Submission**

- You must create a public repository on GitHub.
    
- The final submission is the URL to this GitHub repository.
    
- The repository's `README.md` file should serve as the main project page, containing an overview, a link to the live Droplet IP/domains, and the detailed documentation files for each setup part.
    

**C. Required Deliverables**

The **only deliverable** is the URL to your public GitHub repository. This repository must contain:

1. **Complete Documentation:** A set of Markdown (`.md`) files detailing every step you took, from server creation to final application configuration. Structure your documentation logically (e.g., `01-server-setup.md`, `02-keycloak.md`, etc.).
    
2. **Screenshots:** Embedded directly within your documentation files.
    
3. **Live URLs:** The IP address of your Digital Ocean droplet and the Drupal, Django, and PHP application domains must be clearly listed in the `README.md` file for the evaluation team to access and test.
    

No video demonstration or presentation is required. The live, functional setup serves as the primary proof of your work.

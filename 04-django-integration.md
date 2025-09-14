# 04. Django Integration with Keycloak SSO

## Django Project Setup and Configuration

### Step 1: Database Setup for Django

Created a dedicated database and user for the Django application:

```bash
# Log in to MariaDB
sudo mysql -u root -p

# Create database and user
CREATE DATABASE djangodb;
CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'q4tdqs7a';
GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

![alt text](image-28.png)

### Step 2: Create Django Project Directory

```bash
# Create project directory
sudo mkdir /var/www/django_project
sudo chown xyushman:xyushman /var/www/django_project

# Install required system dependencies
sudo dnf install gcc python3-devel mariadb-devel pkg-config -y
```

### Step 3: Set Up Python Virtual Environment

```bash
# Navigate to project directory
cd /var/www/django_project

# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate
```

### Step 4: Install Python Dependencies

```bash
# Install Django, Gunicorn, OIDC library, and MySQL client
pip install django gunicorn mozilla-django-oidc mysqlclient
```

### Step 5: Create Django Project

```bash
# Create Django project
django-admin startproject mysite .

# Run initial migrations
python manage.py migrate

# Create superuser
python manage.py createsuperuser
# Username: xyushman
# Password: d4tdg57a

# Deactivate virtual environment
deactivate
```
![alt text](image-29.png)
![alt text](image-30.png)

## Django Configuration

### Step 6: Configure Database Settings

Updated `mysite/settings.py` with database configuration:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'djangodb',
        'USER': 'djangouser',
        'PASSWORD': 'q4tdqs7a',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

### Step 7: Configure Keycloak OIDC Settings

Added Keycloak OIDC configuration to `mysite/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'mozilla_django_oidc',  # OIDC authentication
]

AUTHENTICATION_BACKENDS = [
    'mozilla_django_oidc.auth.OIDCAuthenticationBackend',
    'django.contrib.auth.backends.ModelBackend',
]

# Keycloak OIDC configuration
OIDC_RP_CLIENT_ID = "django"
OIDC_RP_CLIENT_SECRET = "IAFIzhole719rQsB6zhimSkCV4n2iR6F"
OIDC_OP_AUTHORIZATION_ENDPOINT = "http://167.172.205.191:8080/realms/master/protocol/openid-connect/auth"
OIDC_OP_TOKEN_ENDPOINT = "http://167.172.205.191:8080/realms/master/protocol/openid-connect/token"
OIDC_OP_USER_ENDPOINT = "http://167.172.205.191:8080/realms/master/protocol/openid-connect/userinfo"
OIDC_RP_SIGN_ALGO = "RS256"
OIDC_OP_JWKS_ENDPOINT = "http://167.172.205.191:8080/realms/master/protocol/openid-connect/certs"

# Redirect URLs
LOGIN_REDIRECT_URL = "/django/"
LOGOUT_REDIRECT_URL = "/django/"

# Additional settings for reverse proxy
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'http')
```

### Step 8: Configure Static Files

Updated static files configuration in `mysite/settings.py`:

```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

### Step 9: Configure URL Routing

Updated `mysite/urls.py` to include OIDC URLs and a simple home view:

```python
from django.contrib import admin
from django.urls import path, include
from django.http import HttpResponse

def home(request):
    return HttpResponse("Welcome to Django with Keycloak SSO!")

urlpatterns = [
    path('admin/', admin.site.urls),
    path('oidc/', include('mozilla_django_oidc.urls')),
    path('', home),
]
```

## Gunicorn Configuration

### Step 10: Create Gunicorn Systemd Service

Created a systemd service file for Gunicorn:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

Added the following configuration:
```ini
[Unit]
Description=Gunicorn daemon for Django project
After=network.target

[Service]
User=xyushman
Group=xyushman
WorkingDirectory=/var/www/django_project
ExecStart=/var/www/django_project/venv/bin/gunicorn \
    --access-logfile - \
    --workers 3 \
    --bind 127.0.0.1:8000 \
    mysite.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
```

Enabled and started the Gunicorn service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn
```

![alt text](image-31.png)

## Apache Reverse Proxy Configuration

### Step 11: Configure Apache for Django

Added Django configuration to the combined Apache configuration:

```apache
# === Django at /django ===
Alias /django/static/ /var/www/django_project/staticfiles/
<Directory /var/www/django_project/staticfiles>
    Require all granted
</Directory>

ProxyPreserveHost On
ProxyPass /django/ http://127.0.0.1:8000/
ProxyPassReverse /django/ http://127.0.0.1:8000/
```

Restarted Apache to apply changes:

```bash
sudo systemctl restart httpd
```

### Step 12: Collect Static Files

```bash
cd /var/www/django_project
source venv/bin/activate
python manage.py collectstatic
deactivate
```

## Keycloak Client Configuration

### Step 13: Configure Keycloak Client for Django

In the Keycloak Admin Console (`http://167.172.205.191:8080`):

1. Navigated to **Clients** > **Create client**
2. Configured the Django client:
   - **Client ID**: django
   - **Client Protocol**: openid-connect
   - **Access Type**: confidential
   - **Valid Redirect URIs**: http://167.172.205.191/django/oidc/callback/
3. Saved the configuration and copied the **Client Secret** from the **Credentials** tab
4. Updated the `OIDC_RP_CLIENT_SECRET` in Django settings with the actual secret

## Testing and Verification

### Step 14: Test Django Application

1. Accessed the Django application at `http://167.172.205.191/django/`
2. Verified the welcome message was displayed
3. Tested the admin interface at `http://167.172.205.191/django/admin/`
4. Verified static files were loading correctly

### Step 15: Test SSO Integration

1. Accessed a protected page to trigger authentication
2. Was redirected to Keycloak login page
3. Entered Keycloak credentials
4. Was successfully redirected back to Django application
5. Verified user was authenticated

### Step 16: Troubleshooting

Encountered and resolved the following issues:

1. **Static files not loading**: Fixed Apache Alias configuration and collected static files
2. **Reverse proxy issues**: Added proper proxy headers to Django settings
3. **OIDC callback errors**: Ensured redirect URI in Keycloak matched Django URL structure
4. **Database connection issues**: Verified MariaDB user permissions and database configuration

## Final Configuration

### Django Application Information
- **URL**: http://167.172.205.191/django/
- **Admin URL**: http://167.172.205.191/django/admin/
- **Static files**: http://167.172.205.191/django/static/
- **Database**: djangodb

### Keycloak Client Configuration
- **Client ID**: django
- **Client Protocol**: openid-connect
- **Access Type**: confidential
- **Redirect URI**: http://167.172.205.191/django/oidc/callback/

The Django integration with Keycloak SSO was successfully completed, providing seamless authentication through Keycloak for the Django application.

---

**Next Step**: [PHP Application Integration with Keycloak SSO](05-php-integration.md)
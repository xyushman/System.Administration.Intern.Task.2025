# Digital Ocean Server with Keycloak SSO Integration

## Live Demo URLs
- **Main Server**: http://167.172.205.191/
- **Drupal Application**: http://167.172.205.191/
- **Django Application**: http://167.172.205.191/django/
- **PHP Application**: http://167.172.205.191/php/login.php
- **Keycloak Admin**: http://167.172.205.191:8080/ (admin/q4tdqs7a)

## Applications Integrated with Keycloak SSO
1. Drupal 11 - Content Management System
2. Django - Python Web Framework  
3. PHP Application - Custom PHP app with OIDC

## Documentation
- [Server Setup](01-server-setup.md)
- [Keycloak Configuration](02-keycloak-configuration.md)
- [Drupal Integration](03-drupal-integration.md)
- [Django Integration](04-django-integration.md)
- [PHP Integration](05-php-integration.md)
- [Troubleshooting](06-troubleshooting.md)

## Technology Stack
- **OS**: Rocky Linux 10
- **Web Server**: Apache HTTPD
- **Database**: MariaDB
- **SSO**: Keycloak 24.0.4
- **Programming**: PHP 8.3, Python 3.12, Django
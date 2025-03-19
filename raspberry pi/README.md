# Setting Up LDAP, Keycloak, and Nginx Reverse Proxy with HTTPS on Raspberry Pi

## Prerequisites
Ensure your Raspberry Pi has:
- Docker and Docker Compose installed
- Nginx installed
- OpenSSL for generating self-signed certificates

## Step 1: Create a Docker Network
```sh
docker network create my_network
```

## Step 2: Set Up OpenLDAP and phpLDAPadmin
Create a `docker-compose.yml` file and add the following services:

```yaml
services:
  ldap:
    image: osixia/openldap:latest
    container_name: ldap_server
    restart: always
    environment:
      LDAP_ORGANISATION: "My Organization"
      LDAP_DOMAIN: "pi5.local"
      LDAP_ADMIN_PASSWORD: "admin"
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - ./ldap_data:/var/lib/ldap
      - ./ldap_config:/etc/ldap/slapd.d
    networks:
      - my_network
  phpldapadmin:
    image: osixia/phpldapadmin
    container_name: ldap_admin
    restart: always
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: "ldap"
      PHPLDAPADMIN_HTTPS: "false"
    ports:
      - "8081:80"
    networks:
      - my_network

networks:
  my_network:
    external: true
```

### LDAP Admin Credentials:
- **User**: `cn=admin,dc=pi5,dc=local`
- **Password**: `admin`

## Step 3: Set Up Keycloak and PostgreSQL

Add the following services to `docker-compose.yml`:

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.1.1
    container_name: keycloak
    environment:
      KC_HTTP_ENABLED: "true"
      KC_HTTPS_ENABLED: "false"
      KC_PROXY: edge
      KC_PROXY_HEADERS: xforwarded
      KC_HOSTNAME_STRICT_HTTPS: "false"
      KC_HOSTNAME_STRICT: "false"
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HEALTH_ENABLED: 'true'
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloakdb:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      KC_HTTPS_KEYSTORE_FILE: /opt/keycloak/certs/keycloak.p12
      KC_HTTPS_KEYSTORE_PASSWORD: keycloak
      KC_HTTP_RELATIVE_PATH: /keycloak
    command: start-dev
    depends_on:
      - keycloakdb
    ports:
      - 8080:8080
    volumes:
      - ./certs:/opt/keycloak/certs
    networks:
      - my_network
  keycloakdb:
    image: postgres:17-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    networks:
      - my_network

volumes:
  postgres_data:

networks:
  my_network:
    external: true
```

## Step 4: Install and Configure Nginx

```sh
sudo apt install -y nginx certbot
```

Create necessary directories:
```sh
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled
```

### Step 4.1: Generate Self-Signed SSL Certificate
```sh
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt -subj "/CN=pi5.local"
```

### Step 4.2: Configure Nginx Reverse Proxy

Create a configuration file:
```sh
sudo nano /etc/nginx/sites-available/reverse-proxy
```

Add the following:

```nginx
server {
    listen 80;
    server_name pi5.local;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name pi5.local;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location /keycloak/ {
        proxy_pass http://localhost:8080;
        proxy_set_header    Host               $host;
        proxy_set_header    X-Real-IP          $remote_addr;
        proxy_set_header    X-Forwarded-For    $proxy_protocol_addr;
        proxy_set_header    X-Forwarded-Host   $host;
        proxy_set_header    X-Forwarded-Server $host;
        proxy_set_header    X-Forwarded-Port   $server_port;
        proxy_set_header    X-Forwarded-Proto  $scheme;
    }

    location /ldapadmin/ {
        proxy_pass http://localhost:8081/;
        proxy_set_header    Host               $host;
        proxy_set_header    X-Real-IP          $remote_addr;
        proxy_set_header    X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Host   $host;
        proxy_set_header    X-Forwarded-Server $host;
        proxy_set_header    X-Forwarded-Port   $server_port;
        proxy_set_header    X-Forwarded-Proto  $scheme;
    }
}
```

Enable the configuration:
```sh
sudo ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
```

Test and restart Nginx:
```sh
sudo nginx -t
sudo systemctl restart nginx
```

## Step 5: Configure `/etc/hosts`
Edit the hosts file on your machine:
```sh
sudo nano /etc/hosts
```
Add:
```sh
192.168.1.27 pi5.local
```

On Windows, edit:
```
C:\Windows\System32\drivers\etc\hosts
```
and add the same line.

## Step 6: Start Services
```sh
docker-compose up -d
```

## Troubleshooting

Check the Keycloak documentation for the latest configuration options:
- https://www.keycloak.org/server/reverseproxy
- https://www.keycloak.org/server/hostname
- https://skycloak.io/blog/how-to-run-keycloak-behind-a-reverse-proxy/ 

---
This setup provides a working LDAP authentication system with Keycloak and a secure reverse proxy using Nginx.


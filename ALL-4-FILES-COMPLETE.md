═══════════════════════════════════════════════════════════════════════════════
                    IERADA COMPLETE SETUP - ALL 4 FILES
═══════════════════════════════════════════════════════════════════════════════

This file contains ALL 4 files you need. Copy each section to respective location.

═══════════════════════════════════════════════════════════════════════════════
FILE 1: docker-compose-prod-proxy.yaml
LOCATION: /root/production/docker-compose-prod-proxy.yaml
═══════════════════════════════════════════════════════════════════════════════

version: '3.8'

services:
  mysql-prod:
    build:
      context: ./database
      dockerfile: Dockerfile
    image: database:prod-v1
    container_name: mysql-prod
    restart: unless-stopped
    ports:
      - "3306:3306"
    networks:
      - ierada-prod
    env_file:
      - ./database/.env
    volumes:
      - /Ierada_DB/production/ierada_database:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  backend-prod:
    build:
      context: ./backend/Ierada_Dev_Ec2_Server
      dockerfile: Dockerfile
    image: backend:prod-v1
    container_name: backend-prod
    restart: unless-stopped
    ports:
      - "3000:3000"
    networks:
      - ierada-prod
    env_file:
      - ./backend/Ierada_Dev_Ec2_Server/.env
    volumes:
      - /Ierada_DB/production/assets:/usr/src/app/assets
    depends_on:
      mysql-prod:
        condition: service_healthy

  frontend-prod:
    build:
      context: ./frontend/Ierada_Dev_Ec2_Client
      dockerfile: Dockerfile
    image: frontend:prod-v1
    container_name: frontend-prod
    restart: unless-stopped
    networks:
      - ierada-prod
    env_file:
      - ./frontend/Ierada_Dev_Ec2_Client/.env
    depends_on:
      - backend-prod

networks:
  ierada-prod:
    driver: bridge
    name: ierada-prod


═══════════════════════════════════════════════════════════════════════════════
FILE 2: docker-compose-test-proxy.yaml
LOCATION: /root/testing/docker-compose-test-proxy.yaml
═══════════════════════════════════════════════════════════════════════════════

version: '3.8'

services:
  mysql-test:
    build:
      context: ./database
      dockerfile: Dockerfile
    image: database:test-v1
    container_name: mysql-test
    restart: unless-stopped
    ports:
      - "3307:3306"
    networks:
      - ierada-test
    env_file:
      - ./database/.env
    volumes:
      - /Ierada_DB/testing/ierada_database:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  backend-test:
    build:
      context: ./backend/Ierada_Dev_Ec2_Server
      dockerfile: Dockerfile
    image: backend:test-v1
    container_name: backend-test
    restart: unless-stopped
    ports:
      - "3001:3000"
    networks:
      - ierada-test
    env_file:
      - ./backend/Ierada_Dev_Ec2_Server/.env
    volumes:
      - /Ierada_DB/testing/assets:/usr/src/app/assets
    depends_on:
      mysql-test:
        condition: service_healthy

  frontend-test:
    build:
      context: ./frontend/Ierada_Test_Client
      dockerfile: Dockerfile
    image: frontend:test-v1
    container_name: frontend-test
    restart: unless-stopped
    networks:
      - ierada-test
    env_file:
      - ./frontend/Ierada_Test_Client/.env
    depends_on:
      - backend-test

networks:
  ierada-test:
    driver: bridge
    name: ierada-test


═══════════════════════════════════════════════════════════════════════════════
FILE 3: docker-compose-reverse-proxy.yaml
LOCATION: /root/reverse-proxy/docker-compose.yaml (rename to this)
═══════════════════════════════════════════════════════════════════════════════

version: '3.8'

services:
  reverse-proxy:
    image: nginx:latest
    container_name: reverse-proxy-ierada
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /root/Ierada_SSL:/etc/nginx/ssl:ro
      - ./nginx-unified.conf:/etc/nginx/conf.d/default.conf:ro
      - ./logs:/var/log/nginx
    networks:
      - ierada-prod
      - ierada-test
    depends_on:
      - frontend-prod
      - backend-prod
      - frontend-test
      - backend-test

networks:
  ierada-prod:
    external: true
  ierada-test:
    external: true


═══════════════════════════════════════════════════════════════════════════════
FILE 4: nginx-unified.conf
LOCATION: /root/reverse-proxy/nginx-unified.conf
═══════════════════════════════════════════════════════════════════════════════

# Nginx Unified Configuration - Production & Testing on Port 443
# ierada.com (Production) + internal-test.ierada.com (Testing)

# HTTP to HTTPS Redirect
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# ====================================================================
# PRODUCTION - ierada.com
# ====================================================================
server {
    listen 443 ssl http2;
    server_name ierada.com www.ierada.com;

    # SSL Configuration
    ssl_certificate     /etc/nginx/ssl/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/ierada.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Logging
    access_log /var/log/nginx/prod_access.log;
    error_log  /var/log/nginx/prod_error.log;

    # Performance & Timeouts
    client_max_body_size 500m;
    client_body_timeout 300s;
    proxy_read_timeout 300s;
    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;

    # Root Location - Serve Frontend (SPA)
    location / {
        proxy_pass         http://frontend-prod:80;
        proxy_http_version 1.1;
        proxy_set_header   Host               $host;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  https;
        proxy_set_header   X-Forwarded-Host   $host;
        try_files $uri /index.html;
    }

    # API Location - Proxy to Backend
    location /api {
        proxy_pass         http://backend-prod:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade            $http_upgrade;
        proxy_set_header   Connection         "upgrade";
        proxy_set_header   Host               $host;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  https;
        proxy_set_header   X-Forwarded-Host   $host;

        # Buffers
        proxy_buffers 16 16k;
        proxy_buffer_size 32k;
        proxy_request_buffering off;

        # Timeouts
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
        proxy_connect_timeout 600s;
    }
}

# ====================================================================
# TESTING - internal-test.ierada.com
# ====================================================================
server {
    listen 443 ssl http2;
    server_name internal-test.ierada.com www.internal-test.ierada.com;

    # SSL Configuration (Same certificate)
    ssl_certificate     /etc/nginx/ssl/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/ierada.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Logging
    access_log /var/log/nginx/test_access.log;
    error_log  /var/log/nginx/test_error.log;

    # Performance & Timeouts
    client_max_body_size 500m;
    client_body_timeout 300s;
    proxy_read_timeout 300s;
    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;

    # Root Location - Serve Frontend (SPA)
    location / {
        proxy_pass         http://frontend-test:80;
        proxy_http_version 1.1;
        proxy_set_header   Host               $host;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  https;
        proxy_set_header   X-Forwarded-Host   $host;
        try_files $uri /index.html;
    }

    # API Location - Proxy to Backend
    location /api {
        proxy_pass         http://backend-test:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade            $http_upgrade;
        proxy_set_header   Connection         "upgrade";
        proxy_set_header   Host               $host;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  https;
        proxy_set_header   X-Forwarded-Host   $host;

        # Buffers
        proxy_buffers 16 16k;
        proxy_buffer_size 32k;
        proxy_request_buffering off;

        # Timeouts
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
        proxy_connect_timeout 600s;
    }
}


═══════════════════════════════════════════════════════════════════════════════
DEPLOYMENT INSTRUCTIONS
═══════════════════════════════════════════════════════════════════════════════

STEP 1: Create Reverse Proxy Folder
─────────────────────────────────────────────────────────────
mkdir -p /root/reverse-proxy/logs

STEP 2: Copy Files to Correct Locations
─────────────────────────────────────────────────────────────
File 1 → /root/production/docker-compose-prod-proxy.yaml
File 2 → /root/testing/docker-compose-test-proxy.yaml
File 3 → /root/reverse-proxy/docker-compose.yaml (rename)
File 4 → /root/reverse-proxy/nginx-unified.conf

STEP 3: Update /etc/hosts
─────────────────────────────────────────────────────────────
sudo nano /etc/hosts

Add:
192.168.x.x ierada.com www.ierada.com
192.168.x.x internal-test.ierada.com www.internal-test.ierada.com

STEP 4: Start Production
─────────────────────────────────────────────────────────────
cd /root/production
docker-compose -f docker-compose-prod-proxy.yaml up -d
sleep 10

STEP 5: Start Testing
─────────────────────────────────────────────────────────────
cd /root/testing
docker-compose -f docker-compose-test-proxy.yaml up -d
sleep 10

STEP 6: Start Reverse Proxy
─────────────────────────────────────────────────────────────
cd /root/reverse-proxy
docker-compose up -d
sleep 5

STEP 7: Verify
─────────────────────────────────────────────────────────────
docker ps

STEP 8: Access
─────────────────────────────────────────────────────────────
✓ Production:  https://ierada.com (Port 443)
✓ Testing:     https://internal-test.ierada.com (Port 443)


═══════════════════════════════════════════════════════════════════════════════
QUICK COMMANDS
═══════════════════════════════════════════════════════════════════════════════

View all logs:
  cd /root/reverse-proxy && docker-compose logs -f

Stop everything:
  cd /root/reverse-proxy && docker-compose down

Restart reverse proxy:
  docker restart reverse-proxy-ierada

Check Nginx config:
  docker exec reverse-proxy-ierada nginx -t

═══════════════════════════════════════════════════════════════════════════════
END OF FILE
═══════════════════════════════════════════════════════════════════════════════

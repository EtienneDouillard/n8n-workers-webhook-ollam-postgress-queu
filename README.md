Deploying n8n (Main/Worker/Webhook) behind a System-Wide Nginx with PostgreSQL, Redis, and Ollama
This tutorial walks you through deploying n8n (with worker, webhook, PostgreSQL, Redis, and Ollama) behind an existing Nginx server. We’ll use the domain n8n.example.com as an example. You’ll end up with:

Nginx (system installation) listening on ports 80 and 443 for n8n.example.com.
A Docker stack hosting:
n8n (main) → 1 CPU
n8n-worker → 1 CPU
n8n-webhook → 0.5 CPU
PostgreSQL, Redis (for queue mode), and Ollama → 1 CPU
n8n is accessible only at 127.0.0.1:5678 inside the server, while Nginx proxies external traffic (443) to that internal port.
HTTPS is managed by Let’s Encrypt (Certbot) on the host.
Table of Contents
Server Preparation
DNS Setup
Certbot & The Challenge Directory
Temporarily Disable the Nginx HTTPS Section
Generate Certificates with Certbot
Install Additional SSL Files
Re-enable the HTTPS Section in Nginx
Create the .env File
Create the docker-compose.yml File
Launch the Docker Stack
Access n8n in HTTPS
Common Errors & Solutions
Final Summary
1) Server Preparation
1.1 Install Docker & Docker Compose
On Ubuntu/Debian (example):

bash
Copier
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io

# Install docker-compose (plugin or standalone)
sudo apt-get install -y docker-compose-plugin

# Check Docker Compose version
docker compose version
1.2 Install Nginx (if not present)
bash
Copier
sudo apt-get install -y nginx
1.3 Open ports 80 and 443 in the firewall
bash
Copier
sudo ufw allow 80
sudo ufw allow 443
sudo ufw reload
2) DNS Setup
At your domain registrar (e.g., OVH, Gandi), create an A record for n8n.example.com pointing to your server IP address.

Wait for DNS propagation:

bash
Copier
nslookup n8n.example.com
dig n8n.example.com
They should return your server’s public IP.

3) Certbot & The Challenge Directory
3.1 Install Certbot
bash
Copier
sudo apt-get update
sudo apt-get install -y certbot
3.2 Create the challenge directory
bash
Copier
sudo mkdir -p /var/www/certbot
We will use this path to store the HTTP-01 challenge files from Let’s Encrypt.

4) Temporarily Disable the Nginx HTTPS Section
Before generating the certificate, comment out the HTTPS block so Nginx won’t fail when it can’t find the SSL files. For example, in /etc/nginx/nginx.conf or /etc/nginx/sites-available/default:

nginx
Copier
# -------------------------------------------------------------------------
# Port 80 for n8n.example.com
# -------------------------------------------------------------------------
server {
    listen 80;
    server_name n8n.example.com;

    # Certbot challenge directory
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# -------------------------------------------------------------------------
# Port 443 (commented out until cert is generated)
# -------------------------------------------------------------------------
# server {
#     listen 443 ssl;
#     server_name n8n.example.com;

#     ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
#     ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;
#     include /etc/letsencrypt/options-ssl-nginx.conf;
#     ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

#     location / {
#         proxy_pass http://127.0.0.1:5678;
#         proxy_http_version 1.1;
#         proxy_set_header Upgrade $http_upgrade;
#         proxy_set_header Connection 'upgrade';
#         proxy_set_header Host $host;
#         proxy_cache_bypass $http_upgrade;
#         proxy_set_header X-Real-IP $remote_addr;
#         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#         proxy_set_header X-Forwarded-Proto $scheme;
#     }
# }
Verify Nginx configuration and reload:

bash
Copier
sudo nginx -t
sudo systemctl reload nginx
5) Generate Certificates with Certbot
Now that Nginx won’t fail on missing SSL files:

bash
Copier
sudo certbot certonly --webroot -w /var/www/certbot -d n8n.example.com
-w /var/www/certbot is the webroot directory for the challenge.
-d n8n.example.com is your domain.
If successful, certificates are stored in /etc/letsencrypt/live/n8n.example.com/.
6) Install Additional SSL Files
If Nginx complains about missing files like /etc/letsencrypt/options-ssl-nginx.conf or /etc/letsencrypt/ssl-dhparams.pem, you need to create/download them:

6.1 Download options-ssl-nginx.conf
bash
Copier
sudo mkdir -p /etc/letsencrypt
sudo curl \
  https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf \
  -o /etc/letsencrypt/options-ssl-nginx.conf
6.2 Generate or download ssl-dhparams.pem
bash
Copier
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
7) Re-enable the HTTPS Section in Nginx
After you have the SSL files:

nginx
Copier
server {
    listen 443 ssl;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
Then:

bash
Copier
sudo nginx -t
sudo systemctl reload nginx
8) Create the .env File
In /home/n8n-dev (or another directory), create .env:

bash
Copier
POSTGRES_USER=n8n
POSTGRES_PASSWORD=myPGpassword
POSTGRES_DB=n8n_database

ENCRYPTION_KEY=MyVerySecretKey
9) Create the docker-compose.yml File
Place this in /home/n8n-dev/docker-compose.yml:

yaml
Copier
version: "3.8"

volumes:
  db_storage:
  n8n_storage:
  redis_storage:
  ollama_storage:

services:
  ##########################################################################
  # PostgreSQL
  ##########################################################################
  postgres:
    image: postgres:16
    container_name: n8n-postgres
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - db_storage:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  ##########################################################################
  # Redis
  ##########################################################################
  redis:
    image: redis:6-alpine
    container_name: n8n-redis
    restart: always
    volumes:
      - redis_storage:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 10

  ##########################################################################
  # N8N MAIN
  ##########################################################################
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n-main
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

    ports:
      - "127.0.0.1:5678:5678"

    environment:
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue mode
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      # Encryption key
      - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

      # n8n configuration
      - N8N_HOST=n8n.example.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.example.com/

    volumes:
      - n8n_storage:/home/node/.n8n
    deploy:
      resources:
        limits:
          cpus: "1"

  ##########################################################################
  # N8N WORKER
  ##########################################################################
  n8n-worker:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n-worker
    restart: always
    depends_on:
      - n8n
    command: worker
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

    deploy:
      resources:
        limits:
          cpus: "1"

  ##########################################################################
  # N8N WEBHOOK
  ##########################################################################
  n8n-webhook:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n-webhook
    restart: always
    depends_on:
      - n8n
    command: webhook
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

    deploy:
      resources:
        limits:
          cpus: "0.5"

  ##########################################################################
  # OLLAMA
  ##########################################################################
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ollama_storage:/root/.ollama
    deploy:
      resources:
        limits:
          cpus: "1"

networks:
  default:
    name: my_n8n_network
10) Launch the Docker Stack
bash
Copier
cd /home/n8n-dev  # the folder containing docker-compose.yml and .env
docker compose up -d
Check container status:

bash
Copier
docker compose ps
You should see Up or healthy for n8n-main, n8n-worker, n8n-webhook, n8n-postgres, n8n-redis, and ollama.

10.1 Test locally
bash
Copier
curl -I http://127.0.0.1:5678
Should return HTTP/1.1 200 OK or similar.

11) Access n8n in HTTPS
Open your browser to https://n8n.example.com:

Nginx receives the request on port 443.
It forwards traffic to 127.0.0.1:5678.
Docker runs n8n (main + worker + webhook).
You should see the n8n UI.
12) Common Errors & Solutions
Missing /etc/letsencrypt/options-ssl-nginx.conf
Download it:

bash
Copier
sudo curl \
  https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf \
  -o /etc/letsencrypt/options-ssl-nginx.conf
Missing /etc/letsencrypt/ssl-dhparams.pem
Generate it:

bash
Copier
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
Database (PostgreSQL) authentication issues

Check .env variables: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB.
Remove the database volume to reset:
bash
Copier
docker compose down
docker volume rm n8n_db_storage
docker compose up -d
Database migration errors (e.g., duplicate key)

Possibly an incomplete or partial migration.
Clear or re-init the DB volume as above.
Nginx fails to start

Ensure you commented out the HTTPS block before generating certificates.
Make sure the certificate files exist after Certbot finishes.
13) Final Summary
Disable the HTTPS block in Nginx before generating Let’s Encrypt certificates.
Generate certs with certbot certonly --webroot ....
Install missing SSL files (options-ssl-nginx.conf, ssl-dhparams.pem) if required.
Re-enable the HTTPS block in Nginx referencing the new certificate paths.
Launch n8n (main, worker, webhook) with PostgreSQL, Redis, and Ollama in Docker, mapping 127.0.0.1:5678 to the container.
Test https://n8n.example.com—Nginx will proxy to n8n in Docker.
Congratulations! Your n8n setup is fully secured with HTTPS and backed by a queue system (Redis + PostgreSQL) and Ollama.

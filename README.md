# Deploying n8n scaling queue mode Behind Nginx with Docker, PostgreSQL, Redis, Workers, Webhook and Ollama

This guide walks through deploying **n8n** (including worker, webhook, PostgreSQL, Redis, and Ollama) behind an existing Nginx server on a Linux host. The goal is to serve **n8n** at **`https://n8n.example.com`** using a secure setup with Let’s Encrypt and Docker.


This tutorial walks you through deploying **n8n** (with , **Webhook processors**, **PostgreSQL**, **Redis**,**Queu Mode**,  and Ollama) behind an existing Nginx server.

This guide will help you to configure this scaling mode : https://docs.n8n.io/hosting/scaling/queue-mode/

We’ll use the domain **n8n.example.com** as an example. You’ll end up with:

Nginx (system installation) listening on ports 80 and 443 for n8n.example.com.
A Docker stack hosting:

    n8n (main) → 1 CPU
    n8n-worker → 1 CPU
    n8n-webhook → 0.5 CPU
    PostgreSQL 
    Redis (for queue mode)
    Ollama → 1 CPU
    
    
n8n is accessible only at 127.0.0.1:5678 inside the server, while Nginx proxies external traffic (443) to that internal port.
HTTPS is managed by Let’s Encrypt (Certbot) on the host.

---

## Prerequisites

- **A Linux server** with a public IP address
- **A domain name** (e.g., `n8n.example.com`)
- **Nginx installed** on the server
- **Docker and Docker Compose** installed
- A working knowledge of basic Linux commands

---

## 1. Install Required Software

### Docker and Docker Compose

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin
```

### Nginx

```bash
sudo apt-get install -y nginx
```

### Certbot

```bash
sudo apt-get install -y certbot
```

---

## 2. Configure DNS

At your domain registrar, create an **A record** pointing your domain (e.g., `n8n.example.com`) to your server’s public IP address.

Verify DNS propagation:

```bash
nslookup n8n.example.com
```

---

## 3. Prepare the Nginx Configuration

We will set up Nginx to handle both HTTP (port 80) and HTTPS (port 443).

### HTTP (Port 80) Configuration

**Edit your Nginx configuration file**:

```bash
sudo nano /etc/nginx/nginx.conf
```



```nginx.conf
server {
    listen 80;
    server_name n8n.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Save the file, then reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Prepare Certbot’s Webroot

Create the directory:

```bash
sudo mkdir -p /var/www/certbot
```

---

## 4. Obtain Let’s Encrypt Certificates

Run Certbot with the **webroot** plugin:

```bash
sudo certbot certonly --webroot -w /var/www/certbot -d n8n.example.com
```

When completed, the certificates will be available in **`/etc/letsencrypt/live/n8n.example.com/`**.

---

## 5. Add HTTPS (Port 443) Configuration

Now that certificates are in place, update the Nginx configuration for HTTPS and add this below your first configuration :

```nginx
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
```



Verify you configuration  and reload Nginx:
```nginx
# ----------------------------------------------------------------------------
# Nginx listening on port 80 for the domain n8n.exemple.com
# ----------------------------------------------------------------------------
server {
    listen 80;
    server_name n8n.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
 # ----------------------------------------------------------------------------
 # Nginx listening on port 443 for the domain n8n.exemple.com
 # ----------------------------------------------------------------------------
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

```


```bash
sudo nginx -t
sudo systemctl reload nginx
```

If Nginx complains about missing files like `/etc/letsencrypt/options-ssl-nginx.conf` or `/etc/letsencrypt/ssl-dhparams.pem`, you need to create/download them:

Download `options-ssl-nginx.conf`

```bash
sudo mkdir -p /etc/letsencrypt
sudo curl \
  https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf \
  -o /etc/letsencrypt/options-ssl-nginx.conf
```
Generate or download `ssl-dhparams.pem`
```bash
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048

```

then 

```bash
sudo nginx -t
sudo systemctl reload nginx
```


---

## 6. Prepare the `.env` File

In your project directory (e.g., `/home/n8n`), create a `.env` file:

```env
POSTGRES_USER=n8n
POSTGRES_PASSWORD=securepassword
POSTGRES_DB=n8n_db
POSTGRES_NON_ROOT_USER=n8n
POSTGRES_NON_ROOT_PASSWORD=securepassword



ENCRYPTION_KEY=mysecretkey

DOMAIN=exemple.com

SSL_EMAIL=youremail.com

GENERIC_TIMEZONE=Europe/Paris

WEBHOOK_URL=https://n8n.exemple.com

REDIS_HOST=redis
REDIS_PORT=6379

ENCRYPTION_KEY=cryptedkey

EXECUTIONS_MODE=queue
N8N_DATABASE_TYPE=postgresdb
N8N_POSTGRES_HOST=postgres
N8N_POSTGRES_PORT=5432
N8N_POSTGRES_USER=${POSTGRES_USER}
N8N_POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
N8N_POSTGRES_DB=${POSTGRES_DB}

QUEUE_BULL_REDIS_HOST=${REDIS_HOST}
QUEUE_BULL_REDIS_PORT=${REDIS_PORT}

N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=name
N8N_BASIC_AUTH_PASSWORD=securepassword


```

---

## 7. Create the Docker Compose File

Create a `docker-compose.yml` in the same directory:

```yaml
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
      # Base de données
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Mode queue
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      # Clé de chiffrement
      - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

      # Configuration n8n
      - N8N_HOST=n8n.exemple.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.exemple.com/

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
      # BDD
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # Queue
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_HEALTH_CHECK_ACTIVE=true

      # Clé de chiffrement
      - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

    deploy:
      resources:
        limits:
          cpus: "1"


  ##########################################################################
  # OLLAMA (CPU)
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

```

---

## 8. Launch the Docker Stack

Start the containers in detached mode:

```bash
docker compose up -d
```

Verify that all services are running:

```bash
docker compose ps
```

You should see **Up** or **healthy** for n8n-main, n8n-worker, n8n-webhook, n8n-postgres, n8n-redis, and ollama.


---

## 9. Test the Setup

Open a browser and navigate to:

```
https://n8n.example.com
```

---

## Troubleshooting

1. **Missing SSL files:**
   - If `options-ssl-nginx.conf` or `ssl-dhparams.pem` are missing, download or generate them:
     ```bash
     sudo curl -o /etc/letsencrypt/options-ssl-nginx.conf \
       https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
     sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
     ```

2. **PostgreSQL authentication errors:**
   - Verify `.env` credentials.
   - If issues persist, reset the database volume:
     ```bash
     docker compose down
     docker volume rm db_storage
     docker compose up -d
     ```

---

## Summary

This guide walks you through:

- Setting up Nginx to serve n8n behind HTTPS.
- Using Let’s Encrypt for certificates.
- Running n8n, worker, webhook, PostgreSQL, Redis, and Ollama in Docker.
- Keeping n8n securely accessible only via localhost and Nginx.

With these steps, you now have a reliable and secure n8n instance accessible at **https://n8n.example.com**.


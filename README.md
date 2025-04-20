# Nginx Reverse Proxy Setup with HTTP to HTTPS Redirection (Cloudflare + Docker)

This guide shows how to set up **Nginx** as a reverse proxy with **HTTP to HTTPS redirection**, using **Cloudflare Origin Certificates**. It also assumes your app (e.g., a Next.js app) is running in a Docker container.

---

## 🌐 Domain & DNS (Cloudflare)

1. Point your domain to your server IP using **Cloudflare DNS**.
2. Example DNS records:

Type: A Name: app Value: <your-server-ip> Proxy status: Proxied (orange cloud)


3. In Cloudflare SSL/TLS settings:
- SSL Mode: **Full** (or **Full (Strict)** if using valid certs)
- Enable **Always Use HTTPS**
- Disable **Automatic HTTPS Rewrites** (optional)

---

## 🔐 SSL Certificate (Cloudflare Origin Cert)

1. Go to Cloudflare → SSL/TLS → Origin Server → **Create Certificate**.
2. Download:
- **Origin Certificate** (e.g., `origin-cert.pem`)
- **Private Key** (e.g., `origin-key.pem`)
3. Place them on your server, e.g.:

/etc/ssl/certs/cloudflare/origin-cert.pem /etc/ssl/certs/cloudflare/origin-key.pem


> 💡 These certs are only valid when traffic comes through Cloudflare.

---

## 🧾 Nginx Configuration

Create or update your `/etc/nginx/nginx.conf`:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
               '$status $body_bytes_sent "$http_referer" '
               '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;

  sendfile on;
  keepalive_timeout 65;

  # HTTP to HTTPS Redirect
  server {
    listen 80;
    server_name app.housepitalcare.com;

     return 301 https://$host$request_uri;
  }

  # HTTPS Server Block
  server {
   listen 443 ssl;
   server_name app.housepitalcare.com;
  
   ssl_certificate     /etc/ssl/certs/cloudflare/origin-cert.pem;
   ssl_certificate_key /etc/ssl/certs/cloudflare/origin-key.pem;
  
   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
     proxy_pass http://nextjs:3000;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```



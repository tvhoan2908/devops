# 🌐 Tài liệu Nginx & HAProxy: Từ Zero đến Chuyên gia

> **Mục tiêu**: Cấu hình production-ready cho reverse proxy, load balancer, SSL termination, performance tuning.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Cài đặt](#p0) | Nginx + HAProxy lab |
| [P1. Nginx cơ bản](#p1) | Server blocks, location, log |
| [P2. Reverse proxy & LB](#p2) | Upstream, load balancing |
| [P3. SSL/TLS](#p3) | HTTPS, cert, security headers |
| [P4. Performance](#p4) | Caching, gzip, buffer |
| [P5. Security](#p5) | Rate limit, WAF, hardening |
| [P6. HAProxy cơ bản](#p6) | Frontend, backend, ACL |
| [P7. HAProxy nâng cao](#p7) | Sticky session, health check |
| [P8. HAProxy logs & stats](#p8) | Logging, monitoring |
| [P9. K8s Ingress](#p9) | Nginx ingress controller |

---

<a id="p0"></a>
## P0. Cài đặt

### Bước 1: Cài Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo dnf install epel-release
sudo dnf install nginx

# macOS
brew install nginx

# Từ source (latest)
wget https://nginx.org/download/nginx-1.25.4.tar.gz
tar xzf nginx-1.25.4.tar.gz
cd nginx-1.25.4
./configure --prefix=/usr/local/nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_realip_module
make && sudo make install
```

### Bước 2: Cài HAProxy

```bash
# Ubuntu
sudo apt install haproxy

# CentOS
sudo dnf install haproxy

# Verify version (cần 2.0+)
haproxy -v
```

### Bước 3: Cấu trúc Nginx

```bash
# Config files
/etc/nginx/
├── nginx.conf                # Main config
├── conf.d/                   # Additional configs
│   └── *.conf
├── sites-available/          # Available site configs
│   └── default
├── sites-enabled/            # Enabled sites (symlink)
│   └── default
├── snippets/                 # Reusable snippets
│   ├── ssl-params.conf
│   └── fastcgi-php.conf
├── modules-available/        # Dynamic modules
├── modules-enabled/
└── mime.types                # MIME types

# Main commands
sudo systemctl enable --now nginx
sudo nginx -t                  # Test config
sudo nginx -s reload           # Reload without downtime
sudo nginx -s stop
sudo nginx -V                  # Version + modules
```

---

<a id="p1"></a>
## P1. Nginx cơ bản

### Bước 1: nginx.conf chính

```nginx
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;                  # Số CPU cores
worker_rlimit_nofile 65535;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 4096;            # Per worker
    multi_accept on;
    use epoll;
}

http {
    # === BASIC ===
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;                  # Không hiện version

    # === LOGGING ===
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'urt="$upstream_response_time"';

    log_format json_combined escape=json
    '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_referrer":"$http_referer",'
        '"http_user_agent":"$http_user_agent"'
    '}';

    access_log /var/log/nginx/access.log main;

    # === PERFORMANCE ===
    sendfile_max_chunk 1m;
    aio threads;

    # === COMPRESSION ===
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # === VIRTUAL HOSTS ===
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Bước 2: Server block đầu tiên

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example.com;
    index index.html index.htm;

    access_log /var/log/nginx/example.com.access.log main;
    error_log  /var/log/nginx/example.com.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }
}
```
```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Bước 3: Location matching

```nginx
server {
    listen 80;
    server_name example.com;

    # 1. Exact match (=)
    location = /favicon.ico {
        # Chỉ match /favicon.ico chính xác
    }

    # 2. Prefix match (^~) - ưu tiên cao
    location ^~ /static/ {
        # Match /static/... và không check regex
        expires 30d;
        access_log off;
    }

    # 3. Case-sensitive regex (~)
    location ~ \.(gif|jpg|png)$ {
        # Match file ảnh
        expires 30d;
    }

    # 4. Case-insensitive regex (~*)
    location ~* \.(css|js)$ {
        # Match CSS/JS
        expires 7d;
    }

    # 5. Prefix match (không modifier) - lowest priority
    location /api/ {
        proxy_pass http://backend;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}

# Thứ tự ưu tiên:
# 1. = exact
# 2. ^~ prefix (nếu match dài nhất và không có regex match)
# 3. ~ hoặc ~* regex (match theo thứ tự khai báo)
# 4. prefix thông thường (match dài nhất)
```

### Bước 4: Variables & headers

```nginx
server {
    # Built-in variables
    # $host, $server_name, $request, $uri, $args
    # $remote_addr, $http_user_agent, $http_referer
    # $status, $body_bytes_sent, $request_time

    location /api {
        # Custom
        set $api_version "v1";

        # Header từ client
        if ($http_x_api_key = "") {
            return 401 "API key required";
        }

        # Custom response header
        add_header X-Custom-Header "Value" always;
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
    }
}
```

### Bước 5: Custom error pages

```nginx
server {
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
        root /usr/share/nginx/html;
        internal;
    }

    location = /50x.html {
        root /usr/share/nginx/html;
        internal;
    }
}
```

---

<a id="p2"></a>
## P2. Reverse Proxy & Load Balancing

### Bước 1: Reverse proxy cơ bản

```nginx
# /etc/nginx/sites-available/app.conf
upstream backend {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffer
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
    }
}
```

### Bước 2: Load balancing methods

```nginx
# Round-robin (mặc định)
upstream backend {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}

# Least connections
upstream backend {
    least_conn;
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}

# IP hash (sticky session)
upstream backend {
    ip_hash;
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}

# Weighted
upstream backend {
    server 10.0.0.10:8080 weight=3;     # 3/4 requests
    server 10.0.0.11:8080 weight=1;     # 1/4 requests
}

# Generic hash (Nginx Plus)
upstream backend {
    hash $request_uri consistent;
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}

# Random (Nginx Plus)
upstream backend {
    random two;
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}
```

### Bước 3: Health check & failover

```nginx
upstream backend {
    server 10.0.0.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:8080 backup;        # Chỉ dùng khi các server khác down

    # Active health check (Nginx Plus only)
    # health_check interval=5s fails=3 passes=2;
}

server {
    location / {
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;

        proxy_pass http://backend;
    }
}
```

### Bước 4: WebSocket

```nginx
upstream websocket {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}

server {
    listen 80;
    server_name ws.example.com;

    location / {
        proxy_pass http://websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # WebSocket timeouts (no timeout)
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### Bước 5: Path-based routing

```nginx
server {
    listen 80;
    server_name example.com;

    # /api -> backend API
    location /api/ {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # /static -> CDN/storage
    location /static/ {
        proxy_pass http://static_backend;
        expires 30d;
        access_log off;
    }

    # / -> frontend
    location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
    }
}

upstream api_backend {
    server 10.0.0.10:3000;
    server 10.0.0.11:3000;
}

upstream static_backend {
    server 10.0.0.20:80;
}

upstream frontend {
    server 10.0.0.30:80;
    server 10.0.0.31:80;
}
```

---

<a id="p3"></a>
## P3. SSL/TLS

### Bước 1: HTTPS với Let's Encrypt

```bash
# Cài certbot
sudo apt install certbot python3-certbot-nginx

# Tạo cert
sudo certbot --nginx -d example.com -d www.example.com

# Auto-renew
sudo certbot renew --dry-run
```

### Bước 2: Cấu hình SSL tối ưu

```nginx
# /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers off;

ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

ssl_stapling on;
ssl_stapling_verify on;

resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

add_header Strict-Transport-Security "max-age=63072000" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### Bước 3: HTTPS server block

```nginx
# Redirect HTTP -> HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

# Main HTTPS server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;

    root /var/www/example.com;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    include snippets/ssl-params.conf;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api {
        proxy_pass http://backend;
        include snippets/proxy-params.conf;
    }
}
```

### Bước 4: Self-signed cert (dev)

```bash
# Tạo cert
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=localhost"

# DH params (tăng tốc độ handshake)
openssl dhparam -out /etc/nginx/dhparam.pem 2048
```
```nginx
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
ssl_dhparam /etc/nginx/dhparam.pem;
```

### Bước 5: Mutual TLS (mTLS)

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /etc/ssl/server.crt;
    ssl_certificate_key /etc/ssl/server.key;

    # Client cert
    ssl_client_certificate /etc/ssl/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;

    location / {
        if ($ssl_client_verify != SUCCESS) {
            return 403;
        }

        proxy_pass http://backend;
        proxy_set_header X-Client-DN $ssl_client_s_dn;
    }
}
```

### Bước 6: SSL test

```bash
# Test SSL Labs
curl -I https://example.com
openssl s_client -connect example.com:443 -servername example.com

# Check cipher
nmap --script ssl-enum-ciphers -p 443 example.com
```

---

<a id="p4"></a>
## P4. Performance

### Bước 1: Caching static files

```nginx
server {
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
        add_header X-Cache-Status $upstream_cache_status;
    }

    location ~* \.(html)$ {
        expires 1h;
        add_header Cache-Control "public, must-revalidate";
    }
}
```

### Bước 2: Proxy caching

```nginx
http {
    proxy_cache_path /var/cache/nginx/proxy
        levels=1:2
        keys_zone=my_cache:10m
        max_size=10g
        inactive=60m
        use_temp_path=off;

    proxy_cache_key "$scheme$request_method$host$request_uri";

    server {
        location /api {
            proxy_cache my_cache;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_bypass $http_cache_control;
            proxy_add_x_forwarded_for;

            # Cache status
            add_header X-Cache-Status $upstream_cache_status;

            # Conditional
            proxy_cache_methods GET HEAD;
            proxy_no_cache $arg_nocache;

            proxy_pass http://backend;
        }
    }
}
```

### Bước 3: FastCGI caching (PHP)

```nginx
http {
    fastcgi_cache_path /var/cache/nginx/fastcgi
        levels=1:2
        keys_zone=php_cache:10m
        max_size=10g
        inactive=60m;

    server {
        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
            include snippets/fastcgi-php.conf;

            fastcgi_cache php_cache;
            fastcgi_cache_valid 200 302 10m;
            fastcgi_cache_valid 404 1m;
            fastcgi_cache_bypass $http_cache_control;
            add_header X-Cache-Status $upstream_cache_status;

            # Cache theo cookie/logged in
            fastcgi_cache_bypass $cookie_logged_in $arg_nocache;
            fastcgi_no_cache $cookie_logged_in $arg_nocache;
        }
    }
}
```

### Bước 4: Gzip & Brotli

```nginx
http {
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Brotli (cần module)
    # brotli on;
    # brotli_comp_level 6;
    # brotli_types text/plain text/css application/json;
}
```

### Bước 5: HTTP/2 & HTTP/3

```nginx
server {
    listen 443 ssl http2;
    # HTTP/3 (cần Nginx Plus hoặc build với quiche)
    # listen 443 ssl http3 quic;
    # add_header Alt-Svc 'h3=":443"; ma=86400';

    server_name example.com;
    # ...
}
```

### Bước 6: Connection tuning

```nginx
http {
    # Keepalive
    keepalive_timeout 65;
    keepalive_requests 1000;

    # Sendfile
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # File descriptors
    worker_rlimit_nofile 65535;
    worker_connections 4096;

    # Buffers
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    client_max_body_size 8m;
    large_client_header_buffers 4 8k;

    # Timeouts
    client_body_timeout 12s;
    client_header_timeout 12s;
    send_timeout 10s;

    # Open file cache
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

### Bước 7: Performance benchmark

```bash
# Apache Bench
ab -n 10000 -c 100 https://example.com/

# wrk
wrk -t 4 -c 100 -d 30s https://example.com/

# hey
hey -n 10000 -c 100 https://example.com/

# với auth
ab -n 1000 -c 10 -A user:pass https://api.example.com/

# Test POST
ab -n 1000 -c 10 -p data.json -T application/json https://api.example.com/
```

---

<a id="p5"></a>
## P5. Security

### Bước 1: Rate limiting

```nginx
http {
    # Định nghĩa zone
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    # Connection limit
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        limit_conn addr 10;

        location / {
            limit_req zone=general burst=20 nodelay;
            limit_req_status 429;
        }

        location /api/ {
            limit_req zone=api burst=50 nodelay;
        }

        location /login {
            limit_req zone=login burst=5;
        }
    }
}
```

### Bước 2: Block bad bots

```nginx
server {
    # Block bad bots
    if ($http_user_agent ~* (ahrefsbot|petalbot|dotbot|mj12bot|scrapy)) {
        return 403;
    }

    # Block referer spam
    if ($http_referer ~* (semalt\.com|buttons-for-website\.com)) {
        return 403;
    }
}
```

### Bước 3: Block SQL injection / XSS attempts

```nginx
server {
    location / {
        # Block common attack patterns
        if ($args ~* "union.*select.*\(") { return 403; }
        if ($args ~* "select.*from.*information_schema") { return 403; }
        if ($args ~* "<script") { return 403; }
        if ($request_uri ~* "\.\./") { return 403; }
        if ($query_string ~* "etc/passwd") { return 403; }
    }
}
```

### Bước 4: IP whitelist/blacklist

```nginx
# /etc/nginx/conf.d/ip-rules.conf
geo $blocked_ip {
    default 0;
    192.0.2.0/24 1;      # Block subnet
    198.51.100.10 1;     # Block IP
}

server {
    if ($blocked_ip) {
        return 403;
    }

    # Allow specific
    location /admin {
        allow 10.0.0.0/8;
        allow 192.168.0.0/16;
        deny all;

        # Hoặc auth
        auth_basic "Admin";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

### Bước 5: ModSecurity WAF

```bash
# Cài
sudo apt install libmodsecurity3
git clone https://github.com/owasp-modsecurity/ModSecurity-nginx.git
cd ModSecurity-nginx
./configure --with-compat
make && sudo make install
```
```nginx
load_module modules/ngx_http_modsecurity_module.so;

server {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity/main.conf;

    location / {
        # ...
    }
}
```
```bash
# OWASP CRS
wget https://github.com/coreruleset/coreruleset/archive/v3.3.4.tar.gz
tar xzf coreruleset-3.3.4.tar.gz -C /etc/nginx/modsecurity/
ln -s /etc/nginx/modsecurity/coreruleset-3.3.4/crs-setup.conf.example /etc/nginx/modsecurity/crs-setup.conf

# /etc/nginx/modsecurity/main.conf
Include /etc/nginx/modsecurity/modsecurity.conf
Include /etc/nginx/modsecurity/crs-setup.conf
Include /etc/nginx/modsecurity/coreruleset-3.3.4/rules/*.conf
```

### Bước 6: Security headers

```nginx
server {
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self';" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}
```

---

<a id="p6"></a>
## P6. HAProxy cơ bản

### Bước 1: Cấu hình tổng quan

```haproxy
# /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # SSL
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    timeout http-request 10s
    timeout queue 30s
    timeout http-keep-alive 10s
    timeout tunnel 1h
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
```

### Bước 2: Frontend & Backend

```haproxy
frontend fe_main
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem alpn h2,http/1.1
    redirect scheme https code 301 if !{ ssl_fc }

    # ACL
    acl is_api path_beg /api
    acl is_static path_beg /static

    # Routing
    use_backend be_api if is_api
    use_backend be_static if is_static
    default_backend be_web

backend be_web
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server web1 10.0.0.10:80 check inter 5s fall 3 rise 2
    server web2 10.0.0.11:80 check inter 5s fall 3 rise 2
    server web3 10.0.0.12:80 check inter 5s fall 3 rise 2 backup

backend be_api
    balance leastconn
    option httpchk GET /api/health
    http-check expect status 200

    server api1 10.0.0.20:3000 check inter 5s
    server api2 10.0.0.21:3000 check inter 5s

backend be_static
    balance roundrobin
    option httpchk

    server static1 10.0.0.30:80 check
    server static2 10.0.0.31:80 check
```

### Bước 3: Load balancing algorithms

```haproxy
backend be_web
    # Round-robin (mặc định)
    balance roundrobin

    # Least connections
    balance leastconn

    # Source IP hash (sticky)
    balance source

    # URI hash
    balance uri

    # URL parameter hash
    balance url_param sessionid

    # Random
    balance random

    # Weighted round-robin
    server web1 10.0.0.10:80 weight 3
    server web2 10.0.0.11:80 weight 1
```

---

<a id="p7"></a>
## P7. HAProxy nâng cao

### Bước 1: Sticky session

```haproxy
backend be_app
    balance roundrobin
    cookie SERVERID insert indirect nocache

    server app1 10.0.0.10:8080 cookie app1 check
    server app2 10.0.0.11:8080 cookie app2 check

# Hoặc stick theo session ID từ URL
backend be_app
    balance roundrobin
    stick-table type string size 100k expire 30m
    stick on url_param session_id

# Hoặc stick theo source IP
backend be_app
    balance source
    stick-table type ip size 50k expire 30m
    stick on src
```

### Bước 2: Health check nâng cao

```haproxy
backend be_api
    option httpchk
    http-check send meth GET hdr Host api.example.com uri /health
    http-check expect status 200
    http-check expect header Content-Type -m end "application/json"

    # Check period
    server api1 10.0.0.10:3000 check inter 3s fall 3 rise 2
    server api2 10.0.0.11:3000 check inter 3s fall 3 rise 2

    # Slow start (warmup sau khi up lại)
    server api1 10.0.0.10:3000 check slowstart 30s

    # Maintenance mode
    server api1 10.0.0.10:3000 check maintenance
```

### Bước 3: ACL nâng cao

```haproxy
frontend fe_main
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem

    # Path-based
    acl is_api path_beg /api
    acl is_admin path_beg /admin
    acl is_static path_end .css .js .jpg .png .gif .ico

    # Domain-based
    acl is_example hdr(host) -i example.com
    acl is_www hdr(host) -i www.example.com

    # Header-based
    acl has_api_key req.hdr(X-API-Key) -m found
    acl is_mobile req.hdr(User-Agent) -m sub android iphone

    # Source IP
    acl is_internal src 10.0.0.0/8 192.168.0.0/16

    # SSL
    acl is_ssl ssl_fc

    # Routing
    use_backend be_api if is_api
    use_backend be_admin if is_admin { src 10.0.0.0/8 } or { req.hdr(X-Admin-Token) -m found }
    use_backend be_static if is_static
    default_backend be_web

    # Block
    acl is_bad_bot req.hdr(User-Agent) -m sub -i (ahrefsbot|petalbot|dotbot)
    http-request deny if is_bad_bot

    # Rate limit (HAProxy 2.2+)
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny if { sc_http_req_rate(0) gt 100 }

    # GeoIP (cần db file)
    acl is_vn src_country -f /etc/haproxy/geoip/country.lst -m str "VN"
```

### Bước 4: SSL termination & passthrough

```haproxy
# SSL termination (HAProxy decrypt)
frontend fe_https
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem crt /etc/ssl/certs/wildcard.pem alpn h2,http/1.1

    # SNI routing
    use_backend be_api if { ssl_fc_sni api.example.com }
    use_backend be_web if { ssl_fc_sni example.com www.example.com }

backend be_web
    server web1 10.0.0.10:80

# SSL passthrough (HAProxy không decrypt, chỉ forward)
frontend fe_ssl_passthrough
    bind *:443
    mode tcp

    use_backend be_web_ssl if { ssl_fc_sni web.example.com }
    use_backend be_api_ssl if { ssl_fc_sni api.example.com }

backend be_web_ssl
    mode tcp
    balance roundrobin
    server web1 10.0.0.10:443 check

backend be_api_ssl
    mode tcp
    balance roundrobin
    server api1 10.0.0.20:443 check
```

### Bước 5: Rewrites & redirects

```haproxy
frontend fe_main
    # HTTP -> HTTPS
    redirect scheme https code 301 if !{ ssl_fc }

    # WWW -> non-WWW
    redirect prefix https://example.com code 301 if { hdr(host) -i www.example.com }

    # Path rewrite
    http-request set-path /api/%[path,regsub(^/old-api/,)] if { path_beg /old-api/ }

    # Response header modify
    http-response set-header Strict-Transport-Security "max-age=31536000"
    http-response set-header X-Frame-Options "DENY"
    http-response set-header X-Content-Type-Options "nosniff"

    # Strip
    http-request del-header Cookie if { path_beg /static/ }
    http-response del-header Server
```

---

<a id="p8"></a>
## P8. HAProxy logs & stats

### Bước 1: Stats page

```haproxy
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if TRUE
    stats auth admin:password

    # Per-process
    stats show-node
    stats show-legends
```

Truy cập: `http://server:8404/stats`

### Bước 2: Logging

```bash
# /etc/rsyslog.d/haproxy.conf
$ModLoad imudp
$UDPServerRun 514

$ModLoad imtcp
$InputTCPServerRun 514

local0.* /var/log/haproxy.log
local1.* /var/log/haproxy.log
& stop
```
```bash
sudo systemctl restart rsyslog
```

### Bước 3: Prometheus metrics

```bash
# Trong haproxy.cfg
frontend prometheus-exporter
    bind *:8405
    http-request use-service prometheus-exporter
```

```bash
# Hoặc dùng haproxy_exporter
docker run -d --name haproxy-exporter \
  -p 9101:9101 \
  prom/haproxy-exporter \
  --haproxy.scrape-uri=http://haproxy:8404/stats;csv
```

### Bước 4: Datadog API integration

```haproxy
# Trong defaults
log /dev/log local0
log "[email protected]:10514" format rfc5424
option log-health-checks
```

---

<a id="p9"></a>
## P9. Nginx Ingress Controller (K8s)

### Bước 1: Cài Nginx Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.resources.limits.cpu=500m,memory=512Mi
```

### Bước 2: Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom: Value";
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

### Bước 3: ConfigMap tuning

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
data:
  proxy-body-size: "50m"
  proxy-read-timeout: "300"
  proxy-send-timeout: "300"
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
  use-forwarded-headers: "true"
  enable-real-ip: "true"
  compute-full-forwarded-for: "true"
  forwarded-for-header: "X-Forwarded-For"
  upstream-keepalive-connections: "100"
```

### Bước 4: Backend Config cho sticky session

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
data:
  # Session affinity
  sticky-session-cookie-name: "SERVERID"
  sticky-session-cookie-expires: "1h"
  sticky-session-cookie-path: "/"

  # Load balancing
  load-balance: "least_conn"
```

---

## 🎯 Bài tập P0-P9

1. Cài Nginx + HAProxy, tạo reverse proxy cho 2 backend
2. Cấu hình HTTPS với Let's Encrypt, đạt A+ trên SSL Labs
3. Setup load balancing với health check
4. Rate limit /api endpoint
5. Caching proxy responses, kiểm tra `X-Cache-Status`
6. HAProxy với ACL routing theo domain
7. Cài Nginx Ingress Controller lên K8s, deploy app

---

> **💡 Tip cuối**: Nginx cho static + reverse proxy thường. HAProxy cho TCP/L4 + sticky session + high throughput. Test với `wrk` trước khi deploy production.

---

*Tạo bởi tài liệu học Nginx & HAProxy - Chúc bạn thành công! 🚀*
